# PostgreSQL catcache / relcache error cleanup
## 课程定位
前置知识：
已经理解 `MemoryContext`、`ResourceOwner`、syscache/catcache lookup、relcache build、shared invalidation、catalog snapshot 和事务 abort 的基本边界。
本节唯一主问题：
ERROR、transaction abort 和 invalidation 风暴下，cache ref、relcache entry 和 ResourceOwner 如何收尾？
核心矛盾：
catcache 和 relcache 必须把 catalog metadata 做成 backend-local 快速缓存。
但 `ereport(ERROR)` 可以在 lookup、build、callback、DDL cleanup 中途 longjmp。
同时，DDL 风暴会持续把旧 cache entry 标成语义过期。
PostgreSQL 不能在每次命中时重新扫 catalog。
也不能在 invalidation 到达时释放正在被调用栈使用的对象。
它把问题拆成三条线：
refcount / `ResourceOwner` 保证指针安全；
`dead` / `rd_isvalid` / invalidation 保证 future lookup 不相信旧语义；
transaction cleanup ordering 保证 commit、abort、subtransaction 的边界一致。
一句话运行模型：
catcache tuple 用 `CatCTup.refcount` 和 `ResourceOwner` 管短期引用，
invalidation 把 stale entry 标 `dead` 或删除；
relcache 用 `RelationData.rd_refcnt`、`rd_isvalid` 和 EOX cleanup 管 relation descriptor；
abort 路径释放 owner 上的 ref，并处理本 backend 已经受失败事务影响的 local invalidation。
学完后应能判断：
一个 `HeapTuple` 或 `Relation *` 是否仍能安全解引用；
它代表的 catalog 语义是否仍然新鲜；
ERROR 后谁释放忘记 release 的 cache ref；
abort 后为什么本 backend 也要清 local cache；
invalidation storm 为什么通常表现为 CPU / rebuild 抖动，而不是跨 backend 指针悬挂。
本课基于 `/home/nail/postgres` 当前源码树。
## 1. 本节在总主线中的位置
第 56 节已经讲过 syscache/catcache lookup lifetime：
```text
SearchSysCache*()
  -> SearchCatCache*()
  -> CatCTup.refcount++
  -> ResourceOwnerRememberCatCacheRef()
  -> ReleaseSysCache()
```
第 57 节已经讲过 relcache build / invalidation：
```text
RelationIdGetRelation()
  -> RelationBuildDesc()
  -> rd_refcnt++
  -> rd_isvalid controls freshness
  -> RelationClose()
```
第 59 节已经讲过 shared invalidation message flow：
```text
catalog tuple change
  -> pending invalidation group
  -> command / commit boundary
  -> LocalExecuteInvalidationMessage()
  -> catcache / relcache local cleanup
```
第 60 节已经讲过 catalog snapshot / syscache consistency。
本节不重复 lookup、build、message propagation。
本节只追 cleanup 问题：
当执行流不是正常返回，
或者 invalidation 比 lookup 更频繁时，
backend-local cache 如何不泄漏 ref、不悬挂 pointer、不继续返回 stale fact。
README 中待补主题提到 `61`、`62` 之后的 `63`。
当前仓库只存在 `56` 到 `60`。
本节按已存在课程承接，不假设不存在的章节内容。
本节的 runtime truth：
其它 backend 不能释放你的 catcache tuple 或 relcache entry。
它们只能发送 invalidation。
receiver backend 在安全点处理消息，
把本地对象删除、标 dead、标 invalid、rebuild 或延迟 cleanup。
## 2. 核心矛盾与运行模型
如果只追求速度，
catcache 可以把 `pg_class`、`pg_type`、`pg_attribute` 等 catalog tuple 放进本地 hash。
relcache 可以把 `pg_class`、`pg_attribute`、index、rule、partition 等信息拼成 `RelationData`。
命中后直接返回本地指针。
如果只追求简单正确，
每次 metadata access 都重新拿 snapshot、扫 catalog、构造 descriptor。
那样不会有 stale local cache，
但 planner、executor、DDL、syscache helper 的热路径成本不可接受。
PostgreSQL 选择缓存，
但把 cleanup 拆成窄机制：
| 机制 | 负责什么 | 不负责什么 |
| --- | --- | --- |
| `MemoryContext` | cache entry 和 descriptor 子对象内存 | 归还 refcount |
| `ResourceOwner` | ERROR / abort 时释放已登记引用 | 判断语义是否 stale |
| refcount / `rd_refcnt` | 使用中的本地对象不能物理释放 | metadata 仍然最新 |
| `dead` / `rd_isvalid` | future lookup 不使用旧 entry | 中断当前 holder |
| invalidation | 把 catalog change 变成本地 stale 信号 | 传输新 tuple / descriptor |
| xact cleanup | 排序 commit / abort / subxact 收尾 | 替代所有模块局部 cleanup |
因此一个对象可以同时满足：
指针安全；
语义过期；
还不能物理释放。
这不是矛盾。
这是 cache cleanup 的基本状态。
本节 mental model：
```text
pointer safety:
  refcount + ResourceOwner + MemoryContext
semantic freshness:
  invalidation + dead/rd_isvalid + command/commit boundary
transaction cleanup:
  xact.c ordering + AtEOXact_* + ResourceOwnerRelease phases
```
`ereport(ERROR)` 让 normal release 不可靠。
代码可能写了：
```c
tuple = SearchSysCache1(TYPEOID, ObjectIdGetDatum(typeoid));
do_something_that_may_error(tuple);
ReleaseSysCache(tuple);
```
如果中间 ERROR，
`ReleaseSysCache()` 不会执行。
所以成功 acquire 后必须登记到 `CurrentResourceOwner`。
abort cleanup 才能兜底释放。
invalidation storm 让 semantic freshness 更频繁变化。
收到 DDL invalidation 时，
receiver 不能直接释放当前调用栈正在用的 entry。
它只能标记 stale，
并等 refcount 归零后清理。
这就是本节核心 tension：
热路径要快，
错误路径要能 longjmp，
invalidation 要能粗粒度过度，
而指针生命周期仍必须精确。
## 3. 核心文件分工与阅读顺序
| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/utils/catcache.h` | `CatCTup`、`CatCList` 的 `refcount`、`dead`、`negative`。 |
| 2 | `src/backend/utils/cache/catcache.c` | `SearchCatCacheInternal()`、`ReleaseCatCache()`、`CatCacheInvalidate()`、in-progress stack。 |
| 3 | `src/include/utils/rel.h` | `RelationData.rd_refcnt`、`rd_isvalid`、`rd_isnailed`。 |
| 4 | `src/include/utils/relcache.h` | `RelationIdGetRelation()`、`RelationClose()`、`AtEOXact_RelationCache()` API。 |
| 5 | `src/backend/utils/cache/relcache.c` | build、flush、rebuild、clear、`in_progress_list`、EOX cleanup。 |
| 6 | `src/include/utils/resowner.h` | release phase、priority、resource kind contract。 |
| 7 | `src/backend/utils/resowner/resowner.c` | `ResourceOwnerRelease()` 三阶段释放。 |
| 8 | `src/backend/utils/cache/inval.c` | command / commit / abort / subxact invalidation ordering。 |
| 9 | `src/include/utils/inval.h` | invalidation 对外边界。 |
| 10 | `src/backend/access/transam/xact.c` | `CommitTransaction()`、`AbortTransaction()`、subtransaction cleanup 顺序。 |
读源码时先定位三类函数。
获取并登记引用：
```text
SearchCatCacheInternal()
SearchCatCacheMiss()
SearchCatCacheList()
RelationIdGetRelation()
RelationIncrementReferenceCount()
```
正常释放和 owner 兜底：
```text
ReleaseCatCache()
ReleaseCatCacheList()
RelationClose()
RelationCloseCleanup()
ResOwnerReleaseCatCache()
ResOwnerReleaseCatCacheList()
ResOwnerReleaseRelation()
```
语义失效和事务收尾：
```text
CatCacheInvalidate()
ResetCatalogCache()
RelationCacheInvalidateEntry()
RelationCacheInvalidate()
RelationFlushRelation()
AtEOXact_RelationCache()
AtEOSubXact_RelationCache()
AtEOXact_Inval()
AtEOSubXact_Inval()
CommandEndInvalidationMessages()
```
不要把源码想象成一个统一 cache framework。
catcache tuple、catcache list、relcache entry、typcache callback、plancache callback 是不同历史层次。
它们共享原则，
不共享同一套结构体。
## 4. 关键数据结构与状态
### 4.1 `CatCTup`
`CatCTup` 是 catcache 的单 tuple entry。
调用者通常拿到的是 entry 内部的 `HeapTupleData` 地址。
本节只抽象几个字段：
```c
typedef struct catctup
{
    int refcount;
    bool dead;
    bool negative;
    HeapTupleData tuple;
} CatCTup;
```
实际结构还有 hash link、key、cache pointer、list pointer 和 magic。
`refcount`：
当前 backend 内有多少活跃引用。
positive hit 会增加它。
`ReleaseCatCache()` 会减少它。
ResourceOwner callback 也走同一套内部 release。
`dead`：
语义已经过期。
`dead=true` 的 entry 不应被后续 search 返回。
但如果 `refcount > 0`，
或者它被 live `CatCList` 持有，
物理删除必须延后。
`negative`：
缓存“不存在”。
insert 也必须 invalidation，
否则旧 negative entry 会继续让 lookup 认为对象不存在。
核心状态组合：
```text
positive + refcount>0 + dead=false:
  当前可返回且正在使用
