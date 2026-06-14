# PostgreSQL cache diagnostics / source entry map
## 课程定位
前置知识：已经读过 syscache / catcache lookup lifetime、relcache build invalidation、typcache / opclass cache、shared invalidation message flow，以及 catalog snapshot / syscache consistency。
本节唯一主问题：
```text
如何从 relcache/syscache 错误、stale metadata 和 invalidation 日志，回到正确的源码入口并验证状态变化？
```
核心矛盾：
```text
runtime 现场给出的通常只是错误字符串、OID、DEBUG4 日志或 stale plan 现象；
真正原因却分散在 backend-local cache、shared invalidation、transaction boundary、plan/type 派生缓存和错误上报路径之间。
```
一句话运行模型：
```text
先把现场现象归类为 syscache miss、relcache stale、derived-cache stale 或 sinval reset；
再把 OID / cache ID / relid / hashvalue 映射到本地 cache 状态；
然后从最窄源码入口设置断点；
最后用 SQL、日志、gdb 或 debug_discard_caches 证明状态按预期推进。
```
学完后应能判断：
- 看到 `cache lookup failed for relation 12345` 时，先找哪个 caller，而不是先怀疑 catcache 本身。
- 看到 stale metadata 时，区分是 `RelationData` 旧、`CachedPlanSource` 旧、`TypeCacheEntry` 旧，还是 catalog snapshot 边界没推进。
- 看到 `cache state reset` 或 commit invalidation 日志时，知道它来自 shared invalidation queue 的哪一层。
- 知道 `debug_discard_caches` 能暴露哪类 stale pointer 假设，不能模拟哪类生产竞态。
- 能从一个 OID、一个 cache ID 或一个 relid 回到 `catcache.c`、`syscache.c`、`relcache.c`、`inval.c`、`plancache.c`、`typcache.c` 的具体入口。
- 能解释为什么 `refcount`、`rd_refcnt`、`is_valid`、`dead`、`rd_isvalid` 不能单独代表“元数据新鲜”。
本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e9`。
本节是诊断地图，不是第 56 到 63 节的重复讲解。
第 56 节讲一个 `CatCTup` 的 lifetime。
第 57 节讲一个 `RelationData` 的 build / invalidation。
第 58 节讲 typcache / opclass 派生状态。
第 59 节讲 shared invalidation message flow。
第 60 节讲 catalog snapshot 和 syscache consistency。
第 61 到 63 节补上 dependency、relmapper 物理身份和 error cleanup。
本节把这些内容压成一条可操作的定位链：
```text
runtime 现象
  -> 现场 token：错误文本 / OID / relid / log line / wait event
  -> cache 状态：entry / refcount / invalid flag / pending message / plan validity
  -> 源码入口：caller -> wrapper -> local cache -> invalidation dispatcher
  -> 验证：SQL catalog query / gdb breakpoint / log level / debug_discard_caches
