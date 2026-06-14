# PostgreSQL syscache / catcache lookup lifetime
## 课程定位
前置知识：已经理解 `MemoryContext` 管 backend-local 内存生命周期，`ResourceOwner` 管 buffer pin、lock、snapshot、catcache ref 这类外部资源，且已经看过 heavyweight lock 和 shared invalidation 在事务边界中的位置。
本节唯一主问题：
```text
syscache lookup 如何从 catalog tuple 构造 cache entry，refcount 与 memory context 如何避免悬挂 tuple？
```
核心矛盾：catalog 元数据 lookup 是 parser、planner、executor 和 DDL 的高频热路径，不能每次都扫描系统表；但 catalog tuple 会被 DDL、inplace update、VACUUM FULL、abort 和 shared invalidation 改变语义，调用者拿到的又是一个普通 `HeapTuple` 指针。
一句话运行模型：
```text
SearchSysCache() 只是 syscache 包装层；
SearchCatCache*() 在 backend-local CatCache hash 中找 CatCTup；
miss 时扫描 catalog 并把 tuple copy 到 CacheMemoryContext；
返回前增加 CatCTup.refcount 并登记到 CurrentResourceOwner；
invalidation 只把仍被持有的 entry 标成 dead，等 ReleaseSysCache() 或 ResourceOwner cleanup 后再物理删除。
```
学完后应能判断：
- `SearchSysCache()` 返回的 tuple 为什么不能 `pfree()`、不能修改、不能跨 release 使用。
- `CacheMemoryContext` 解决的是 cache entry 内存生命周期，不解决正在使用的引用归还。
- `refcount` 解决的是物理悬挂指针，不等于保证 catalog 语义仍然最新。
- `CatCacheInvalidate()` 为什么可以先 `dead = true`，而不是立刻释放所有匹配 tuple。
- 正常路径、ERROR 路径和 transaction abort 分别由谁降低 catcache refcount。
- 哪些现象可以从 `pg_backend_memory_contexts`、`debug_discard_caches`、gdb 和日志看到，哪些只能推断。
本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e9`。
## 1. 本节在总主线中的位置
前面几组基础设施课已经建立三条边界。
第一条是 `MemoryContext`：
```text
palloc chunk 属于当前 backend；
context reset/delete 批量回收内存；
普通指针不能跨 backend 传递。
```
第二条是 `ResourceOwner`：
```text
外部资源需要 owner 账本；
ERROR longjmp 后由 owner bulk release；
正常路径应该显式 forget/release。
```
第三条是 transaction cleanup 和 invalidation ordering：
```text
catalog change 需要在 command / transaction boundary 传播；
等待锁的 backend 被唤醒前，应该能看到必要 invalidation；
但已经被当前 backend 持有的本地引用不能被别人直接释放。
```
syscache / catcache 把这三条边界放到同一条热路径上。
典型调用看起来很普通：
```text
tuple = SearchSysCache1(RELOID, ObjectIdGetDatum(relid));
if (!HeapTupleIsValid(tuple))
    elog(ERROR, ...);
form = (Form_pg_class) GETSTRUCT(tuple);
... use form fields ...
ReleaseSysCache(tuple);
```
这里隐藏了一个危险点。
`tuple` 不是 heap page 上的 pinned tuple。
它也不是调用者刚刚 `palloc()` 出来的私有副本。
它是 catcache entry 内部的 `HeapTupleData`。
它的 `t_data` 指向 `CacheMemoryContext` 中的 cache entry payload。
所以本节不把 syscache 当成“查系统表的方便 API”来讲。
本节只追一个对象：
```text
一个 catalog row 被扫描到
  -> 复制成 CatCTup
  -> 作为 HeapTuple 返回给调用者
  -> 被 invalidation 标记 dead
  -> 等所有 ref 归还后删除