positive + refcount>0 + dead=true:
  指针安全，但 future lookup 跳过
positive + refcount=0 + dead=true:
  可物理删除
negative + dead=false:
  miss fact 可复用
negative + dead=true:
  miss fact 已过期
```
不变量：
`dead` 管语义新鲜；
`refcount` 管物理生命周期；
两者不能互相替代。
### 4.2 `CatCList`
`CatCList` 缓存一组匹配 key 前缀的 `CatCTup`。
本节抽象为：
```c
typedef struct catclist
{
    int refcount;
    bool dead;
    int n_members;
    CatCTup *members[FLEXIBLE_ARRAY_MEMBER];
} CatCList;
```
实际结构还包含 key、hash、cache pointer 和 magic。
list 的 cleanup 比单 tuple 难。
list 引用 member entry。
只要 list 仍被 holder 使用，
member 即使 dead 也不能随便释放。
当前源码里 `CatCacheInvalidate()` 会粗粒度 invalidate 同一个 catcache 的所有 `CatCList`。
源码注释说很难判断哪些 list search 仍然正确，
所以直接 zap all lists。
这比 tuple invalidation 更粗。
tuple 仍按 hash bucket 匹配。
不变量：
list dead 后不会被 future `SearchCatCacheList()` 返回。
list refcount 归零后，
删除 list 时再检查 member 是否也能删除。
### 4.3 `catcache_in_progress_stack`
`catcache_in_progress_stack` 记录正在构造的 entry 或 list。
构造过程可能扫 catalog、打开 index、拷贝 tuple、访问 toast。
这些步骤可能处理 invalidation。
如果同一个 cache/hash 在构造中被 invalidated，
in-progress 记录会被标 dead。
构造逻辑随后重试或丢弃。
抽象流程：
```text
SearchCatCacheMiss()
  -> push CatCInProgress
  -> scan catalog
  -> invalidation may mark in-progress dead
  -> build result discarded or retried
```
这个栈不是锁。
它只防止“出生即 stale”的 entry 进入 cache。
### 4.4 `RelationData`
relcache 的对象是 backend-local `RelationData`。
调用者用 `Relation` 指针访问它。
本节只抽象几个字段：
```c
typedef struct RelationData
{
    int rd_refcnt;
    bool rd_isvalid;
    bool rd_isnailed;
    Oid rd_id;
} RelationData;
```
实际结构很大，
包括 tupledesc、relation kind、index AM、trigger、rule、partition、storage manager、options 等子状态。
`rd_refcnt`：
当前 backend 有多少打开引用。
`RelationIdGetRelation()` 和 `relation_open()` 增加它。
`RelationClose()` 减少它。
`rd_isvalid`：
descriptor 语义是否有效。
收到 invalidation 后，
entry 可能被 clear、rebuild，
也可能保留对象但标 invalid。
`rd_isnailed`：
系统关键 relation 的常驻 entry。
nailed entry 不能像普通 relation 一样随意从 hash 删除。
核心状态组合：
```text
rd_refcnt>0 + rd_isvalid=true:
  正在使用且当前语义有效
