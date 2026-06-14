# PostgreSQL relmapper / relfilenode cache
## 课程定位
前置知识：已经理解 `MemoryContext`、`ResourceOwner`、syscache/catcache、relcache、shared invalidation、WAL 与 relation storage 的基本边界。
本节唯一主问题：
```text
relfilenode、relmapper 和 mapped relation 如何连接 catalog identity 与物理文件 identity？
```
核心矛盾：
```text
上层代码希望用稳定的 relation OID 表示一个 catalog 对象；
存储层却必须用会在 rewrite、VACUUM FULL、CLUSTER、TRUNCATE、REINDEX 中变化的物理文件号定位数据文件。
```
一句话运行模型：
```text
普通 relation 用 pg_class.relfilenode 记录当前物理文件号；
mapped relation 在 pg_class.relfilenode 中保存 0，并通过 pg_filenode.map 把 relid 映射到 RelFileNumber；
relcache build 时把 catalog identity 转成 rd_locator；
storage manager 只消费 RelFileLocator；
RelidByRelfilenumber() 反向缓存从物理 filenumber 找 relid 的诊断/解码路径。
```
学完后应能判断：
- `pg_class.oid` 为什么不是稳定的磁盘文件名。
- `pg_class.relfilenode = 0` 为什么不是“没有文件”，而是“走 relmapper”。
- `RelationData.rd_locator` 为什么才是 relcache 交给 smgr 的物理定位结果。
- 普通 relation rewrite 为什么通过更新 `pg_class.relfilenode` 提交物理身份变化。
- mapped catalog rewrite 为什么不能依赖更新自己的 `pg_class` 行。
- `pg_filenode.map` 为什么需要 WAL、CRC、atomic rename、sinval 和 `RelationMappingLock`。
- `relfilenumbermap.c` 的反向 cache 为什么不能替代 relmapper。
- stale relfilenode 诊断应该从 relcache、relmapper、smgr、WAL/recovery 还是 logical decoding 边界入手。
本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e9`。
## 1. 本节在总主线中的位置
前面几节已经建立了三个事实：catalog tuple 通过 syscache/catcache 进入 backend-local cache，`RelationData` 通过 relcache 组合多个 catalog row，shared invalidation 只传播“哪些语义过期”。本节沿同一条 metadata 主线继续向下追：一个稳定的 relation OID 最终怎样变成 storage manager 可以打开的物理文件。
一个 relation 至少有两种身份：
| 身份 | 状态 | 语义 |
| --- | --- | --- |
| catalog identity | `pg_class.oid` | SQL 对象和依赖关系里的 relation。 |
| physical identity | `RelFileLocator` | smgr、buffer、WAL 使用的文件定位。 |
本节主链路：
```text
relid
  -> relcache build
  -> pg_class.relfilenode or relmapper
  -> RelationData.rd_locator
  -> smgropen()/smgrcreate()
  -> filesystem path and WAL records
