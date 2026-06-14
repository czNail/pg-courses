# PostgreSQL catalog snapshot / syscache consistency
## 课程定位
前置知识：已经理解 `MemoryContext`、`ResourceOwner`、MVCC snapshot、command counter、catcache/syscache、relcache、shared invalidation 和 heavyweight lock 的基本边界。

本节唯一主问题：

```text
catalog snapshot、syscache 命中和 invalidation 顺序如何共同保证 DDL/DML 看到一致的元数据？
```

核心矛盾：

```text
元数据读取必须像 backend-local cache hit 一样便宜；
但元数据事实来自会被 DDL 修改的 catalog tuple，
并且同一 backend 的 command boundary 与其它 backend 的 commit boundary 不同。
```

一句话运行模型：

```text
catalog scan 使用可复用但会被 invalidation 丢弃的 CatalogSnapshot；
syscache hit 返回 backend-local CatCTup，不重新扫描 catalog；
catalog tuple 修改只先排队 invalidation；
CommandCounterIncrement() 让本 backend 清旧 snapshot/cache；
commit 后 AtEOXact_Inval(true) 再把过期事实发给其它 backend；
relation/object lock 把 DDL/DML 的对象级并发排进这条时间线。
```

学完后应能判断：为什么 syscache hit 不等于绕过一致性；为什么同一事务内 DDL 后下一条 DML 能看到新 metadata；为什么其它 backend 要等 commit 后 shared invalidation；为什么 name lookup 在 lock wait 后还要根据 invalidation retry；以及 stale metadata 问题该从 snapshot、syscache、relcache、plan cache 还是 lock ordering 入手。

本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e9`。

## 1. 本节在总主线中的位置
前面三节已经分别建立局部模型。

第 56 节的主线是：

```text
SearchSysCache()
  -> SearchCatCacheInternal()
  -> hit or miss
  -> CatCTup.refcount + ResourceOwner
  -> invalidation marks dead
```

第 57 节的主线是：

```text
RelationData 是 backend-local 描述符；
relcache invalidation 只传播 relid；
open Relation 不能被其它 backend 直接释放。
```

第 59 节的主线是：

```text
catalog change 先进入 pending invalidation；
command end 本地执行；
commit 后发送 shared invalidation；
receiver 在安全点本地清 cache。
```

本节把它们合成一个问题：

```text
一个 backend 执行 DDL 改了 catalog，
为什么自己下一条命令能看到新元数据，
而另一个 backend 不会在旧 syscache hit、新 catalog snapshot 和 lock wait 之间拼出不一致事实？
```

本节不重复完整 catcache 或 relcache 生命周期，而是看三类状态的交叉顺序。

| 状态 | 所在位置 | 本节关注点 |
| --- | --- | --- |
| `CatalogSnapshot` | backend-local static snapshot | catalog scan 何时复用，何时被 invalidation 丢弃。 |
| `CatCTup` / syscache entry | backend-local `CacheMemoryContext` | hit path 如何便宜返回，以及如何被标过期。 |
| pending/shared invalidation | backend-local transaction state + shared SI queue | catalog 修改如何在 command/commit 边界推进。 |

这三者共同解决的不是“有没有缓存”，而是：

```text
读 catalog 的时间点、
命中 cache 的时间点、
处理 DDL 失效的时间点，
必须落在同一个 command/transaction ordering 模型里。
```

本节锚定的 runtime truth：

```text
同一事务中 ALTER TABLE ADD COLUMN 后，下一条 SELECT 能看到新列；
另一个 backend 的 DML 或 prepared statement 只有在 DDL commit 后、
并在安全点处理 shared invalidation 后，才丢弃旧元数据。
```

这个现象不能只用 MVCC 解释。它还需要 `CatalogSnapshot` 在 command boundary 被丢弃，`CatCTup.dead` 阻止 future hit 返回旧 tuple，`AtEOXact_Inval(true)` 在释放 locks 前发送 shared messages，以及 `RangeVarGetRelidExtended()` 在 name lookup 和 lock wait 之间做 retry。

## 2. 核心矛盾与一句话运行模型
如果每次需要元数据都重新取 snapshot 并扫描系统表，parser、planner、executor 的高频 metadata lookup 会被 catalog IO 和 CPU 淹没。类型名、函数名、操作符、namespace、relation descriptor、partition 信息都可能反复访问 catalog。

如果只靠 backend-local cache，DDL 会制造 stale metadata。`CREATE TABLE` 后 negative syscache entry 可能继续说对象不存在；`ALTER TABLE ADD COLUMN` 后旧 `pg_attribute` 或 relcache 可能继续命中；`DROP` / `RENAME` 后 name lookup 可能锁住旧 OID；DDL abort 后本 backend 可能保留基于未提交 catalog 状态构造的 cache。

PostgreSQL 的实际模型是分层的。

| 层次 | 作用 | 不能替代什么 |
| --- | --- | --- |
| MVCC visibility | 决定 catalog tuple 版本是否对某个 snapshot 可见。 | 不负责清 backend-local cache。 |
| `CatalogSnapshot` | 让 catalog scan 在安全窗口内复用 snapshot。 | 不保证跨 command 永远新鲜。 |
| syscache/catcache | 让高频 tuple lookup 命中本地 hash。 | 不负责 DDL 互斥。 |
| command counter | 推进同一事务内 command 可见性。 | 不把未提交变化发给其它 backend。 |
| invalidation | 标记本地/远端 cache 语义过期。 | 不释放被 refcount 持有的指针。 |
| heavyweight lock | 排序 DDL/DML 对逻辑对象的并发访问。 | 不知道本地 `CatCTup` 是否 dead。 |

本节最短模型：

```text
syscache hit 是 fast path；
catalog snapshot 是 miss/scan path 的 visibility context；
invalidation 是 fast path 和 scan path 重新同步到 command/commit boundary 的时钟。
```

核心不变量：

```text
一个 backend 可以在一个 command 内复用旧 CatalogSnapshot 和 syscache entry；
但只要 catalog change 越过本 backend 的 CommandCounterIncrement()，
或其它 backend 的 commit invalidation 已被接收，
future lookup 就必须避开旧 entry 并使用新的 catalog snapshot。
```

同一 backend 的时间线：

```text
catalog update
  -> CurrentCmdInvalidMsgs
  -> CommandCounterIncrement()
  -> LocalExecuteInvalidationMessage()
  -> InvalidateCatalogSnapshot()
  -> SysCacheInvalidate() / RelationCacheInvalidateEntry()
  -> 下一条 command 重新 lookup
```

其它 backend 的时间线：

```text
DDL transaction commit
  -> AtEOXact_Inval(true)
  -> SendSharedInvalidMessages()
  -> release locks
  -> waiting backend wakes or next transaction starts
  -> AcceptInvalidationMessages()
  -> local cache cleanup