rd_refcnt>0 + rd_isvalid=false:
  指针安全，但 future lookup 不应信任
rd_refcnt=0 + rd_isvalid=false:
  可 clear、rebuild 或延迟清理
rd_isnailed=true:
  常驻 entry，flush 多数表现为重建或标 invalid
```
不变量：
`rd_refcnt` 不是 freshness。
持有 relation ref 只保证 descriptor 内存不被释放。
### 4.5 `in_progress_list`
`relcache.c` 的 `in_progress_list` 记录正在 `RelationBuildDesc()` 的 relid。
它解决的问题和 catcache in-progress 类似。
relcache build 可能递归访问 catalog、syscache、index、partition metadata。
如果 build 中途收到 relcache invalidation，
构造出的 descriptor 不能当作 fresh entry。
抽象流程：
```text
RelationBuildDesc(relid)
  -> push InProgressEnt
  -> read catalog metadata
  -> invalidation marks invalidated
  -> build exits
  -> retry, discard, or leave invalid state
```
`AtEOXact_RelationCache()` 和 `AtEOSubXact_RelationCache()` 会在 abort 场景忘记 in-progress list。
源码断言 commit 时不应还在 build 中。
### 4.6 `ResourceOwner`
`ResourceOwner` 管必须显式归还的资源。
它不是 MemoryContext。
本节涉及的 owner 资源：
catcache tuple ref。
catcache list ref。
relcache relation ref。
tupledesc ref。
snapshot ref。
buffer pin。
lock。
当前源码中：
catcache tuple ref 的 release phase 是 `RESOURCE_RELEASE_AFTER_LOCKS`。
catcache list ref 也是 `RESOURCE_RELEASE_AFTER_LOCKS`。
relcache relation ref 的 release phase 是 `RESOURCE_RELEASE_BEFORE_LOCKS`。
这很重要。
commit / abort path 中，
relcache refs 先在 before-locks 阶段释放。
`AtEOXact_RelationCache()` 随后检查 relcache 状态。
catcache refs 则在 after-locks 阶段兜底释放。
所以不要泛泛说“所有 cache refs 都在 invalidation 前释放”。
源码只明确要求 relcache refs 在处理 relcache invalidation 前掉到正常计数。
catcache 通过 `dead` + refcount 支持 invalidation 先到、release 后到。
## 5. 主流程 walkthrough：catcache ref
普通 syscache lookup 进入 catcache：
```text
SearchSysCache1()
  -> SearchCatCache1()
  -> SearchCatCacheInternal()
```
命中 positive 且未 dead 的 entry：
```text
ResourceOwnerEnlarge(CurrentResourceOwner)
ct->refcount++
ResourceOwnerRememberCatCacheRef(CurrentResourceOwner, &ct->tuple)
return &ct->tuple
```
`ResourceOwnerEnlarge()` 在真正增加 refcount 前调用。
这是 acquire-before-ERROR 原则。
如果 remember 需要空间，
先把空间准备好，
避免“资源已获取但登记失败”的窗口。
caller 正常释放：
```text
ReleaseSysCache(tuple)
  -> ReleaseCatCache(tuple)
  -> ReleaseCatCacheWithOwner(tuple, CurrentResourceOwner)
  -> ct->refcount--
  -> ResourceOwnerForgetCatCacheRef()
```
如果 entry 已经 dead，
且 tuple refcount 为零，
且相关 list 也没有 ref，
`ReleaseCatCacheWithOwner()` 会调用 `CatCacheRemoveCTup()`。
ResourceOwner callback 走：
```text
ResOwnerReleaseCatCache()
  -> ReleaseCatCacheWithOwner(tuple, NULL)
```
传 `NULL` 是为了避免 release 时再从 owner 里 forget。
因为 ResourceOwner 正在释放这个资源。
### 5.1 miss path
miss path 进入 `SearchCatCacheMiss()`。
它会扫 catalog，
构造 positive entry 或 negative entry。
positive entry 返回前：
```text
ResourceOwnerEnlarge(CurrentResourceOwner)
ct->refcount++
ResourceOwnerRememberCatCacheRef()
```
negative entry 通常不返回给 caller，
但仍留在 cache 中代表 miss fact。
insert/update/delete 都可能让它过期。
### 5.2 list path
`SearchCatCacheList()` 返回 `CatCList *`。
命中 live list 时：
```text
ResourceOwnerEnlarge(CurrentResourceOwner)
cl->refcount++
ResourceOwnerRememberCatCacheListRef()
```
释放：
```text
ReleaseCatCacheList()
  -> cl->refcount--
  -> ResourceOwnerForgetCatCacheListRef()
  -> if cl->dead && cl->refcount==0:
       CatCacheRemoveCList()
```
构造 list 时，
源码会临时 bump member refcount，
并用局部 error cleanup 保护。
这些临时 ref 不完全等同于 caller 持有的 ResourceOwner ref。
## 6. 主流程 walkthrough：relcache ref
高层常见入口是 `relation_open()`。
本节简化为：
```text
relation_open()
  -> RelationIdGetRelation(relid)
  -> RelationIncrementReferenceCount()
```
当前源码中 `RelationIncrementReferenceCount()` 做：
```text
ResourceOwnerEnlarge(CurrentResourceOwner)
rel->rd_refcnt++
ResourceOwnerRememberRelationRef(CurrentResourceOwner, rel)
```
bootstrap mode 是例外，
它禁用 owner tracking。
正常关闭：
```text
relation_close()
  -> RelationClose()
  -> RelationDecrementReferenceCount()
  -> ResourceOwnerForgetRelationRef()
  -> RelationCloseCleanup()