```
这个对象的生命周期就是本节主线。
后续 relcache、typcache、plancache 都会依赖这条基础。
relcache 会把多个 catalog row 组合成 `Relation`。
typcache 会把 type、opclass、opfamily 信息组合成更高层判断。
shared invalidation 课程会单独展开消息队列和 delivery。
这里先建立最小不变量：
```text
cache tuple 指针的内存安全，由 CacheMemoryContext + refcount + ResourceOwner 共同保证；
cache tuple 的语义新鲜度，由 command boundary + invalidation + relookup 共同推进。
```
## 2. 核心矛盾与运行模型
本节 tension 可以压缩成一句话：
```text
catalog lookup 要像读本地 hash 一样便宜；
但返回给调用者的 tuple 又必须在 DDL invalidation 和 ERROR longjmp 下不悬挂。
```
如果每次 lookup 都扫描 catalog：
- parser 解析类型名、函数名、操作符名会频繁打开系统表。
- planner 查询 `pg_proc`、`pg_type`、`pg_operator`、`pg_amop` 的成本会被放大。
- executor 或权限检查中的 metadata lookup 会把 catalog index scan 推到热点。
- 系统表自身 lookup 又可能递归触发 relcache / syscache 初始化。
如果只做一个简单 hash cache：
- DDL 修改 catalog 后，旧 entry 可能继续被返回。
- caller 正在使用的 tuple 可能被 invalidation 释放。
- `ERROR` 可能跳过 `ReleaseSysCache()`。
- negative cache entry 可能在新 tuple 插入后继续说“不存在”。
- by-reference key 或 toasted attribute 可能指向已经不可用的内存。
PostgreSQL 的实际模型分成五层。
| 层次 | 状态 | 解决的问题 |
| --- | --- | --- |
| `syscache.c` | `SysCache[cacheId]` | 把 enum cache ID 映射到一个 `CatCache`。 |
| `catcache.c` | `CatCache` hash buckets | 在当前 backend 内复用 catalog tuple copy。 |
| `CacheMemoryContext` | `CatCache`、`CatCTup`、key copy | 让 cache entry 活过单条语句和事务。 |
| `CatCTup.refcount` | active references | caller 正在使用时不能物理删除 entry。 |
| `ResourceOwner` | catcache reference record | `ERROR`、abort、Portal close 时兜底降 refcount。 |
这五层分别回答不同问题。
`CacheMemoryContext` 回答：
```text
cache entry 的内存为什么不会在当前表达式或当前语句 context reset 时消失？
```
`refcount` 回答：
```text
一个 entry 被 invalidation 命中时，为什么不能直接 pfree？
```
`dead` flag 回答：
```text
这个 entry 为什么不能再被新的 lookup 返回？
```
`ResourceOwner` 回答：
```text
调用者没有走到 ReleaseSysCache() 时，谁负责把 refcount 还回去？
```
`invalidation` 回答：
```text
catalog 更新后，哪些 backend-local cache entry 需要被标记过期？
```
把这些字段混成一个“cache 是否有效”会误导诊断。
一个 `CatCTup` 可以同时满足：
```text
dead = true
refcount = 1
tuple memory still allocated
not visible to future lookup
still safe for current holder to read until ReleaseSysCache()
```
这就是本课要建立的核心 mental model。
## 3. 核心文件分工与阅读顺序
| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/utils/catcache.h` | `CatCache`、`CatCTup`、`CatCList` 字段语义，尤其是 `refcount`、`dead`、`negative`、`tuple`。 |
| 2 | `src/include/utils/syscache.h` | syscache public API：`SearchSysCache*()`、`ReleaseSysCache()`、copy / exists / list helper。 |
| 3 | `src/backend/utils/cache/syscache.c` | `SysCache[cacheId]` 初始化、`SearchSysCache*()` 到 `SearchCatCache*()` 的包装、`SysCacheInvalidate()`。 |
| 4 | `src/backend/utils/cache/catcache.c` | lookup hit/miss、entry creation、refcount、ResourceOwner callback、`CatCacheInvalidate()`。 |
| 5 | `src/backend/utils/cache/lsyscache.c` | 常见元数据 helper 如何隐藏 lookup/release，哪些 helper 返回 scalar，哪些仍返回 tuple。 |
| 6 | `src/backend/utils/cache/inval.c` | catalog tuple 修改如何排队 invalidation，command end 本地处理，commit 后 shared invalidation。 |
| 7 | `src/include/utils/resowner.h` | catcache ref 的 release phase 和 priority。 |
| 8 | `src/backend/utils/resowner/resowner.c` | `ResourceOwnerRelease()` 如何在 after-locks 阶段调用 catcache release callback。 |
| 9 | `src/backend/access/transam/xact.c` | commit/abort/subtransaction cleanup 中 invalidation 和 ResourceOwner release 的相对顺序。 |
| 10 | `src/include/catalog/pg_*.h` | `MAKE_SYSCACHE(...)` 定义每个 syscache 的 catalog、unique index 和初始 bucket 数。 |
| 11 | `src/backend/catalog/genbki.pl` | 生成 `syscache_ids.h` 和 `syscache_info.h`；本地源码树中它们通常不是手写文件。 |
推荐阅读顺序不是从 `syscache.c` 顶部顺着读。
更有效的顺序是：
```text
catcache.h 的 CatCTup
  -> syscache.c 的 SearchSysCache1()
  -> catcache.c 的 SearchCatCacheInternal()
  -> catcache.c 的 SearchCatCacheMiss()
  -> catcache.c 的 CatalogCacheCreateEntry()
  -> catcache.c 的 ReleaseCatCacheWithOwner()
  -> inval.c 的 CacheInvalidateHeapTupleCommon()
  -> catcache.c 的 CatCacheInvalidate()
  -> xact.c 的 AtEOXact_Inval / ResourceOwnerRelease ordering
```
注意一个版本细节。
`syscache.h` include 了生成的 `catalog/syscache_ids.h`。
`syscache.c` include 了生成的 `catalog/syscache_info.h`。
这些 generated headers 来自各 `include/catalog/pg_*.h` 中的 `MAKE_SYSCACHE`。
不要在源码树里找不到 `syscache_info.h` 就误以为 `cacheinfo[]` 是运行时动态生成的。
它是 build-time catalog metadata 的产物。
## 4. 关键数据结构与状态
### 4.1 `CatCache`
`CatCache` 是一个 backend-local cache descriptor。
它不是 shared memory 对象。
每个 backend 有自己的 catcache entry 集合。
关键字段组合如下。
| 字段 | 语义 |
| --- | --- |
| `id` | syscache ID，对应 generated `SysCacheIdentifier`。 |
| `cc_reloid` | 这个 cache 缓存哪个 catalog relation。 |
| `cc_indexoid` | miss 时正常用哪个 unique index scan。 |
| `cc_nkeys` / `cc_keyno[]` | lookup key 数量和对应 catalog attribute number。 |
| `cc_hashfunc[]` / `cc_fastequal[]` | key 类型对应的快速 hash / equality。 |
| `cc_tupdesc` | 从 relcache copy 出来的 tuple descriptor，放在 `CacheMemoryContext`。 |
| `cc_bucket` | 单 tuple lookup hash buckets。 |
| `cc_lbucket` | partial-key list lookup buckets，第一次 list search 才分配。 |
| `cc_relisshared` | invalidation message 中 dbId 选择需要它。 |
| `cc_ntup` / `cc_nlist` | 当前 backend 内该 cache 的 entry / list 数。 |
它回答的是：
```text
给定一组 key，如何在当前 backend 的本地 hash 中找到对应 catalog tuple copy？
```
它不回答：
```text
其他 backend 是否也有这个 tuple？
这个 tuple 是否被某个事务修改？
这个 tuple 对某个 snapshot 是否可见？
```
syscache 的“system”不是 shared cache 的意思。
它是 system catalog cache。
### 4.2 `CatCTup`
`CatCTup` 是本节主角。
公开 API 返回的是 `HeapTuple`，但真实对象是：
```text
CatCTup header
  + embedded HeapTupleData tuple
  + tuple.t_data pointing into payload after CatCTup
```
正向指针是：
```text
return &ct->tuple;
```
释放时反向找回 entry：
```text
ct = (CatCTup *) (((char *) tuple) - offsetof(CatCTup, tuple));
```
关键字段组合如下。
| 字段 | 语义 |
| --- | --- |
| `ct_magic` | debug safety，确认传入的是 catcache entry。 |
| `hash_value` | lookup key 的 hash，用于 bucket 和 invalidation。 |
| `keys[]` | 正 entry 的 by-ref key 指向 tuple payload；negative entry 的 by-ref key 单独分配。 |
| `refcount` | 当前 backend 内 active references。 |
| `dead` | 已被 invalidation 命中，不再给未来 lookup 返回。 |
| `negative` | 表示“不存在这个 key”的 negative cache entry。 |
| `tuple` | 返回给 caller 的 `HeapTupleData`。 |
| `c_list` | 如果 entry 属于某个 `CatCList`，指向该 list。 |
| `my_cache` | 回到 owning `CatCache`。 |
最重要的不变量：
```text
dead entry 可以继续被当前 holder 读取；
refcount 为 0 且 dead 时才能物理删除；
negative entry 不返回给 caller，所以正常 refcount 保持 0；
positive entry 的 tuple payload 和 CatCTup 一起分配。
```
`dead` 不是“内存已经释放”。
`refcount` 也不是“语义仍然最新”。
两者组合才表达 lifetime。
### 4.3 `SysCache[cacheId]`
`syscache.c` 的 `SysCache` 是：
```text
static CatCache *SysCache[SysCacheSize];
```
`InitCatalogCache()` 遍历 generated `cacheinfo[]`：
```text
cacheinfo[cacheId].reloid
cacheinfo[cacheId].indoid
cacheinfo[cacheId].nkeys
cacheinfo[cacheId].key[]
cacheinfo[cacheId].nbuckets
```
然后调用：
```text
InitCatCache(cacheId, reloid, indoid, nkeys, key, nbuckets)
```
`SearchSysCache1()` 到 `SearchSysCache4()` 主要做两件事。
第一，确认 `cacheId` 范围和 `SysCache[cacheId]` 已初始化。
第二，确认指定 variant 的 key 数量和 `cc_nkeys` 一致。
然后转发给对应 `SearchCatCacheN()`。
所以 syscache 不是另一套 cache 算法。
它是：
```text
cache ID + generated metadata + catcache implementation
```
### 4.4 `CacheMemoryContext`
catcache descriptor、bucket array、tuple descriptor copy、entry payload 都放在 `CacheMemoryContext`。
`InitCatCache()` 会在第一次需要时调用 `CreateCacheMemoryContext()`。
这个 context 挂在 `TopMemoryContext` 下。
语义是：
```text
cache entry 可以跨 command、跨 transaction 留在当前 backend 中；
它不随 query context、portal context 或 transaction context reset 自动消失。
```
这正是 cache 的价值。
但这也带来诊断误区。
`CacheMemoryContext` 变大不一定是 leak。
它可能只是这个 backend 查过很多 catalog objects。
也可能是 invalidation 暂时不能释放被 refcount pin 住的 dead entries。
还可能是 list cache 或 relcache / typcache 等其它 cache 共用这个 context。
### 4.5 ResourceOwner 里的 catcache ref
`catcache.c` 定义了两个 `ResourceOwnerDesc`。
一个是 `catcache reference`。
一个是 `catcache list reference`。
它们都在：
```text
RESOURCE_RELEASE_AFTER_LOCKS
```
并使用 priority：
```text
RELEASE_PRIO_CATCACHE_REFS = 100
RELEASE_PRIO_CATCACHE_LIST_REFS = 200
```
这说明 catcache ref cleanup 是当前 backend 内部 refcount 归还。
它不是 catalog change 的跨 backend 可见性边界。
catalog change 的可见性由 `AtEOXact_Inval()` 和 shared invalidation ordering 处理。
## 5. 主流程源码 walkthrough
本节主流程选最常见的单 tuple lookup。
假设调用方要通过 relation OID 查 `pg_class`：
```text
SearchSysCache1(RELOID, ObjectIdGetDatum(relid))
```
真实路径是：
```text
SearchSysCache1()
  -> SearchCatCache1()
     -> SearchCatCacheInternal()
        -> hit: bump refcount + ResourceOwnerRemember
        -> miss: SearchCatCacheMiss()
                 -> systable scan
                 -> CatalogCacheCreateEntry()
                 -> bump refcount + ResourceOwnerRemember
```
### 5.1 syscache wrapper
`SearchSysCache1()` 在 `syscache.c` 中非常薄。
它确认：
```text
cacheId >= 0
cacheId < SysCacheSize
SysCache[cacheId] != NULL
SysCache[cacheId]->cc_nkeys == 1
```
然后：
```text
return SearchCatCache1(SysCache[cacheId], key1);
```
`ReleaseSysCache()` 同样只是：
```text
ReleaseCatCache(tuple);
```
这层的价值不在复杂算法。
它把调用点从“知道 catalog relation、index、key attribute”降到“知道 cache ID”。
### 5.2 第一次使用 cache：lazy initialize
`SearchCatCacheInternal()` 开头调用：
```text
ConditionalCatalogCacheInitializeCache(cache)
```
如果 `cache->cc_tupdesc == NULL`，进入 `CatalogCacheInitializeCache()`。
初始化动作包括：
```text
table_open(cache->cc_reloid, AccessShareLock)
switch to CacheMemoryContext
CreateTupleDescCopyConstr(RelationGetDescr(relation))
copy relation name
copy relisshared
set hash/equality functions for each key type
initialize ScanKeyData for systable scans
cache->cc_tupdesc = copied tupdesc
table_close(relation, AccessShareLock)
```
这里有一个重要设计点。
catcache copy 了 relcache 的 tuple descriptor。
它不长期依赖 relcache entry 里的 descriptor 指针。
这避免某些 catalog cache flush 和 relcache rebuild 交叉时出现 tuple descriptor 悬挂。
### 5.3 hit path：本地 hash 命中
初始化完成后，`SearchCatCacheInternal()` 组装参数数组，计算 hash：
```text
CatalogCacheComputeHashValue(cache, nkeys, v1, v2, v3, v4)
hashIndex = HASH_INDEX(hashValue, cache->cc_nbuckets)
```
然后扫描 bucket。
每个 candidate 依次检查：
```text
ct->dead
ct->hash_value
CatalogCacheCompareTuple(cache, nkeys, ct->keys, arguments)
```
命中后移动到 bucket 头部。
这是一个简单 LRU-ish 局部性优化。
如果 entry 是 negative：
```text
return NULL;
```
如果 entry 是 positive：
```text
ResourceOwnerEnlarge(CurrentResourceOwner);
ct->refcount++;
ResourceOwnerRememberCatCacheRef(CurrentResourceOwner, &ct->tuple);
return &ct->tuple;
```
顺序很关键。
`ResourceOwnerEnlarge()` 先跑。
它可能因为 OOM 抛 `ERROR`。
但此时 refcount 还没增加。
一旦 refcount 增加，后面的 `ResourceOwnerRemember...` 应该已经有空间，不应再因为扩容失败留下“refcount 已加但 owner 没记录”的窗口。
这就是 acquire-before-ERROR 安全在 catcache 上的具体形态。
### 5.4 miss path：打开 catalog 并扫描
bucket 没找到时，进入 `SearchCatCacheMiss()`。
这段被标成 `pg_noinline`，目的是让 hit path 尽量小。
miss path 先打开 catalog：
```text
relation = table_open(cache->cc_reloid, AccessShareLock)
```
然后复制预先初始化好的 scan keys。
每个调用只填入当前 lookup arguments。
扫描入口：
```text
systable_beginscan(relation,
                   cache->cc_indexoid,
                   IndexScanOK(cache),
                   NULL,
                   nkeys,
                   cur_skey)
```
正常情况下用 unique index scan。
但启动早期和少数关键 cache 不能安全使用 indexscan。
例如 relcache 初始化还没完成时，`pg_index` 的某些 lookup 改走 heap scan，避免用还没建好的 index support 递归自己。
所以 `IndexScanOK()` 是启动和递归边界的 fallback。
### 5.5 miss path：positive entry 构造
如果 scan 找到 tuple，调用：
```text
CatalogCacheCreateEntry(cache, ntp, NULL, hashValue, hashIndex)
```
positive entry 的关键动作：
```text
如果有 external toasted fields:
  toast_flatten_tuple(ntp, cache->cc_tupdesc)
MemoryContextAlloc(CacheMemoryContext,
                   MAXALIGN(sizeof(CatCTup)) + tuple length)
ct->tuple.t_len = dtp->t_len
ct->tuple.t_self = dtp->t_self
ct->tuple.t_tableOid = dtp->t_tableOid
ct->tuple.t_data = payload after CatCTup
memcpy tuple header/data into payload
for each key:
  ct->keys[i] = heap_getattr(&ct->tuple, key attno, cc_tupdesc, ...)
```
positive entry 的 by-reference key 指向这个 copied tuple payload。
它不需要再为 key 单独分配内存。
这也是为什么 tuple payload 必须和 entry 一起活在 `CacheMemoryContext`。
构造完成后：
```text
ct->ct_magic = CT_MAGIC
ct->my_cache = cache
ct->refcount = 0
ct->dead = false
ct->negative = false
ct->hash_value = hashValue
dlist_push_head(bucket, &ct->cache_elem)
cache->cc_ntup++
CacheHdr->ch_ntup++
maybe RehashCatCache(cache)
```
注意 `CatalogCacheCreateEntry()` 返回时 refcount 仍是 0。
真正要把它交给 caller 时，`SearchCatCacheMiss()` 才执行：
```text
ResourceOwnerEnlarge(CurrentResourceOwner);
ct->refcount++;
ResourceOwnerRememberCatCacheRef(CurrentResourceOwner, &ct->tuple);
return &ct->tuple;
```
### 5.6 miss path：negative entry 构造
如果 catalog scan 没找到 tuple，catcache 会创建 negative entry。
前提是当前不是 bootstrap processing mode。
bootstrap mode 不建 negative entry，因为 cache invalidation 机制还没完整工作。
negative entry 的用途是缓存“不存在”。
例如反复解析一个不存在的对象名时，不必每次都扫描 catalog。
negative entry 没有真实 tuple payload。
它会：
```text
allocate CatCTup in CacheMemoryContext
CatCacheCopyKeys(...) into ct->keys
ct->negative = true
ct->refcount = 0
insert into bucket
return NULL to caller
```
这里 by-reference keys 必须单独分配。
因为没有 positive tuple payload 可以指向。
当后续插入了 matching catalog tuple，invalidation 仍会按 tuple hash 清掉对应 negative entry。
这是 `PrepareToInvalidateCacheTuple()` 必须对 insert 也登记 catcache inval 的原因。
### 5.7 caller 使用 tuple
返回给 caller 后，常见用法是：
```text
form = (Form_pg_class) GETSTRUCT(tuple);
use fixed-width fields directly
value = SysCacheGetAttr(cacheId, tuple, attrnum, &isnull);
ReleaseSysCache(tuple);
```
`SysCacheGetAttr()` 只是拿到对应 cache 的 `cc_tupdesc`，然后调用 `heap_getattr()`。
如果返回的是 pass-by-reference datum，它通常指向 tuple 内部。
所以它和 `HeapTuple` 有同样生命周期：
```text
ReleaseSysCache(tuple) 后不能继续用这个 datum 指针。
```
需要修改或长期保存时，用：
```text
SearchSysCacheCopy()
SearchSysCacheLockedCopy1()
heap_copytuple()
datumCopy()
pstrdup()
```
copy path 会先复制 tuple，再 `ReleaseSysCache()` 原始 cache tuple。
调用者之后负责 `heap_freetuple()` copy。
### 5.8 release path
正常释放路径是：
```text
ReleaseSysCache(tuple)
  -> ReleaseCatCache(tuple)
     -> ReleaseCatCacheWithOwner(tuple, CurrentResourceOwner)
```
`ReleaseCatCacheWithOwner()` 反向计算 `CatCTup *`，做 assert：
```text
ct->ct_magic == CT_MAGIC
ct->refcount > 0
```
然后：
```text
ct->refcount--;
ResourceOwnerForgetCatCacheRef(resowner, &ct->tuple);
```
最后判断是否可以物理删除：
```text
if (ct->dead &&
    ct->refcount == 0 &&
    (ct->c_list == NULL || ct->c_list->refcount == 0))
    CatCacheRemoveCTup(ct->my_cache, ct);
```
如果没有被 invalidation 标记 dead，release 后 entry 仍留在 cache 中供未来 lookup 命中。
如果已经 dead，最后一个 holder release 才触发真正删除。
这就是避免悬挂 tuple 的中心机制。
## 6. 生命周期 / ownership / cleanup
本节可以把一个 syscache tuple 生命周期画成这样：
```text
InitCatalogCache()
  -> InitCatCache()
     -> CatCache descriptor in CacheMemoryContext
first lookup
  -> CatalogCacheInitializeCache()
     -> copied TupleDesc in CacheMemoryContext
miss
  -> systable scan gets heap tuple under catalog snapshot rules
  -> CatalogCacheCreateEntry()
     -> CatCTup + tuple payload in CacheMemoryContext
     -> refcount = 0
return to caller
  -> ResourceOwnerEnlarge()
  -> refcount++
  -> ResourceOwnerRemember(catcache reference)
  -> return &ct->tuple
normal use
  -> GETSTRUCT / SysCacheGetAttr
  -> ReleaseSysCache()
     -> refcount--
     -> ResourceOwnerForget
invalidation while held
  -> CatCacheInvalidate()
     -> dead = true
     -> do not return to future lookups
     -> do not pfree while refcount > 0
last release after dead
  -> CatCacheRemoveCTup()
     -> unlink
     -> free keys if negative
     -> pfree CatCTup
```
### 谁创建？
`InitCatalogCache()` 创建 syscache descriptors。
它只分配结构，不访问数据库。
每个 `CatCache` 的完整初始化延迟到第一次使用。
`CatalogCacheCreateEntry()` 创建具体 `CatCTup`。
positive entry 来自 catalog scan 的 tuple copy。
negative entry 来自 lookup keys。
### 谁持有？
caller 持有的是：
```text
HeapTuple pointer: &ct->tuple
```
catcache 持有的是：
```text
CatCTup in hash bucket
```
ResourceOwner 持有的是：
```text
PointerGetDatum(&ct->tuple)
```
这三个不是同一个层次。
caller 没有 owning memory。
caller 只有一个 active reference。
### 谁释放？
正常路径：
```text
caller calls ReleaseSysCache(tuple)
```
`ReleaseSysCache()` 必须在同一个 logical owner 下执行。
如果中途切换了 `CurrentResourceOwner`，`ResourceOwnerForget()` 可能找不到这条记录。
这类 bug 通常不是内存 leak，而是 owner 边界错乱。
ERROR 路径：
```text
ResourceOwnerRelease(..., RESOURCE_RELEASE_AFTER_LOCKS, ...)
  -> ResOwnerReleaseCatCache()
     -> ReleaseCatCacheWithOwner(tuple, NULL)
```
传 `NULL` 是因为 bulk release 正在处理 owner 自己的账本，不需要再 `Forget`。
### 谁避免悬挂？
`CacheMemoryContext` 避免 entry 被短生命周期 context reset。
`refcount` 避免 dead entry 在 caller 使用期间被 `pfree()`。
`ResourceOwner` 避免 ERROR 后 refcount 永远不归还。
`dead` 避免旧 entry 被 future lookup 再次拿到。
四者缺一不可。
### 谁推进语义过期？
`inval.c` 在 catalog tuple insert/update/delete 时登记 invalidation。
command end 本地执行当前命令的 invalidation。
commit 后把消息发送到 shared invalidation queue。
其他 backend 在安全点调用 `AcceptInvalidationMessages()` 处理。
`CatCacheInvalidate()` 只处理当前 backend 本地 cache。
它不会直接释放其他 backend 的内存。
## 7. invalidation 如何连接 lookup lifetime
catalog tuple 修改不能立即 flush cache。
`inval.c` 文件头把原因说得很清楚。
同一个 command 内，旧 tuple 仍可能按系统 catalog visibility 规则有效。
如果 `heap_update()` 立刻 flush，后续同一 command 又可能重新加载旧 tuple。
所以正常路径是：
```text
heap_insert / heap_update / heap_delete on catalog
  -> CacheInvalidateHeapTuple()
     -> CacheInvalidateHeapTupleCommon()
        -> PrepareInvalidationState()
        -> PrepareToInvalidateCacheTuple()
           -> for each CatCache on relation
              -> compute old hash
              -> compute new hash if update and changed
              -> RegisterCatcacheInvalidation()
CommandCounterIncrement()
  -> CommandEndInvalidationMessages()
     -> LocalExecuteInvalidationMessage()
        -> SysCacheInvalidate()
           -> CatCacheInvalidate()
commit
  -> AtEOXact_Inval(true)
     -> SendSharedInvalidMessages()
other backend
  -> AcceptInvalidationMessages()
     -> ReceiveSharedInvalidMessages()
     -> LocalExecuteInvalidationMessage()
```
这条链路有几个边界。
第一，invalidation message 记录的是 cache id 和 hash value。
它不是 `CatCTup *`。
因为每个 backend 的 catcache entry 地址都不同。
第二，`CatCacheInvalidate()` 不用 TID 精确匹配。
源码注释说明，VACUUM FULL 可能让 inval 事件创建时的 TID 和处理时的 TID 不一致。
所以当前实现按 hash value 匹配。
代价是可能有 false positive invalidation。
正确性上可以接受。
第三，tuple insert 也必须登记 invalidation。
因为别的 backend 或当前 backend 可能已有 matching negative entry。
第四，update 如果 old tuple 和 new tuple 的 hash 不同，要登记两个 hash。
如果 hash 相同，可以只登记一次。
第五，syscache relation 和 supporting index OID 被 `syscache.c` 记录在 sorted arrays 中。
`RelationHasSysCache()`、`RelationSupportsSysCache()` 用这些数组判断 catalog relation 是否有 syscache 或支持 syscache。
### `CatCacheInvalidate()` 的动作
给定一个 `CatCache *cache` 和 `hashValue`：
```text
for every CatCList in this cache:
  if list refcount > 0:
    list.dead = true
  else:
    CatCacheRemoveCList()
bucket = cache->cc_bucket[HASH_INDEX(hashValue)]
for every CatCTup in bucket:
  if ct->hash_value == hashValue:
    if ct->refcount > 0 or list refcount > 0:
      ct->dead = true
    else:
      CatCacheRemoveCTup()
for every in-progress entry:
  if same cache and same hash or list:
    mark in-progress dead
```
这段代码把“语义过期”和“物理释放”分开。
这是避免悬挂 tuple 的关键。
### in-progress stack
`catcache.c` 还有一个容易忽略的状态：
```text
catcache_in_progress_stack
```
当 entry 正在构造，尤其是 detoast 期间，可能调用路径会处理 invalidation。
如果这个 invalidation 命中新 entry，不能让它“出生就 stale”后仍返回给 caller。
所以构造过程中登记一个 `CatCInProgress`。
invalidation 会把它标成 dead。
`CatalogCacheCreateEntry()` 发现后返回 `NULL`。
caller 重新扫描 catalog。
这条 retry path 是一个重要错误路径，不是边角优化。
## 8. 正确性机制层次
catcache lifetime 不是单一机制保证的。
它是多层正确性边界叠加。
| 机制 | 保证 | 不保证 |
| --- | --- | --- |
| catalog snapshot / visibility | miss scan 时读到哪个 catalog tuple version | 已返回 pointer 永远代表最新 metadata。 |
| `CacheMemoryContext` | entry 内存可跨语句/事务留在 backend | caller 会正确归还引用。 |
| `refcount` | active holder 使用期间不物理释放 entry | entry 语义仍然新鲜。 |
| `dead` flag | future lookup 不返回已 invalidated entry | 当前 holder 不能继续根据旧语义做危险操作。 |
| `ResourceOwner` | ERROR/abort/Portal cleanup 能降 refcount | 自动帮你复制 tuple 或延长业务语义生命周期。 |
| invalidation | catalog change 后通知本地/其他 backend 标记过期 | 阻塞 DDL 或替代 lock。 |
| heavyweight lock | DDL/DML 在对象层建立互斥和等待顺序 | 直接保护 catcache entry 内存。 |
几个边界要特别分开。
### 内存安全不是语义新鲜
持有 `HeapTuple` 指针时，invalidation 可以把 underlying `CatCTup` 标成 dead。
这不会让当前 pointer 悬挂。
但它意味着：
```text
后续 lookup 不应该再拿到它；
当前 holder 如果跨越可能处理 invalidation 的动作，必须重新判断是否还能使用旧语义。
```
### invalidation 不是 lock
`CatCacheInvalidate()` 不阻塞正在读 entry 的 caller。
它也不等待 refcount 归零。
它只是改变本 backend 的 cache state。
对象级并发仍然靠 relation lock、tuple lock、transaction visibility 和 command counter。
### ResourceOwner 不是 MemoryContext
如果漏掉 `ReleaseSysCache()`：
```text
MemoryContext 不会知道需要 ct->refcount--
```
如果只 reset `CurrentMemoryContext`：
```text
CatCTup 在 CacheMemoryContext 中仍存在；
ResourceOwner 账本仍记录 active ref。
```
真正兜底的是 `ResourceOwnerRelease()`。
### `CurrentResourceOwner` 必须匹配
`SearchCatCacheInternal()` 记到 `CurrentResourceOwner`。
`ReleaseCatCache()` 默认从当前 owner forget。
如果一段代码跨 owner 传递 tuple，再在另一个 owner 中 release，就破坏了 owner 账本。
需要 copy，或者保证 release 发生在获取时的 owner 范围内。
## 9. 错误路径 / 异常路径 / fallback
### 9.1 `ResourceOwnerEnlarge()` 失败
hit path 的顺序是：
```text
ResourceOwnerEnlarge()
ct->refcount++
ResourceOwnerRemember()
```
如果 `ResourceOwnerEnlarge()` OOM，refcount 未增加。
没有需要 cleanup 的 active catcache ref。
miss path 在创建 entry 后也先 `ResourceOwnerEnlarge()`，再增加 refcount。
如果这里 OOM，新 entry 可能已在 cache 中，但 refcount 为 0。
这不是悬挂引用。
它只是一个普通 unreferenced cache entry。
后续 invalidation 或 cache reset 可以处理它。
### 9.2 caller ERROR 后没调用 `ReleaseSysCache()`
典型路径：
```text
SearchSysCache()
  -> refcount++
  -> ResourceOwnerRemember
... deeper code ereport(ERROR)
```
普通 C 栈不会回到 caller。
`ReleaseSysCache()` 不会执行。
事务 abort 或 portal cleanup 进入：
```text
ResourceOwnerRelease(..., RESOURCE_RELEASE_AFTER_LOCKS, false, ...)
  -> ResOwnerReleaseCatCache()
  -> ReleaseCatCacheWithOwner(tuple, NULL)
```
如果 entry 已经 dead 且 refcount 归零，马上删除。
如果 entry 没 dead，留在 cache 继续复用。
### 9.3 commit 时仍持有 catcache ref
正常 commit 时，绝大多数 query-lifespan catcache refs 应该已经释放。
如果还有残留，`ResourceOwnerRelease()` 在 commit cleanup 中会按资源类型 release，并可能打印 WARNING。
这通常说明代码漏了 `ReleaseSysCache()`。
abort 时不会把它当成同样级别的 bug 报出来。
abort 的工作就是清理未走完的路径。
### 9.4 invalidation 命中 active entry
路径：
```text
CatCacheInvalidate()
  -> ct->refcount > 0
  -> ct->dead = true
  -> keep memory
```
未来同 key lookup 会跳过 dead entry。
如果需要，会重新扫描 catalog，创建新 entry。
旧 holder release 后：
```text
refcount--
dead && refcount == 0
  -> CatCacheRemoveCTup()
```
这条路径解释了为什么可以同时存在两个 key 相同或语义相关的 cache entries。
旧 entry 被 holder pin 住。
新 entry 服务未来 lookup。
### 9.5 invalidation 命中正在构造的 entry
构造过程中可能 detoast。
TOAST access 可能处理 pending invalidation。
如果 invalidation 命中 in-progress entry：
```text
in_progress_ent.dead = true
CatalogCacheCreateEntry() returns NULL
SearchCatCacheMiss() sets stale = true
retry table scan
```
如果没有这条 retry，新 entry 可能一创建就 stale，却被返回给 caller。
### 9.6 bootstrap fallback
bootstrap processing mode 不创建 negative entries。
原因是 invalidation 机制尚不能保证后续创建的 tuple 会清掉 negative cache。
所以缺失 lookup 直接返回 `NULL`。
这是启动阶段 correctness 优先于 cache 命中率。
### 9.7 index scan fallback
`IndexScanOK()` 会在启动早期禁止某些 syscache 使用 index scan。
例如 `pg_index` cache 在 critical relcache 尚未建立时改用 heap scan。
否则 index support 初始化可能递归依赖自身。
这条 fallback 说明 catcache 的“unique index 支撑 lookup”是正常条件，不是所有阶段的绝对规则。
### 9.8 sinval queue overflow
如果 shared invalidation queue overflow，backend 可能不知道丢了哪些具体消息。
fallback 是：
```text
InvalidateSystemCaches()
  -> InvalidateCatalogSnapshot()
  -> ResetCatalogCachesExt(false)
  -> RelationCacheInvalidate(false)
```
这会牺牲局部性，换取 correctness。
它也说明 invalidation 不是精确消息必须全部可靠逐条送达才正确。
丢失精确信息时，系统退回全量 reset。
## 10. 成本、资源与跨模块传播
### 10.1 hot path 成本
命中路径主要成本：
- 计算 key hash。
- 定位 bucket。
- 遍历 bucket dlist。
- 对 hash 匹配 candidate 做 key equality。
- move-to-head。
- `ResourceOwnerEnlarge()` 快路径。
- `refcount++` 和 `ResourceOwnerRemember()`。
成本随这些变量放大：
| 变量 | 放大方式 |
| --- | --- |
| catalog object 数 | 某些 cache entry 数增加，bucket 链变长，内存增长。 |
| lookup key 分布 | 热 key 命中 bucket 头部，冷 key 更容易扫链。 |
| miss 率 | 每次 miss 都可能打开 catalog、做 systable scan、copy tuple。 |
| DDL/invalidation 频率 | entry 被标 dead，后续 lookup 重新 scan。 |
| long transaction / leaked ref | dead entry 无法物理删除，CacheMemoryContext retention 增加。 |
| list lookup | partial-key list invalidation 更粗，DDL 后更容易重建。 |
catcache 没有跨 backend 锁竞争。
每个 backend 本地维护 hash。
但 miss path 会触碰 catalog relation、relcache、buffer manager、lock manager 和 snapshot。
所以它不是“纯 CPU cache”。
### 10.2 memory cost
`CacheMemoryContext` 中可能包含：
- `CatCache` descriptors。
- bucket arrays。
- copied tuple descriptors。
- positive `CatCTup` + tuple payload。
- negative `CatCTup` + copied keys。
- `CatCList`。
- relcache / typcache / plancache 等其它 cache 相关对象。
因此看 `CacheMemoryContext` 增长时要先问：
```text
是查了很多不同 catalog keys？
是有高 churn DDL 导致 dead entries 等 refcount？
是 list cache 被大量 partial-key lookup 填充？
是 relcache/typcache 而不是 catcache？
```
不能仅凭 context 名字下结论。
### 10.3 与相邻模块边界
与 relcache：
```text
catcache 提供 pg_class / pg_attribute 等 tuple lookup；
relcache 把多个 catalog facts 构造成 Relation；
relcache invalidation 比 catcache invalidation 粒度更高。
```
与 typcache：
```text
typcache 通过 syscache 查询 pg_type、pg_operator、pg_opclass 等；
typcache callback 依赖 syscache invalidation 来标记高层缓存过期。
```
与 planner：
```text
planner 大量通过 lsyscache helper 获取 type/operator/function metadata；
多数 helper 返回 scalar 并内部释放 syscache tuple；
需要长期保存的结构必须 copy。
```
与 lock manager：
```text
DDL 的对象互斥靠 heavyweight lock；
catcache invalidation 不提供等待语义；
commit 中 invalidation 要发生在释放 locks 前。
```
与 WAL/recovery：
```text
logical decoding 需要 per-command invalidations；
commit invalidation messages 可进入 commit record；
recovery replay 会处理 committed invalidation messages。
```
本节不涉及 bgwriter 或 checkpointer 推进 catcache。
catcache 是 backend-local。
shared state 主要是 shared invalidation queue。
普通 backend 自己在安全点消费消息。
startup/recovery 路径会重放 commit invalidation。
## 11. 观测与诊断入口
### 11.1 能直接看到什么
`pg_backend_memory_contexts` 可以看到当前 backend 的 context 统计。
例如：
```sql
SELECT name, level, total_bytes, used_bytes, free_bytes
FROM pg_backend_memory_contexts
WHERE name = 'CacheMemoryContext';
```
这能说明 cache 相关 memory retention。
它不能告诉你：
- 有多少 `CatCTup`。
- 哪些 entry 是 dead。
- 哪些 entry refcount 大于 0。
- 哪个 `ResourceOwner` 持有某个 tuple。
- 哪个 invalidation message 命中过它。
`pg_log_backend_memory_contexts(pid)` 可以让目标 backend 打印 memory context tree。
粒度仍然是 context，不是 catcache entry。
### 11.2 能通过 debug build 看到什么
assert build 默认启用 `DISCARD_CACHES_ENABLED`。
可以设置：
```sql
SET debug_discard_caches = 1;
```
这会在能够处理 invalidation 的地方更积极 flush syscache/relcache。
它适合暴露“代码保存 cache tuple 指针过久”的 bug。
它不适合生产诊断。
源码注释明确说它会极大拖慢 regression test。
更激进的源码测试开关包括：
```text
CATCACHE_FORCE_RELEASE
CLOBBER_FREED_MEMORY
CACHEDEBUG
CATCACHE_STATS
```
这些通常需要本地开发构建。
`CATCACHE_FORCE_RELEASE` 会让 entry 在 refcount 归零后更积极释放。
配合 clobber freed memory，可以抓住 release 后继续使用 tuple 的 bug。
### 11.3 gdb 断点
最直接的生命周期观察靠 gdb。
常用断点：
```gdb
break SearchSysCache1
break SearchCatCacheInternal
break SearchCatCacheMiss
break CatalogCacheCreateEntry
break CatCacheInvalidate
break ReleaseCatCacheWithOwner
break ResOwnerReleaseCatCache
break CommandEndInvalidationMessages
break AcceptInvalidationMessages
```
观察点：
```text
SearchCatCacheInternal hit:
  ct->refcount before/after
  ct->dead
  CurrentResourceOwner
CatalogCacheCreateEntry:
  ntp != NULL or NULL
  ct->negative
  ct->tuple.t_data
  ct->keys[]
CatCacheInvalidate:
  hashValue
  ct->hash_value
  ct->refcount
  ct->dead
ReleaseCatCacheWithOwner:
  resowner == CurrentResourceOwner or NULL
  ct->refcount before/after
  removal condition
```
### 11.4 日志和 WARNING
如果 commit 时还有未释放的 catcache ref，ResourceOwner 的 debug print 可能给出类似信息：
```text
cache <name> (<id>), tuple <block>/<offset> has count <n>
```
这说明正常路径漏了 `ReleaseSysCache()`。
abort path 清理同类资源时不一定打印同样 warning。
因为 abort 的目标就是兜底 cleanup。
### 11.5 一个具体 runtime truth
可以观察到的现象：
```text
当前 backend 第一次解析或访问大量新 catalog objects 后，CacheMemoryContext 增长；
后续重复 lookup 不再按同样幅度增长；
DDL 或 debug_discard_caches 触发 invalidation 后，部分后续 lookup 会重新构造 entry；
如果某处漏 ReleaseSysCache，commit cleanup 可能 warning，abort cleanup 会释放 ref。
```
源码解释：
```text
首次 miss:
  CatalogCacheCreateEntry() allocates in CacheMemoryContext
重复 hit:
  SearchCatCacheInternal() returns existing &ct->tuple
invalidation:
  CatCacheInvalidate() dead/remove
leaked ref:
  ResourceOwnerRelease(AFTER_LOCKS) calls ResOwnerReleaseCatCache()
```
边界：
```text
SQL 视图不能直接看到 CatCTup.refcount；
要确认 tuple 是否 dead/refcount，需要 gdb、临时日志或开发构建。
```
## 12. 常见误区
### 误区一：syscache 是 shared cache
不是。
syscache/catcache entry 是 backend-local。
shared invalidation 只是让各 backend 自己清理本地 cache。
不要把某个 backend 中的 `CatCTup *` 当成跨进程地址。
### 误区二：`CacheMemoryContext` 保证 caller 可以一直用 tuple
不对。
`CacheMemoryContext` 只保证 entry 不随短生命周期 context reset。
caller 的使用权由 refcount 表达。
`ReleaseSysCache()` 后继续使用 tuple 或 by-ref datum，仍然是 use-after-release。
### 误区三：`refcount > 0` 表示 tuple 语义仍然有效
不对。
`refcount` 只保护物理内存。
invalidation 可以把 entry 标成 `dead`。
当前 holder 内存安全，但不能把它当作永远新鲜的 catalog truth。
### 误区四：invalidation 应该立刻 pfree 旧 tuple
如果 caller 正持有 `&ct->tuple`，立刻 pfree 就会制造悬挂指针。
正确做法是：
```text
dead = true
future lookup skips it
last release removes it
```
### 误区五：negative cache entry 不需要 invalidation
恰恰相反。
insert 一个 catalog tuple 时，必须 flush matching negative entry。
否则“不存在”的缓存会掩盖新对象。
### 误区六：`SearchSysCacheCopy()` 只是更方便
它改变 ownership。
返回 copy 后，原始 cache tuple 已经 release。
caller 拥有一份普通 heap tuple copy，可以修改，并需要 `heap_freetuple()`。
这是跨 release 保存 tuple 内容的正确方式。
### 误区七：看到 `CacheMemoryContext` 大就是 leak
可能是正常 cache retention。
也可能是查了大量 catalog objects。
也可能是 relcache/typcache。
也可能是 dead entries 被 refcount pin 住。
必须结合 workload、DDL churn、gdb 或临时计数器判断。
## 13. 课堂实验
### 实验一：gdb 跟一条 `SearchSysCache()` lifetime
目标：看到 miss 构造 entry、返回 refcount、正常 release。
准备一个本地 debug build。
连接一个 backend，找到 PID 后 attach gdb。
设置断点：
```gdb
break SearchSysCache1
break SearchCatCacheMiss
break CatalogCacheCreateEntry
break ReleaseCatCacheWithOwner
```
在 SQL 会话运行：
```sql
SELECT pg_relation_filenode('pg_class'::regclass);
SELECT pg_relation_filenode('pg_class'::regclass);
```
观察：
```text
第一次更可能进入 SearchCatCacheMiss()；
CatalogCacheCreateEntry() 分配 CatCTup；
返回 caller 前 refcount 变成 1；
ReleaseCatCacheWithOwner() 后 refcount 归零；
第二次更可能 hit SearchCatCacheInternal()，不再构造 entry。
```
需要画出的状态：
```text
CatCTup:
  negative
  dead
  refcount
  tuple.t_self
  tuple.t_data
  my_cache->cc_relname
```
如果第二次仍 miss，不要急着判断错误。
可能被 `debug_discard_caches`、invalidation、启动阶段或不同 cache path 影响。
实验目标是跟生命周期，不是证明某条 SQL 必然命中。
### 实验二：观察 `CacheMemoryContext` retention
目标：看到 catalog lookup 造成 backend-local cache memory 增长，但 SQL 视图看不到 refcount。
在单个会话中运行：
```sql
SELECT name, total_bytes, used_bytes
FROM pg_backend_memory_contexts
WHERE name = 'CacheMemoryContext';
CREATE SCHEMA cc_demo;
DO $$
BEGIN
  FOR i IN 1..200 LOOP
    EXECUTE format('CREATE TABLE cc_demo.t%s(id int)', i);
  END LOOP;
END
$$;
SELECT count(to_regclass(format('cc_demo.t%s', g)))
FROM generate_series(1, 200) AS g;
SELECT name, total_bytes, used_bytes
FROM pg_backend_memory_contexts
WHERE name = 'CacheMemoryContext';
```
解释：
```text
CREATE TABLE 和 to_regclass 会触发 catalog/relcache/syscache 路径；
当前 backend 的 CacheMemoryContext 可能增长；
增长不等于 leak；
SQL 视图不能告诉你哪些 CatCTup 是 dead 或 refcount > 0。
```
清理：
```sql
DROP SCHEMA cc_demo CASCADE;
```
再看一次 memory context。
不要期望它完全回到初始值。
cache retention 本来就是为了复用。
### 实验三：用 `debug_discard_caches` 暴露 stale pointer 假设
目标：理解 aggressive invalidation 为什么能抓住错误缓存使用。
要求：assert build 或启用了 `DISCARD_CACHES_ENABLED` 的开发构建。
运行：
```sql
SET debug_discard_caches = 1;
```
然后跑一组会频繁解析对象、创建/删除对象、调用函数的 SQL。
观察：
```text
系统明显变慢；
cache hit 变少；
一些隐藏的 stale metadata 假设更容易暴露。
```
源码对应点：
```text
AcceptInvalidationMessages()
  -> InvalidateSystemCachesExtended(true)
     -> ResetCatalogCachesExt(true)
     -> RelationCacheInvalidate(true)
```
边界：
```text
这是开发诊断工具；
不是生产性能开关；
它改变 timing，不应把所有现象直接外推到正常构建。
```
## 14. 讨论题
1. 为什么 `SearchCatCacheInternal()` 必须在 `ct->refcount++` 前调用 `ResourceOwnerEnlarge()`？
2. `CacheMemoryContext` 和 `ResourceOwner` 分别解决 catcache tuple lifetime 的哪一半问题？
3. 一个 `CatCTup` 已经 `dead = true`，但 `refcount = 1`。此时它对当前 holder 和未来 lookup 分别意味着什么？
4. 为什么 `CatCacheInvalidate()` 只按 hash value invalidation，而不是始终按 TID 精确匹配？
5. negative cache entry 为什么在 catalog insert 时也必须被 invalidation？
6. 如果一个 helper 从 `SearchSysCache()` 得到 pass-by-reference datum，想在 `ReleaseSysCache()` 后保存它，应该怎么做？
7. 为什么 catcache ref 的 ResourceOwner release phase 是 `AFTER_LOCKS`，而不是和 relcache ref 一样放在锁前？
8. `pg_backend_memory_contexts` 能帮助判断什么？它为什么不能直接证明某个 syscache entry 泄漏？
## 15. 本节小结
本节主线是一个 catalog row 从 heap tuple 变成 catcache entry，再作为 `HeapTuple` 指针被 caller 临时持有。
关键链路：
```text
SearchSysCache()
  -> SearchCatCacheInternal()
  -> hit or SearchCatCacheMiss()
  -> CatalogCacheCreateEntry()
  -> refcount++ and ResourceOwnerRemember
  -> caller uses tuple
  -> ReleaseSysCache() or ResourceOwner cleanup
  -> invalidation marks dead
  -> last release physically removes
```
核心状态：
- `CatCache` 是 backend-local descriptor 和 hash buckets。
- `CatCTup` 是真正返回给 caller 的 tuple 所在对象。
- `CacheMemoryContext` 保存长寿 cache entry 内存。
- `refcount` 表示 active holders。
- `dead` 表示语义过期，不再服务未来 lookup。
- `ResourceOwner` 记录 refcount 归还责任。
ownership 边界：
```text
caller 不拥有 tuple memory；
caller 拥有一次 active reference；
正常路径必须 ReleaseSysCache；
ERROR/abort 由 ResourceOwnerRelease(AFTER_LOCKS) 兜底；
需要长期保存或修改时必须 copy。
```
错误路径：
- OOM 前置扩容避免 refcount 和 owner 账本脱节。
- invalidation 命中 active entry 时只标 `dead`。
- detoast 或 list build 期间收到 invalidation 会 retry。
- sinval overflow 退回全量 cache reset。
- bootstrap 阶段不创建 negative cache entry。
观测边界：
```text
pg_backend_memory_contexts 能看到 CacheMemoryContext retention；
gdb/临时日志能看到 CatCTup.refcount 和 dead；
debug_discard_caches 能放大 stale pointer 假设；
普通 pg_stat 视图不能直接展示 syscache entry lifetime。
```
可迁移规律：
```text
一个高频 backend-local cache 如果要返回内部对象指针，
必须把“内存生命周期”“active 引用”“语义失效”“异常 cleanup”拆成不同机制；
不能指望一个 memory context 或一个 invalidation flag 同时解决所有问题。
```
本节判断仍然依赖 workload 和版本。
catalog object 数、DDL churn、长事务、assert/debug 构建、`debug_discard_caches`、以及具体 helper 是否 copy 返回值，都会改变你看到的 memory 和 miss/hit 现象。
下一节进入 relcache build/invalidation 时，要带着本节的不变量继续看：
```text
返回给调用者的高层 descriptor 可以过期；
但它不能在 holder 使用期间物理悬挂。
```