```

因此 README 主问题的答案是：

```text
CatalogSnapshot 约束 catalog scan 看到哪个 tuple version；
syscache hit 提供便宜的 backend-local metadata reuse；
invalidation ordering 决定什么时候旧 snapshot/cache 不再可用于 future lookup；
lock ordering 防止 DDL/DML 在对象级并发上越过这些边界。
```

## 3. 核心源码与阅读顺序
推荐按 runtime 时间线读，不按文件名排序。

| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/utils/time/snapmgr.c` | `GetCatalogSnapshot()`、`GetNonHistoricCatalogSnapshot()`、`InvalidateCatalogSnapshot()`。 |
| 2 | `src/backend/access/table/tableam.c` | `table_beginscan_catalog()` 如何绑定 `GetCatalogSnapshot()`。 |
| 3 | `src/backend/utils/cache/syscache.c` | `SearchSysCache*()`、`SysCacheInvalidate()`、`RelationHasSysCache()`、`RelationInvalidatesSnapshotsOnly()`。 |
| 4 | `src/backend/utils/cache/catcache.c` | `SearchCatCacheInternal()`、`SearchCatCacheMiss()`、`CatCacheInvalidate()`、`PrepareToInvalidateCacheTuple()`。 |
| 5 | `src/backend/utils/cache/inval.c` | `LocalExecuteInvalidationMessage()`、`CommandEndInvalidationMessages()`、`AtEOXact_Inval()`。 |
| 6 | `src/backend/access/transam/xact.c` | `CommandCounterIncrement()`、`AtCCI_LocalCache()`、commit/abort cleanup 顺序。 |
| 7 | `src/backend/catalog/namespace.c` | `RangeVarGetRelidExtended()` 的 name lookup、lock、invalidation retry。 |
| 8 | `src/backend/commands/tablecmds.c` | `ATExecAddColumn()` 等 DDL 如何更新 catalog 并显式 CCI。 |
| 9 | `src/backend/catalog/heap.c` | `AddNewAttributeTuples()`、`CatalogTupleUpdate()` 后的 catalog change。 |
| 10 | `src/backend/storage/ipc/sinvaladt.c` | shared invalidation ring、catchup、overflow reset。 |
| 11 | `src/include/storage/sinval.h` | `SharedInvalidationMessage` 类型和 payload。 |

最小阅读链：

```text
table_beginscan_catalog()
  -> RegisterSnapshot(GetCatalogSnapshot(relid))
  -> GetNonHistoricCatalogSnapshot(relid)
  -> maybe reuse CatalogSnapshot
```

syscache fast path：

```text
SearchSysCache1()
  -> SearchCatCache1()
  -> SearchCatCacheInternal()
     -> hit: refcount++ and return
     -> miss: systable_beginscan(... snapshot NULL ...)
```

DDL 失效路径：

```text
CatalogTupleUpdate()
  -> heap_update()
  -> CacheInvalidateHeapTuple()
  -> PrepareToInvalidateCacheTuple()
  -> CurrentCmdInvalidMsgs
  -> CommandCounterIncrement()
  -> CommandEndInvalidationMessages()
  -> LocalExecuteInvalidationMessage()
```

跨 backend 边界：

```text
CommitTransaction()
  -> AtEOXact_Inval(true)
  -> SendSharedInvalidMessages()
  -> ResourceOwnerRelease(... LOCKS ...)
```

不要把 `CatalogSnapshot` 单独读成 snapshot 课程。本节只问：syscache miss 或系统表扫描发生时，catalog scan 用哪个 MVCC 时间点。也不要把 `CatCacheInvalidate()` 单独读成 cache 课程。本节只问：invalidation 到达后，future syscache hit 为什么不会继续返回旧 entry。

## 4. 关键结构与状态
### 4.1 `CatalogSnapshot`
`CatalogSnapshot` 定义在 `snapmgr.c`，是 backend-local static pointer。相关状态如下。

| 状态 | 语义 |
| --- | --- |
| `CatalogSnapshotData` | 一个 `SNAPSHOT_MVCC` 类型的 static `SnapshotData`。 |
| `CatalogSnapshot` | 当前可复用 catalog snapshot；NULL 表示下次要重新获取。 |
| `RegisteredSnapshots` | pairing heap；catalog snapshot 会被手动加入其中参与 xmin 推进判断。 |
| `HistoricSnapshot` | logical decoding 活跃时 `GetCatalogSnapshot()` 返回 historic snapshot。 |

`GetNonHistoricCatalogSnapshot(relid)` 的关键规则：

```text
如果 CatalogSnapshot 为 NULL，调用 GetSnapshotData(&CatalogSnapshotData)。
如果 relid 既没有 syscache，也不会发送 snapshot-only invalidation，
则不能复用旧 CatalogSnapshot，必须先 InvalidateCatalogSnapshot()。
```

`CatalogSnapshot` 不是普通 `RegisterSnapshot()` 结果。源码把它手动加入 `RegisteredSnapshots`，因为它要影响 `PGPROC->xmin` / horizon，但不应该绑定某个 `ResourceOwner`。因此 `InvalidateCatalogSnapshot()` 必须做：

```text
pairingheap_remove(&RegisteredSnapshots, &CatalogSnapshot->ph_node)
CatalogSnapshot = NULL
SnapshotResetXmin()
```

这说明 catalog snapshot cleanup 不是普通 pfree，而是 visibility horizon 相关的 backend-local state transition。

### 4.2 catalog scan snapshot
`table_beginscan_catalog()` 在 `tableam.c` 中很短：

```text
snapshot = RegisterSnapshot(GetCatalogSnapshot(relid))
table_beginscan_common(... SO_TEMP_SNAPSHOT ...)
```

含义：

- catalog scan 使用 `GetCatalogSnapshot(relid)`。
- scan 自己注册一个临时 snapshot 引用。
- scan 结束后按 table scan 生命周期注销。
- `GetCatalogSnapshot()` 可能复用 static snapshot，也可能新建。

`systable_beginscan()` 传入 `snapshot = NULL` 时也走 catalog snapshot 语义。所以 syscache miss 不是裸读系统表，它仍然落在 catalog snapshot 的 visibility 模型里。

### 4.3 `CatCTup` 与 syscache hit
syscache 对外暴露：

```text
SearchSysCache1(RELOID, ObjectIdGetDatum(relid))
ReleaseSysCache(tuple)
```

真实 fast path 在 `SearchCatCacheInternal()`。本节关心这些字段。