```
ResourceOwner callback：
```text
ResOwnerReleaseRelation()
  -> rel->rd_refcnt--
  -> RelationCloseCleanup(rel)
```
callback 不再调用 `ResourceOwnerForgetRelationRef()`。
因为 owner 正在释放该资源。
如果 hash 中没有 relcache entry，
`RelationIdGetRelation()` 调用 `RelationBuildDesc()`。
build 过程读 `pg_class`、`pg_attribute`、index 和其它派生 metadata。
如果 entry 存在但 invalid，
relcache 可能 rebuild、invalidate 或在下次 lookup 继续处理。
具体分支依赖 refcount、transaction state、nailed 状态和 relation 类型。
关键区别：
catcache entry 是 catalog tuple copy。
relcache entry 是多 catalog tuple 拼成的 descriptor。
因此 relcache flush 不是简单 free。
它可能要清 smgr、tupledesc、index list、partition descriptor、rule/trigger state 等子对象。
## 7. invalidation 到达时如何收尾
shared invalidation message 不携带其它 backend 的 cache 指针。
receiver backend 在本地执行 cleanup。
常见入口：
```text
AcceptInvalidationMessages()
CommandEndInvalidationMessages()
AtEOXact_Inval()
AtEOSubXact_Inval()
```
### 7.1 catcache invalidation
`CatCacheInvalidate(cache, hashValue)` 做三件事。
第一，粗粒度 invalidate 该 cache 的所有 `CatCList`：
```text
if cl->refcount > 0:
  cl->dead = true
else:
  CatCacheRemoveCList()
```
第二，按 hash bucket 处理 tuple：
```text
if hashValue == ct->hash_value:
  if ct->refcount > 0 or live list holds it:
    ct->dead = true
  else:
    CatCacheRemoveCTup()
```
第三，处理正在构造的 entry：
```text
for e in catcache_in_progress_stack:
  if same cache and (list or same hash):
    e->dead = true
```
所以 catcache invalidation 的效果是：
删除无人持有的 stale entry；
标记有人持有的 stale entry；
阻止正在构造的 stale entry 以 fresh 状态出生。
它不会等待 holder。
它不会释放 holder 的 ref。
它不会让当前调用栈自动重跑 lookup。
### 7.2 whole catcache reset
`ResetCatalogCache()` 处理某个 cache 的所有 list 和 tuple。
有 ref 的标 dead。
无 ref 的删除。
如果不是 `debug_discard_caches`，
它还会把 matching in-progress build 标 dead。
whole reset 是 overflow / debug / 粗粒度 fallback 的基础。
原则是：
过度 invalidation 可以接受；
missed invalidation 不可接受。
### 7.3 relcache invalidation
`RelationCacheInvalidateEntry(relid)`：
```text
RelationIdCacheLookup(relid)
if relation exists:
  relcacheInvalsReceived++
  RelationFlushRelation(relation)
else:
  mark matching in_progress_list invalidated