```
## 1. 本节在总主线中的位置
前面八节已经把 catalog/cache 子系统拆成对象级机制。
这对理解源码很有用。
但生产诊断不会先给你一个整洁对象。
它通常给你这些碎片：
```text
ERROR: cache lookup failed for relation 16442
ERROR: cache lookup failed for type 16620
ERROR: cached plan must not change result type
DEBUG4: cache state reset
DEBUG4: replaying commit with 8 messages
wait_event = SInvalRead
某个 backend 在 ALTER TABLE 后仍按旧列布局执行
某个 prepared statement 在 DDL 后重新分析时报错
```
这些碎片不能直接对应一个模块。
同一句 `cache lookup failed` 可能来自：
- `lsyscache.c` 的标量 helper。
- `relcache.c` 构造 index relation 时查不到 `pg_index`。
- `fmgr.c` 初始化函数调用信息时查不到 `pg_proc`。
- `typcache.c` 构造 range / multirange 元数据时查不到 `pg_type`。
- `objectaddress.c` 把 object address 还原成具体对象时查不到 catalog row。
同一个 stale metadata 也可能来自不同层：
- syscache 返回了仍被 caller 持有但已 `dead` 的 tuple。
- relcache entry 收到 invalidation 后 `rd_isvalid = false`，但仍因 `rd_refcnt > 0` 保留。
- plan cache callback 只把 `CachedPlanSource.is_valid` 标 false，下一次取 plan 才 revalidate。
- typcache callback 只清 flag，下一次请求相同能力才重新填字段。
- catalog snapshot 没有在 command boundary 之前推进。
所以本节不问：
```text
catcache 怎么实现？
relcache 怎么实现？
shared invalidation 怎么实现？
```
本节只问：
```text
看到一个 runtime 现象时，怎样最短路径回到正确源码入口？
```
这个问题的答案不是一个万能函数。
它是一张 source entry map。
它要保留四个边界。
第一，错误字符串的 caller 才是语义入口。
`SearchSysCache1()` 返回 NULL 只是事实。
为什么 NULL 被解释成 ERROR、FATAL、ignore、retry 或 fallback，要看 caller。
第二，cache entry 是 backend-local。
你在一个 backend 的 gdb 里看到的 `CatCTup`、`RelationData`、`TypeCacheEntry`，不是其它 backend 的对象。
shared invalidation 只传播过期事实。
第三，invalidation 是安全边界上的语义推进，不是锁。
它不阻止 DDL。
它也不保证接收方已经重建。
第四，诊断必须验证状态变化。
只找到函数名是不够的。
要能看到 `dead`、`rd_isvalid`、`is_valid`、pending message、reset 或 callback 确实发生。
## 2. 核心矛盾与诊断运行模型
诊断 cache 问题最容易走偏的地方，是把所有现象都叫“缓存不一致”。
这个词太宽。
它会把完全不同的状态混在一起。
更好的第一步是把现场压成四类。
| 类别 | runtime 现象 | 先看状态 | 第一源码入口 |
| --- | --- | --- | --- |
| syscache lookup miss | `cache lookup failed for type/function/operator/...` | cache ID、catalog row 是否存在、caller 是否要求必须存在 | `rg "cache lookup failed for ..." src/backend` |
| relcache lookup/build stale | `cache lookup failed for relation/index/access method`、relation open 后旧结构 | `RelationData.rd_isvalid`、`rd_refcnt`、in-progress build | `RelationIdGetRelation()` / `RelationBuildDesc()` |
| derived cache stale | prepared statement、typcache、operator cache、RLS/search_path 相关重分析 | `CachedPlanSource.is_valid`、`CachedPlan.is_valid`、`TypeCacheEntry.flags` | `PlanCacheRelCallback()` / `TypeCache*Callback()` |
| shared invalidation reset/log | `cache state reset`、`replaying commit with N messages`、`SInvalRead/Write` | `SISeg`、`ProcState.resetState`、message queue | `ReceiveSharedInvalidMessages()` / `SIGetDataEntries()` |
这四类不是互斥的根因分类。
它们是诊断入口分类。
一个 `ALTER TABLE` 可以同时触发 relcache invalidation、plan cache invalidation、typcache reset 和 catalog snapshot invalidation。
但现场通常只有一个最窄入口。
诊断时要沿这条线推进：
```text
1. 复制现场 token
2. 找到最窄 caller
3. 判断 caller 期望的 catalog 事实
4. 找到 cache 层的状态
5. 找到 invalidation 是否应该到达
6. 找到 retry / fallback 是否发生
7. 用一次可重复实验验证判断
```
如果你跳过第 2 步，直接看 `catcache.c`，容易误判。
`catcache.c` 只知道：
```text
给定 cache descriptor 和 key，返回 tuple 或 NULL。
```
它不知道这个 NULL 对上层是不是错误。
例如某些 helper 用 `SearchSysCacheExists()` 做 existence check，NULL 是正常 false。
另一些 helper 立刻 `elog(ERROR, "cache lookup failed ...")`，因为调用者已有 OID 依赖，按语义该对象必须存在。
如果你跳过第 5 步，只看 catalog row 是否存在，也容易误判。
其它 backend 可能还没接收 invalidation。
当前 backend 可能还在同一 command 内。
prepared statement 可能已经被 callback 标 invalid，但还没重新分析。
relcache entry 可能已经 stale，但因 `rd_refcnt > 0` 不能直接 free。
所以本节的诊断模型是：
```text
错误文本不是根因；
OID 不是根因；
cache miss 不是根因；
invalidation message 也不是根因。
根因要在 caller 的语义假设和 cache 状态推进之间找。
```
## 3. 核心文件分工与阅读顺序
本节的文件表按诊断入口排序。
不要按文件名字母顺序读。
| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/utils/error/elog.c` | 错误如何形成 `ErrorData`、默认 SQLSTATE、为什么错误上报层不应反向依赖 relcache/syscache。 |
| 2 | `src/backend/utils/cache/syscache.c` | `SearchSysCache*()` wrapper、`SearchSysCacheCopy()`、`SearchSysCacheLocked1()`、cache ID 到 catcache 的桥。 |
| 3 | `src/backend/utils/cache/catcache.c` | hit / miss、negative entry、`CatCTup.dead`、`refcount`、in-progress stale retry、`PrepareToInvalidateCacheTuple()`。 |
| 4 | `src/backend/utils/cache/relcache.c` | `RelationIdGetRelation()`、`RelationBuildDesc()`、`RelationCacheInvalidateEntry()`、`RelationCacheInvalidate()`、relcache init file。 |
| 5 | `src/backend/utils/cache/inval.c` | `LocalExecuteInvalidationMessage()`、`CommandEndInvalidationMessages()`、`AtEOXact_Inval()`、callback registry、`debug_discard_caches`。 |
| 6 | `src/backend/storage/ipc/sinval.c` | `ReceiveSharedInvalidMessages()`、`cache state reset` 日志、catchup interrupt。 |
| 7 | `src/backend/storage/ipc/sinvaladt.c` | `SISeg`、`ProcState`、`SIInsertDataEntries()`、`SIGetDataEntries()`、overflow reset。 |
| 8 | `src/backend/utils/cache/plancache.c` | relcache/syscache callback 如何把 plan source / generic plan 标 invalid。 |
| 9 | `src/backend/utils/cache/typcache.c` | typcache callback、opclass/domain/composite 派生状态如何失效。 |
| 10 | `src/backend/utils/misc/guc_parameters.dat` | `debug_discard_caches` GUC 的 context、默认值、是否可用。 |
| 11 | `src/include/utils/inval.h` | `debug_discard_caches` 的编译开关和 public invalidation API。 |
| 12 | `src/include/storage/sinval.h` | `SharedInvalidationMessage` 的 message 类型和 payload。 |
| 13 | `src/include/utils/catcache.h` | `CatCache`、`CatCTup`、`CatCList` 字段语义。 |
| 14 | `src/include/utils/rel.h` | `RelationData` 字段语义。 |
| 15 | `src/include/utils/plancache.h` | `CachedPlanSource`、`CachedPlan` 的 validity 和 refcount。 |
| 16 | `src/include/utils/typcache.h` | `TypeCacheEntry` flags 和派生字段。 |
推荐阅读顺序：
```text
现场错误字符串
  -> rg 定位 caller
  -> caller 的 SearchSysCache / RelationIdGetRelation / plan cache helper
  -> syscache.c 或 relcache.c wrapper
  -> catcache.c hit/miss 或 RelationBuildDesc
  -> inval.c LocalExecuteInvalidationMessage
  -> sinval.c / sinvaladt.c message 接收或 reset
  -> plancache.c / typcache.c 派生 callback
```
这个顺序有两个好处。
第一，它先保留 caller 语义。
第二，它让你只在需要时进入共享队列。
很多 `cache lookup failed` 并不需要先看 `sinvaladt.c`。
很多 stale plan 问题也不需要先看 `catcache.c`。
## 4. 关键 runtime 现象与 source entry
### 4.1 `cache lookup failed` 不是一个统一错误
源码中有大量：
```text
elog(ERROR, "cache lookup failed for ...")
```
这类错误一般表示：
```text
caller 手里已经有一个 OID 或 key；
按它所在路径的语义，这个对象应该存在；
但 syscache / relcache / helper 没有找到对应 catalog row。
```
它不是 catcache 自己抛出的通用错误。
很多时候是 caller 在 `SearchSysCache1()` 后检查 `HeapTupleIsValid()`。
诊断步骤：
```bash
rg -n "cache lookup failed for relation %u" /home/nail/postgres/src/backend
rg -n "cache lookup failed for type %u" /home/nail/postgres/src/backend
rg -n "cache lookup failed for function %u" /home/nail/postgres/src/backend
```
不要只搜 `SearchSysCache1`。
应该先搜完整错误文本。
错误文本告诉你 caller 的语义上下文。
例如 `lsyscache.c` 的 helper 多半是在把 OID 转成属性、类型、操作符属性。
`relcache.c` 的错误可能发生在 relation descriptor build / rebuild。
`fmgr.c` 的错误常和函数调用信息初始化有关。
`objectaddress.c` 的错误常和 DDL、依赖或对象地址还原有关。
然后再看 caller 调用了哪个 cache ID。
例如：
```text
SearchSysCache1(RELOID, ObjectIdGetDatum(relid))
SearchSysCache1(TYPEOID, ObjectIdGetDatum(typeOid))
SearchSysCache1(PROCOID, ObjectIdGetDatum(funcid))
SearchSysCache2(ATTNUM, ObjectIdGetDatum(relid), Int16GetDatum(attnum))
```
cache ID 决定 `syscache.c` 中的 `SysCache[cacheId]`。
`SysCache[cacheId]` 决定 catcache 的 catalog relation、unique index、key columns。
这个映射来自 generated `catalog/syscache_info.h`，源头是 `include/catalog/pg_*.h` 中的 `MAKE_SYSCACHE(...)`。
### 4.2 stale metadata 要先问“哪一层旧”
stale metadata 不是单点状态。
需要先判断旧的是哪个对象。
| 旧状态 | 常见现象 | 源码入口 |
| --- | --- | --- |
| `CatCTup` | helper 读到旧 catalog tuple、negative entry 未刷新 | `CatCacheInvalidate()`、`SearchCatCacheInternal()` |
| `RelationData` | 列布局、index list、trigger、partition descriptor 旧 | `RelationCacheInvalidateEntry()`、`RelationFlushRelation()` |
| `CachedPlanSource` | prepared statement 仍以旧 query tree / result desc 判断 | `PlanCacheRelCallback()`、`RevalidateCachedQuery()` |
| `CachedPlan` | generic plan 仍被引用或需要释放重建 | `CachedPlanIsSimplyValid()`、`ReleaseGenericPlan()` |
| `TypeCacheEntry` | equality/hash/range/domain/composite 能力判断旧 | `TypeCacheTypCallback()`、`TypeCacheRelCallback()` |
| catalog snapshot | 当前命令或事务边界上的 catalog scan 可见性旧 | `InvalidateCatalogSnapshot()`、`CommandEndInvalidationMessages()` |
正确问题不是：
```text
为什么缓存没刷新？
```
而是：
```text
哪个 backend 的哪个本地对象，在哪个安全边界之前仍可被合法使用？
```
例如 relcache entry 收到 invalidation 后：
- `rd_refcnt == 0` 时可以清掉。
- `rd_refcnt > 0` 时可能 rebuild in place 或标 invalid。
- nailed relation 有额外约束。
- `RelationBuildDesc()` 正在构造时，in-progress list 需要触发 retry。
这和 syscache entry 很像，但字段不同。
catcache 用 `CatCTup.dead` 和 `refcount`。
relcache 用 `rd_isvalid`、`rd_refcnt`、`rd_isnailed` 和子结构上下文。
plan cache 用 `is_valid` 和 plan refcount。
typcache 用 flags。
### 4.3 invalidation 日志要区分 sender、receiver 和 redo
看到 invalidation 相关日志时，先判断日志在哪一层。
| 日志或 wait event | 源码入口 | 语义 |
| --- | --- | --- |
| `cache state reset` | `ReceiveSharedInvalidMessages()` | `SIGetDataEntries()` 返回 reset，receiver 要 reset 本地 cache。 |
| `replaying commit with N messages` | `ProcessCommittedInvalidationMessages()` | recovery / standby 重放 commit invalidation。 |
| `removing relcache init files for database ...` | `ProcessCommittedInvalidationMessages()` | recovery 路径中 relcache init file invalidation。 |
| `SInvalRead` | `SIGetDataEntries()` | backend 正在读 shared invalidation queue。 |
| `SInvalWrite` | `SIInsertDataEntries()` | backend 正在写 shared invalidation queue。 |
`cache state reset` 不是某个 catalog row 的具体消息。
它表示 receiver 落后到需要整体 reset。
之后会调用 reset function。
在 invalidation dispatcher 里，reset function 是 `InvalidateSystemCaches()`。
它会走：
```text
InvalidateSystemCaches()
  -> InvalidateSystemCachesExtended(false)
     -> ResetCatalogCaches()
     -> RelationCacheInvalidate(false)
     -> callbacks with InvalidOid / hashvalue 0
```
这是一种 correctness fallback。
它牺牲局部性，换取不会漏掉过期事实。
### 4.4 `debug_discard_caches` 是压力工具，不是生产重放器
`debug_discard_caches` 定义在 `guc_parameters.dat`。
它是 `PGC_SUSET`，属于 `DEVELOPER_OPTIONS`。
但它只有在编译启用 `DISCARD_CACHES_ENABLED` 时才真正可用。
`src/include/utils/inval.h` 中：
```text
DISCARD_CACHES_ENABLED off -> MAX_DEBUG_DISCARD_CACHES = 0
DISCARD_CACHES_ENABLED on  -> MAX_DEBUG_DISCARD_CACHES = 5
```
assert build 默认倾向启用相关调试能力。
这个 GUC 的诊断意义是：
```text
在可以接收 invalidation 的地方主动 aggressive flush；
暴露代码是否错误依赖长寿命 cache pointer、旧 tupledesc、旧 relcache 子结构或未复制数据。
```
它不能说明生产中一定会按同样时机 flush。
它也不能替代 relation lock、事务边界、logical decoding 语义分析。
当 `debug_discard_caches > 0` 时，`relcache.c` 的 `RelationBuildDesc()` 会更积极使用临时 workspace context，避免反复 build 泄漏太多临时内存。
当 `debug_discard_caches > 2` 时，relcache 的 opclass cache 也会更激进地失效。
这个工具很适合暴露：
- caller 保存 `HeapTuple` 指针跨 release。
- caller 保存 relcache 内部子结构指针但没有保持 relation open。
- plan path 没有正确处理 callback invalidation。
- typcache caller 假设 flags 永远不变。
它不适合直接证明：
- 某个生产错误一定由 SI queue overflow 引起。
- 某个 catalog row 一定被错误删除。
- 某个 DDL 一定缺少 lock。
## 5. 关键结构与状态边界
### 5.1 `CatCache` 与 `CatCTup`
`CatCache` 是 backend-local。
它不是 shared memory。
`SysCache[cacheId]` 指向对应的 `CatCache`。
诊断时最有用的 `CatCTup` 字段组合是：
| 字段 | 诊断语义 |
| --- | --- |
| `hash_value` | invalidation message 只带 hash value，不带 tuple 指针。 |
| `keys[]` | cache entry 的 key copy，用于比较和定位。 |
| `tuple` | 返回给 caller 的 cache copy。 |
| `negative` | 表示这个 key 曾经查不到，后续 insert 需要 invalidation 清掉它。 |
| `dead` | 不应再返回给新的 lookup，但旧 holder 仍可读到 release。 |
| `refcount` | 本 backend 内有多少 active holder。 |
| `c_list` | list cache entry 与 tuple entry 的交织 lifetime。 |
核心组合语义：
```text
dead = true
refcount > 0
tuple memory still allocated
future lookup skips it
current holder can still finish
```
如果 gdb 里看到 `dead = true`，不要立刻判断内存错误。
它可能正是 invalidation 安全工作的证据。
### 5.2 `SysCache[cacheId]`
`syscache.c` 是包装层。
`SearchSysCache1()` 只做：
```text
Assert(cacheId valid)
Assert(key count matches)
return SearchCatCache1(SysCache[cacheId], key1)
```
所以 `SearchSysCache1()` 不是根因入口。
它是从 caller 进入 catcache 的桥。
但是它非常适合打断点。
当不知道哪个 helper 正在查某个 OID 时，可以在 `SearchSysCache1()` 或 `SearchSysCache2()` 上加条件断点。
示例：
```gdb
break SearchSysCache1 if cacheId == RELOID
break SearchSysCache1 if cacheId == TYPEOID
break SearchSysCache2 if cacheId == ATTNUM
```
实际枚举值来自 `catalog/syscache_ids.h`。
如果 gdb 无法识别 enum 名，可以先打印：
```gdb
p cacheId
bt
```
然后回到 caller 读具体 `SearchSysCache*()` 参数。
### 5.3 `RelationData`
`RelationData` 也是 backend-local。
它活在 `CacheMemoryContext` 及其子 context 中。
诊断 relcache stale 时，优先看这些字段：
| 字段 | 诊断语义 |
| --- | --- |
| `rd_id` | relation OID，和错误中的 relid 对齐。 |
| `rd_refcnt` | 当前 backend 有多少 open 引用。 |
| `rd_isvalid` | 语义是否需要 rebuild / reload。 |
| `rd_isnailed` | 是否 nailed，影响是否可删除。 |
| `rd_att` | tuple descriptor，列布局错误首先看这里。 |
| `rd_indexlist` | heap relation 的 index list 是否已 lazy 构造。 |
| `rd_rules` / `trigdesc` / `rd_rsdesc` | rewrite、trigger、RLS 派生状态。 |
| `rd_partkey` / `rd_partdesc` | partition 相关 stale metadata。 |
| `rd_createSubid` / `rd_droppedSubid` | 当前事务创建或 drop 的特殊 cleanup 状态。 |
`rd_refcnt` 不是锁。
其它 backend 看不到它。
它只保证当前 backend 内这个 `Relation *` 不能随便释放。
DDL 并发安全仍要看 heavyweight relation lock。
`rd_isvalid` 不是内存安全判断。
`rd_isvalid = false` 的 relation 仍可能被当前代码安全持有。
问题是后续需要在合适边界 rebuild 或报错。
### 5.4 `SharedInvalidationMessage`
`SharedInvalidationMessage` 是小 payload。
它故意不带新 tuple。
它也不带本地指针。
关键类型：
| 类型 | payload | receiver 动作 |
| --- | --- | --- |
| catcache tuple | cache id、dbId、hashValue | `SysCacheInvalidate()`，再调用 syscache callbacks。 |
| whole catalog | dbId、catId | `CatalogCacheFlushCatalog()`。 |
| relcache | dbId、relId | `RelationCacheInvalidateEntry()` 或 whole reset。 |
| smgr | relfile locator | 关闭 smgr handle。 |
| relmap | dbId | 重新读 relation map。 |
| catalog snapshot | dbId、relId | `InvalidateCatalogSnapshot()`。 |
| relsync | dbId、relid | logical decoding relation sync cache callback。 |
诊断 invalidation 时要记住：
```text
message is a possibility, not a replacement object.
```
它表示“这个 key 可能过期”。
receiver 本地可能没有对应 entry。
receiver 也可能已经先被 reset。
所以 message 不需要精确携带所有状态。
### 5.5 `CachedPlanSource` 与 `CachedPlan`
Plan cache 是 derived cache。
它依赖 relcache、syscache 和环境状态。
`plancache.c` 文件头给了核心语义：
```text
sinval event 只标记 matching CachedPlanSource / CachedPlan invalid；
下一次需要 cached plan 时，再重新 parse analysis / rewrite / planning。
```
诊断字段：
| 字段 | 诊断语义 |
| --- | --- |
| `CachedPlanSource.is_valid` | query tree / rewrite 结果是否仍可复用。 |
| `CachedPlanSource.gplan` | generic plan 指针。 |
| `CachedPlan.is_valid` | generic plan 是否仍可复用。 |
| `CachedPlan.refcount` | 当前是否有执行者持有 plan。 |
| `dependsOnRLS`、`rewriteRoleId`、`search_path` | invalidation 之外的环境失效条件。 |
所以 prepared statement 的 stale 诊断不要只看 relcache。
要看 callback 是否标 invalid，下一次 `GetCachedPlan()` 是否进入 revalidate。
### 5.6 `TypeCacheEntry`
typcache 也是 backend-local derived cache。
它通过 callbacks 接收 relcache/syscache invalidation。
典型状态是 flags。
例如：
```text
TCFLAGS_HAVE_PG_TYPE_DATA
TCFLAGS_OPERATOR_FLAGS
TCFLAGS_CHECKED_DOMAIN_CONSTRAINTS
```
`TypeCacheTypCallback()` 清 `pg_type` 数据相关 flags。
`TypeCacheOpcCallback()` 对 `pg_opclass` invalidation 采取较粗粒度策略，重置 operator 相关 flags。
`TypeCacheRelCallback()` 处理 composite type 的 tuple descriptor。
这解释了为什么同一条 DDL 可能不直接让 SQL 报 relcache 错误，却改变 operator 或 type capability 判断。
## 6. Source Entry Map：从现场 token 到入口函数
本节最实用的表是下面这张。
| 现场 token | 第一个命令 | 第一入口 | 下一层入口 | 需要验证的状态 |
| --- | --- | --- | --- | --- |
| `cache lookup failed for relation %u` | `rg` 完整错误文本 | caller 文件 | `SearchSysCache1(RELOID)` 或 `RelationIdGetRelation()` | catalog row、relation lock、`rd_isvalid` |
| `cache lookup failed for type %u` | `rg` 完整错误文本 | caller 文件 | `SearchSysCache1(TYPEOID)` / `lookup_type_cache()` | `pg_type` row、typcache flags |
| `cache lookup failed for function %u` | `rg` 完整错误文本 | caller 文件 | `SearchSysCache1(PROCOID)` / fmgr path | `pg_proc` row、dependency、extension DDL |
| `cache lookup failed for attribute ...` | `rg` 完整错误文本 | `lsyscache.c` 或 caller | `SearchSysCache2(ATTNUM)` | `pg_attribute` row、attnum、dropped column |
| stale column layout | `bt` + `p rel->rd_att` | `RelationIdGetRelation()` | `RelationBuildDesc()` / `RelationRebuildRelation()` | `rd_att`、`rd_isvalid`、invalidation callback |
| stale prepared statement | `bt` + `p plansource->is_valid` | `GetCachedPlan()` | `PlanCacheRelCallback()` / `RevalidateCachedQuery()` | plan source invalidation、locks、result desc |
| stale type/operator behavior | `bt` + `p typentry->flags` | `lookup_type_cache()` | `TypeCache*Callback()` | flags 是否被清、是否重新填充 |
| `cache state reset` | 搜日志字符串 | `ReceiveSharedInvalidMessages()` | `SIGetDataEntries()` | `ProcState.resetState`、whole cache reset |
| `SInvalRead` / `SInvalWrite` | wait event | `SIGetDataEntries()` / `SIInsertDataEntries()` | `SICleanupQueue()` | queue pressure、slow receiver |
| `replaying commit with N messages` | 搜日志字符串 | `ProcessCommittedInvalidationMessages()` | `SendSharedInvalidMessages()` | recovery / standby invalidation replay |
| `debug_discard_caches` 触发问题 | 查 GUC 是否可用 | `AcceptInvalidationMessages()` | `InvalidateSystemCachesExtended(true)` | caller 是否持有 stale pointer |
这张表的使用方式：
```text
不要从“所有 cache 源码”开始。
先从现场 token 找最窄入口。
然后只沿一条时间线展开。
```
例如错误文本是：
```text
cache lookup failed for opclass 12345
```
第一步不是看 `CatCacheInvalidate()`。
第一步是：
```bash
rg -n "cache lookup failed for opclass %u" /home/nail/postgres/src/backend
```
找到 caller 后，再看它是 planner、index build、typcache 还是 ruleutils。
不同 caller 对“opclass 必须存在”的假设不同。
同一个 OID 缺失，在 DDL 进行中、extension upgrade、catalog corruption、错误依赖、snapshot 边界或 bug 下，解释完全不同。
## 7. 主流程 walkthrough 一：从 `cache lookup failed` 回到 syscache caller
这一节选一个常见现场：
```text
ERROR: cache lookup failed for type 16620
```
诊断目标不是立刻证明 catalog corruption。
目标是找到：
```text
哪个 caller 手里拿到了 type OID 16620；
它为什么认为这个 type 必须存在；
syscache miss 是正常 NULL 还是违背语义；
是否有 stale metadata 或 invalidation timing 参与。
```
### 7.1 搜错误文本
第一步：
```bash
rg -n "cache lookup failed for type %u" /home/nail/postgres/src/backend
```
你会看到多处 caller。
不要急着读所有。
先用日志里的上下文缩小：
- 错误发生在 query parse / rewrite / plan / execute 哪一阶段？
- 错误附近是否有 `STATEMENT:`？
- 是否是 extension 函数、operator、index build、ruleutils、logical decoding 或 statistics？
- OID 是 type、function、relation 还是 attribute？
如果错误堆栈可用，直接 `bt`。
如果没有 core 或 gdb，可以从 `log_min_error_statement` 和 SQL 语境推断 caller。
### 7.2 这个 walkthrough 的状态闭环
这个错误链的闭环是：
```text
错误文本
  -> caller
  -> caller 的 OID 来源
  -> SearchSysCache cache ID
  -> CatCache hit/miss/negative/stale retry
  -> catalog row 当前事实
  -> invalidation 是否应该清掉旧 entry
```
如果无法解释 OID 来源，只盯着 catcache，会漏掉真正问题。
很多 `cache lookup failed` 是上层保存了旧 OID 或依赖关系不完整。
catcache 只是在最后一刻暴露了矛盾。
## 8. 主流程 walkthrough 二：从 stale metadata 回到 relcache / plan cache
这一节选另一个常见现场：
```text
会话 A PREPARE 了一条查询；
会话 B ALTER TABLE；
会话 A 再 EXECUTE 时报错、重规划，或行为与旧列布局相关。
```
这个现场不应该先归因给 syscache。
更可能路径是：
```text
DDL 修改 catalog
  -> relcache invalidation
  -> plan cache callback
  -> CachedPlanSource.is_valid = false
  -> 下一次 GetCachedPlan()
  -> RevalidateCachedQuery()
  -> 重新 parse/rewrite/plan
```
### 8.1 sender 侧：DDL 产生 invalidation
catalog tuple 修改路径最终会注册 invalidation。
主入口：
```text
CacheInvalidateHeapTuple()
  -> CacheInvalidateHeapTupleCommon()
     -> PrepareToInvalidateCacheTuple()
     -> RegisterRelcacheInvalidation()
```
如果 DDL 没直接修改某个被 `CacheInvalidateHeapTupleCommon()` 识别的 tuple，但仍需要 relcache rebuild，会显式调用：
```text
CacheInvalidateRelcache()
CacheInvalidateRelcacheByRelid()
CacheInvalidateRelcacheAll()
```
例如 index 变化会影响 heap relation 的 index list。
这类路径不能只看 `pg_class` update。
要在 caller 处找显式 relcache invalidation。
### 8.2 command boundary：本 backend 先执行本地消息
`inval.c` 文件头强调：
```text
tuple update/delete 在同一 command 内按 visibility 仍可能有效；
不能在 heap_update() 立刻 flush cache；
必须到 CommandCounterIncrement() 后处理当前命令消息。
```
源码入口：
```text
CommandEndInvalidationMessages()
  -> ProcessInvalidationMessages(CurrentCmdInvalidMsgs, LocalExecuteInvalidationMessage)
  -> AppendInvalidationMessages(PriorCmdInvalidMsgs, CurrentCmdInvalidMsgs)
```
本 backend 的 stale metadata 问题，经常发生在“当前 command”与“下一个 command”的边界理解错了。
同一 command 内看到旧 catalog tuple 不一定是 bug。
下一 command 仍看到旧派生状态，才要追 invalidation / rebuild。
### 8.3 receiver 侧：relcache message 本地执行
receiver 处理 relcache message：
```text
LocalExecuteInvalidationMessage()
  -> msg->id == SHAREDINVALRELCACHE_ID
  -> RelationCacheInvalidateEntry(relid)
  -> relcache callbacks
```
`RelationCacheInvalidateEntry()`：
```text
RelationIdCacheLookup(relid, relation)
if relation:
  RelationFlushRelation(relation)
else:
  mark in-progress RelationBuildDesc invalidated
```
这一步仍不等于 plan 已经重建。
它只是本地 relcache 状态被清理、标记或重建。
### 8.4 plan cache callback
`inval.c` 的 relcache callback registry 会调用 `plancache.c` 注册的 callback。
入口：
```text
PlanCacheRelCallback(Datum arg, Oid relid)
```
它遍历 saved plan list。
匹配 relid 的 plan source 会被标：
```text
plansource->is_valid = false
plansource->gplan->is_valid = false
```
这解释了一个常见误区：
```text
invalidation 不会立刻重新 planning。
它只让下一次需要 plan 时无法直接复用旧 plan。
```
### 8.5 下一次执行：revalidate 与 race check
下一次 `GetCachedPlan()` 会进入 revalidate。
关键路径：
```text
RevalidateCachedQuery()
  -> 如果 plansource->is_valid，先 AcquirePlannerLocks()
  -> 再检查是否有 invalidation callback 已经把它标 invalid
  -> 如果失效，释放无用 locks，重新 parse analysis / rewrite
```
这个 double check 是防 race 的。
它处理：
```text
invalidation message 在拿锁前到达；
拿锁后才发现 query tree 已经被 callback 标 invalid。
```
所以 stale plan 诊断要看两件事：
- callback 有没有把 plan source 标 invalid。
- revalidate 有没有在拿锁后再次检查。
### 8.6 这个 walkthrough 的状态闭环
完整闭环：
```text
ALTER TABLE
  -> catalog update / explicit relcache invalidation
  -> CurrentCmdInvalidMsgs
  -> CommandEndInvalidationMessages()
  -> commit 后 SendSharedInvalidMessages()
  -> receiver AcceptInvalidationMessages()
  -> RelationCacheInvalidateEntry()
  -> PlanCacheRelCallback()
  -> next GetCachedPlan()
  -> RevalidateCachedQuery()
  -> 新语义或报错
```
如果中间任意一步没有发生，要按入口往回查。
不要把最终报错直接归因给 relcache。
prepared statement 的报错可能来自重新 parse analysis 后旧列不存在。
这恰好说明 invalidation 工作了。
## 9. 主流程 walkthrough 三：从 invalidation 日志回到 shared queue
这一节选日志现场：
```text
DEBUG4: cache state reset
```
这个日志来自 `ReceiveSharedInvalidMessages()`。
它表示：
```text
SIGetDataEntries() 返回 -1；
当前 backend 被要求 reset cache state；
receiver 不再逐条处理旧消息。
```
### 9.1 sender 写入 shared queue
提交路径会把消息送入 shared queue。
主入口：
```text
AtEOXact_Inval(true)
  -> SendSharedInvalidMessages()
     -> SIInsertDataEntries()
```
`SIInsertDataEntries()` 每次最多处理 `WRITE_QUANTUM` 批。
它持有 `SInvalWriteLock`，把 message 放进 `SISeg.buffer`。
然后把所有 active receiver 的 `hasMessages` 设为 true。
如果 buffer 太满，会调用 `SICleanupQueue()`。
### 9.2 receiver 读取 shared queue
receiver：
```text
AcceptInvalidationMessages()
  -> ReceiveSharedInvalidMessages(LocalExecuteInvalidationMessage, InvalidateSystemCaches)
     -> SIGetDataEntries()
```
返回值语义：
```text
0    no message
n>0  取到 n 条 message
-1   reset message
```
如果返回 `-1`，`ReceiveSharedInvalidMessages()` 打：
```text
cache state reset
```
然后调用 reset function。
### 9.3 queue overflow / slow receiver
`sinvaladt.c` 的 `SISeg` 包含：
```text
minMsgNum
maxMsgNum
buffer[MAXNUMMESSAGES]
procState[]
```
每个 receiver 的 `ProcState` 有：
```text
nextMsgNum
resetState
signaled
hasMessages
sendOnly
```
如果某个 backend 太久不读，队列不能无限保留旧消息。
`SICleanupQueue()` 会：
- 计算所有 active backend 的最小 `nextMsgNum`。
- 找出落后需要 signal 的 backend。
- 必要时把过慢 backend 标成 reset。
- 通过 `PROCSIG_CATCHUP_INTERRUPT` 促使 backend 处理 invalidation。
这不是 correctness failure。
它是从精确逐条消息退化到 whole-cache reset。
### 9.4 wait event 如何解释
`SInvalRead` 表示在读 invalidation queue。
`SInvalWrite` 表示在写 invalidation queue。
它们能说明 queue 层面有活动或等待。
但它们不能告诉你：
- 哪个 catalog row 被改。
- 哪个 relid 被 invalidated。
- 哪个 backend 的 plan 被标 invalid。
- 哪个 SQL 是根因。
如果要把 wait event 回到 SQL，需要结合：
- 当前 backend 的 `pg_stat_activity.query`。
- DDL 日志。
- `log_statement = 'ddl'` 或审计日志。
- gdb 断点中的 message payload。
- 对 `SharedInvalidationMessage` 的打印。
## 10. 生命周期 / ownership / cleanup
### 10.1 syscache / catcache entry
谁创建：
```text
SearchCatCacheMiss()
  -> CatalogCacheCreateEntry()
```
谁持有：
```text
SearchCatCacheInternal() 或 SearchCatCacheMiss()
  -> ct->refcount++
  -> ResourceOwnerRememberCatCacheRef()
```
谁释放：
```text
ReleaseSysCache()
  -> ReleaseCatCache()
  -> refcount--
  -> if dead and refcount == 0: remove
```
ERROR / abort 兜底：
```text
ResourceOwner release callback
  -> release catcache refs
```
语义过期：
```text
LocalExecuteInvalidationMessage()
  -> SysCacheInvalidate()
  -> CatCacheInvalidate()
  -> mark dead or remove
```
诊断重点：
```text
ResourceOwner 负责归还 ref；
MemoryContext 负责 entry memory；
invalidation 负责语义过期；
三者缺一不可，但互不替代。
```
### 10.2 relcache entry
谁创建：
```text
RelationIdGetRelation()
  -> RelationBuildDesc()
```
谁持有：
```text
table_open() / relation_open()
  -> RelationIncrementReferenceCount()
  -> relation close path decrement
```
谁释放：
```text
RelationFlushRelation()
RelationClearRelation()
AtEOXact_RelationCache()
```
ERROR / abort 兜底：
```text
ResourceOwner / transaction cleanup 释放 relation references；
relcache 自己在 EOX cleanup 处理 pending tupledesc、new relfilenumber、dropped state。
```
语义过期：
```text
RelationCacheInvalidateEntry(relid)
  -> RelationFlushRelation(relation)
```
in-progress build：
```text
RelationBuildDesc() pushes relid into in_progress_list；
RelationCacheInvalidateEntry() 找不到 relation 时也会标记 matching in-progress build invalidated；
build 结束前发现 invalidated 就 retry。
```
## 11. 正确性机制层次
cache diagnostics 必须分层看 correctness。
| 层次 | 机制 | 保证什么 | 不保证什么 |
| --- | --- | --- | --- |
| MVCC / command ID | catalog tuple 在 command boundary 前后的可见性 | 当前命令内旧 tuple 何时仍有效 | 本地 cache 指针何时释放 |
| relation lock | DDL/DML 逻辑并发顺序 | schema change 与使用者互斥或等待 | cache entry 自动更新 |
| catcache refcount | `HeapTuple` cache copy 不悬挂 | active holder 可安全读到 release | tuple 语义仍最新 |
| relcache `rd_refcnt` | `Relation *` 不被当前 backend 提前释放 | open relation 指针安全 | DDL 被阻塞、元数据新鲜 |
| invalidation | 过期事实传播 | receiver 最终不继续相信旧 entry | 立即重建、立即阻塞 caller |
| callback | 派生 cache 被标 invalid | plan / typcache 不再盲目复用 | 新对象已构造完成 |
| ResourceOwner | ERROR-safe release | skipped release path 可兜底 | 语义 freshness |
| MemoryContext | 批量内存 lifetime | entry memory 可长期存在 | active ref 自动归还 |
| WAL / redo | standby / recovery 获得 invalidation 事实 | crash/recovery 后语义传播 | 前台延迟低 |
这张表的用法：
```text
看到 stale metadata 时，先问缺的是哪一层保证。
不要把 invalidation 当 lock。
不要把 refcount 当 freshness。
不要把 MemoryContext 当 ResourceOwner。
```
例如：
```text
rd_refcnt > 0
```
只说明当前 backend 有 open reference。
它不能说明 DDL 不会提交。
也不能说明 plan cache 仍 valid。
再例如：
```text
CatCTup.dead = true
```
说明这个 entry 不该被新的 lookup 返回。
但已有 holder 仍能安全读到 release。
## 12. 错误路径 / 异常路径 / fallback
### 12.1 active catcache entry 被 invalidated
路径：
```text
LocalExecuteInvalidationMessage()
  -> SysCacheInvalidate()
  -> CatCacheInvalidate()
```
如果 matching `CatCTup.refcount > 0`：
```text
ct->dead = true
```
不会立即释放。
未来 lookup 跳过它。
当前 holder release 后再删除。
诊断信号：
- `dead = true`。
- `refcount > 0`。
- 当前 holder 继续完成。
- 后续 lookup 走 miss / rebuild。
### 12.2 negative cache entry 需要 invalidation
如果某个 key 查不到，会产生 negative entry。
后续 insert 对应 catalog tuple 时，必须清掉它。
否则 backend 会继续相信“不存在”。
`PrepareToInvalidateCacheTuple()` 明确说明：
```text
即使当前 backend 没有 matching entry，也要记录 invalidation；
因为到 command end 之前可能出现 matching negative entry；
其它 backend 也可能有。
```
诊断 negative entry 时，不要只查 positive tuple。
要看 insert 是否产生了 catcache invalidation message。
### 12.3 catcache entry 构造中被 invalidated
`CatalogCacheCreateEntry()` 处理 TOAST flatten 时可能触发 `AcceptInvalidationMessages()`。
如果正构造的 entry 被 invalidated，in-progress stack 会标 `dead`。
caller 要 retry。
路径：
```text
SearchCatCacheMiss()
  -> CatalogCacheCreateEntry()
     -> return NULL if stale
  -> stale = true
  -> restart scan
```
这类 retry 很少见。
`debug_discard_caches` 和 assert build 随机失败路径可以帮助覆盖它。
### 12.4 RelationBuildDesc 构造中被 invalidated
`RelationBuildDesc()` 注册 in-progress relid。
如果收到 relcache invalidation：
```text
RelationCacheInvalidateEntry()
  -> relation not in cache
  -> mark matching in_progress_list invalidated
```
build 完成前发现 invalidated，会 retry。
这对 `CREATE INDEX CONCURRENTLY` 等不能错过的变化很关键。
诊断 stale relcache build 时，要看：
- `in_progress_list` 是否被标 invalidated。
- build 是否 retry。
- retry 期间是否递归处理 invalidation。
## 13. 成本、资源与跨模块传播
### 13.1 hot path 成本
正常 hit path 很短：
```text
hashValue
bucket scan
key compare
refcount++
ResourceOwner remember
```
成本随这些变量增长：
- bucket 中冲突 entry 数。
- 当前 backend cache 大小。
- caller 是否忘记及时 release，导致 dead entry retention。
- key 类型是否 by-value / by-reference。
- `debug_discard_caches` 是否强制频繁 miss。
### 13.2 miss path 成本
miss path 会打开 catalog relation 并走 sys scan。
它可能触发：
- relcache lookup。
- index scan。
- TOAST flatten。
- memory allocation in `CacheMemoryContext`。
- recursive invalidation processing。
在 `debug_discard_caches` 打开时，miss / rebuild 频率会显著上升。
性能现场不要把这种调试模式下的成本外推到生产。
### 13.3 invalidation fan-out 成本
一个 DDL 的 invalidation 传播可能触发：
- catcache entry flush。
- relcache entry flush / rebuild。
- plan cache saved plan list scan。
- typcache hash scan或 domain linked list scan。
- smgr handle close。
- relcache init file invalidation。
fan-out 成本随这些变量增长：
- active backend 数。
- prepared statement / saved plan 数。
- backend-local cache entry 数。
- DDL 频率。
- 分区表的 relation / partition descriptor 数。
- extension 或 migration 是否批量修改 catalog。
### 13.4 shared queue 成本
`SISeg.buffer` 是固定大小 ring。
sender 写入要持有 `SInvalWriteLock`。
receiver 读取要持有 `SInvalReadLock`。
慢 receiver 会拖住 `minMsgNum` 推进。
如果拖太久，fallback 是 reset。
成本不是简单“消息越多越慢”。
真正要看：
```text
消息产生速率
active receiver 数
receiver 接收时机
是否频繁触发 cleanup / catchup / reset
```
## 14. 观测与诊断入口
### 14.1 能直接看到什么
SQL 能直接看到 catalog 当前状态：
```sql
SELECT oid, relname, relnamespace::regnamespace, relkind
FROM pg_class
WHERE oid = 16442;
SELECT oid, typname, typnamespace::regnamespace, typtype, typrelid
FROM pg_type
WHERE oid = 16620;
SELECT oid, proname, pronamespace::regnamespace, proargtypes
FROM pg_proc
WHERE oid = 12345;
```
SQL 能看到 lock 和 wait：
```sql
SELECT pid, wait_event_type, wait_event, state, query
FROM pg_stat_activity
WHERE wait_event IN ('SInvalRead', 'SInvalWrite');
SELECT locktype, relation::regclass, mode, granted, pid
FROM pg_locks
WHERE relation IS NOT NULL;
```
SQL 能看到 memory context 大小：
```sql
SELECT name, total_bytes, used_bytes
FROM pg_backend_memory_contexts
WHERE name LIKE '%Cache%';
```
但 SQL 看不到：
- 某个 `CatCTup.dead`。
- 某个 `RelationData.rd_isvalid`。
- 某个 `CachedPlanSource.is_valid`。
- 某条 SI message 的 hash value。
- receiver 是否刚从逐条 message fallback 到 reset。
这些需要日志、gdb 或源码 instrumentation。
### 14.2 日志入口
有用设置：
```text
log_min_messages = debug4
log_min_error_statement = error
log_statement = 'ddl'
```
`DEBUG4` 下可以看到：
- `cache state reset`。
- recovery replay invalidation 相关日志。
- 某些 relcache init file invalidation 日志。
注意：
```text
PostgreSQL 默认不会记录每一条 SharedInvalidationMessage。
```
如果需要逐条 message，需要 gdb 或临时源码日志。
临时日志应加在：
- `RegisterRelcacheInvalidation()`。
- `AddCatcacheInvalidationMessage()`。
- `LocalExecuteInvalidationMessage()`。
- `SIInsertDataEntries()`。
- `SIGetDataEntries()`。
## 15. 课堂实验
### 实验 1：从 syscache lookup 到 caller
目标：证明 `cache lookup failed` 的语义在 caller，而不是在 `SearchSysCache1()`。
准备一个 debug build。
在 gdb 中：
```gdb
break SearchSysCache1
commands
  silent
  bt 6
  p cacheId
  p key1
  continue
end
```
执行几条会触发 metadata lookup 的 SQL：
```sql
SELECT 'pg_class'::regclass;
SELECT format_type('int4'::regtype::oid, NULL);
SELECT proname FROM pg_proc WHERE oid = 'now'::regproc;
```
观察：
- 同样是 `SearchSysCache1()`，caller 分布在不同 helper。
- 有些 caller 只是取标量并 release。
- 有些 caller 如果 NULL 会 ERROR。
- `SearchSysCache1()` 自己不决定 ERROR。
扩展练习：
```bash
rg -n "cache lookup failed for .*%u" /home/nail/postgres/src/backend/utils /home/nail/postgres/src/backend/catalog
```
把 5 个错误文本映射到 caller 语义。
### 实验 2：DDL 到 plan cache invalidation
目标：观察 `ALTER TABLE` 如何让 prepared statement 的 plan source 失效。
会话 A：
```sql
CREATE TABLE cache_diag_t(a int);
PREPARE s AS SELECT * FROM cache_diag_t;
EXECUTE s;
```
会话 B：
```sql
ALTER TABLE cache_diag_t ADD COLUMN b text;
```
会话 A 再执行：
```sql
EXECUTE s;
```
gdb 断点：
```gdb
break LocalExecuteInvalidationMessage
break RelationCacheInvalidateEntry
break PlanCacheRelCallback
break RevalidateCachedQuery
```
观察：
- 会话 B 产生 relcache invalidation。
- 会话 A 接收后执行本地 relcache invalidation。
- `PlanCacheRelCallback()` 把 matching plan source 标 invalid。
- 下一次 `EXECUTE` 进入 revalidate。
讨论：
```text
如果 EXECUTE 报 result type 改变相关错误，这是 invalidation 工作后的重分析结果；
不是必然说明 invalidation 失败。
```
### 实验 3：观察 command boundary 的本地 invalidation
目标：证明 catalog update 不是在 `heap_update()` 立刻 flush cache，而是在 command boundary 处理。
断点：
```gdb
break CacheInvalidateHeapTupleCommon
break CommandEndInvalidationMessages
break LocalExecuteInvalidationMessage
```
执行：
```sql
CREATE TABLE cache_diag_cci(a int);
ALTER TABLE cache_diag_cci ADD COLUMN b int;
SELECT attname
FROM pg_attribute
WHERE attrelid = 'cache_diag_cci'::regclass
ORDER BY attnum;
```
观察：
- DDL 修改 catalog 时进入 `CacheInvalidateHeapTupleCommon()`。
- `CommandEndInvalidationMessages()` 在 CCI 后本地处理当前命令消息。
- 后续 SQL 能看到新 catalog 状态。
这个实验对应 `inval.c` 文件头的 visibility 规则。
## 16. 常见误区
- 把 `cache lookup failed` 当成 catcache bug；它首先是 caller 的“已有 OID 必须存在”假设被 lookup 否定。
- 看到当前 catalog 查不到 OID 就断定 corruption；还要检查 stale plan、extension upgrade、临时对象、standby replay、依赖和并发 DDL。
- 把 invalidation 当成 lock；它只传播过期事实，互斥仍要看 relation lock、tuple lock 和事务可见性。
- 把 `refcount > 0` 或 `rd_refcnt > 0` 当成 freshness；它们只保证当前 backend 的指针安全。
- 认为 plan cache invalidation 会立刻 replan；通常只是标 invalid，下一次使用才 revalidate / rebuild。
- 认为 `debug_discard_caches` 能重放生产竞态；它适合暴露 stale pointer 假设，不等于真实时序。
- 把 `cache state reset` 当成 correctness 丢失；reset 是 whole-cache fallback。
- 在 `elog.c` 里反查 relcache；错误诊断应从 caller 和 `ErrorData` 回去，避免错误层反向依赖 cache。
## 17. 讨论题
1. 为什么错误文本 caller 比 `SearchSysCache1()` 更接近 `cache lookup failed` 的语义根？
2. `CatCTup.dead = true` 且 `refcount = 1` 时，当前 holder 和未来 lookup 分别允许什么？
3. `RelationData.rd_refcnt`、`rd_isvalid`、relation lock 三者各自保证什么？
4. prepared statement 在 DDL 后报错，为什么可能说明 invalidation 正常工作？
5. shared invalidation queue reset 为什么是 fallback，而不是消息丢失后的 silent corruption？
6. 只看到 `SInvalRead` wait event 时，还缺哪些证据才能定位到具体 DDL？
7. `debug_discard_caches` 能暴露哪些假设？哪些结论仍然不能外推到生产？
8. typcache 为什么接受部分粗粒度 invalidation？
## 18. 本节小结
本节把 cache 诊断压成一条链：
```text
runtime 现象 -> 现场 token -> cache 状态 -> 源码入口 -> 验证
```
`cache lookup failed` 要先找 caller；syscache / catcache 只返回 lookup 事实。
stale metadata 要先判断旧的是 `CatCTup`、`RelationData`、`CachedPlanSource`、`CachedPlan`、`TypeCacheEntry` 还是 catalog snapshot。
`dead`、`refcount`、`rd_isvalid`、`rd_refcnt`、`is_valid`、flags 都不能单独代表完整语义。
shared invalidation 只传播过期事实；receiver 在本地 cleanup、mark invalid、rebuild 或 whole-cache reset。
正确性来自 MVCC / command boundary、relation lock、ResourceOwner、MemoryContext、invalidation callback 和 WAL / redo 的组合。
最短入口是：错误文本找 caller，OID 找 catalog 和 cache ID，stale plan 找 callback 和 revalidate，invalidation log 找 receiver / queue / reset。
最后必须回到 SQL、日志、gdb 断点、message payload 或 `debug_discard_caches` 做验证；timing、workload、版本、extension 和 standby/recovery 相关判断要保留推断边界。