| 字段 | 本节语义 |
| --- | --- |
| `ct->hash_value` | invalidation message 用同一 hash 定位候选 entry。 |
| `ct->dead` | entry 已过期，future lookup 必须跳过。 |
| `ct->negative` | 缓存“不存在”，insert 也必须 invalidation。 |
| `ct->refcount` | 当前 backend 内 active holder，保护内存不悬挂。 |
| `ct->tuple` | 返回给 caller 的 `HeapTupleData`。 |
| `ct->c_list` | partial-key list cache 的成员关系。 |

hit path 的核心顺序：

```text
skip dead entries
match hash
compare key
if positive:
  ResourceOwnerEnlarge(CurrentResourceOwner)
  ct->refcount++
  ResourceOwnerRememberCatCacheRef(...)
  return &ct->tuple
if negative:
  return NULL
```

这个 hit path 不重新取 catalog snapshot。它依赖 invalidation ordering 保证：如果旧 entry 已经越过语义失效边界，它会被标 dead 或移除，hit path 会跳过它。

### 4.4 pending invalidation groups
`inval.c` 把事务性消息分成两个时间组。

| group | 意义 |
| --- | --- |
| `CurrentCmdInvalidMsgs` | 当前 command 产生，尚未本地执行。 |
| `PriorCmdInvalidMsgs` | 之前 command 已本地执行，commit 时还要发给其它 backend。 |

同一事务内：

```text
CurrentCmdInvalidMsgs
  -> CommandEndInvalidationMessages()
  -> LocalExecuteInvalidationMessage()
  -> append to PriorCmdInvalidMsgs
```

事务 commit 时：

```text
PriorCmdInvalidMsgs + CurrentCmdInvalidMsgs
  -> SendSharedInvalidMessages()
```

事务 abort 时：

```text
PriorCmdInvalidMsgs
  -> LocalExecuteInvalidationMessage()
CurrentCmdInvalidMsgs
  -> discard
```

原因：prior messages 已经影响过本 backend 的 cache，abort 后必须清掉可能基于未提交 catalog 状态的本地 cache；current messages 尚未影响本地 cache，abort 时不需要通知任何 backend。

### 4.5 `SharedInvalidationMessage`
本节主要涉及这些类型。

| 类型 | payload | 本节含义 |
| --- | --- | --- |
| catcache | `id`、`dbId`、`hashValue` | 某个 syscache key hash 对应事实过期。 |
| whole catalog | `catId`、`dbId` | 某个 catalog 的所有 catcache entry 过期。 |
| relcache | `relId`、`dbId` | 某个 relation descriptor 过期。 |
| snapshot | `relId`、`dbId` | 没有 syscache 的 catalog scan snapshot 需要失效。 |

message 不携带新 tuple、旧 tuple、新 tuple descriptor、DDL 类型、`CatCTup *` 或 `RelationData *`。receiver 可能没有 entry，可能 entry refcount > 0，可能 entry 已经 dead，可能正在构造 entry，也可能只有 negative entry。shared message 只能传递最小过期事实。

### 4.6 name lookup retry 状态
`RangeVarGetRelidExtended()` 用 `SharedInvalidMessageCounter` 把名称解析和锁等待连起来。

```text
inval_count = SharedInvalidMessageCounter
lookup name -> relId
callback checks permissions or extra locks
LockRelationOid(relId, lockmode)
if SharedInvalidMessageCounter changed:
  retry lookup
if relId changed:
  unlock oldRelId and repeat
```

这个逻辑保护 name lookup race。`LockRelationOid()` 可能等待，等待期间另一个 DDL 可能 commit 并发送 invalidation。拿到锁后如果继续使用等待前 lookup 出来的 OID，就可能把 DML 绑定到已经重命名、替换或删除后的旧对象。

## 5. 主流程 walkthrough：DML 读取元数据
假设执行：

```sql
SELECT * FROM t WHERE id = 1;
```

简化元数据链路：

```text
parser/analyzer
  -> RangeVarGetRelidExtended("t", AccessShareLock, ...)
     -> namespace lookup through syscache
     -> LockRelationOid()
     -> maybe AcceptInvalidationMessages()
     -> retry if SharedInvalidMessageCounter changed
  -> relation_open/table_open
     -> RelationIdGetRelation()
     -> relcache hit or RelationBuildDesc()
  -> planner/executor helper lookups
     -> SearchSysCache*()
     -> lsyscache helpers
```

### 5.1 name lookup 不是单纯 syscache hit
`RangeVarGetRelidExtended()` 先解析 schema。显式 schema 走 `LookupExplicitNamespace()`，未显式 schema 走 search path 和 `RelnameGetRelid()`。这些路径可能命中 `pg_namespace` 或 `pg_class` syscache。

但函数不会把 syscache hit 结果当作永久稳定事实。它记录 `SharedInvalidMessageCounter`，随后拿 relation lock。如果锁等待期间处理了 invalidation，就重做 name lookup。

这把 syscache hit 放回 lock ordering 中：

```text
name lookup may hit syscache
  -> wait for lock
  -> process invalidation
  -> if namespace/name answer may have changed, lookup again
```

### 5.2 relation lock 的角色
`AccessShareLock` 不保护 catcache 内存，它保护对象级并发语义。普通 DML 的 `AccessShareLock` 与许多 DDL 的 `AccessExclusiveLock` 冲突。

因此 `ALTER TABLE ... ADD COLUMN` 持有 `AccessExclusiveLock` 时，普通 DML 会等待。等待结束后，DML 不能只继续使用等待前 metadata；它需要处理 invalidation。`LockRelationOid()` 路径会接收 pending invalidation，`RangeVarGetRelidExtended()` 再用 counter 判断是否 retry。

### 5.3 relcache build 仍然会扫 catalog
`RelationIdGetRelation()` 命中有效 relcache 时，DML 不需要扫 `pg_class` / `pg_attribute`。miss 或 invalid 时，`RelationBuildDesc()` 会扫系统表，通常通过 `systable_beginscan(... snapshot NULL ...)` 或 `RegisterSnapshot(GetNonHistoricCatalogSnapshot(RelationRelationId))` 进入 catalog snapshot 语义。

因此 relcache build 的一致性依赖三层：

```text
relation lock:
  避免构造期间对象定义被并发 DDL 改掉
catalog snapshot:
  决定本次系统表扫描看到哪个 committed/command 状态
invalidation:
  确保旧 relcache/syscache entry 不再作为 future lookup 来源
```

### 5.4 syscache hit 的安全边界
假设 planner 调用：

```text
get_rel_relkind(relid)
  -> SearchSysCache1(RELOID, ObjectIdGetDatum(relid))
```

如果 `CatCTup` 存在且不 dead，hit path 直接返回。它没有重新检查 tuple visibility。安全性来自：任何影响 `RELOID` cache key/hash 对应事实的 catalog change，都会在 command/commit boundary 触发 `SysCacheInvalidate(RELOID, hash)`。