```
`RelationFlushRelation()` 根据状态决定 clear、rebuild 或 invalidate。
如果 entry 没有 ref，
可以更积极地清。
如果有 ref，
必须保留 pointer safety。
whole relcache reset 走 `RelationCacheInvalidate(debug_discard)`。
源码注释说：
它会 blow away zero-ref descriptors，
rebuild positive-ref descriptors，
也会 reset smgr cache 和重新读取 relation mapping。
在 phase 2 中，
如果不在 transaction state，
或者 nailed relation 只有 nailed baseline ref，
源码会 `RelationInvalidateRelation()`。
否则尝试 `RelationRebuildRelation()`。
whole reset 最后会把所有 in-progress build 标 invalidated，
除非是 debug discard。
### 7.4 callback 层
typcache、plancache 等高层 cache 不直接成为 SI message payload。
它们注册 syscache / relcache callbacks。
基础层处理后，
callback 把“某个 catalog tuple 或 relid stale”翻译成：
typcache flags 清理；
cached plan 标 stale；
partition / opclass 派生状态失效。
这降低 shared invalidation message 的耦合。
代价是 receiver 本地 callback fan-out。
## 8. transaction cleanup ordering
### 8.1 main transaction commit
`xact.c` 的 `CommitTransaction()` 中，
本节关心的顺序是：
```text
CurrentResourceOwner = NULL
ResourceOwnerRelease(TopTransactionResourceOwner, BEFORE_LOCKS, true, true)
AtEOXact_Buffers(true)
AtEOXact_RelationCache(true)
AtEOXact_TypeCache()
AtEOXact_Inval(true)
ResourceOwnerRelease(TopTransactionResourceOwner, LOCKS, true, true)
ResourceOwnerRelease(TopTransactionResourceOwner, AFTER_LOCKS, true, true)
```
relcache ref 是 `BEFORE_LOCKS` resource。
因此 `AtEOXact_RelationCache(true)` 看到的 relcache refcount 应回到正常非事务状态：
普通 relation 是 0；
nailed relation 是 1。
catcache tuple/list ref 是 `AFTER_LOCKS` resource。
如果 caller 忘记 release，
它会在 later phase 被兜底释放。
catcache invalidation 依靠 `dead` 和 refcount 可以跨过这个顺序。
`AtEOXact_Inval(true)` 在释放 locks 前。
源码注释明确说：
catalog changes 要在等待 relation lock 的 backend 开始使用 relation 之前可见。
commit 时，
`AtEOXact_Inval(true)` 把 prior 和 current invalidation messages 合并并发送到 shared invalidation queue。
本 backend 下一次 transaction start 也会通过 `AcceptInvalidationMessages()` 读到它们。
### 8.2 main transaction abort
`AbortTransaction()` 中对应顺序是：
```text
ResourceOwnerRelease(TopTransactionResourceOwner, BEFORE_LOCKS, false, true)
AtEOXact_Buffers(false)
AtEOXact_RelationCache(false)
AtEOXact_TypeCache()
AtEOXact_Inval(false)
ResourceOwnerRelease(TopTransactionResourceOwner, LOCKS, false, true)
ResourceOwnerRelease(TopTransactionResourceOwner, AFTER_LOCKS, false, true)
```
`AtEOXact_RelationCache(false)` 必须在 invalidation 前。
源码注释说明：
abort 时不应尝试 rebuild invalidated cache entries，
因为不能安全访问数据库。
所以先把 relcache refcount reset 到正常状态，
再处理 pending invalidation。
`AtEOXact_Inval(false)` 不发送消息给其它 backend。
其它 backend 看不到本事务未提交 catalog change。
但本 backend 可能已经在 prior command boundary 处理过 local invalidation，
并构造了基于失败事务的 cache state。
abort 需要 locally process `PriorCmdInvalidMsgs`。
`CurrentCmdInvalidMsgs` 尚未 touch caches，可以忘掉。
### 8.3 command boundary
`CommandEndInvalidationMessages()` 在 `CommandCounterIncrement()` 后运行。
它本地处理 `CurrentCmdInvalidMsgs`，
然后把它们 append 到 `PriorCmdInvalidMsgs`。
这解释了同一事务内 DDL 后下一条 command 能看到新 metadata：
不是因为 shared invalidation 已经通知其它 backend，
而是因为本 backend 在 command boundary 清掉了旧 local cache。
### 8.4 subtransaction
subcommit：
```text
ResourceOwnerRelease(subowner, BEFORE_LOCKS, true, false)
AtEOSubXact_RelationCache(true, mySubid, parentSubid)
AtEOSubXact_TypeCache()
AtEOSubXact_Inval(true)
ResourceOwnerRelease(subowner, LOCKS, true, false)
ResourceOwnerRelease(subowner, AFTER_LOCKS, true, false)
```
`AtEOSubXact_Inval(true)` 会先调用 `CommandEndInvalidationMessages()`，
再把本层 prior messages 挂到 parent 的 prior list。
已经本地处理过的 invalidation 继续成为父事务的问题。
subabort：
```text
ResourceOwnerRelease(subowner, BEFORE_LOCKS, false, false)
AtEOSubXact_RelationCache(false, mySubid, parentSubid)
AtEOSubXact_TypeCache()
AtEOSubXact_Inval(false)
ResourceOwnerRelease(subowner, LOCKS, false, false)
ResourceOwnerRelease(subowner, AFTER_LOCKS, false, false)
```
`AtEOSubXact_Inval(false)` locally process 本层 prior messages，
并丢弃 current messages。
PL/pgSQL exception block 是典型触发器。
内层子事务创建对象、lookup metadata、随后 ERROR 回滚。
外层事务继续运行时，
future lookup 不能继续相信内层失败对象。
## 9. 生命周期 / ownership / cleanup
### 9.1 catcache tuple ref
谁创建：
`SearchCatCacheMiss()` 在 miss path 中创建 `CatCTup`。
谁持有：
caller 持有一个 ref。
`CurrentResourceOwner` 记录这个 ref。
`CatCList` 也可能间接持有 member entry。
谁释放：
正常路径 `ReleaseSysCache()` / `ReleaseCatCache()`。
ERROR / abort 兜底路径 `ResourceOwnerRelease(... AFTER_LOCKS ...)`。
长期如何失效：
`CatCacheInvalidate()` 标 `dead` 或删除。
whole reset 批量标记或删除。
使用边界：
释放 catcache ref 后不能继续用 `HeapTuple` 内部指针。
需要长期保存时 copy 需要的字段。
### 9.2 catcache list ref
谁创建：
`SearchCatCacheList()`。
谁持有：
caller 持有 list ref。
list 关联 member entry。
owner 记录 list ref。
谁释放：
正常路径 `ReleaseCatCacheList()`。
ERROR / abort 兜底路径 catcache-list ResourceOwner callback。
长期如何失效：
当前源码对同 cache 的所有 lists 粗粒度 invalidation。
有 ref 的 list 标 dead。
无 ref 的 list 删除。
### 9.3 relcache relation ref
谁创建：
`RelationBuildDesc()`、relcache init、nailed relation initialization。
谁持有：
caller 通过 `RelationIdGetRelation()` / `relation_open()` 持有 ref。
owner 记录 relation ref。
nailed relation 有 baseline ref。
谁释放：
正常路径 `RelationClose()`。
ERROR / abort 兜底路径 `ResourceOwnerRelease(... BEFORE_LOCKS ...)`。
长期如何失效：
`RelationCacheInvalidateEntry()`、`RelationCacheInvalidate()`、init file invalidation。
EOX cleanup：
`AtEOXact_RelationCache()` 主要做 debug cross-check 和特殊 relation cleanup。
源码注释说 PG 8.1 之后 relcache refcount 应由 ResourceOwner 释放。
### 9.4 half-built state
ResourceOwner 只知道成功登记的资源。
半成品 cache entry 还需要模块自己的 cleanup。
catcache 用 `catcache_in_progress_stack`。
relcache 用 `in_progress_list`。
list build 用局部 `PG_TRY` / `PG_CATCH` 保护临时 member ref。
因此不要把 ResourceOwner 当成万能 finally。
它只覆盖 owner 认识的资源。
## 10. 错误路径 / 异常路径 / fallback
### 10.1 lookup 成功后 ERROR
场景：
```text
tuple = SearchSysCache1(...)
  -> refcount++
  -> remember owner
later function ERROR
abort cleanup
  -> ResourceOwnerRelease(AFTER_LOCKS)
  -> ResOwnerReleaseCatCache()
  -> refcount--
```
这就是为什么很多内核代码不为每次 syscache lookup 写局部 `PG_TRY`。
前提是 lookup API 正确登记 owner。
### 10.2 relation open 后 ERROR
场景：
```text
rel = relation_open(...)
  -> rd_refcnt++
  -> remember owner
later function ERROR
abort cleanup
  -> ResourceOwnerRelease(BEFORE_LOCKS)
  -> ResOwnerReleaseRelation()
  -> rd_refcnt--
  -> RelationCloseCleanup()
