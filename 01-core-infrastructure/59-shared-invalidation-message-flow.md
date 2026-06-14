# PostgreSQL shared invalidation message flow
## 课程定位
前置知识：已经理解 PostgreSQL 多进程模型、`MemoryContext`、`ResourceOwner`、catcache/syscache、relcache、heavyweight lock 和基本事务边界。
本节唯一主问题：
```text
catalog 更新如何产生 invalidation message，
backend 如何在 command boundary 消费并清理本地 cache？
```
核心矛盾：
```text
catalog cache 必须是 backend-local，才能低成本命中；
但 catalog 更新是全局事实，必须让其它 backend 的本地 cache 及时失效。
```
PostgreSQL 的选择不是共享一个全局 catalog cache。
它选择：
```text
本地 cache 保存对象和指针；
shared invalidation 只传播“哪些语义过期”；
每个 backend 在安全边界本地清理或重建。
```
学完后应能判断：
- 为什么 `heap_update()` 不能立刻删除 syscache entry。
- 为什么 `CommandCounterIncrement()` 是本 backend catalog 可见性的关键边界。
- 为什么 commit 后才向其它 backend 发送事务性 invalidation。
- 为什么 shared invalidation message 不携带新 tuple、新 schema 或本地指针。
- 为什么队列 overflow 时 fallback 是 reset 全部 invalidatable cache。
- 为什么 plan cache 通过 callback 被间接 invalidated。
- 哪些行为可以通过 SQL、日志、gdb、源码断点观察，哪些只能推断。
本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e9`。
## 1. 本节在总主线中的位置
前面两节已经分别看过 syscache/catcache 和 relcache。
那两节回答的是：
```text
一个 backend 如何把 catalog tuple 复制成本地 cache entry？
一个 backend 如何把多个 catalog row 拼成 RelationData？
```
本节把视角换成跨 backend 的时间线。
一个 backend 执行 DDL 或 catalog update。
它会修改系统表。
另一个 backend 可能已经缓存了旧 `CatCTup`、旧 `RelationData` 或旧 generic plan。
这两个 backend 之间不能传 C 指针。
也不能把所有 cache entry 放进 shared memory。
所以 PostgreSQL 只在 shared memory 中维护一条 invalidation message ring。
消息的语义很窄：
```text
这个 catcache hash 可能过期。
这个 catalog 的所有 catcache entry 可能过期。
这个 relcache relid 可能过期。
这个 smgr 或 relmap 状态可能过期。
```
消息的执行很本地：
```text
LocalExecuteInvalidationMessage()
  -> SysCacheInvalidate()
  -> RelationCacheInvalidateEntry()
  -> callbacks
  -> 当前 backend 自己删除、标 dead、重建或 reset cache