所以 syscache hit 的前提是：

```text
本 backend 尚未越过需要处理该 invalidation 的安全边界。
```

这也是为什么“什么时候接收 invalidation”比“hit path 有没有 MVCC check”更重要。

### 5.5 catalog snapshot 的角色
syscache miss 时，catcache 打开 catalog relation：

```text
table_open(cache->cc_reloid, AccessShareLock)
systable_beginscan(relation, cache->cc_indexoid, IndexScanOK(cache), NULL, ...)
```

`snapshot = NULL` 让系统表扫描使用 catalog snapshot。同一 command 内复用 snapshot 是允许的，因为同一 command 的 catalog 修改不应立刻改变当前 command 的 catalog visibility。下一条 command 通过 `CommandCounterIncrement()` 推进；在该边界，pending invalidation 会调用 `InvalidateCatalogSnapshot()`、`SysCacheInvalidate()` 和 `RelationCacheInvalidateEntry()`。

因此新 command 的 miss path 会拿新 catalog snapshot，hit path 会避开 dead entry。这就是 catalog snapshot 与 syscache hit 的配合点。

## 6. 主流程 walkthrough：DDL 修改 catalog
以 `ALTER TABLE t ADD COLUMN c int` 为例，`tablecmds.c` 的 `ATExecAddColumn()` 展示了典型顺序。

简化链路：

```text
ATExecAddColumn()
  -> table_open(AttributeRelationId, RowExclusiveLock)
  -> SearchSysCacheCopy1(RELOID, relid)
  -> InsertPgAttributeTuples()
  -> CatalogTupleUpdate(pg_class, reltup)
  -> CommandCounterIncrement()
  -> maybe AddRelationNewConstraints()
  -> CommandCounterIncrement()
```

### 6.1 为什么 DDL 先更新 catalog
新增列至少要写 `pg_attribute` 中的新列和 `pg_class.relnatts`，可能还要写 `pg_attrdef`、`pg_constraint`、dependency 记录。`CatalogTupleUpdate()` / insert wrapper 最终走 heap AM，catalog heap 修改会调用：

```text
CacheInvalidateHeapTuple()
  -> CacheInvalidateHeapTupleCommon()
```

这一步不立刻清 syscache，而是登记 invalidation。原因是同一 command 内旧 tuple 仍可能按 command visibility 规则有效，且其它 backend 不能看到未提交变化。

### 6.2 `PrepareToInvalidateCacheTuple()`
catcache invalidation 信息由 `PrepareToInvalidateCacheTuple()` 计算。它遍历已经初始化的 catcache descriptor，对每个缓存同一 catalog relation 的 cache 计算 tuple hash 并登记 `cache id + hash + dbId`。

```text
compute old tuple hash
register cache id + hash + dbId
if update and new tuple hash differs:
  register new hash too
```

它不要求当前 backend 已经加载了对应 `CatCTup`。当前 backend 可能稍后在本 command 内加载；其它 backend 可能已经有 positive entry；也可能有 negative entry。insert/delete/update 都只表达“这个 key 的事实过期”。

这解释了为什么 insert 也要 invalidation：它要清 negative entry。

### 6.3 relcache invalidation
一些 catalog tuple 修改还会注册 relcache invalidation。

| catalog | 常见影响 |
| --- | --- |
| `pg_class` | relation 自身 fixed fields、relkind、relnatts、relfilenode 等。 |
| `pg_attribute` | 对应 `attrelid` 的 tuple descriptor。 |
| `pg_index` | index relation 或 heap relation index list。 |
| `pg_constraint` | constraint、trigger、partition 或 RI 相关 descriptor。 |

`ATExecAddColumn()` 更新 `pg_attribute` 和 `pg_class` 后，下一条命令必须刷新 relcache；否则同一事务内后续 `SELECT c FROM t` 仍可能看到旧 tuple descriptor。

### 6.4 `CommandCounterIncrement()` 的双重作用
`CommandCounterIncrement()` 先判断 `currentCommandIdUsed`。如果当前 command 确实写过 tuple 或登记过 invalidation，它会：

```text
currentCommandId++
SnapshotSetCommandId(currentCommandId)
AtCCI_LocalCache()
```

`AtCCI_LocalCache()` 做：

```text
AtCCI_RelationMap()
CommandEndInvalidationMessages()
```

`CommandEndInvalidationMessages()` 做：

```text
ProcessInvalidationMessages(CurrentCmdInvalidMsgs, LocalExecuteInvalidationMessage)
AppendInvalidationMessages(PriorCmdInvalidMsgs, CurrentCmdInvalidMsgs)
```

关键不是 `currentCommandId++` 单独生效，而是：

```text
command id 推进 MVCC command visibility；
local invalidation 清掉旧 cache 和旧 CatalogSnapshot；
两者在同一个 CCI 边界发生。
```

### 6.5 `LocalExecuteInvalidationMessage()`
本 backend 在 CCI 处理自己的 message 时，走和远端接收相同的 dispatcher。

catcache message：

```text
InvalidateCatalogSnapshot()
SysCacheInvalidate(cacheId, hashValue)
CallSyscacheCallbacks(cacheId, hashValue)
```

whole catalog message：

```text
InvalidateCatalogSnapshot()
CatalogCacheFlushCatalog(catId)
```

relcache message：

```text
RelationCacheInvalidateEntry(relId)
relcache callbacks
```

snapshot message：

```text
InvalidateCatalogSnapshot()
```

同一 backend 的 DDL 不需要等 commit 才看到新元数据，因为 CCI 已经推进 command visibility、丢弃旧 `CatalogSnapshot`、标记或删除旧 syscache entry，并标记或刷新 relcache entry。

### 6.6 commit 后再通知其它 backend
同一事务内的 DDL 可能 abort，所以其它 backend 不能在 command end 看到未提交 catalog change。commit path 中，`xact.c` 的顺序是：

```text
AtEOXact_RelationCache(true)
AtEOXact_TypeCache()
AtEOXact_Inval(true)
ResourceOwnerRelease(... RESOURCE_RELEASE_LOCKS ...)
```

`AtEOXact_Inval(true)` 会把 current messages 追加到 prior messages，再通过 `SendSharedInvalidMessages()` 发送。它发生在释放 heavyweight locks 之前。

因此等待 DDL lock 的 backend 被唤醒前，shared invalidation 已经进入队列。等待者接下来在 lock path 或 transaction start 接收消息，避免“DDL commit 已完成、lock 已释放、waiter 继续命中旧 cache”的窗口。

## 7. 生命周期 / ownership / cleanup
### 7.1 `CatalogSnapshot`
创建：

```text
GetNonHistoricCatalogSnapshot()
  -> GetSnapshotData(&CatalogSnapshotData)
```