```
relcache ref 在 before-locks 阶段释放。
这使 `AtEOXact_RelationCache()` 能检查 refcount 是否回到正常状态。
### 10.3 构造过程中 ERROR
miss path 或 build path 中可能 ERROR：
catalog scan 失败。
allocation 失败。
toast fetch 失败。
打开 index 失败。
callback 或 helper 失败。
这些状态可能还没登记到 ResourceOwner。
因此模块必须用 in-progress 和局部 cleanup 保证半成品不污染 cache。
诊断时要区分：
成功返回给 caller 的 ref 泄漏；
构造中半成品被 ERROR 打断；
invalidation 到达导致 build 结果 stale。
### 10.4 active catcache ref 遇到 invalidation
场景：
```text
Session A holds CatCTup ref
Session B commits DDL
Session A receives catcache invalidation
```
如果 `ct->refcount > 0`，
receiver 不能删除 entry。
它设置 `ct->dead = true`。
当前 holder 仍有指针安全。
future lookup 跳过 dead entry。
holder release 后，
dead 且无 list ref 的 entry 被删除。
### 10.5 active relcache ref 遇到 invalidation
场景：
```text
Session A holds Relation *
Session B commits DDL
Session A receives relcache invalidation
```
relcache 不能释放 active descriptor。
它可能 flush 子状态、rebuild、invalidate 或延迟 clear。
常见 stale metadata bug 不是“其它 backend free 了我的指针”。
更常见的是某个路径在应该重新 lookup / retry / rebuild 的边界继续使用旧派生事实。
### 10.6 transaction abort
DDL 在本 backend 内可能已经过 command boundary：
```sql
BEGIN;
CREATE TABLE abort_demo(a int);
SELECT 'abort_demo'::regclass;
ROLLBACK;
```
`SELECT` 可能让本 backend 构造关于 `abort_demo` 的 cache。
ROLLBACK 后这些 metadata 不能继续作为 committed fact。
abort path 中：
`AtEOXact_Inval(false)` locally process prior messages。
`CurrentCmdInvalidMsgs` 还没本地处理，可以忘掉。
其它 backend 不需要收到消息。
### 10.7 subtransaction abort
PL/pgSQL exception block 常见：
```sql
BEGIN;
DO $$
BEGIN
  BEGIN
    CREATE TABLE subxact_cache_demo(a int);
    PERFORM 'subxact_cache_demo'::regclass;
    RAISE EXCEPTION 'rollback subxact';
  EXCEPTION WHEN others THEN
    NULL;
  END;