```
这就是本节主线。
后续再看 plan cache、logical decoding、relmap、smgr 时，都要回到这条边界：
```text
shared message 只传播过期事实；
实际 cleanup 发生在 receiver 的本地状态机里。
```
## 2. 核心矛盾与一句话运行模型
一句话运行模型：
```text
catalog heap insert/update/delete 调用 CacheInvalidateHeapTuple() 排队消息；
CommandCounterIncrement() 后本 backend 先执行当前命令的消息；
commit 记录完成后 AtEOXact_Inval(true) 把消息送入 shared sinval ring；
其它 backend 在事务开始、cache 敏感路径或 catchup interrupt 中接收消息；
receiver 按消息类型清理本地 catcache、relcache、plancache 等派生 cache。
```
这里的 tension 有两层。
第一层是 MVCC/command visibility tension：
```text
同一 command 内，旧 catalog tuple 仍可能按当前命令可见性规则有效；
下一个 command 才应该看到刚才 catalog 修改后的新状态。
```
所以不能在 `heap_update()` 里直接冲掉本地 syscache。
`inval.c` 文件头把这个约束讲得很清楚。
系统表通常用 current snapshot 类语义扫描。
一个 command 修改 catalog tuple 后，旧版本到 command boundary 才真正停止对本 backend 的后续 catalog lookup 有效。
第二层是 transaction visibility tension：
```text
其它 backend 不能在修改事务 commit 前相信新 catalog tuple；
但 commit 后等待锁的 backend 又必须在继续执行前知道旧 cache 已过期。
```
所以发送给其它 backend 的事务性消息必须发生在 commit 事实建立之后。
但又必须早于释放会唤醒等待者的 relation locks。
源码在 `CommitTransaction()` 中把这个顺序写死：
```text
AtEOXact_RelationCache(true)
AtEOXact_TypeCache()
AtEOXact_Inval(true)
ResourceOwnerRelease(... RESOURCE_RELEASE_LOCKS ...)
```
这是 correctness ordering。
## 3. 核心文件分工与阅读顺序
| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/utils/cache/inval.c` | pending message 分组、command boundary、本地执行、commit/abort/subxact 处理、callback registry。 |
| 2 | `src/include/storage/sinval.h` | `SharedInvalidationMessage` union，各类 message 的最小 payload。 |
| 3 | `src/backend/storage/ipc/sinvaladt.c` | shared sinval ring、`SISeg`、`ProcState`、发送、接收、overflow reset、catchup interrupt。 |
| 4 | `src/backend/access/transam/xact.c` | `CommandCounterIncrement()`、`AtCCI_LocalCache()`、commit/abort/subtransaction cleanup ordering。 |
| 5 | `src/backend/access/heap/heapam.c` | `heap_insert`/`heap_update`/`heap_delete` 对 catalog tuple 调用 invalidation 的入口。 |
| 6 | `src/backend/catalog/indexing.c` | `CatalogTupleInsert()`、`CatalogTupleUpdate()`、`CatalogTupleDelete()` 的 catalog 修改包装层。 |
| 7 | `src/backend/utils/cache/catcache.c` | `PrepareToInvalidateCacheTuple()`、`CatCacheInvalidate()`、`CatalogCacheFlushCatalog()`。 |
| 8 | `src/backend/utils/cache/syscache.c` | `SysCacheInvalidate()` 如何转发到 catcache。 |
| 9 | `src/backend/utils/cache/relcache.c` | `RelationCacheInvalidateEntry()`、`RelationCacheInvalidate()`、init file invalidation、EOX cleanup。 |
| 10 | `src/backend/utils/cache/plancache.c` | relcache/syscache callback 如何让 cached plan 失效。 |
| 11 | `src/include/utils/inval.h` | 本节 public API 边界。 |
推荐阅读顺序不是从 `sinvaladt.c` 顶部开始。
更有效的顺序是：
```text
inval.c 文件头
  -> CacheInvalidateHeapTupleCommon()
  -> PrepareToInvalidateCacheTuple()
  -> CommandEndInvalidationMessages()
  -> AtEOXact_Inval()
  -> SendSharedInvalidMessages()
  -> SIInsertDataEntries()
  -> SIGetDataEntries()
  -> LocalExecuteInvalidationMessage()
  -> CatCacheInvalidate() / RelationCacheInvalidateEntry()
  -> plancache callbacks
```
这条顺序按时间推进。
它能避免一个常见误读：
```text
shared invalidation 不是一个后台线程不断清 cache；
它是每个 backend 在事务边界和安全点主动接收并本地执行。
```
## 4. 关键数据结构与状态
### 4.1 `SharedInvalidationMessage`
`SharedInvalidationMessage` 定义在 `src/include/storage/sinval.h`。
它是一个 union。
第一个 `int8 id` 同时是类型 tag。
`id >= 0` 表示 catcache message。
负数表示其它类型。
当前版本主要类型如下。
| 类型 | message | payload | 语义 |
| --- | --- | --- | --- |
| catcache tuple | `SharedInvalCatcacheMsg` | `id`、`dbId`、`hashValue` | 某个 syscache/catcache hash 可能过期。 |
| whole catalog | `SharedInvalCatalogMsg` | `dbId`、`catId` | 某个 catalog 的所有 catcache entry 可能过期。 |
| relcache | `SharedInvalRelcacheMsg` | `dbId`、`relId` | 某个 relation 的 relcache entry 或 whole relcache 可能过期。 |
| smgr | `SharedInvalSmgrMsg` | relfile locator + backend bits | 某个物理 relation 的 smgr handle 需要关闭。 |
| relmap | `SharedInvalRelmapMsg` | `dbId` | relation map 文件需要重读。 |
| catalog snapshot | `SharedInvalSnapshotMsg` | `dbId`、`relId` | catalog snapshot 可能不再适合扫描该 catalog。 |
| relsync | `SharedInvalRelSyncMsg` | `dbId`、`relid` | logical decoding output plugin 的 relation sync cache 失效。 |
这个结构体故意很小。
它不包含：
- catalog tuple 的新值。
- catalog tuple 的旧值。
- DDL 类型。
- SQL command text。
- `RelationData *`。
- `CatCTup *`。
- plan cache entry 指针。
原因很直接。
所有这些对象都是 backend-local。
其它 backend 不能安全解引用。
并且 receiver 的本地状态不同：
```text
可能没有这个 cache entry；
可能有 entry 但 refcount > 0；
可能正在构造 entry；
可能已经被更早消息清过；
可能只需要标 invalid，不能立即 pfree。
```
message 的稳定语义只能是“某个 key 可能过期”。
### 4.2 `TransInvalidationInfo`
事务性 invalidation 的 pending state 在 `inval.c`。
核心是三层结构：
```text
InvalMessageArray
  -> SharedInvalidationMessage 数组，放在 TopTransactionContext
InvalidationMsgsGroup
  -> 每个 subgroup 的 firstmsg / nextmsg 下标
TransInvalidationInfo
  -> CurrentCmdInvalidMsgs
  -> PriorCmdInvalidMsgs
  -> parent
  -> my_level
```
它把消息分成两个时间组。
第一组是当前 command 刚产生、还没有本地执行的消息：
```text
CurrentCmdInvalidMsgs
```
第二组是前面 commands 已经在本 backend 执行过，但事务还没结束的消息：
```text
PriorCmdInvalidMsgs
```
这个分组回答两个问题。
对本 backend：
```text
当前 command 结束后，需要执行 CurrentCmdInvalidMsgs，
让下一条 command 的 syscache/relcache lookup 看到新状态。
```
对其它 backend：
```text
事务 commit 后，需要广播 PriorCmdInvalidMsgs + CurrentCmdInvalidMsgs，
让其它 backend 清理旧 cache。
```
对 abort：
```text
PriorCmdInvalidMsgs 已经影响过本 backend 的 cache，
abort 时必须本地执行一遍反向意义上的 invalidation，
以清掉本事务中加载的、现在不再有效的 cache state。
CurrentCmdInvalidMsgs 尚未影响本地 cache，
abort 时可以直接忘掉。
```
这里容易误解。
invalidation message 本身不区分 insert/delete/update。
所以 abort 的“反向”不是恢复旧 entry。
它只是再次告诉本地 cache：
```text
你可能因为当前事务的 catalog 状态加载了不该长期存在的 entry；
现在把这些 key 相关的 entry 清掉。
```
### 4.3 catcache 与 relcache 分组
`inval.c` 把 pending messages 分成两个物理数组：
```text
CatCacheMsgs
RelCacheMsgs
```
`ProcessInvalidationMessages()` 总是先处理 catcache，再处理 relcache。
这不是随意排序。
relcache rebuild 可能会查 catalog。
如果先 rebuild relcache，再处理 catcache invalidation，就可能刚加载了一个旧 catcache tuple，马上又被冲掉。
先处理 catcache 可以减少这类无意义加载。
另外 relcache message 会去重。
`AddRelcacheInvalidationMessage()` 会检查同一 group 中是否已经有同 relid 或 whole relcache invalidation。
catcache message 一般不做同样去重。
原因是 catcache invalidation 是按 cache id + hash 传播。
重复检测成本未必值得。
### 4.4 `SISeg` 与 `ProcState`
shared ring 在 `sinvaladt.c`。
核心共享结构是 `SISeg`。
它包含：
| 字段 | 语义 |
| --- | --- |
| `minMsgNum` | 仍可能被某个 backend 需要的最老消息号。 |
| `maxMsgNum` | 下一个写入消息号。 |
| `nextThreshold` | 下次触发 cleanup 的队列长度阈值。 |
| `msgnumLock` | 保护 `maxMsgNum` 的 spinlock，也提供 memory barrier。 |
| `buffer[MAXNUMMESSAGES]` | 4096 个 message 的 circular buffer。 |
| `numProcs` / `pgprocnos` | 当前参与 SI 的 backend 槽位集合。 |
| `procState[]` | 每个 backend 的接收位置和状态。 |
每个 active backend 有一个 `ProcState`。
关键字段如下。
| 字段 | 语义 |
| --- | --- |
| `procPid` | 该 slot 是否 active，以及 catchup signal 目标 PID。 |
| `nextMsgNum` | 这个 backend 下次要读取的消息号。 |
| `resetState` | 已经落后太多，不能逐条补消息，必须全 cache reset。 |
| `signaled` | 是否已发送过 catchup interrupt。 |
| `hasMessages` | 快速判断是否可能有未读消息。 |
| `sendOnly` | startup process 等只发送不接收的特殊进程。 |
| `nextLXID` | slot 重用时维持 local transaction id 连续性。 |
这套结构不是每个 database 一份。
它是 cluster-wide shared memory。
message 里用 `dbId` 判断 receiver 是否需要处理。
## 5. 主流程源码 walkthrough：从 catalog update 到本地 command boundary
本节主流程从一个普通 catalog update 开始。
例如 `CREATE INDEX`、`ALTER TABLE`、`CREATE FUNCTION`、`GRANT` 这类命令最终会修改系统表。
许多 catalog 修改通过 `src/backend/catalog/indexing.c` 的包装函数：
```text
CatalogTupleInsert()
  -> simple_heap_insert()
  -> CatalogIndexInsert()
CatalogTupleUpdate()
  -> simple_heap_update()
  -> CatalogIndexInsert()
CatalogTupleDelete()
  -> simple_heap_delete()
```
这些 wrapper 负责 heap tuple 和 catalog index 的一致更新。
真正触发 invalidation 的位置在 heap AM。
`heapam.c` 在 catalog heap insert/update/delete 路径中调用：
```text
CacheInvalidateHeapTuple(relation, tuple, newtuple)
```
入口进入 `inval.c`：
```text
CacheInvalidateHeapTuple()
  -> CacheInvalidateHeapTupleCommon()
```
`CacheInvalidateHeapTupleCommon()` 先过滤。
它只关心 system catalog relation。
用户表 tuple 不进 catcache。
system catalog 的 TOAST table 也被忽略。
然后它调用：
```text
PrepareInvalidationState()
```
这里会为当前 transaction nesting level 创建 `TransInvalidationInfo`。
内存放在 `TopTransactionContext`。
这是因为消息需要活到 transaction end。
### 5.1 catcache 消息如何产生
下一步是 catcache。
`CacheInvalidateHeapTupleCommon()` 对普通 cacheable catalog 调用：
```text
PrepareToInvalidateCacheTuple()
```
这个函数在 `catcache.c`。
它遍历当前 backend 已初始化的 catcache descriptor。
对每个 `cc_reloid == catalog relid` 的 cache：
```text
ConditionalCatalogCacheInitializeCache()
CatalogCacheComputeTupleHashValue()
RegisterCatcacheInvalidation()
```
如果是 update，并且新旧 tuple 的 catcache key hash 不同，就为旧 hash 和新 hash 都登记。
如果 hash 相同，就只登记一次。
这解释了一个重要事实：
```text
invalidation 不是根据 tuple TID 匹配 catcache entry；
它主要根据 catcache ID + key hash 匹配。
```
`sinval.h` 注释说明了原因。
`VACUUM FULL` 或 `CLUSTER` 可能移动 system catalog tuple。
如果 message 只记 TID，排队时的 TID 和处理时的 TID 可能不一致。
hash match 会有小概率 false positive。
PostgreSQL 接受不必要 invalidation。
它不接受 stale cache。
### 5.2 negative entry 为什么也需要 invalidation
catcache 有 negative entry。
它表示：
```text
这个 key 当前不存在。
```
插入一个 catalog tuple 时，本 backend 或其它 backend 可能已经缓存了“不存在”。
所以 insert 也必须发 invalidation。
message 不说“insert”。
receiver 只按 hash 删除 positive 和 negative entry。
这就是 `CatCacheInvalidate()` 的语义：
```text
给定 cache id 和 hashValue，
所有匹配 hash 的 positive/negative entry 都不能继续当新鲜事实。
```
### 5.3 relcache 消息如何产生
`CacheInvalidateHeapTupleCommon()` 处理完 catcache 后，会判断这个 tuple 是否定义了某个 relcache entry。
当前版本直接识别这些 catalog：
| catalog | relcache invalidation 目标 |
| --- | --- |
| `pg_class` | `pg_class.oid` 对应 relation。 |
| `pg_attribute` | `pg_attribute.attrelid` 对应 relation。 |
| `pg_index` | `pg_index.indexrelid` 对应 index relation。 |
| `pg_constraint` | foreign key 的 `conrelid` 对应 relation。 |
如果命中，就调用：
```text
RegisterRelcacheInvalidation()
```
它把 `SharedInvalRelcacheMsg` 加入 `CurrentCmdInvalidMsgs` 的 relcache subgroup。
它还会调用：
```text
GetCurrentCommandId(true)
```
这一步很细。
某些显式 relcache invalidation 不一定伴随普通 heap tuple update。
为了保证下一次 `CommandCounterIncrement()` 不被当成 read-only no-op，源码强制把当前 command id 标记为 used。
否则 `AtCCI_LocalCache()` 可能不会运行。
### 5.4 command boundary 本地消费
catalog heap 操作排队后，当前 command 还没结束。
此时不应该清当前 command 可见的旧 cache。
真正本地消费发生在 `CommandCounterIncrement()`。
源码路径在 `xact.c`：
```text
CommandCounterIncrement()
  -> currentCommandId++
  -> SnapshotSetCommandId(currentCommandId)
  -> AtCCI_LocalCache()
```
`AtCCI_LocalCache()` 做两件事：
```text
AtCCI_RelationMap()
CommandEndInvalidationMessages()
```
先 relation map，后 invalidation。
原因是 relcache invalidation 处理时可能需要看到刚激活的 relmap change。
然后进入 `CommandEndInvalidationMessages()`。
它的核心逻辑是：
```text
ProcessInvalidationMessages(CurrentCmdInvalidMsgs, LocalExecuteInvalidationMessage)
if logical WAL active:
    LogLogicalInvalidations()
AppendInvalidationMessages(PriorCmdInvalidMsgs, CurrentCmdInvalidMsgs)
```
这一步只影响当前 backend。
它不向 shared queue 发送。
因为事务还没 commit。
如果之后 abort，其它 backend 不应该知道这个未提交 catalog change。
## 6. `LocalExecuteInvalidationMessage()` 如何清本地 cache
`LocalExecuteInvalidationMessage()` 是 receiver-side dispatcher。
本 backend 在 command boundary 处理自己的消息时也调用它。
其它 backend 从 shared queue 收到消息时也调用它。
所以本地和远端的最终执行入口一致。
这有一个好处：
```text
同一个 backend 自己的 DDL 在 CCI 时看到的 cache 行为，
和其它 backend 在 commit 后收到 SI 时看到的 cache 行为尽量一致。
```
### 6.1 catcache tuple message
如果 `msg->id >= 0`，它是 catcache message。
dispatcher 先检查 database：
```text
msg->cc.dbId == MyDatabaseId || msg->cc.dbId == InvalidOid
```
命中后执行：
```text
InvalidateCatalogSnapshot()
SysCacheInvalidate(msg->cc.id, msg->cc.hashValue)
CallSyscacheCallbacks(msg->cc.id, msg->cc.hashValue)
```
`SysCacheInvalidate()` 在 `syscache.c`。
它只是定位 `SysCache[cacheId]` 并调用：
```text
CatCacheInvalidate()
```
真正删除或标记 entry 的逻辑在 `catcache.c`。
### 6.2 `CatCacheInvalidate()`
`CatCacheInvalidate()` 先处理 `CatCList`。
它会把该 cache 的所有 list cache entry 都 invalidated。
原因是 partial-key list lookup 很难精确判断哪一个 list 仍正确。
然后它计算 hash bucket：
```text
HASH_INDEX(hashValue, cache->cc_nbuckets)
```
遍历 bucket 中 hash 相同的 `CatCTup`。
如果 entry 没有 active reference，就物理删除。
如果 `refcount > 0`，或者它属于一个 refcount > 0 的 `CatCList`，就只设置：
```text
dead = true
```
这正是前面 catcache 课程中的生命周期边界：
```text
invalidation 管语义过期；
refcount 管内存安全。
```
当前 holder 仍可安全读到它已经拿到的 tuple copy。
未来 lookup 不应该再返回这个 dead entry。
等 holder `ReleaseSysCache()` 或 ResourceOwner cleanup 后，entry 才能物理删除。
`CatCacheInvalidate()` 还会标记正在构造的 entry。
`catcache_in_progress_stack` 上如果有同 cache/hash 的 in-progress entry，会被设置 dead。
这样可以避免一个 tuple 在 detoast 或 catalog access 中途收到 invalidation 后，被构造成出生即 stale 的 cache entry。
### 6.3 whole-catalog message
`SHAREDINVALCATALOG_ID` 走：
```text
CatalogCacheFlushCatalog(msg->cat.catId)
```
它遍历所有 catcache。
凡是 `cc_reloid == catId` 的 cache，整体 reset。
这个路径用于 `VACUUM FULL` / `CLUSTER` 等场景。
这些操作可能没有“某个 key 的值变了”这么简单。
catalog tuple 的 TID 和物理位置都可能变化。
whole-catalog invalidation 比精确 invalidation 粗，但更安全。
### 6.4 relcache message
`SHAREDINVALRELCACHE_ID` 也先检查 database。
然后分两种：
```text
relId == InvalidOid
  -> RelationCacheInvalidate(false)
relId != InvalidOid
  -> RelationCacheInvalidateEntry(relId)
```
`RelationCacheInvalidateEntry()` 在 `relcache.c`。
如果本地 `RelationIdCache` 有该 relation：
```text
relcacheInvalsReceived++
RelationFlushRelation(relation)
```
如果没有 entry，但当前 backend 正在 `RelationBuildDesc()` 构造这个 relid，它会把 `in_progress_list[i].invalidated = true`。
构造完成时会发现并重试。
这个细节很重要。
否则一个 backend 可能在构造 relcache 的中途收到 concurrent DDL invalidation，然后把已经 stale 的 `RelationData` 插入本地 hash。
whole relcache reset 走 `RelationCacheInvalidate(false)`。
它两阶段处理。
第一阶段删除 refcount 为 0 的 entry。
对有 refcount 的 entry，收集到 rebuild list。
第二阶段 rebuild 仍被打开的 entry。
它还先处理 nailed 或 mapped relation 的特殊顺序。
这不是普通 hash 清空。
原因是 relcache entry 内部有大量指针和子 memory context，且可能正被当前调用栈持有。
### 6.5 plan cache callback
`LocalExecuteInvalidationMessage()` 处理 catcache/relcache 后会调用 callback。
`plancache.c` 在 `InitPlanCache()` 注册：
```text
CacheRegisterRelcacheCallback(PlanCacheRelCallback, ...)
CacheRegisterSyscacheCallback(PROCOID, PlanCacheObjectCallback, ...)
CacheRegisterSyscacheCallback(TYPEOID, PlanCacheObjectCallback, ...)
CacheRegisterSyscacheCallback(NAMESPACEOID, PlanCacheSysCallback, ...)
...
```
relcache invalidation 进入 `PlanCacheRelCallback()`。
它扫描 `saved_plan_list` 和 `cached_expression_list`。
如果 dependency list 包含该 relid，就把 `CachedPlanSource` 或 generic plan 标成 invalid。
PROCOID/TYPEOID 等 syscache invalidation 进入 `PlanCacheObjectCallback()`。
它用 `PlanInvalItem` 的 `cacheId` 和 `hashValue` 判断依赖。
其它一些 syscache callback 直接 `ResetPlanCache()`。
所以 plan cache 不是 shared invalidation message 的直接 payload。
它是本地 callback 的派生影响。
## 7. commit 后如何发送给其它 backend
当前事务每个 command 的消息，在 CCI 后已经从 `CurrentCmdInvalidMsgs` 移到了 `PriorCmdInvalidMsgs`。
到 top-level commit 时，`xact.c` 调用：
```text
AtEOXact_RelationCache(true)
AtEOXact_TypeCache()
AtEOXact_Inval(true)
```
`AtEOXact_Inval(true)` 的关键步骤是：
```text
if RelcacheInitFileInval:
    RelationCacheInitFilePreInvalidate()
AppendInvalidationMessages(PriorCmdInvalidMsgs, CurrentCmdInvalidMsgs)
ProcessInvalidationMessagesMulti(PriorCmdInvalidMsgs, SendSharedInvalidMessages)
if RelcacheInitFileInval:
    RelationCacheInitFilePostInvalidate()
transInvalInfo = NULL
```
这里有三个关键 ordering。
第一，commit path 中 invalidation 发生在 relcache refs cleanup 之后。
`AtEOXact_RelationCache()` 注释说明了这点。
如果 abort，也需要先 reset refcounts，再处理 pending invalidation。
第二，invalidation 发生在释放 heavyweight locks 之前。
等待 relation lock 的 backend 被唤醒后，不应该继续使用旧 relcache。
第三，relcache init file unlink 被包在发送 SI message 前后。
`RelationCacheInitFilePreInvalidate()` 拿 `RelCacheInitLock` 并 unlink init file。
caller 在锁内发送 SI message。
`RelationCacheInitFilePostInvalidate()` 再释放锁。
这样避免新 backend 读取旧 `pg_internal.init`，然后又错过对应 invalidation。
### 7.1 `SendSharedInvalidMessages()`
`SendSharedInvalidMessages()` 声明在 `sinval.h`。
底层进入 `sinvaladt.c` 的：
```text
SIInsertDataEntries()
```
它把消息分批写入 shared ring。
每批最多：
```text
WRITE_QUANTUM = 64
```
这样避免一次持有 `SInvalWriteLock` 太久。
写入流程是：
```text
LWLockAcquire(SInvalWriteLock, LW_EXCLUSIVE)
if buffer close to full:
    SICleanupQueue(true, nthistime)
copy messages into buffer[max % MAXNUMMESSAGES]
SpinLockAcquire(msgnumLock)
maxMsgNum = max
SpinLockRelease(msgnumLock)
for each active ProcState:
    hasMessages = true
LWLockRelease(SInvalWriteLock)
```
`maxMsgNum` 的 spinlock 不只是保护一个 int。
注释明确说它提供 memory barrier。
writer 必须先把 message 写入 ring，再让 receiver 看到新的 `maxMsgNum`。
弱内存序平台上不能依赖普通 int store。
### 7.2 commit WAL 与 logical decoding
`xactGetCommittedInvalidationMessages()` 在 `RecordTransactionCommit()` 前收集消息，用于 commit record。
注释强调它必须在 `AtEOXact_Inval()` 前运行。
因为 `AtEOXact_Inval()` 会清掉 pending state。
`ProcessCommittedInvalidationMessages()` 用于 redo commit 或 standby redo。
它会在 replay 端发送 shared invalidation。
这让 hot standby query backend 能看到 schema change。
在 `effective_wal_level = logical` 时，`CommandEndInvalidationMessages()` 还可能调用 `LogLogicalInvalidations()`。
这是为了支持 decoding in-progress transactions。
所以 invalidation 不只是前台 backend 之间的内存消息。
它也和 WAL/redo/logical decoding 的可见性传播有交叉。
但普通事务性 SI 的核心规则仍然是：
```text
commit 后广播；
receiver 本地解释；
cache 内容不写入 WAL。
```
## 8. receiver 如何接收 shared queue
receiver 的主要入口是：
```text
AcceptInvalidationMessages()
```
它调用：
```text
ReceiveSharedInvalidMessages(LocalExecuteInvalidationMessage,
                             InvalidateSystemCaches)
```
`ReceiveSharedInvalidMessages()` 再循环调用 `SIGetDataEntries()`。
`SIGetDataEntries()` 每次返回：
| 返回值 | 含义 |
| --- | --- |
| `0` | 当前没有消息。 |
| `n > 0` | 读到 n 条普通消息。 |
| `-1` | 当前 backend 已进入 reset state，必须全量 reset。 |
### 8.1 接收位置
`SIGetDataEntries()` 使用当前 backend 的 `ProcState.nextMsgNum`。
它先用 unlocked `hasMessages` 做快速返回。
如果可能有消息，拿：
```text
SInvalReadLock, LW_SHARED
```
这里的 shared lock 不是纯读。
源码注释特别提醒：
```text
多个 reader 可以并行，
但每个 reader 只修改自己的 ProcState。
```
然后读取当前 `maxMsgNum`。
因为 reader 没有 `SInvalWriteLock`，所以读取 `maxMsgNum` 必须拿 `msgnumLock`。
接着从：
```text
buffer[nextMsgNum % MAXNUMMESSAGES]
```
拷贝消息，并推进自己的 `nextMsgNum`。
### 8.2 接收时机
`AcceptInvalidationMessages()` 的主要调用点包括：
| 调用点 | 目的 |
| --- | --- |
| transaction start 的 `AtStart_Cache()` | 每个新事务开始前先接收外部 catalog change。 |
| `syscache.c` 的 inplace update lock loop | 等待 inplace update 后确保处理对应 syscache inval。 |
| relcache init file 写出前 | 写 `pg_internal.init` 前确认没有未处理 relcache inval。 |
| catchup interrupt 处理路径 | 后台提示某 backend 赶快读 SI queue。 |
| debug discard cache 测试路径 | 故意放大 cache flush 时机，暴露 stale pointer 问题。 |
最关键的是 transaction start。
这让一个 backend 在开始新事务前处理其它事务已经 commit 的 catalog change。
在长事务内部，PostgreSQL 不承诺每个瞬间都主动处理所有外部 invalidation。
系统依赖 lock ordering、command boundary 和 cache lookup 路径中的安全点。
### 8.3 catchup interrupt
当一个 backend 落后太多，`SICleanupQueue()` 会选一个最落后的未 signaled backend，发送：
```text
PROCSIG_CATCHUP_INTERRUPT
```
signal handler 设置：
```text
catchupInterruptPending
```
后续 `ProcessCatchupInterrupt()` 会调用接收逻辑。
设计上通常一次只推动一个落后 backend。
这样避免大量 backend 同时醒来抢 `SInvalReadLock` / `SInvalWriteLock`。
如果某 backend 长期卡在不能处理 signal 的位置，它最终可能被 reset。
系统不会让一个慢 receiver 永久阻塞所有 sender。
## 9. 生命周期 / ownership / cleanup
### 9.1 message 谁创建
事务性 message 由 catalog tuple 修改路径创建。
主要入口：
```text
CacheInvalidateHeapTuple()
CacheInvalidateCatalog()
CacheInvalidateRelcache()
CacheInvalidateRelcacheByTuple()
CacheInvalidateRelcacheByRelid()
CacheInvalidateRelSync()
```
非事务性 message 由特定底层状态变化立即发送：
```text
CacheInvalidateSmgr()
CacheInvalidateRelmap()
CacheInvalidateHeapTupleInplace()
```
它们的区别是：
```text
事务性消息等 command / transaction boundary；
非事务性消息在实际物理或 inplace change 周围立即发送。
```
### 9.2 pending message 谁持有
`TransInvalidationInfo` 和 message arrays 放在 `TopTransactionContext`。
它们由当前 backend 持有。
其它 backend 看不到。
subtransaction 也用同一套 arrays，通过 index group 表示边界。
这样避免搬动 message。
subtransaction commit 时，只需要调整 parent group 的 `firstmsg`/`nextmsg`。
subtransaction abort 时，本地处理 prior messages，然后丢掉当前层 state。
### 9.3 shared queue 谁持有
`SISeg` 在 main shared memory。
它由 postmaster 启动期通过 `SharedInvalShmemCallbacks` 申请和初始化。
每个 backend 在 `SharedInvalBackendInit()` 注册自己的 `ProcState` slot。
退出时通过 `on_shmem_exit()` 调用：
```text
CleanupInvalidationState()
```
它会：
```text
保存 nextLXID
procPid = 0
nextMsgNum = 0
resetState = false
signaled = false
从 pgprocnos dense array 移除 MyProcNumber
```
这不是事务级 cleanup。
这是 backend 生命周期 cleanup。
### 9.4 cache entry 谁释放
shared invalidation 不释放其它 backend 的对象。
本地 cache 的释放仍由各 cache 自己决定。
catcache：
```text
refcount == 0
  -> CatCacheRemoveCTup()
refcount > 0
  -> dead = true
  -> ReleaseSysCache() 或 ResourceOwner cleanup 后再移除
```
relcache：
```text
rd_refcnt == 0
  -> RelationClearRelation()
rd_refcnt > 0
  -> RelationFlushRelation() 选择 rebuild / mark invalid / reload subset
```
plancache：
```text
callback 只把 is_valid 置 false；
下一次执行或 revalidation 时重新 parse/analyze/rewrite/plan。
```
### 9.5 ERROR / abort 谁兜底
`ERROR` longjmp 不会自动帮你发 SI。
它通过事务 abort cleanup 兜底。
abort path 中：
```text
AtEOXact_RelationCache(false)
AtEOXact_TypeCache()
AtEOXact_Inval(false)
```
`AtEOXact_Inval(false)` 不发送给其它 backend。
因为其它 backend 没看到未提交 catalog change。
但它会本地处理：
```text
PriorCmdInvalidMsgs
```
原因是本 backend 之前 command boundary 已经按未提交 catalog state 清过或加载过 cache。
abort 后这些本地 cache state 也不可信。
当前 command 尚未处理的 `CurrentCmdInvalidMsgs` 可以丢弃。
`TopTransactionContext` 随后会 reset。
message arrays 不需要逐项释放。
## 10. 正确性机制层次
shared invalidation 的 correctness 不是一个锁保证的。
它由多个层次拼起来。
| 层次 | 机制 | 保证 | 不保证 |
| --- | --- | --- | --- |
| MVCC / command id | `CommandCounterIncrement()`、catalog current snapshot | 本 backend 下一个 command 看到 catalog change。 | 其它 backend 立即看到。 |
| transaction commit | commit record / transaction end ordering | 其它 backend 只在 commit 后收到事务性消息。 | receiver 立刻重建所有 cache。 |
| heavyweight lock | DDL 持有 relation/object lock | 等待者在锁释放前会先有机会收到必要 invalidation。 | cache entry 内存安全。 |
| local refcount | `CatCTup.refcount`、`Relation.rd_refcnt` | 正在使用的本地指针不悬挂。 | 元数据语义仍然新鲜。 |
| shared ring locks | `SInvalWriteLock`、`SInvalReadLock`、`msgnumLock` | ring buffer 并发读写和 memory ordering。 | 消息永不丢失。 |
| overflow reset | `resetState` + `InvalidateSystemCaches()` | 丢失精确消息后恢复正确性。 | 保留精确 invalidation 成本。 |
| callbacks | `CacheRegister*Callback()` | 高层 cache 依赖 catcache/relcache 失效。 | callback 一定精确或低成本。 |
一个常见错误是把 invalidation 当成 lock。
它不是。
invalidation 不阻止并发 DDL。
它也不等待 reader 停止使用旧指针。
它只是让 reader 在安全边界知道：
```text
你本地某些语义缓存不应继续当事实。
```
并发互斥由 lock 管。
可见性由 MVCC/command id 管。
内存安全由 refcount/ResourceOwner 管。
跨 backend 传播由 SI queue 管。
## 11. 错误路径 / 异常路径 / fallback
### 11.1 subtransaction abort
subtransaction 有自己的 `TransInvalidationInfo` 栈层。
如果 subtransaction commit：
```text
CommandEndInvalidationMessages()
Append child PriorCmdInvalidMsgs to parent PriorCmdInvalidMsgs
RelcacheInitFileInval 传给 parent
```
如果 subtransaction abort：
```text
ProcessInvalidationMessages(child PriorCmdInvalidMsgs,
                            LocalExecuteInvalidationMessage)
pop child state
```
不向其它 backend 发送。
因为 subtransaction 的 catalog change 不会 commit。
但本 backend 可能在子事务内经历过 command boundary，并加载过基于子事务 catalog 状态的 cache。
这些 cache 必须本地清掉。
### 11.2 two-phase commit
`PostPrepare_Inval()` 在 PREPARE 成功后调用：
```text
AtEOXact_Inval(false)
```
它故意像 abort 一样处理本地 state。
原因是 PREPARE 后，外部世界还不认为事务 commit。
如果 prepared transaction 后续 commit，相关 invalidation 会像普通 commit 一样被接收。
如果 abort，就没有更多要做。
这再次说明：
```text
本 backend 本地 cache cleanup 和全局 commit 广播不是同一件事。
```
### 11.3 queue overflow reset
`sinvaladt.c` 的 ring 只有：
```text
MAXNUMMESSAGES = 4096
```
如果某些 backend 落后太多，sender 调用 `SICleanupQueue()` 时可能发现必须释放空间。
对于阻止空间回收的 backend：
```text
stateP->resetState = true
```
之后这个 backend 调用 `SIGetDataEntries()` 会返回 `-1`。
上层 `ReceiveSharedInvalidMessages()` 会调用 reset function：
```text
InvalidateSystemCaches()
```
它会：
```text
InvalidateCatalogSnapshot()
ResetCatalogCachesExt(false)
RelationCacheInvalidate(false)
syscache callbacks with hash 0
relcache callbacks with InvalidOid
relsync callbacks with InvalidOid
```
这是一条 correctness fallback。
精确消息丢了。
系统不尝试补猜。
它直接把所有 invalidatable local state 当不可信。
成本很高，但正确。
### 11.4 in-place update
`CacheInvalidateHeapTupleInplace()` 用 `PrepareInplaceInvalidationState()`。
inplace invalidation 不走普通 transaction command 分组。
相关流程：
```text
PreInplace_Inval()
AtInplace_Inval()
ForgetInplace_Inval()
```
`AtInplace_Inval()` 要求在 critical section 中：
```text
Assert(CritSectionCount > 0)
```
它把消息直接 `SendSharedInvalidMessages()`。
原因是 inplace update 属于非事务性缓存语义变化。
底层 tuple 已经原地变化，不能等到 transaction end 才告诉其它 backend。
`syscache.c` 的 inplace update lock loop 在等待 tuple lock 后会调用：
```text
AcceptInvalidationMessages()
```
确保刚完成的 inplace update 对当前 backend 的 syscache 生效。
### 11.5 relcache init file 失效失败
如果 relcache invalidation 影响 init file 中的 critical relation，`RegisterRelcacheInvalidation()` 会设置：
```text
RelcacheInitFileInval = true
```
commit 时 `RelationCacheInitFilePreInvalidate()` 会 unlink local/shared `pg_internal.init`。
这一步在发送 SI message 前发生。
如果 unlink 遇到非 `ENOENT` 错误，会 ERROR。
此时事务还没释放锁，仍能 abort。
这是为了避免旧 init file 成为新 backend 的 stale metadata 来源。
## 12. 成本、资源与跨模块传播
### 12.1 发送成本
发送成本随消息数增长。
常见放大变量：
- 一个 catalog relation 上有多个 catcache。
- update key 变化时可能旧 hash + 新 hash 都登记。
- DDL 触发 relcache、catcache、snapshot、relsync 多类消息。
- 大事务中多 command 累积 `PriorCmdInvalidMsgs`。
- commit 时一次性发送给 shared ring。
`SIInsertDataEntries()` 每批最多 64 条消息。
这控制了单次持有 `SInvalWriteLock` 的时间。
但并不消除总发送成本。
如果某 workload 在一个事务内修改大量 catalog rows，commit latency 可能包含明显 SI 发送开销。
### 12.2 接收成本
receiver 每条消息至少要做 type dispatch 和 database filter。
如果消息命中本 database，成本继续扩散。
catcache message：
```text
按 hash bucket 扫 CatCTup；
清所有 CatCList；
调用 syscache callbacks。
```
relcache message：
```text
hash lookup RelationIdCache；
可能 RelationFlushRelation()；
可能 rebuild open relation；
调用 relcache callbacks；
plan cache 扫 saved_plan_list。
```
whole relcache reset 或 overflow reset 成本更高。
它会扫描很多本地 cache。
如果 backend 保存了大量 prepared statements 或 cached plans，callback 扫描也会放大。
### 12.3 与相邻模块的边界
与 catcache 的边界：
```text
inval.c 只计算并发送 cache id + hash；
catcache.c 决定删除、dead 标记、in-progress 标记。
```
与 relcache 的边界：
```text
inval.c 只发送 relid；
relcache.c 决定 flush、rebuild、init file unlink、in-progress retry。
```
与 plancache 的边界：
```text
inval.c 只维护 callback list；
plancache.c 用 dependency list 决定哪些 plan invalid。
```
与 transaction manager 的边界：
```text
xact.c 决定 command id、commit/abort/subxact cleanup ordering；
inval.c 只在这些边界处理 pending messages。
```
与 storage/ipc 的边界：
```text
inval.c 给出 messages；
sinvaladt.c 只负责可靠地让各 backend 读到或 reset。
```
与 WAL/redo 的边界：
```text
commit record 可携带 invalidation messages；
redo/standby 通过 ProcessCommittedInvalidationMessages() 再发 SI。
```
## 13. 观测与诊断入口
### 13.1 能直接看到什么
普通 SQL 不能直接查询 shared sinval ring。
但可以观察到结果。
第一类是 prepared statement 被 DDL invalidated。
实验中可以看到同一个 prepared query 在 DDL 后重新规划。
第二类是 catalog cache 或 relcache flush 后的 CPU 抖动。
这通常需要 `perf`、gdb、trace log 或临时计数器。
第三类是 wait event。
高 catalog churn 下可能看到：
```text
SInvalReadLock
SInvalWriteLock
RelCacheInitLock
```
具体 wait event 名称取决于版本和观测工具展示。
wait event 只能说明等待位置。
不能证明 stale cache 或具体消息类型。
### 13.2 gdb 断点建议
源码跟读时，最有用的断点是：
```text
CacheInvalidateHeapTupleCommon
CommandEndInvalidationMessages
AtEOXact_Inval
SIInsertDataEntries
SIGetDataEntries
LocalExecuteInvalidationMessage
CatCacheInvalidate
RelationCacheInvalidateEntry
PlanCacheRelCallback
PlanCacheObjectCallback
```
推荐跟一条 `ALTER TABLE ... ADD COLUMN`。
观察点：
```text
CacheInvalidateHeapTupleCommon() 中 tupleRelId 是哪个 catalog。
PrepareToInvalidateCacheTuple() 产生了哪些 cache id/hash。
RegisterRelcacheInvalidation() 产生的 relId 是哪个 relation。
CommandEndInvalidationMessages() 何时本地执行。
AtEOXact_Inval(true) 何时发送 shared messages。
另一个 backend 何时进入 LocalExecuteInvalidationMessage()。
```
### 13.3 只能推断什么
以下状态通常不能从 SQL 直接看到：
- 某个 backend 的 `TransInvalidationInfo` 当前有多少 pending messages。
- shared ring 的 `minMsgNum` / `maxMsgNum`。
- 某个 backend 是否刚被置为 `resetState`。
- 某个 `CatCTup` 是否 `dead = true`。
- 某个 relcache entry 是被删除还是 rebuild。
- plan cache invalidation 是由 relid callback 还是 syscache callback 触发。
这些需要 gdb、临时日志、DTrace/eBPF、perf probe 或定制计数器。
不要把 `pg_stat_activity` 中的等待状态过度解释成完整因果。
## 14. 课堂实验
### 实验 1：单 backend command boundary
目标：观察当前 backend 在 CCI 后本地清 cache，而不是等 commit。
步骤：
```sql
BEGIN;
CREATE TABLE si_demo(a int);
SELECT relname FROM pg_class WHERE relname = 'si_demo';
ALTER TABLE si_demo ADD COLUMN b text;
SELECT attname FROM pg_attribute
WHERE attrelid = 'si_demo'::regclass
ORDER BY attnum;
COMMIT;
```
断点：
```text
CacheInvalidateHeapTupleCommon
CommandEndInvalidationMessages
LocalExecuteInvalidationMessage
```
观察：
```text
CREATE TABLE 和 ALTER TABLE 都会产生 catalog invalidation。
同一事务中后续 SELECT 能看到新 catalog state。
这个可见性来自 CommandCounterIncrement() 后本地处理 pending inval。
```
源码回扣：
```text
CommandCounterIncrement()
  -> AtCCI_LocalCache()
  -> CommandEndInvalidationMessages()
```
### 实验 2：双 backend commit 后传播
目标：观察其它 backend 在 commit 后接收 SI。
Backend A：
```sql
CREATE TABLE si_demo2(a int);
PREPARE p AS SELECT * FROM si_demo2;
EXECUTE p;
```
Backend B：
```sql
ALTER TABLE si_demo2 ADD COLUMN b int;
COMMIT;
```
Backend A：
```sql
EXECUTE p;
```
断点：
```text
AtEOXact_Inval
SIInsertDataEntries
AcceptInvalidationMessages
LocalExecuteInvalidationMessage
PlanCacheRelCallback
```
观察：
```text
Backend B commit 后发送 relcache invalidation。
Backend A 下次事务开始或执行路径接收 message。
relcache callback 标记 cached plan invalid。
下一次 EXECUTE 需要重新验证或重建计划。
```
注意：
```text
shared message 没有携带 plan id。
plan cache 是 callback 派生失效。
```
### 实验 3：队列 overflow 的源码实验
目标：理解 reset fallback，不建议在生产库做。
做法：
```text
在测试编译中临时把 MAXNUMMESSAGES 改小，例如 32。
启动多个 backend。
让一个 backend 长时间不处理 invalidation。
另一个 backend 执行大量 catalog churn。
在 SIGetDataEntries() 和 InvalidateSystemCaches() 断点观察 reset。
```
预期：
```text
落后 backend 的 ProcState.resetState 被置 true。
下次接收返回 -1。
上层调用 InvalidateSystemCaches()。
精确消息丢失，但 correctness 通过全量 reset 恢复。
```
## 15. 常见误区
### 误区 1：invalidation message 表示 tuple 删除
不是。
catcache message 不区分 insert、delete、update。
它只表示：
```text
这个 cache id + hashValue 对应的本地事实不可信。
```
insert 需要 invalidation，是为了清 negative entry。
delete 需要 invalidation，是为了清 positive entry。
update 可能两者都需要。
### 误区 2：command 内 catalog update 后应立刻清 cache
不对。
同一 command 内旧 tuple 仍可能按 command visibility 规则有效。
如果 `heap_update()` 立刻 flush，后续同 command 又可能重新加载不该在下一 command 保留的状态。
正确边界是 command end。
### 误区 3：commit 发送 SI 后 receiver 已经重建好所有 cache
不对。
发送只是把 message 写入 ring。
receiver 要在自己的安全点接收。
接收后也不一定立即重建。
catcache 可能只是删除 entry。
relcache 可能标 invalid。
plan cache 可能只是 `is_valid = false`。
### 误区 4：refcount 能保证 cache 语义新鲜
不对。
refcount 保证当前 backend 不释放正在被使用的本地对象。
invalidation 可能已经把 entry 标 dead。
holder 仍能读旧对象直到 release。
未来 lookup 必须重新加载。
### 误区 5：队列 overflow 意味着 correctness 丢失
不对。
精确消息丢失。
correctness 通过 reset fallback 保住。
代价是 receiver 丢弃更大范围的本地 cache，并在后续请求中重建。
### 误区 6：plan cache 由 SI message 直接命中
不对。
SI message 只到 catcache/relcache/snapshot/smgr/relmap/relsync 这类基础类型。
plan cache 通过 registered callback 和 dependency list 派生失效。
## 16. 讨论题
1. 为什么 `CacheInvalidateHeapTuple()` 只排队，不在 heap tuple 修改时立刻调用 `CatCacheInvalidate()`？
2. 为什么 catcache invalidation 用 hash value，而不是 tuple TID？
3. insert catalog tuple 时，为什么也必须 invalidate catcache？
4. `AtEOXact_Inval(true)` 为什么必须在释放 heavyweight locks 前执行？
5. 一个 `CatCTup` 同时 `dead = true` 且 `refcount > 0` 时，系统分别保证了什么？
6. 队列 overflow 后，为什么 reset 全部 invalidatable cache 比尝试补偿缺失消息更合理？
7. 为什么 relcache invalidation message 只带 relid，而不带新的 tuple descriptor？
8. plan cache 为什么注册 relcache/syscache callback，而不是让 shared invalidation 认识每个 cached plan？
## 17. 本节小结
本节主链路是：
```text
catalog heap change
  -> CacheInvalidateHeapTupleCommon()
  -> PrepareToInvalidateCacheTuple() / RegisterRelcacheInvalidation()
  -> CurrentCmdInvalidMsgs
  -> CommandCounterIncrement()
  -> CommandEndInvalidationMessages()
  -> LocalExecuteInvalidationMessage()
  -> PriorCmdInvalidMsgs
  -> AtEOXact_Inval(true)
  -> SendSharedInvalidMessages()
  -> SIInsertDataEntries()
  -> other backend AcceptInvalidationMessages()
  -> SIGetDataEntries()
  -> LocalExecuteInvalidationMessage()
  -> catcache/relcache/plancache local cleanup
```
核心状态边界是：
```text
pending message 是 backend-local transaction state；
shared sinval ring 是 cluster-wide delivery state；
catcache/relcache/plancache entry 是 receiver 的 backend-local cache state。
```
ownership 规则是：
```text
TopTransactionContext 持有 pending transactional messages；
SISeg 持有 shared ring 和 per-backend cursors；
各 cache 自己持有本地对象；
ResourceOwner/refcount 防止正在使用的本地对象悬挂。
```
错误路径的关键不是恢复旧对象。
而是：
```text
abort 清掉本 backend 已经按未提交 catalog 状态加载的 cache；
subtransaction abort 清掉子事务已影响的 local cache；
overflow reset 丢弃精确性，保住 correctness；
inplace update 走立即发送路径。
```
能直接观测的是 commit latency、wait、replan、cache miss 后的性能现象。
很难直接观测的是每个 backend 的 pending group、ring cursor、`CatCTup.dead` 和 relcache rebuild 决策。
本节可迁移的系统规律是：
```text
跨进程 cache invalidation 不应传递本地对象，
而应传递稳定、最小、可幂等解释的过期事实；
receiver 在自己的生命周期、refcount 和安全边界内完成 cleanup。
```
这个规律会在 relmap、smgr、plan cache、logical decoding 和 extension cache 设计中反复出现。