持有者是当前 backend 的 static `CatalogSnapshot` 指针。它还会手动进入 `RegisteredSnapshots`，参与 xmin horizon 判断。

失效：

```text
InvalidateCatalogSnapshot()
  -> remove from RegisteredSnapshots
  -> CatalogSnapshot = NULL
  -> SnapshotResetXmin()
```

`table_beginscan_catalog()` 会对返回 snapshot 做 `RegisterSnapshot()`，这是 scan 层自己的临时引用；static `CatalogSnapshot` 的根生命周期仍由 `InvalidateCatalogSnapshot()` 控制。

### 7.2 syscache entry
创建：

```text
SearchCatCacheMiss()
  -> systable_beginscan()
  -> CatalogCacheCreateEntry()
  -> allocate in CacheMemoryContext
```

持有：

```text
SearchCatCacheInternal()
  -> ct->refcount++
  -> ResourceOwnerRememberCatCacheRef()
```

正常释放：

```text
ReleaseSysCache()
  -> ReleaseCatCache()
  -> refcount--
```

ERROR/abort 兜底：

```text
ResourceOwnerRelease(... AFTER_LOCKS ...)
  -> ResOwnerReleaseCatCache()
```

语义失效：

```text
CatCacheInvalidate()
  -> refcount == 0: CatCacheRemoveCTup()
  -> refcount > 0: ct->dead = true
```

所以 lifecycle 分两条线：

```text
memory lifetime:
  CacheMemoryContext + refcount + ResourceOwner
semantic lifetime:
  command/commit invalidation + dead flag
```

### 7.3 invalidation message
事务性 pending messages 放在 `TopTransactionContext`，属于当前 backend。subtransaction 通过 `TransInvalidationInfo.parent` 和 group index 形成栈。commit/abort 后 `transInvalInfo = NULL`，`TopTransactionContext` reset，消息不需要逐条释放。

shared queue 在 main shared memory。`sinvaladt.c` 中的 `SISeg` 持有 circular buffer、`minMsgNum` / `maxMsgNum`、per-backend `ProcState` 和 catchup signal 状态。backend 初始化时注册 SI slot，退出时 `CleanupInvalidationState()` 清 slot。

### 7.4 lock ownership
relation lock 由 lock manager 和 transaction resource owner 管。DDL 常见模式：

```text
RangeVarGetRelidExtended(... AccessExclusiveLock ...)
relation_open(... NoLock or already locked ...)
catalog updates
commit
AtEOXact_Inval(true)
release locks
```

DML 常见模式：

```text
RangeVarGetRelidExtended(... AccessShareLock ...)
relation_open()
execute with Relation *
ResourceOwnerRelease closes refs
release locks at xact end
```

locks 不释放 syscache entry，syscache ref 不释放 lock。它们在 commit/abort cleanup 中有相对顺序要求，但 ownership 不相同。

## 8. 正确性机制层次
### 8.1 分层表
| 机制 | 保证 | 不保证 |
| --- | --- | --- |
| MVCC snapshot | catalog scan 看到哪个 tuple version。 | syscache hit 自动重查 visibility。 |
| `CatalogSnapshot` reuse | catalog scan 避免每次取新 snapshot。 | catalog 永远最新。 |
| command counter | 同一事务下一条 command 能看到上一条 command 的 catalog 写入。 | 其它 backend 看到未提交变化。 |
| local invalidation | 本 backend CCI 后旧 cache/snapshot 不再服务 future lookup。 | 阻塞并发 DDL。 |
| shared invalidation | commit 后其它 backend 得知过期事实。 | 立即重建所有 cache。 |
| refcount / ResourceOwner | 本地指针不悬挂，ERROR 后归还引用。 | 语义仍然最新。 |
| heavyweight lock | DDL/DML 对同一 relation 的对象级互斥。 | cache entry 内存安全。 |
| retry loop | name lookup 和 lock wait 后验证答案。 | `NoLock` caller 的正确性。 |

### 8.2 为什么不是单一机制
只靠 MVCC 不够，因为 syscache hit 不访问 heap tuple visibility。只靠 syscache invalidation 不够，因为 miss path 仍要决定 catalog scan 用哪个 snapshot。只靠 command counter 不够，因为其它 backend 需要 commit 后通知。只靠 shared invalidation 不够，因为 DDL/DML 对象级互斥还要靠 relation lock。只靠 relation lock 不够，因为等待者被唤醒后仍可能持有旧 syscache、relcache 或 plan cache。

PostgreSQL 把每个机制限制在自己的边界，以换取 hot path 低成本；代价是 ordering 必须严格。

### 8.3 command boundary
在本节语境中，command boundary 的核心函数是 `CommandCounterIncrement()`。它只有在 `currentCommandIdUsed` 为 true 时才真正推进。`RegisterRelcacheInvalidation()` 等路径会调用 `GetCurrentCommandId(true)`，确保下一次 CCI 不被当作 read-only no-op。

否则可能出现：

```text
catalog invalidation 已登记；
currentCommandIdUsed 仍 false；
CommandCounterIncrement() 不处理 AtCCI_LocalCache()。
```

这把 command counter 优化和 invalidation ordering 连接起来。

### 8.4 commit boundary
`AtEOXact_Inval(true)` 必须在释放 locks 前。如果先释放 lock，等待者可能继续执行；如果 shared invalidation 还没发出，等待者可能继续命中旧 cache。因此 commit 顺序是 correctness boundary，不是性能微调。

### 8.5 snapshot invalidation 与 syscache invalidation
`LocalExecuteInvalidationMessage()` 对 catcache message 会先 `InvalidateCatalogSnapshot()`，再 `SysCacheInvalidate()`。否则即使旧 syscache entry 被删了，miss path 也可能用旧 snapshot 重新扫描出旧 tuple version。

这就是本节标题中三者的组合：

```text
cache invalidation 必须同时让 fast path 和 scan path 失效。
```

## 9. 错误路径 / 异常路径 / fallback
### 9.1 abort
场景：

```sql
BEGIN;
CREATE TABLE abort_demo(a int);
SELECT 'abort_demo'::regclass;
ROLLBACK;
```

`CREATE TABLE` 的 CCI 已让本 backend cache 看到 `abort_demo`。ROLLBACK 时不能只丢弃 pending messages，而要：

```text
AtEOXact_Inval(false)
  -> ProcessInvalidationMessages(PriorCmdInvalidMsgs, LocalExecuteInvalidationMessage)
```

它不发送给其它 backend，因为其它 backend 从未看到未提交 catalog change；它只清当前 backend 的本地 cache。

### 9.2 subtransaction abort
子事务可能经历 command boundary。子事务 abort 时：