END $$;
SELECT to_regclass('subxact_cache_demo');
ROLLBACK;
```
预期 `to_regclass` 返回 `NULL`。
因为 subabort cleanup 清掉本 backend 已受内层失败事务影响的 cache state。
### 10.8 invalidation storm
invalidation storm 不是特殊代码路径。
它是正常 invalidation 在高频 DDL / temp table churn / partition churn / extension script 下的退化。
常见表现：
catcache entry 频繁 dead / remove。
catcache lists 粗粒度失效。
relcache entry 频繁 flush / rebuild。
typcache flags 清理。
plancache 标 stale。
catalog snapshot 失效。
CPU 从 hit path 转移到 miss / rebuild / callback path。
如果消息太多或 receiver 落后，
系统倾向于 coarse reset。
重建成本高，
但 correctness 更重要。
### 10.9 relcache init file cleanup
relcache init file 优化系统 relation descriptor 加载。
影响 init file 中 relation 的 DDL commit 时，
源码在发送 SI messages 前后处理 init file invalidation。
`RelationCacheInitFilePreInvalidate()` 会拿 `RelCacheInitLock` 并 unlink init file。
如果 unlink 遇到非 `ENOENT` 错误，
可以 ERROR。
此时事务仍可 abort，
并走资源与 invalidation cleanup。
## 11. 正确性机制层次
| 层次 | 机制 | 保证 | 不保证 |
| --- | --- | --- | --- |
| 指针生命周期 | refcount、`rd_refcnt`、MemoryContext | holder 不悬挂 | metadata 最新 |
| ERROR-safe release | `ResourceOwner` | longjmp 后归还已登记 ref | 半成品自动一致 |
| 语义失效 | `dead`、`rd_isvalid` | future lookup 不相信旧 entry | 当前 holder 立即重跑 |
| 消息传播 | shared invalidation | commit 后其它 backend 得知 stale | 发送新 descriptor |
| 本地顺序 | command / xact boundary | pending state 在安全点推进 | 每条 C 语句同步 |
| 并发互斥 | heavyweight lock | DDL / DML 对象级排序 | cache memory safety |
| 可见性 | MVCC / command id | catalog tuple visibility | cache hit 每次重做 MVCC |
为什么不能只靠 ResourceOwner：
它只释放引用，
不知道 catalog tuple 是否被修改。
为什么不能只靠 invalidation：
它只能标 stale 或删除无人持有的 entry，
不能释放 holder 的 ref。
为什么不能只靠 locks：
locks 排序对象级并发，
但不释放 catcache tuple，
也不清 typcache / plancache 派生状态。
为什么不每次 hit 重验证：
那会让 cache 退化成 catalog scan wrapper。
PostgreSQL 把成本移到 DDL / invalidation boundary。
## 12. 成本、资源与跨模块传播
catcache hit 成本：
hash 计算。
bucket 遍历。
key equality。
refcount 增减。
ResourceOwner remember / forget。
catcache miss 成本：
catalog index open。
系统表 scan。
tuple copy。
可能 detoast。
negative entry 构造。
MemoryContext allocation。
relcache rebuild 成本：
读 `pg_class`。
读 `pg_attribute`。
读 index metadata。
读 rule / trigger / partition / AM 派生信息。
清理和重建多个子对象。
ResourceOwner 成本随这些变量增长：
事务内打开 relation 数。
持有 catcache ref 数。
catcache list ref 数。
子事务层级。
ERROR 后集中 cleanup 的资源数。
invalidation fan-out 变量：
backend 数。
每个 backend 已初始化的 cache 数。
relcache entry 数。
高层 callback 数。
DDL 频率。
partition / index / type / opclass 数。
跨模块连接：
| 模块 | 连接点 | 本节边界 |
| --- | --- | --- |
| MemoryContext | cache entry 内存 | 不归还 refcount |
| ResourceOwner | ERROR-safe ref cleanup | 不判断 stale |
| xact | cleanup ordering | 不理解 entry 内容 |
| inval | pending/shared message | 不释放其它 backend 对象 |
| lock manager | DDL/DML 排序 | 不替代 invalidation |
| typcache | callback 清派生 flags | 不拥有 catcache tuple |
| plancache | callback 标 plan stale | 不直接处理 SI payload |
| catalog snapshot | scan path snapshot | 不保护 cache pointer |
涉及 shared state 时，
普通 backend 是主要推进者。
DDL backend 在 command / commit boundary 登记和发送 invalidation。
其它 backend 在 transaction start、command boundary、lock wait 返回等安全点接收。
recovery replay 也可能推进 invalidation 相关处理。
## 13. 观测与诊断入口
### 13.1 SQL 和系统视图能看到什么
能看到：
DDL 后下一条 DML 是否看到新 metadata。
prepared statement 是否重新规划或报错。
`cache lookup failed`、`relation does not exist`、cached plan 失效类错误。
`pg_stat_activity` 的 lock wait。
`pg_locks` 的 DDL / DML 锁冲突。
`pg_stat_database` 的 abort 增量。
看不到：
某个 `CatCTup.dead`。
某个 `RelationData.rd_isvalid`。
某个 backend 的 ResourceOwner 内部 ref 列表。
每个 invalidation callback 的精确 fan-out。
### 13.2 gdb 断点
catcache 断在 `SearchCatCacheInternal`、`ReleaseCatCache`、`CatCacheInvalidate`、`CatCacheRemoveCTup`、`ResOwnerReleaseCatCache`。
观察 `ct->refcount`、`ct->dead`、`ct->negative`、`ct->c_list`。
relcache 断在 `RelationIdGetRelation`、`RelationClose`、`RelationCacheInvalidateEntry`、`RelationFlushRelation`、`AtEOXact_RelationCache`、`ResOwnerReleaseRelation`。
观察 `rd_refcnt`、`rd_isvalid`、`rd_isnailed`、`rd_id`。
transaction / invalidation 断在 `ResourceOwnerRelease`、`AtEOXact_Inval`、`AtEOSubXact_Inval`、`CommandEndInvalidationMessages`、`AcceptInvalidationMessages`。
观察 release phase、commit/abort 参数、`CurrentResourceOwner`、prior/current invalidation group。
### 13.3 临时日志和计数器
实验库可临时记录：
`CatCacheInvalidate()` 中 mark dead vs remove。
`ReleaseCatCache()` 中 dead + refcount zero。
`RelationFlushRelation()` 中 clear / rebuild / invalidate。
`AtEOXact_RelationCache()` 中 nonzero refcount warning。
`ResourceOwnerRelease()` 中 resource kind 和 phase。
invalidation storm 下日志会改变时序。
只在实验库或单 backend 中做。
### 13.4 perf
CPU profile 中可能看到 `SearchCatCacheInternal`、`SearchCatCacheMiss`、`CatCacheInvalidate`、`RelationIdGetRelation`、`RelationBuildDesc`、`RelationFlushRelation`、`LocalExecuteInvalidationMessage`、`PlanCacheRelCallback`。
有高频 DDL / temp table churn 时这些函数上升是合理结果。
普通 OLTP 无 DDL 却持续出现在顶部时，再查 extension、prepared statement invalidation、schema churn 或 ref 泄漏。
### 13.5 诊断顺序
看到 stale metadata 或 cache lookup 错误时：
1. 是否有并发 DDL、temp table churn、partition churn、extension script？
2. 是否有长事务长时间不处理 invalidation？
3. 是否有 lock wait 后 name lookup 未 retry？
4. 是否有 C extension 缓存 `HeapTuple` / `Relation *` 超过生命周期？
5. 是否有自定义资源未登记 ResourceOwner？
6. 是否有 subtransaction exception block 构造 metadata 后 abort？
7. 是否是 plan cache / typcache 派生状态没被 callback 清理？
不要先假设 shared invalidation 丢消息。
多数问题来自生命周期边界错误或 workload 制造的高频失效。
## 14. 课堂实验
### 实验 1：catcache ref 在 ERROR 后由 ResourceOwner 释放
在临时源码分支中写测试函数：`SearchSysCache1(TYPEOID, ...)` 后直接 `elog(ERROR, ...)`，不调用 `ReleaseSysCache()`。
断在 `SearchCatCacheInternal`、`ResOwnerReleaseCatCache`、`ReleaseCatCache`、`ResourceOwnerRelease`。
预期：abort path 进入 `ResourceOwnerRelease(... AFTER_LOCKS ...)`，`ResOwnerReleaseCatCache()` 调用 `ReleaseCatCacheWithOwner(tuple, NULL)`，`ct->refcount` 被减回。
解释：MemoryContext 不释放 ref，ResourceOwner 才是兜底。
### 实验 2：active catcache ref 遇到 invalidation
在 `CatCacheInvalidate()` 和 `ReleaseCatCache()` 加临时日志。
Session A 命中某个 `pg_class` 或 `pg_type` syscache，并在释放 tuple 前停住；Session B 执行相关 DDL 并 commit。
预期：active `CatCTup` 被标 `dead`，A 释放后 dead 且 refcount 为零的 entry 才能删除；list 会被同 cache 粗粒度 invalidated，tuple 仍按 hash 匹配。
### 实验 3：relcache ref 的 before-locks cleanup
断在 `RelationIncrementReferenceCount`、`ResOwnerReleaseRelation`、`AtEOXact_RelationCache`、`ResourceOwnerRelease`。
在函数或调试会话中打开 relation 后触发 ERROR。
预期：`ResourceOwnerRelease(... BEFORE_LOCKS ...)` 先调用 relcache release callback，随后 `AtEOXact_RelationCache(false)` 看到 refcount 回到正常状态。
### 实验 4：relcache invalidation 下 `rd_isvalid`
断在 `RelationCacheInvalidateEntry`、`RelationFlushRelation`、`RelationCacheInvalidate`、`RelationClose`。
Session A 打开或访问目标 relation；Session B 执行 `ALTER TABLE`；让 A 在安全点处理 invalidation。
预期：active descriptor 不能直接释放，relcache 会根据状态标 invalid、rebuild 或延迟 cleanup。
### 实验 5：subtransaction abort 清 local cache
SQL：
```sql
BEGIN;
DO $$
BEGIN
  BEGIN
    CREATE TABLE subxact_cache_demo(a int);
    PERFORM 'subxact_cache_demo'::regclass;
    RAISE EXCEPTION 'rollback subxact';
  EXCEPTION WHEN others THEN
    NULL;
  END;