```
本节不重复 tuple descriptor、heap page layout、fork number、segment file 编号，只回答这个边界问题：
```text
当 catalog 中的 logical relation identity 和磁盘上的 physical file identity 不再一一等同，
PostgreSQL 如何让各模块仍然读到同一个“当前文件”？
```
## 2. 核心矛盾与一句话运行模型
最容易误读的是：
```text
relation OID 就是 relation 文件名。
```
这个说法只在少量启动早期、未 rewrite 的系统对象上近似成立。正常运行中，PostgreSQL 明确区分：
```text
relid:      catalog object identity
filenumber: physical storage identity
```
普通 relation 的正向映射通常来自 `pg_class.relfilenode`。
mapped relation 的正向映射来自 `pg_filenode.map`。
relcache 把两者统一成 `RelationData.rd_locator`。
storage manager 只看 `rd_locator`，不关心它来自 catalog 还是 relmapper。
这个设计要同时满足三件事：catalog 对象身份在 rewrite 后仍稳定；物理文件可以被 `VACUUM FULL`、`CLUSTER`、`TRUNCATE`、`REINDEX` 替换；核心/共享 catalog 不能依赖“先打开自己的 `pg_class` 行，再知道自己的文件在哪里”。
| 层次 | 正向问题 | 状态位置 |
| --- | --- | --- |
| `pg_class` | 普通 relation 的 relid -> filenumber | catalog tuple。 |
| relmapper | mapped relation 的 relid -> filenumber | `pg_filenode.map` 文件 + backend-local copy。 |
| relcache | relid -> `RelationData` -> `rd_locator` | backend-local `CacheMemoryContext`。 |
| smgr | `RelFileLocator` -> storage handle | backend-local smgr hash / file descriptor layer。 |
| relfilenumber map cache | filenumber -> relid | backend-local `RelfilenumberMapHash`。 |
这五层不能互相替代：`pg_class.relfilenode` 不是所有 relation 的权威物理身份，relmapper 不是通用 object catalog，`rd_locator` 不是持久状态，smgr handle 不是 catalog identity，`RelidByRelfilenumber()` 也不是打开 relation 的主路径。
本节核心不变量：
```text
上层按 relid 持有逻辑对象；
relcache 在当前 backend 内把 relid 解析成当前 rd_locator；
物理文件身份变化必须同时让持久映射、relcache 和相关反向 cache 在正确边界失效；
storage 层只相信传入的 RelFileLocator，不负责理解 catalog 语义。
```
## 3. 核心源码文件与阅读顺序
推荐按正向打开路径读，再读 rewrite/update 路径，最后读反向 cache。
| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/catalog/pg_class.h` | `relfilenode` 字段、`RELKIND_HAS_STORAGE()`、mapped relation 的 catalog 表示。 |
| 2 | `src/include/utils/rel.h` | `RelationData.rd_locator`、`rd_backend`、`RelationIsMapped()`、`RelationGetSmgr()`。 |
| 3 | `src/backend/utils/cache/relcache.c` | `RelationInitPhysicalAddr()`、`RelationBuildLocalRelation()`、`RelationSetNewRelfilenumber()`。 |
| 4 | `src/include/utils/relmapper.h` | relmapper public API、relmap WAL record。 |
| 5 | `src/backend/utils/cache/relmapper.c` | `pg_filenode.map` 格式、load/write/update、CCI/EOXact、WAL、sinval、redo。 |
| 6 | `src/backend/utils/cache/relfilenumbermap.c` | `(tablespace, relfilenumber) -> relid` 反向 cache。 |
| 7 | `src/backend/catalog/storage.c` | `RelationCreateStorage()`、`RelationDropStorage()`、`RelationPreserveStorage()`、smgr WAL。 |
| 8 | `src/backend/storage/smgr/smgr.c` | `smgrcreate()`、`smgropen()` 消费 `RelFileLocator`。 |
| 9 | `src/include/catalog/pg_database.h` | `dattablespace` 如何进入 database 默认 tablespace 与 database path。 |
| 10 | `src/backend/replication/logical/reorderbuffer.c` | logical decoding 从 WAL 的 physical locator 反查 relid。 |
| 11 | `src/backend/utils/adt/dbsize.c` | `pg_filenode_relation()` 这类诊断函数如何走反向 cache。 |
最小阅读链：
```text
pg_class.h 的 relfilenode 注释
  -> rel.h 的 RelationIsMapped()
  -> relcache.c 的 RelationInitPhysicalAddr()
  -> relmapper.c 的 RelationMapOidToFilenumber()
  -> storage.c 的 RelationCreateStorage()
  -> relcache.c 的 RelationSetNewRelfilenumber()
  -> relmapper.c 的 AtCCI_RelationMap() / AtEOXact_RelationMap()
  -> relfilenumbermap.c 的 RelidByRelfilenumber()
```
不要从 `smgr.c` 顶部开始读。
`smgr` 是物理访问抽象，不知道 relation OID 的 catalog 意义。
也不要先读 `pg_filenode.map` 文件格式。
文件格式只是状态载体。
真正的系统问题是：
```text
哪个时间点可以相信新的 filenumber 已经代表同一个 relation？
```
## 4. 关键数据结构与状态
### 4.1 `pg_class.oid` 与 `pg_class.relfilenode`
`pg_class.oid` 是 relation 的 catalog identity。
依赖记录、权限检查、规则、触发器、统计信息、planner 结构、lock tag 的 relation 部分都围绕它展开。
`pg_class.relfilenode` 是普通 relation 的当前物理文件号。
本地源码中 `pg_class.h` 对这个字段的关键注释是：
```text
relfilenode == 0 means it is a "mapped" relation
```
因此语义不是：
```text
relfilenode = 0 -> 没有 storage
```
而是：
```text
RELKIND_HAS_STORAGE(relkind) && relfilenode = 0
  -> 这个 relation 有 storage，但 filenumber 要从 relmapper 查
```
`RELKIND_HAS_STORAGE()` 当前覆盖：
```text
ordinary table
index
sequence
toast table
materialized view
```
partitioned table 和 partitioned index 有 catalog identity 和 relcache entry，但没有自己的 heap/index storage 文件。
所以判断物理文件不能只看 `relfilenode`。
必须组合：
```text
relkind
relpersistence
reltablespace
relfilenode
relisshared
database default tablespace
```
这也是 `RelationInitPhysicalAddr()` 存在的原因。
### 4.2 `RelFileNumber`、`RelFileLocator` 与 `rd_locator`
`RelFileNumber` 是 relation 物理文件号。
它不是 SQL object OID 的别名。
一个 relation 的 `relid` 可以稳定存在，而 `relfilenumber` 被 rewrite 替换。
`RelFileLocator` 才是 storage 层真正需要的三元组：
```text
spcOid
dbOid
relNumber
```
`spcOid` 表示 tablespace。
`dbOid` 表示 database。
shared relation 位于 `pg_global`，物理层通常用 `InvalidOid` 表示 database 部分。
`relNumber` 是当前 filenumber。
`RelationData.rd_locator` 是 relcache 对这个三元组的 backend-local 计算结果。
普通 relation：
```text
rd_locator.relNumber = rd_rel->relfilenode
```
mapped relation：
```text
rd_locator.relNumber =
    RelationMapOidToFilenumber(rd_id, rd_rel->relisshared)
```
这意味着 `rd_locator` 是快照式结果。
它会随着 relcache build/rebuild、relmap invalidation、`RelationInitPhysicalAddr()` 重跑而变化。
它不是磁盘上的持久权威状态。
### 4.3 `RelationData` 中与物理身份相关的字段
本节只关心 `RelationData` 的一组字段。
| 字段 | 语义 |
| --- | --- |
| `rd_id` | relation 的 catalog OID。 |
| `rd_rel` | `pg_class` fixed fields 的本地拷贝。 |
| `rd_locator` | 当前 backend 计算出的物理 locator。 |
| `rd_backend` | local/temp relation 的 backend identity。 |
| `rd_smgr` | 已打开的 smgr handle。 |
| `rd_refcnt` | 当前 backend 中 open reference 数。 |
| `rd_isvalid` | relcache 语义是否仍可用于 future lookup。 |
| `rd_createSubid` | relation 是否在当前事务/子事务中新建。 |
| `rd_newRelfilelocatorSubid` | 当前事务中最近一次 relfilenumber 变化。 |
| `rd_firstRelfilelocatorSubid` | 当前事务中最早仍存活的 relfilenumber 变化边界。 |
`rd_smgr` 依赖 `rd_locator`。
如果 `rd_locator` 变化，旧 smgr handle 不能继续代表当前文件。
relcache invalidation、smgr invalidation 和 storage pending-delete 共同维护这个边界。
`rd_refcnt` 只保证这个 `Relation *` 在本 backend 中不会被物理释放。
它不保证 `rd_locator` 代表最新文件。
和前几节一样，要分清：
```text
pointer safety != semantic freshness
```
### 4.4 relmapper 文件：`pg_filenode.map`
`relmapper.c` 维护两个层面的 map 文件。
| map | 位置 | 覆盖对象 |
| --- | --- | --- |
| shared map | `global/pg_filenode.map` | shared mapped catalog。 |
| local map | database path 下的 `pg_filenode.map` | 当前 database 的 local mapped catalog。 |
文件内容不是无限增长的 hash table。
源码中 `RelMapFile` 固定包含：
```text
magic
num_mappings
mappings[MAX_MAPPINGS]
crc
```
`MAX_MAPPINGS` 当前是 64。
这表达了设计意图：
```text
relmapper 只服务少量 nailed/mapped catalog，
不是用户 relation 的通用 metadata store。
```
每个 entry 是：
```text
mapoid -> mapfilenumber
```
map 文件用 CRC 检测损坏。
写入时先写临时文件，再 `durable_rename()` 到正式文件名。
原因是 map 文件是 critical data。
如果丢失或读到半写入内容，系统可能连核心 catalog 的物理文件都找不到。
### 4.5 relmapper 的 backend-local 状态
每个 backend 会在静态变量中保存当前已加载的 map。
```text
shared_map
local_map
```
事务中还有两组增量状态：
```text
active_shared_updates
active_local_updates
pending_shared_updates
pending_local_updates
```
`pending_*` 的语义是：
```text
已经被告知有 mapping 更新，
但要到下一个 CommandCounterIncrement 才对本 backend 的 future lookup 可见。
```
`active_*` 的语义是：
```text
当前事务已经可见的 mapping 更新；
RelationMapOidToFilenumber() 应该优先相信它们。
```
这个设计让 relmapper 模拟 `pg_class.relfilenode` 的 command visibility。
普通 catalog update 在 `CCI` 后对本事务下一条命令可见。
relmap update 也需要类似边界。
否则同一事务内部可能出现：
```text
pg_class 相关状态还停在旧 command 可见性；
relmapper 却提前让 future lookup 看到新文件。
```
### 4.6 `RelfilenumberMapHash`
`relfilenumbermap.c` 是反向 cache。
key 是：
```text
reltablespace
relfilenumber
```
value 是：
```text
relid
```
它回答的问题是：
```text
我已经从 WAL、文件路径或 SQL 函数拿到了 physical filenumber，
它对应哪个 pg_class.oid？
```
它不回答：
```text
打开 relid 时应该访问哪个当前文件？
```
正向打开仍然走 relcache + `pg_class` / relmapper。
反向 cache 的典型使用者包括：
- `pg_filenode_relation()`。
- `pg_relation_filepath()` 相关诊断路径。
- logical decoding reorder buffer 从 WAL change 的 `RelFileLocator` 找 relation OID。
反向 cache 会注册 relcache callback。
当 `pg_class` 相关 relcache invalidation 发生时，它删除匹配 relation 的 entry。
如果 `relid == InvalidOid` 表示完整 reset，它清空全部 entry。
它还总是删除 negative cache entry，因为一个之前不存在的 filenumber 可能在 catalog change 后出现。
## 5. 主流程源码 walkthrough
### 5.1 普通 relation 打开：relid 到 `rd_locator`
主路径从 `relation_open()` 或 `table_open()` 进入 relcache。
本节抽象掉 lock acquisition 和完整 relcache build，只看物理身份。
```text
relation_open(relid, lockmode)
  -> RelationIdGetRelation(relid)
     -> cache hit or RelationBuildDesc()
        -> pg_class tuple copied into rd_rel
        -> RelationInitPhysicalAddr(relation)
           -> decide tablespace
           -> decide database OID
           -> decide filenumber
  -> caller uses RelationGetSmgr()
     -> smgropen(rd_locator, rd_backend)
```
`RelationInitPhysicalAddr()` 先处理没有 storage 的 relation kind。
没有 storage 就直接返回。
然后它计算 tablespace。
```text
if reltablespace is set:
    spcOid = reltablespace
else:
    spcOid = MyDatabaseTableSpace
```
接着计算 database 部分。
```text
if spcOid == GLOBALTABLESPACE_OID:
    dbOid = InvalidOid
else:
    dbOid = MyDatabaseId
```
这里的注释很重要：
```text
at the physical level, relations in pg_global tablespace must be treated as shared,
even if relisshared isn't set
```
也就是说，物理路径不完全由 `relisshared` 决定。
`relisshared` 参与 relmapper lookup。
`spcOid == GLOBALTABLESPACE_OID` 决定物理 locator 的 database 部分。
最后决定 filenumber。
普通 relation：
```text
if rd_rel->relfilenode:
    rd_locator.relNumber = rd_rel->relfilenode
```
mapped relation：
```text
else:
    rd_locator.relNumber =
        RelationMapOidToFilenumber(rd_id, rd_rel->relisshared)
```
如果 map 找不到，源码报错：
```text
could not find relation mapping for relation ..., OID ...
```
这类错误不是普通 cache miss。
它意味着 catalog 说这是 mapped relation，但 relmapper 没有对应物理文件号。
### 5.2 mapped relation lookup：active updates 优先
`RelationMapOidToFilenumber()` 的查找顺序不是只读磁盘 map。
它先看当前事务 active updates。
shared case：
```text
active_shared_updates
  -> shared_map
```
local case：
```text
active_local_updates
  -> local_map
```
原因是当前事务可能刚刚 rewrite 了 mapped relation。
在下一条 command 中，它应该看到自己已经激活的 mapping。
但是这个 mapping 还没写入永久 map 文件。
事务 commit 前，`active_*` 是当前 backend 的局部事实。
commit 时才会变成 map 文件事实，并通过 sinval 通知其它 backend。
### 5.3 新建 relation：普通 relation 与 mapped relation 的分叉
`RelationBuildLocalRelation()` 创建本地 relcache entry 时就分叉。普通 relation 直接把 filenumber 放入 `rd_rel->relfilenode`：
```text
rd_rel->relfilenode = relfilenumber
RelationInitPhysicalAddr(rel)
```
mapped relation 则把 `relfilenode` 置 0，并把真实 filenumber 放入 relmapper：
```text
rd_rel->relfilenode = InvalidRelFileNumber
RelationMapUpdateMap(relid, relfilenumber, shared_relation, true)
RelationInitPhysicalAddr(rel)
```
`immediate = true` 是为了让当前构造过程立刻能完成 `rd_locator` 初始化。
### 5.4 普通 rewrite：更新 `pg_class.relfilenode`
`RelationSetNewRelfilenumber()` 是理解物理身份变化的主入口。
它用于把一个 relation 指向新的物理文件号。
调用者必须已经持有 relation 的 exclusive lock。
主流程：
```text
RelationSetNewRelfilenumber(relation, persistence)
  -> GetNewRelFileNumber()
  -> SearchSysCacheLockedCopy1(RELOID, relid)
  -> RelationDropStorage(old relation)
  -> create storage for new RelFileLocator
  -> if mapped:
         RelationMapUpdateMap(..., immediate = false)
         CacheInvalidateRelcache(relation)
     else:
         classform->relfilenode = newrelfilenumber
         reset relpages/reltuples if needed
         CatalogTupleUpdate(pg_class, tuple)
```
普通 relation 的持久映射变化就是 `pg_class` tuple update。
这个 update 通过 catalog update 机制产生 syscache/relcache invalidation。
commit/abort 时，storage pending-delete 机制决定删除旧文件还是新文件。
所以普通 rewrite 的提交语义可以压缩成：
```text
pg_class row commit
  -> new relfilenode becomes authoritative
  -> relcache invalidation makes future opens recompute rd_locator
  -> pending storage delete removes no-longer-current files at the right boundary
```
### 5.5 mapped rewrite：更新 relmapper 而不是 `pg_class`
mapped relation 不能简单更新自己的 `pg_class.relfilenode`：`pg_class` 或核心 index 不能依赖“先打开 `pg_class` 再知道自己的文件在哪里”，shared catalog relocation 也不能只改某个 database 的 `pg_class` 行。因此 mapped rewrite 走：
```text
RelationMapUpdateMap(relid, newrelfilenumber, shared, false)
CacheInvalidateRelcache(relation)
```
`immediate = false` 表示先进入 pending map updates。
下一次 `CommandCounterIncrement()`：
```text
AtCCI_RelationMap()
  -> merge pending updates into active updates
```
事务 commit：
```text
AtEOXact_RelationMap(true, false)
  -> perform_relmap_update(shared/local)
  -> write pg_filenode.map
  -> send relmap sinval
  -> clear active updates
```
abort：
```text
AtEOXact_RelationMap(false, ...)
  -> discard active and pending updates
```
这条路径的限制是：
```text
mapped catalogs can only be relocated by operations such as VACUUM FULL and CLUSTER,
which make no transactionally-significant changes
```
原因是 map 文件更新本身几乎就是 physical relocation 的 commit。源码注释承认存在窗口：
```text
map file update 已经提交了 physical relocation，
但外层事务仍理论上可能 abort。
```
PostgreSQL 通过限制 mapped catalog relocation 的语义来接受这个复杂性。
### 5.6 relmap 文件写入：WAL、rename、sinval、preserve
`write_relmap_file()` 是 relmapper 正确性的核心。
它要求 caller 持有 `RelationMappingLock` exclusive。
写入流程：
```text
fill magic and CRC
write pg_filenode.map.tmp
close temp file
if write_wal:
    START_CRIT_SECTION()
    XLogInsert(RM_RELMAP_ID, XLOG_RELMAP_UPDATE)
    XLogFlush(lsn)
durable_rename(tmp, pg_filenode.map)
if send_sinval:
    CacheInvalidateRelmap(dbid)
if preserve_files:
    RelationPreserveStorage(each mapped file)
if write_wal:
    END_CRIT_SECTION()
```
几个边界必须分开看。
CRC 解决文件损坏检测。
临时文件 + durable rename 解决 overwrite-in-place 的 torn write 风险。
WAL record 解决 crash recovery。
`XLogFlush(lsn)` 保证 WAL 先于 data update 落盘。
sinval 让其它 backend 重新读取 map 文件。
`RelationPreserveStorage()` 防止外层事务 abort 时把已经被 map 文件指向的新文件删掉。
这些机制各自解决不同问题。
不能把它们合并成一句“写 map 文件就行”。
### 5.7 relmap invalidation：重新读文件，而不是传新 map
shared invalidation message 对 relmap 的 payload 很小。
它只表达：
```text
某个 database 或 shared relmap 文件过期了。
```
receiver 执行：
```text
RelationMapInvalidate(shared)
  -> if map already loaded:
         load_relmap_file(shared, false)
```
如果是 overflow：
```text
RelationMapInvalidateAll()
  -> reload loaded shared/local maps
```
注意这个条件：
```text
reload only if the map is valid now
```
例如 autovacuum launcher 没有 attach 到某个具体 database。
它不应该尝试读取 local map。
这和前几节 shared invalidation 的规律一致：
```text
shared message 只告诉 receiver “你本地某类语义可能过期”；
receiver 根据自己是否拥有相关本地状态决定怎么清。
```
### 5.8 反向路径：filenumber 到 relid
`RelidByRelfilenumber()` 服务的是另一种场景。
例如 `pg_filenode_relation(tablespace, filenumber)` 已经拿到了物理文件号。
或者 logical decoding 从 WAL change 里拿到了 `RelFileLocator`。
主流程：
```text
RelidByRelfilenumber(reltablespace, relfilenumber)
  -> normalize MyDatabaseTableSpace to 0 for pg_class scan
  -> check RelfilenumberMapHash
  -> if global tablespace:
         RelationMapFilenumberToOid(relfilenumber, true)
     else:
         scan pg_class by (reltablespace, relfilenode)
         ignore temp relations
         if not found:
             RelationMapFilenumberToOid(relfilenumber, false)
  -> cache relid or InvalidOid
```
这个 cache 有 negative entry。
如果找不到，也缓存 `InvalidOid`。
但是 invalidation callback 会删除 negative entry。
因为一个 filenumber 之前不存在，不代表 catalog change 后仍不存在。
这条路径的一个重要边界：
```text
temporary relation may share its relfilenumber with permanent or other backend temp relation。
```
函数不知道其它 backend 的 proc number。
所以它忽略临时 relation。
这解释了为什么某些诊断函数对 temp relation 返回 NULL 或不能可靠反查。
## 6. 生命周期 / ownership / cleanup
### 6.1 谁创建普通 relation 的物理文件
普通 relation 创建或 rewrite 时，会先分配新的 `RelFileNumber`。
随后构造 `RelFileLocator`。
`RelationCreateStorage()` 负责创建 main fork。
```text
RelationCreateStorage(rlocator, relpersistence, register_delete)
  -> decide procNumber and needs_wal
  -> smgropen(rlocator, procNumber)
  -> smgrcreate(MAIN_FORKNUM)
  -> if permanent:
         log_smgrcreate()
  -> if register_delete:
         add PendingRelDelete
```
`relpersistence` 决定 WAL 和 backend identity。
temporary relation：
```text
procNumber = ProcNumberForTempRelations()
needs_wal = false
```
unlogged relation：
```text
procNumber = INVALID_PROC_NUMBER
needs_wal = false
```
permanent relation：
```text
procNumber = INVALID_PROC_NUMBER
needs_wal = true
```
`register_delete` 把新建文件登记到 pending delete 链表。
如果事务 abort，新文件会被删掉。
如果 commit，新文件保留。
### 6.2 谁持有 relation 的物理身份
持久权威状态分两类。
普通 relation：
```text
pg_class.relfilenode
```
mapped relation：
```text
pg_filenode.map
```
运行时消费状态是：
```text
RelationData.rd_locator
SMgrRelation.smgr_rlocator
BufferTag.rlocator
WAL record locator
```
这些运行时状态都可能比持久权威状态滞后。
它们靠 relcache invalidation、smgr invalidation、buffer invalidation、redo 和 checkpoint 边界逐步收敛。
### 6.3 谁释放或删除旧文件
旧文件删除通常不是直接在 catalog update 那一行完成。
`RelationDropStorage()` 会把 relation 的当前 storage 记录到 pending delete。
事务结尾根据 `atCommit` 决定删除时机。
`RelationPreserveStorage()` 则用于反向保护：
```text
这个 RelFileLocator 原本可能在 abort cleanup 中被删除，
但现在 relmap 已经指向它，必须保留。
```
mapped relation commit map 文件时会调用它。
这说明 mapped relation 的 ownership 比普通 relation 更拧巴：
```text
map 文件一旦 durable rename 成功，
物理文件就必须被当成新权威；
事务 abort cleanup 不能再删除它。
```
### 6.4 ERROR / abort 时谁兜底
普通 relation：
```text
new storage created
  -> pending delete registered for abort
  -> pg_class update aborts
  -> new storage deleted
  -> old pg_class.relfilenode remains authoritative
```
mapped relation：
```text
pending/active relmap updates exist only in backend-local state
  -> abort before map file write
  -> AtEOXact_RelationMap(false) discards them
  -> storage cleanup handles transient files
```
commit map file write 之后：
```text
write_relmap_file()
  -> WAL flushed
  -> durable rename completed
  -> sinval sent
  -> files preserved
```
如果中间失败并且已经进入 critical section，`ERROR` 可能升级为 `PANIC`。
这是有意的。
让其它 backend 带着 stale mapping 继续运行，比数据库重启恢复更危险。
## 7. 正确性机制层次
这个主题的 correctness 不是一个机制保证的。
需要把层次拆开。
| 层次 | 机制 | 保证 | 不保证 |
| --- | --- | --- | --- |
| catalog identity | `pg_class.oid` | 逻辑对象身份稳定。 | 文件号稳定。 |
| physical identity | `RelFileLocator` | smgr/buffer/WAL 可以定位文件。 | relation 语义新鲜。 |
| normal mapping | `pg_class.relfilenode` | 普通 relation 当前文件号事务化更新。 | mapped catalog bootstrap。 |
| mapped mapping | `pg_filenode.map` | 核心/共享 catalog 不依赖自身 `pg_class` 行。 | 任意 relation 都可走 map。 |
| command visibility | `pending_*` -> `AtCCI_RelationMap()` -> `active_*` | relmap 更新模拟 catalog CCI 可见性。 | 跨 backend 可见。 |
| transaction boundary | `AtEOXact_RelationMap()` | commit 时写 map，abort 时丢弃本地 updates。 | 消除所有物理窗口。 |
| durability | relmap WAL + `XLogFlush()` + `durable_rename()` | crash 后 map 文件可恢复到一致状态。 | 避免所有启动失败；CRC 错仍会 FATAL。 |
| cache freshness | relcache/relmap sinval | 其它 backend 重新读 map 或重建 locator。 | 阻塞并发读者。 |
| pointer safety | relcache refcount / smgr pin | 本 backend 指针不被提前释放。 | 指针代表最新语义。 |
| storage cleanup | pending deletes / preserve | abort/commit 删除正确一侧文件。 | 解释 SQL 对象身份。 |
### 7.1 为什么需要 relation lock
`RelationSetNewRelfilenumber()` 要求 caller 已经持有 exclusive lock。
这不是为了保护 `rd_locator` 的 C 指针。
它保护的是逻辑对象级 rewrite/relocation 并发。
relmapper 的 `perform_relmap_update()` 注释也强调：
```text
Anyone updating a relation's mapping info should take exclusive lock on that rel and hold it until commit.
```
同一个 map 文件里不同 relation 的并发更新由 `RelationMappingLock` 串行化。
同一个 relation 的并发 rewrite 由 relation lock 排序。
这两个锁保护的对象不同。
### 7.2 为什么 `RelationMappingLock` 是 LWLock
map 文件是少量 critical metadata。
读文件和写文件需要防止并发 rename/open 的平台差异。
`read_relmap_file()` 在读文件前拿 shared `RelationMappingLock`。
`write_relmap_file()` 要求 exclusive lock。
checkpoint 的 `CheckPointRelationMap()` 只拿 shared 再释放。
这不是为了读 map 的 hash lookup 热路径。
`RelationMapOidToFilenumber()` 查的是 backend-local copy，不拿 LWLock。
锁保护的是磁盘文件读写与 checkpoint/recovery 的边界。
### 7.3 为什么 relmap 需要 WAL
`pg_filenode.map` 是普通文件，不是 heap page。
不能依赖 heap WAL 记录重放它的变化。
因此 relmapper 定义自己的 WAL record：
```text
RM_RELMAP_ID / XLOG_RELMAP_UPDATE
```
record 包含：
```text
dbid
tsid
nbytes
RelMapFile data
```
redo 时重建目标 path，并调用写 map 文件逻辑。
这保证 crash recovery 能恢复 map update。
如果没有这个 WAL，base backup 或 crash restart 可能看到 catalog 和物理文件状态不一致。
## 8. 错误路径 / 异常路径 / fallback
### 8.1 relmap 文件读不到或 CRC 错
`load_relmap_file()` 对 shared/local map 使用 `FATAL` 级别。
原因很直接：
```text
无法读 mapping 文件，就无法可靠打开核心 mapped catalog。
```
`read_relmap_file()` 检查：
- 文件必须能 open/read/close。
- 读取字节数必须等于 `sizeof(RelMapFile)`。
- `magic` 必须正确。
- `num_mappings` 必须在范围内。
- CRC 必须匹配。
任何失败都不是普通 cache miss。
这是 cluster critical metadata 损坏或不可访问。
### 8.2 map 文件写入过程中失败
写入 map 文件的危险区间是：
```text
WAL inserted and flushed
  -> durable rename
  -> sinval
  -> preserve files
```
一旦进入 critical section，很多 `ERROR` 会转为 `PANIC`。
这看起来激烈，但符合系统目标。
如果 map 文件已经被替换，而 sinval 没发出去，别的 backend 可能继续用旧 filenumber 读核心 catalog。
如果 map 指向新文件，但 abort cleanup 删除了新文件，重启后也会坏。
在这些场景中，立即崩溃并走 WAL recovery 是更可控的选择。
### 8.3 subtransaction 和 parallel mode 禁止修改 relmap
`RelationMapUpdateMap()` 明确拒绝：
```text
GetCurrentTransactionNestLevel() > 1
IsInParallelMode()
```
源码注释说可以通过更多 bookkeeping 支持，但当前不值得。
原因是 relmap update 已经有 pending/active 两层事务状态。
如果允许 subtransaction，还要记录每个 subxact 的映射可见性、回滚、上移、parallel serialization。
对于少量 mapped catalog rewrite，这个复杂度不划算。
### 8.4 PREPARE TRANSACTION 禁止 relmap 修改
`AtPrepare_RelationMap()` 如果发现 active 或 pending updates，就报：
```text
cannot PREPARE a transaction that modified relation mapping
```
two-phase commit 要求 prepare 后状态能跨崩溃、跨时间恢复。
relmap 更新的物理提交语义和文件 preserve/delete 边界太特殊。
PostgreSQL 选择直接禁止。
### 8.5 logical decoding 中 relmapper 没有 historic view
`reorderbuffer.c` 从 WAL change 的 `RelFileLocator` 调用：
```text
RelidByRelfilenumber(spcOid, relNumber)
```
源码注释提醒：
```text
relmapper has no historic view
```
普通 catalog 可以用 historic snapshot 解码旧 catalog tuple。
relmapper 只保存当前 map 文件和当前 backend-local updates。
如果 mapped catalog 被重复 rewrite，logical decoding 可能无法把旧 filenumber 反查到 relid。
这是可以接受的，因为 catalog table changes 通常不会被逻辑解码输出。
这也是本节最重要的版本/时间边界之一：
```text
relmap 是当前物理身份映射，不是历史映射数据库。
```
## 9. 成本、资源与跨模块传播
### 9.1 正向打开的成本
普通 relation 的正向 filenumber 解析在 relcache build 时完成。
hot path 命中 relcache 时，caller 通常直接使用已有 `RelationData`。
mapped relation lookup 是对最多 64 个 mapping 的线性扫描。
这看起来不是 O(1)，但对象数量极少。
源码甚至注释说没有必要按 OID 排序。
真正成本不在 `RelationMapOidToFilenumber()` 本身。
它在：
- relcache miss 需要读 catalog。
- relmap invalidation 后需要重新读 `pg_filenode.map`。
- rewrite 需要新建文件、WAL、fsync/rename、sinval。
- 反向 cache miss 可能扫描 `pg_class` index。
### 9.2 relmap update 的成本
relmap update 很重。
一次 commit 需要：
```text
RelationMappingLock exclusive
load current map file
merge updates
write temp file
WAL insert
WAL flush
durable rename
send sinval
preserve files
```
但是它只发生在 mapped catalog relocation 这类低频操作中。
因此 PostgreSQL 接受简单的全局 `RelationMappingLock`，没有为每个 map 文件做复杂锁分片。
### 9.3 relfilenumbermap 反向 cache 的成本
`RelidByRelfilenumber()` miss 时可能：
```text
open pg_class
scan ClassTblspcRelfilenodeIndexId
skip temp relations
fallback to RelationMapFilenumberToOid()
enter positive or negative cache
```
这个成本不适合每个 buffer access 都做。
它适合诊断函数、logical decoding、少量文件路径反查。
cache invalidate 时用 hash_seq 全表扫描删除匹配 entry。
这也说明该 cache 预期很小。
### 9.4 cache invalidation 扩散
一次物理身份变化会触发多层失效。
普通 relation rewrite：
```text
pg_class update
  -> catcache/syscache invalidation
  -> relcache invalidation
  -> relfilenumbermap callback 删除反向 entry
  -> smgr/storage pending delete
```
mapped relation rewrite：
```text
RelationMapUpdateMap
  -> relcache manual invalidation
  -> AtCCI activates local map update
  -> commit writes map file
  -> CacheInvalidateRelmap
  -> receivers reload relmap
  -> relcache rebuild recomputes rd_locator
```
这条传播不是同步全局锁步。
每个 backend 在自己的安全点消费 invalidation。
已经持有的 `Relation *` 可能暂时仍指向旧 `rd_locator`。
正确性依赖上层 lock、防止 rewrite 与访问越界并发，以及 relcache 在 future lookup 上刷新。
## 10. 观测与诊断入口
### 10.1 能直接看到什么
SQL 层可以看到一部分状态。
普通 relation：
```sql
SELECT oid, relname, relfilenode, reltablespace, relkind, relpersistence
FROM pg_class
WHERE relname = 'your_table';
```
如果 `relfilenode` 非 0，它通常就是当前物理文件号。
mapped relation：
```sql
SELECT oid, relname, relfilenode
FROM pg_class
WHERE relfilenode = 0
ORDER BY oid;
```
这里看到的是“走 relmapper”，不是当前文件号。
当前文件号可以用：
```sql
SELECT pg_relation_filenode('pg_class'::regclass);
SELECT pg_relation_filepath('pg_class'::regclass);
```
反向查询：
```sql
SELECT pg_filenode_relation(0, pg_relation_filenode('some_table'::regclass));
```
注意 temp relation 和 mapped historic state 可能查不到。
### 10.2 gdb 断点入口
建议断点：
```text
RelationInitPhysicalAddr
RelationMapOidToFilenumber
RelationSetNewRelfilenumber
RelationMapUpdateMap
AtCCI_RelationMap
AtEOXact_RelationMap
write_relmap_file
RelidByRelfilenumber
RelfilenumberMapInvalidateCallback
```
观察变量：
```text
relation->rd_id
relation->rd_rel->relfilenode
relation->rd_rel->relisshared
relation->rd_locator
active_local_updates.num_mappings
pending_local_updates.num_mappings
newrelfilenumber
```
### 10.3 诊断 stale filenode 的顺序
遇到 “文件找不到”、“OID 能查到但 filepath 变化”、“logical decoding 无法反查 relation” 这类问题，按顺序拆。
第一，确认对象是否有 storage。
```sql
SELECT relkind, relpersistence, relfilenode
FROM pg_class
WHERE oid = 'obj'::regclass;
```
第二，确认是否 mapped。
```text
RELKIND_HAS_STORAGE && relfilenode = 0
```
第三，确认当前路径。
```sql
SELECT pg_relation_filenode('obj'::regclass),
       pg_relation_filepath('obj'::regclass);
```
第四，判断是否刚经历 rewrite。
典型命令：
```text
VACUUM FULL
CLUSTER
REINDEX
TRUNCATE
ALTER TABLE ... SET TABLESPACE
```
第五，区分前台 lookup 问题和 recovery/replication 问题。
如果只有 logical decoding 报旧 filenumber 反查失败，重点看 historic boundary。
如果所有 backend 打不开 mapped catalog，重点看 relmap 文件、CRC、WAL recovery。
## 11. 常见误区
### 误区 1：把 relation OID 当成文件名
`pg_class.oid` 是逻辑对象身份。
物理文件名由 `RelFileNumber` 决定。
在新建早期或一些系统对象上二者可能相等，但这不是不变量。
rewrite 后最容易打破这个假设。
### 误区 2：看到 `relfilenode = 0` 就认为 relation 没有 storage
是否有 storage 先看 `relkind`。
如果 `RELKIND_HAS_STORAGE(relkind)` 为真，`relfilenode = 0` 表示 mapped relation。
它的 filenumber 在 relmapper 中。
### 误区 3：认为 relmapper 是普通用户表的替代 catalog
relmapper 的 `MAX_MAPPINGS` 很小。
它为 nailed/shared mapped catalog 解决 bootstrap 和共享 catalog relocation 问题。
普通用户表不应该进入 relmapper。
### 误区 4：认为 relcache invalidation 会立即修改所有 backend 的 `rd_locator`
shared invalidation 只发送过期事实。
receiver 在安全点本地处理。
正在持有的 `Relation *` 可能延迟 flush 或 rebuild。
对象级并发安全靠 relation lock 与事务边界配合。
### 误区 5：把 `RelidByRelfilenumber()` 当成无副作用的小函数
它可能打开 `pg_class`、扫描 index、触发 cache 初始化和 invalidation callback。
在底层锁持有期间调用可能造成死锁风险。
它是诊断/解码辅助，不是 buffer hot path 工具。
### 误区 6：认为 relmapper 有历史版本
relmapper 保存当前 mapping。
它没有 MVCC historic snapshot。
logical decoding 中 mapped catalog rewrite 可能导致旧 filenumber 反查失败。
这是已知边界。
### 误区 7：认为 WAL 只需要记录 relation data page
relmap 文件不是 heap page。
它需要 `RM_RELMAP_ID` 自己的 WAL record。
storage create/truncate 还会走 `RM_SMGR_ID`。
不同持久状态有不同 WAL resource manager。
## 12. 课堂实验
### 实验 1：普通 relation rewrite 后 OID 不变、filenode 改变
目标：看到 catalog identity 与 physical identity 分离。
步骤：
```sql
CREATE TABLE t_relmap_demo (id int, payload text);
INSERT INTO t_relmap_demo
SELECT g, repeat('x', 100)
FROM generate_series(1, 1000) AS g;
SELECT oid, relfilenode, pg_relation_filenode(oid), pg_relation_filepath(oid)
FROM pg_class
WHERE oid = 't_relmap_demo'::regclass;
VACUUM FULL t_relmap_demo;
SELECT oid, relfilenode, pg_relation_filenode(oid), pg_relation_filepath(oid)
FROM pg_class
WHERE oid = 't_relmap_demo'::regclass;
```
预期：
```text
oid 不变；
relfilenode / pg_relation_filenode() 可能改变；
filepath 随 filenumber 改变。
```
回到源码：
```text
RelationSetNewRelfilenumber()
  -> classform->relfilenode = newrelfilenumber
  -> CatalogTupleUpdate(pg_class, ...)
  -> relcache invalidation
```
### 实验 2：观察 mapped relation 的 `relfilenode = 0`
目标：理解 mapped relation 的 catalog 表示。
步骤：
```sql
SELECT oid, relname, relkind, relisshared, relfilenode,
       pg_relation_filenode(oid) AS current_filenode,
       pg_relation_filepath(oid) AS path
FROM pg_class
WHERE relfilenode = 0
  AND relkind IN ('r', 'i', 'S', 't', 'm')
ORDER BY oid
LIMIT 20;
```
预期：
```text
relfilenode = 0；
pg_relation_filenode() 仍能返回当前 filenumber；
pg_relation_filepath() 仍能返回真实路径。
```
回到源码：
```text
RelationIsMapped(relation)
RelationInitPhysicalAddr()
RelationMapOidToFilenumber()
```
### 实验 3：断点跟踪 relcache 如何得到 `rd_locator`
目标：把 SQL 现象落到 `RelationInitPhysicalAddr()`。
建议在测试实例上：
```text
break RelationInitPhysicalAddr
commands
  print relation->rd_id
  print relation->rd_rel->relfilenode
  print relation->rd_rel->relisshared
  continue
end
```
分别执行：
```sql
SELECT count(*) FROM t_relmap_demo;
SELECT count(*) FROM pg_class;
```
观察：
```text
普通表 relfilenode 非 0，直接进入 rd_locator.relNumber。
mapped catalog relfilenode 为 0，调用 RelationMapOidToFilenumber()。
```
### 实验 4：反向 cache 与 `pg_filenode_relation()`
目标：理解 `(tablespace, filenumber) -> relid` 的边界。
步骤：
```sql
SELECT pg_relation_filenode('t_relmap_demo'::regclass) AS fn \gset
SELECT pg_filenode_relation(0, :fn);
VACUUM FULL t_relmap_demo;
SELECT pg_filenode_relation(0, :fn);
SELECT pg_filenode_relation(0, pg_relation_filenode('t_relmap_demo'::regclass));
```
预期：
```text
旧 filenumber 可能不再反查到 relation；
新 filenumber 能反查到同一个 relid。
```
回到源码：
```text
RelidByRelfilenumber()
RelfilenumberMapInvalidateCallback()
ClassTblspcRelfilenodeIndexId
```
## 13. 讨论题
1. 为什么 `pg_class.oid` 不能直接作为长期文件名？
2. `relfilenode = 0` 的语义为什么必须和 `relkind` 一起解释？
3. 如果 `pg_class` 自己的物理文件号也只存在 `pg_class.relfilenode` 中，启动会遇到什么循环？
4. shared catalog relocation 为什么不能要求修改每个 database 的 `pg_class`？
5. `RelationMappingLock` 和 relation exclusive lock 各自保护什么？
6. 为什么 relmap update 要在 `CommandCounterIncrement()` 后才从 pending 变 active？
7. `write_relmap_file()` 为什么在发送 sinval 失败时宁可 PANIC？
8. `RelidByRelfilenumber()` 为什么不能可靠处理其它 backend 的 temporary relation？
9. logical decoding 为什么可以接受 mapped catalog filenumber 反查失败？
10. 如果一个问题表现为 `pg_relation_filepath()` 与文件系统不一致，应先查 relcache、relmapper、smgr 还是 WAL recovery？为什么？
## 14. 本节小结
本节只围绕一个问题：
```text
PostgreSQL 如何把稳定的 catalog identity 连接到可变化的物理文件 identity？
```
核心链路是：
```text
relid
  -> relcache
  -> pg_class.relfilenode or pg_filenode.map
  -> rd_locator
  -> smgr
```
普通 relation 的当前物理文件号由 `pg_class.relfilenode` 持久化。
mapped relation 的 `pg_class.relfilenode` 为 0，当前物理文件号由 relmapper 的 `pg_filenode.map` 持久化。
`RelationInitPhysicalAddr()` 是正向路径的关键汇合点。
它把 tablespace、database、filenumber 计算成 `RelationData.rd_locator`。
storage manager 后续只消费 `RelFileLocator`，不再理解 catalog identity。
rewrite 让物理 identity 变化。
普通 relation 通过 `pg_class` tuple update 提交变化。
mapped relation 通过 pending/active relmap updates、CCI、EOXact、WAL、durable rename 和 sinval 提交变化。
这条路径的正确性依赖多层机制：
```text
relation lock 排序对象级 rewrite；
RelationMappingLock 串行化 map 文件读写；
WAL + durable rename 保证 crash safety；
sinval 让其它 backend 重新读 map；
pending storage delete / preserve 决定 abort/commit 后哪些文件留下；
relcache refcount 保证指针安全，但不保证语义新鲜。
```
`relfilenumbermap.c` 提供反向 cache。
它从 `(tablespace, relfilenumber)` 找 `relid`，服务诊断函数和 logical decoding。
它不是正向打开路径，也不是通用对象目录。
它没有 temp backend identity，也没有 relmapper historic view。
本节可迁移的系统规律是：
```text
当系统同时需要稳定逻辑身份和可替换物理身份时，
不要让一个字段承担所有语义；
用持久映射表达权威关系，
用本地 cache 压缩热路径，
用 invalidation 推进语义新鲜，
用 WAL/rename/cleanup 处理 crash 与 abort，
并明确哪些反向查询只是诊断近似。
```
下一步继续追 storage persistence 时，要把这里的 `RelFileLocator` 带入 buffer tag、fork、block、SMgrRelation 和 WAL record。