```text
child PriorCmdInvalidMsgs
  -> LocalExecuteInvalidationMessage()
child CurrentCmdInvalidMsgs
  -> discard
```

已经影响本地 cache 的未提交状态必须清；尚未本地执行的消息不需要处理。

### 9.3 queue overflow
shared invalidation ring 可能 overflow。如果 receiver 落后太多，`SIGetDataEntries()` 返回 reset 信号，上层调用：

```text
InvalidateSystemCaches()
  -> InvalidateCatalogSnapshot()
  -> ResetCatalogCachesExt(false)
  -> RelationCacheInvalidate(false)
  -> callbacks
```

这牺牲精确性但保住 correctness。一旦精确消息丢失，系统不能猜哪些 cache 安全。

### 9.4 relation without syscache
如果要扫的 catalog relation 没有 syscache，也不在 snapshot-only invalidation 列表，`GetNonHistoricCatalogSnapshot(relid)` 不能复用旧 `CatalogSnapshot`，必须刷新。

原因是没有 syscache invalidation message 能顺手调用 `InvalidateCatalogSnapshot()`，也没有 snapshot-only message 通知它。`syscache.c` 中的 `RelationInvalidatesSnapshotsOnly()` 正是为少数没有 syscache但需要 snapshot invalidation 的 catalog 提供边界。

### 9.5 `SearchSysCacheLocked1()` 与 inplace update
`SearchSysCacheLocked1()` 用于 inplace-updated catalog tuple 的锁定读取。核心 loop：

```text
SearchSysCache1()
remember tuple TID
ReleaseSysCache()
LockTuple(InplaceUpdateTupleLock)
AcceptInvalidationMessages()
SearchSysCache1()
confirm TID did not change
```

这说明有些 catalog 更新路径不能只依赖普通 transaction-end invalidation。inplace update 可能刚完成，拿 tuple lock 后必须处理 syscache invalidation，确保返回 content 是锁之后的。

### 9.6 entry 构造中被 invalidated
`SearchCatCacheMiss()` 注释提到，`CatalogCacheCreateEntry()` 可能在 detoast 期间触发 `AcceptInvalidationMessages()`。如果正在构造的 entry 被 invalidation 命中，create 返回 NULL，miss path 重新扫描。否则可能创建一个出生即 stale 的 entry。

### 9.7 lock wait 后 negative cache
`RangeVarGetRelidExtended()` 中，即使 lookup 结果是 `InvalidOid`，如果请求了 lock，也会调用 `AcceptInvalidationMessages()`。原因是可能有 lingering negative catcache entry，而并发 `CREATE` 已经 commit。对象不存在这个事实也会过期。

### 9.8 historic snapshot
logical decoding 活跃时：

```text
GetCatalogSnapshot()
  -> HistoricSnapshot
```

decoding 需要按历史时间解释 catalog。普通 backend 不走这条默认路径，但要记住 historic snapshot 是 catalog snapshot 语义的显式例外；decoding 结束后需要 reset system caches，避免 backend-local cache 混入 historic catalog 解释。

## 10. 成本、资源与跨模块传播
### 10.1 hot path 与 snapshot 成本
syscache hit 的主要成本是 hash 计算、bucket dlist scan、key equality、move-to-head、`ResourceOwnerEnlarge()` 快路径、`refcount++` 和 owner remember。它不获取 shared lock，不取新 snapshot，不扫 catalog。这就是 PostgreSQL 愿意把复杂度放到 invalidation ordering 的原因。

`GetSnapshotData()` 不是免费操作。虽然本节不是 ProcArray 课，但频繁重新获取 snapshot 会触碰事务可见性全局状态。`CatalogSnapshot` 复用减少 catalog scan 的 snapshot 获取成本，代价是必须精确知道何时不能复用。

### 10.2 invalidation fan-out
DDL 修改一个 catalog tuple，成本可能扩散到当前 backend 本地 cache flush、commit 时 shared SI queue 写入、每个 receiver 的 catcache/relcache cleanup、relcache callbacks、plan cache invalidation、typcache 或其它 callback cache。

| 变量 | 放大方式 |
| --- | --- |
| backend 数 | 每个 backend 都有自己的 cache，需要各自接收或 reset。 |
| DDL 频率 | 更多 invalidation message 和 cache rebuild。 |
| catalog row 数 | 大 DDL 或 extension install 会产生大量 catalog changes。 |
| prepared statement 数 | relcache/syscache callback 扫描 saved plans。 |
| 分区数 | relcache/partition descriptor rebuild 成本高。 |
| long transaction | snapshot horizon 和 cache 状态滞留更久。 |

### 10.3 namespace、relcache、plancache
名称解析常命中 `pg_namespace`、`pg_class` syscache。对象很多时单个 lookup 仍是 hash，但 DDL churn 会提高 miss/rebuild 频率。search path 多 schema、临时 schema、频繁 create/drop 会让 retry 和 negative cache invalidation 更常见。

relcache 构造依赖 syscache 和 catalog snapshot；relcache invalidation 又通过 callbacks 影响 plan cache。边界是：

```text
syscache 负责单个 catalog tuple；
relcache 负责组合后的 RelationData；
CatalogSnapshot 负责 scan visibility；
invalidation 负责把它们同时推过 command/commit 边界。
```

plan cache 不直接收到 shared message payload，而是通过 `CacheRegisterRelcacheCallback()` 和 `CacheRegisterSyscacheCallback()` 被本地通知。DDL 后第一次 `EXECUTE` 变慢或报 result type 变化，通常来自本地 revalidation / replan / descriptor check。

### 10.4 horizon、WAL 与后台进程
`CatalogSnapshot` 会进入 `RegisteredSnapshots`，可能参与 xmin horizon。`InvalidateCatalogSnapshotConditionally()` 在等待 client input 前，如果 catalog snapshot 是唯一 registered snapshot，会丢弃它，避免空闲 backend 阻碍 horizon 推进。

catalog changes 自身由 WAL 保证 crash safety。invalidation messages 可以作为 commit side effect 进入 commit record；redo/recovery 通过 committed invalidation 推进 standby/backend cache。cache entry 是 backend-local memory，WAL 不记录 cache 内容。

没有后台进程替所有 backend 清 syscache。参与状态推进的是发起 DDL 的 backend、接收 invalidation 的 backend、startup/recovery 进程在 redo/standby 场景处理 committed invalidation，以及作为普通 backend 访问 catalog 的 autovacuum 等 worker。checkpointer、bgwriter 不推进 catalog snapshot/syscache freshness。

## 11. 观测与诊断入口
### 11.1 SQL 能看到什么
同一 backend CCI 现象：