END $$;
SELECT to_regclass('subxact_cache_demo');
ROLLBACK;
```
预期：
`to_regclass` 返回 `NULL`。
gdb 中可观察 `AtEOSubXact_Inval(false)` 和 `AtEOSubXact_RelationCache(false, ...)`。
### 实验 6：invalidation storm CPU 画像
Session A 反复执行依赖 relation open、type lookup 或 prepared plan 的查询；Session B 高频创建 / 删除临时表或执行 partition DDL。
用 perf 采样，预期看到 catcache lookup、relcache build、invalidation processing 或 plan callback 成本上升。
## 15. 常见误区
误区一：认为 ResourceOwner 会让 cache entry 失效。纠正：它释放引用；失效由 invalidation 和 flags 表达。
误区二：认为 `dead=true` 表示当前 holder 不能读 tuple。纠正：它表示 future lookup 不应返回；pointer safety 仍由 refcount 保证。
误区三：认为 `rd_refcnt>0` 表示 relation metadata 最新。纠正：`rd_refcnt` 是生命周期；`rd_isvalid` 才是 freshness 线索。
误区四：认为 abort 会清空所有 catcache / relcache。纠正：abort 释放 owner 资源、处理 local invalidation、清 in-progress state，不无条件清空 cache。
误区五：认为 invalidation 是 lock。纠正：lock 排序 DDL / DML；invalidation 清 stale local fact。
误区六：认为 shared invalidation 释放其它 backend 对象。纠正：消息只让 receiver backend 处理自己的本地 cache。
误区七：认为 MemoryContext reset 足够处理 ERROR。纠正：cache ref、relation ref、buffer pin、snapshot 都需要 ResourceOwner。
误区八：认为 invalidation storm 一定是 bug。纠正：它可能只是 schema churn 的自然成本。
误区九：C extension 长期保存 `HeapTuple` 或 `Relation *`。纠正：这些是 backend-local 内部指针；长期保存应保存 OID、name 或自己的 copy。
误区十：认为 `pg_stat_*` 能直接证明 catcache ref 泄漏。纠正：多数内部状态需要 gdb、临时日志、assert 或 perf。
## 16. 讨论题
1. 为什么 catcache invalidation 遇到 `refcount>0` 时标 `dead`，而不是等待 refcount 归零？
2. 为什么 `CatCTup.refcount` 不能说明 tuple 语义仍然新鲜？
3. `ResourceOwnerRelease()` 能兜底释放 ref，为什么正常路径仍必须 `ReleaseSysCache()` / `RelationClose()`？
4. relcache 为什么需要 `rd_isvalid`，而不是只靠 `rd_refcnt`？
5. abort 时为什么要 locally process prior invalidation messages？
6. subtransaction abort 后，哪些 cache state 不能继续留给外层事务相信？
7. invalidation storm 下为什么 coarse reset 比 missed invalidation 更可接受？
8. extension 缓存 `HeapTuple` / `Relation *` 超过生命周期会破坏哪些不变量？
9. 为什么 relcache refs 要在 before-locks phase 释放？
10. 如何区分 cache ref 泄漏和 invalidation storm 导致的 rebuild 成本上升？
## 17. 本节小结
本节唯一主问题是：
ERROR、transaction abort 和 invalidation 风暴下，cache ref、relcache entry 和 ResourceOwner 如何收尾？
核心链路：
```text
SearchSysCache / RelationIdGetRelation
  -> ResourceOwnerEnlarge()
  -> refcount++
  -> ResourceOwnerRemember()
  -> caller uses backend-local pointer
  -> normal Release or ResourceOwner abort cleanup
  -> invalidation marks dead / rd_isvalid=false
  -> refcount reaches zero
  -> physical removal / rebuild / delayed cleanup
```
catcache 的核心状态是 `CatCTup.refcount`、`CatCTup.dead`、`CatCTup.negative` 和 `CatCList.refcount/dead`。
当前源码中 tuple 按 hash invalidation，
list 在同 cache 内粗粒度 invalidation。
catcache ref 的 ResourceOwner release phase 是 `AFTER_LOCKS`。
relcache 的核心状态是 `RelationData.rd_refcnt`、`rd_isvalid`、`rd_isnailed` 和 `in_progress_list`。
relcache ref 的 ResourceOwner release phase 是 `BEFORE_LOCKS`。
`AtEOXact_RelationCache()` 在 invalidation 前运行，
主要做 refcount cross-check、in-progress cleanup 和事务创建/删除 relation 的特殊 cleanup。
ERROR path 的关键不是每段代码都写 finally。
成功 acquire 的资源必须进入 ResourceOwner。
半成品构造仍需要 catcache / relcache 自己的 in-progress cleanup。
abort path 的关键不是清空所有 cache。
它释放本事务 owner 资源，
处理本 backend 已经受失败事务影响的 prior invalidation，
并丢弃尚未 touch cache 的 current invalidation。
invalidation storm 的关键不是跨 backend 释放对象。
它是大量 backend-local dead marking、flush、callback、rebuild 和 coarse reset。
正确性允许过度 invalidation；
不允许 missed invalidation。
能直接观测的是 DDL 可见性、lock wait、abort、错误、plan 失效和 CPU profile。
难以直接观测的是 `CatCTup.dead`、`rd_isvalid`、ResourceOwner 内部列表和 callback fan-out。
可迁移规律：
高性能本地 cache 不应把“对象能否解引用”和“对象语义是否新鲜”合成一个状态。
用 refcount / owner 管生命周期；
用 invalidation / valid bit 管语义；
用 command / transaction boundary 管顺序；
用 ERROR-safe cleanup 让异常路径不依赖普通返回。