```sql
DROP TABLE IF EXISTS cs_cci_demo;
CREATE TABLE cs_cci_demo(a int);
BEGIN;
ALTER TABLE cs_cci_demo ADD COLUMN b int;
SELECT b FROM cs_cci_demo;
ROLLBACK;
```

两会话 commit 后传播：

```sql
PREPARE p AS SELECT * FROM cs_plan_demo;
EXECUTE p;
```

另一个会话执行 DDL 后，第一次再执行可能触发 plan invalidation 或 result type 错误。

`pg_locks` 可以看对象锁：

```sql
SELECT locktype, relation::regclass, mode, granted
FROM pg_locks
WHERE relation = 'cs_plan_demo'::regclass;
```

它只能看到 lock ordering，看不到 syscache entry 是否 dead。

### 11.2 SQL 看不到什么
`pg_backend_memory_contexts` 可以看当前 backend cache memory：

```sql
SELECT name, total_bytes, used_bytes
FROM pg_backend_memory_contexts
WHERE name = 'CacheMemoryContext';
```

它不能告诉你哪个 `CatalogSnapshot` 正在复用、哪个 `CatCTup` 是 dead、哪个 syscache hit 发生过、`PriorCmdInvalidMsgs` 中有多少 message、shared SI ring 中某 backend 落后多少。

### 11.3 gdb 断点
建议断点：

```gdb
break GetCatalogSnapshot
break GetNonHistoricCatalogSnapshot
break InvalidateCatalogSnapshot
break table_beginscan_catalog
break SearchCatCacheInternal
break SearchCatCacheMiss
break CatCacheInvalidate
break PrepareToInvalidateCacheTuple
break CommandCounterIncrement
break CommandEndInvalidationMessages
break LocalExecuteInvalidationMessage
break AtEOXact_Inval
break AcceptInvalidationMessages
break RangeVarGetRelidExtended
```

关键观察：

```gdb
p CatalogSnapshot
p currentCommandId
p currentCommandIdUsed
p SharedInvalidMessageCounter
p ct->dead
p ct->refcount
p msg->id
```

`CatalogSnapshot` 是 `snapmgr.c` file-local static 变量，直接访问可能需要在对应调试上下文或通过符号定位。

### 11.4 perf / flamegraph
如果怀疑元数据机制带来 CPU 抖动，关注：

- `SearchCatCacheInternal`
- `SearchCatCacheMiss`
- `CatalogCacheCreateEntry`
- `GetSnapshotData`
- `CommandEndInvalidationMessages`
- `LocalExecuteInvalidationMessage`
- `RelationBuildDesc`
- `PlanCacheRelCallback`
- `RangeVarGetRelidExtended`

大量 `SearchCatCacheMiss` 可能表示 cache 被频繁 invalidated、catalog object churn 高、`debug_discard_caches` 打开，或连接频繁新建导致 cache 尚未 warm。大量 `GetSnapshotData` 可能表示 catalog snapshot 复用被打断，或扫描没有 syscache/snapshot invalidation 覆盖的 catalog。

### 11.5 日志与错误
有代表性的错误：

- `cache lookup failed for relation ...`
- `cache lookup failed for attribute ...`
- `cached plan must not change result type`
- `relation ... does not exist`
- `tuple concurrently updated`
- relcache init file warning。

诊断时不要直接归因。先问：是 catalog snapshot 太旧，syscache entry 没 invalidated，relcache/plan cache 没处理 callback，没有持正确 relation lock，name lookup 在 lock wait 后没 retry，还是 historic snapshot / logical decoding 特殊路径？

### 11.6 runtime truth
本节锚定的 runtime truth：

```text
DDL 的 catalog tuple 修改不会同步改掉所有 backend 的 cache；
一致性来自 command/commit boundary 上同时推进 CatalogSnapshot 失效、
syscache/relcache invalidation 和 lock wait retry。
```

能直接看到的是 CCI 后 schema 可见、commit 后 plan invalidation、lock wait 和 perf 中 cache miss/rebuild 抖动。不能直接看到的是某条 shared invalidation 是否已被某 backend 消费、某个 `CatCTup.dead` 翻转时间、`CatalogSnapshot` 具体复用次数或某个 relcache entry 的 `rd_isvalid`。

## 12. 常见误区
误区一：catalog snapshot 总是最新。实际 `CatalogSnapshot` 的价值就是复用；“足够新”不是“每次都新”。

误区二：syscache hit 绕过一致性。它绕过的是 catalog scan 成本；一致性前移到 entry 创建时的 snapshot 和 future lookup 前的 invalidation processing。

误区三：`CommandCounterIncrement()` 只影响 heap tuple visibility。catalog 修改路径还依赖它触发 `AtCCI_LocalCache()` 和 `CommandEndInvalidationMessages()`。

误区四：invalidation 是 lock。它不阻塞 reader，也不等待 `CatCTup.refcount` 归零；对象级互斥靠 heavyweight lock。

误区五：refcount 表示 metadata 新鲜。`refcount` 只保护内存；`dead` 和 invalidation 才表达 future lookup 不能继续使用。

误区六：DDL commit 后其它 backend 已重建所有 cache。commit 只是发送 invalidation；receiver 在安全点接收，通常只标 invalid，下一次访问才 rebuild。

误区七：没有 syscache 的 catalog 可以一直复用 `CatalogSnapshot`。如果没有 syscache invalidation 也没有 snapshot-only invalidation，`GetNonHistoricCatalogSnapshot()` 会强制刷新。

误区八：`pg_backend_memory_contexts` 能证明 syscache 一致性。它只能显示 context memory，看不到 snapshot、dead flag、pending invalidation、shared queue cursor 或 lock retry。

## 13. 课堂实验
### 实验 1：同一事务内 CCI 后看到新列
目标：把 SQL 现象和 `CommandCounterIncrement()` / local invalidation 对上。

SQL：

```sql
DROP TABLE IF EXISTS cs_cci_demo;
CREATE TABLE cs_cci_demo(a int);
BEGIN;
ALTER TABLE cs_cci_demo ADD COLUMN b int;
SELECT b FROM cs_cci_demo;
ROLLBACK;
```

断点：

```gdb
break ATExecAddColumn
break CatalogTupleUpdate
break CommandCounterIncrement
break CommandEndInvalidationMessages
break LocalExecuteInvalidationMessage
break InvalidateCatalogSnapshot
break CatCacheInvalidate
```

观察：

```text
ATExecAddColumn() 更新 pg_attribute / pg_class。
CommandCounterIncrement() 推进 command id。
LocalExecuteInvalidationMessage() 清 CatalogSnapshot 和 syscache/relcache。
SELECT b 的 relcache build 不再使用旧 tuple descriptor。
```

思考：如果没有 CCI，`SELECT b` 会在哪一层拿到旧元数据？

### 实验 2：两会话观察 commit 后传播
Session A：

```sql
DROP TABLE IF EXISTS cs_plan_demo;
CREATE TABLE cs_plan_demo(a int);
PREPARE p AS SELECT * FROM cs_plan_demo;
EXECUTE p;
```

Session B：

```sql
ALTER TABLE cs_plan_demo ADD COLUMN b int;
COMMIT;
```

Session A：

```sql
EXECUTE p;
```

断点：

```gdb
break AtEOXact_Inval
break SendSharedInvalidMessages
break AcceptInvalidationMessages
break LocalExecuteInvalidationMessage
break RelationCacheInvalidateEntry
break PlanCacheRelCallback
```

观察：Session B commit 前发送 shared invalidation；Session A 下次安全点接收；plan cache 被 relcache callback 标 invalid；下一次 `EXECUTE` 重新验证，结果取决于语句和协议是否允许 result type 改变。

### 实验 3：name lookup + lock wait retry
目标：观察 `RangeVarGetRelidExtended()` 的 retry。

步骤：

```text
Session B 持有 ALTER TABLE 或 LOCK TABLE 的 AccessExclusiveLock。
Session A 执行 SELECT * FROM t，卡在 AccessShareLock。
Session B 在同一事务中 RENAME 或 DROP/CREATE 后 COMMIT。
Session A 被唤醒。
```

断点：

```gdb
break RangeVarGetRelidExtended
break LockRelationOid
break AcceptInvalidationMessages
```

观察：

```text
inval_count 记录 lookup 前 counter。
LockRelationOid() 等待期间可能处理 invalidation。
SharedInvalidMessageCounter 改变后 repeat lookup。
如果 relId 变化，旧锁释放，新 OID 重新锁。
```

## 14. 讨论题
1. 为什么 `SearchCatCacheInternal()` hit path 不重新做 MVCC visibility check，仍然能保持 metadata 一致？
2. `CatalogSnapshot` 为什么要手动进入 `RegisteredSnapshots`，而不是完全交给 `ResourceOwner`？
3. 为什么 catcache invalidation message 到达时要先 `InvalidateCatalogSnapshot()`，再 `SysCacheInvalidate()`？
4. `CommandCounterIncrement()` 中 `currentCommandIdUsed` 优化为什么需要 invalidation 路径调用 `GetCurrentCommandId(true)` 配合？
5. 如果 `AtEOXact_Inval(true)` 放在释放 locks 之后，会破坏哪个 DDL/DML ordering？
6. `RangeVarGetRelidExtended()` 为什么要在 lock wait 后根据 `SharedInvalidMessageCounter` retry name lookup？
7. 一个 backend 持有 `CatCTup.refcount = 1` 且该 entry 已 `dead = true`，当前 holder 和 future lookup 分别能做什么？
8. 为什么 insert catalog tuple 也必须发 invalidation？
9. 没有 syscache 的 catalog relation 为什么会影响 `GetNonHistoricCatalogSnapshot()` 是否复用旧 snapshot？
10. 线上看到 DDL 后第一次查询变慢，你会如何区分 lock wait、SI receive、relcache rebuild、plan revalidation 和 catalog snapshot 获取成本？

## 15. 本节小结
本节主链路：

```text
DML metadata lookup:
  RangeVarGetRelidExtended()
  -> syscache/namespace lookup
  -> relation lock
  -> invalidation counter retry
  -> relcache/syscache hit or catalog scan

DDL catalog change:
  CatalogTupleInsert/Update/Delete
  -> CacheInvalidateHeapTupleCommon()
  -> CurrentCmdInvalidMsgs
  -> CommandCounterIncrement()
  -> LocalExecuteInvalidationMessage()
  -> InvalidateCatalogSnapshot()
  -> SysCacheInvalidate() / RelationCacheInvalidateEntry()
  -> AtEOXact_Inval(true)
  -> SendSharedInvalidMessages()
```

核心状态和边界：

- `CatalogSnapshot` 是 backend-local 可复用 MVCC snapshot，失效时从 `RegisteredSnapshots` 移除。
- syscache/catcache entry 是 backend-local cache object，hit path 靠 `dead` flag 避开过期 entry。
- pending invalidation 是当前 transaction state，shared invalidation ring 只传播最小过期事实。
- command boundary 让本 backend 的下一条命令看到 catalog change。
- commit boundary 让其它 backend 在锁释放前收到可接收的 invalidation。
- relation/object lock 排序 DDL/DML 的对象级并发。

ownership / cleanup：

```text
CatalogSnapshot:
  GetSnapshotData creates
  InvalidateCatalogSnapshot clears

CatCTup:
  CacheMemoryContext owns memory
  refcount + ResourceOwner owns active use
  CatCacheInvalidate owns semantic invalidation

Invalidation messages:
  TopTransactionContext owns pending groups
  SISeg owns shared delivery
  receiver local cache owns cleanup/rebuild
```

错误路径：

- abort 只清本 backend 已受影响的 prior invalidations，不通知其它 backend。
- subtransaction abort 同理清本地 prior state。
- shared queue overflow 退回 `InvalidateSystemCaches()`。
- inplace update 需要 `SearchSysCacheLocked1()` 中锁后 `AcceptInvalidationMessages()`。
- catcache entry 构造中被 invalidated 要 retry scan。
- 没有 syscache/snapshot invalidation 覆盖的 catalog 不能复用旧 `CatalogSnapshot`。

观测边界：

```text
SQL 能看到 CCI 后 schema 可见、commit 后 plan invalidation、lock wait；
pg_locks 能看到对象锁；
pg_backend_memory_contexts 只能看到 CacheMemoryContext 大小；
gdb/trace 才能看到 CatalogSnapshot、CatCTup.dead、pending invalidation group 和 retry counter。
```

可迁移 mental model：

```text
当一个高频本地 cache 代表可被事务修改的全局元数据时，
不要在 hit path 重做所有正确性检查；
应把 visibility、local cache lifetime、semantic invalidation 和 object lock
拆成独立机制，
再用 command/commit boundary 把它们排成同一条时间线。
```

PostgreSQL 同时需要 catalog snapshot、syscache hit 和 shared invalidation，因为三者分别负责：

```text
scan 看到什么，
hit 如何便宜，
何时不能再相信旧答案。
```

判断具体问题时仍要标注边界：workload 是否频繁 DDL，backend 数和 prepared statement 数是否放大 invalidation fan-out，是否有 long transaction 或 historic snapshot，是否打开 debug discard cache，具体版本中 syscache、relcache、plan cache callback 是否发生实现变化。

下一步如果继续追 plan cache，应带着本节不变量看：

```text
plan cache 不是直接接收新 schema；
它只在 relcache/syscache callback 中知道依赖事实过期，
然后在下一次使用时重新验证。
```
