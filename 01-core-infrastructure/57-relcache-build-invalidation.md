# PostgreSQL relcache build invalidation
## 课程定位
前置知识：已经理解 `MemoryContext`、`ResourceOwner`、`PGPROC`、heavyweight lock、catcache/syscache 的基本作用。
本节唯一主问题：
```text
relcache 如何从 pg_class、pg_attribute、index、rule、partition 信息构造 Relation，
shared invalidation 如何让旧描述失效？
```
核心矛盾：执行器、优化器和存储层希望通过一个稳定的 `Relation *` 快速访问表结构、索引、规则、触发器、分区和物理文件信息；但这些信息来自可被 DDL 修改的系统目录，而且每个 backend 都有自己的本地指针，不能把指针直接共享给其它进程。
一句话运行模型：
```text
Relation 是 backend-local 的长寿命描述符；
构造时在持有 relation lock 的前提下从多个 catalog 拼出一致可用的本地对象；
shared invalidation 只传播“哪个 relid 过期”，各 backend 在安全边界本地清理、重建或延迟标记。
```
学完后应能判断：
- `Relation` 里哪些字段来自 `pg_class`，哪些来自 `pg_attribute`、`pg_index`、`pg_rewrite`、partition catalog。
- 为什么 relcache entry 是 backend-local，而失效消息走 shared invalidation。
- 为什么一个 open relation 不能直接释放，只能 in-place rebuild 或标记 invalid。
- 为什么 refcount 只能保证指针安全，不能保证元数据语义仍然新鲜。
- 为什么同一个 DDL 既可能刷新 relcache，也可能传播到 plan cache、typcache、relfilenumber cache。
- 如何从 SQL 现象、gdb 断点、日志和源码入口定位 stale metadata 问题。
本课基于本地源码 `/home/nail/postgres`，分支 `master`，提交 `0e1f1ed157e9`。
## 1. 本节在总主线中的位置
前面课程已经讲过三类基础设施。
`MemoryContext` 管 backend-local 内存生命周期。
`ResourceOwner` 管 refcount、pin、lock 等外部资源的 ERROR-safe cleanup。
heavyweight lock 管 relation 级并发语义。
relcache 把这些基础设施组合到 catalog/cache 层。
它不是一个共享哈希表。
每个 backend 都有自己的 `RelationIdCache`。
`RelationData` 和它的子结构主要活在 `CacheMemoryContext`。
其它 backend 不能解引用这个地址。
但是 catalog 更新是全局事实。
一个 backend 执行 `ALTER TABLE` 后，别的 backend 的旧 `RelationData` 必须失效。
这就是本节要追的链路：
```text
relation_open/table_open
  -> RelationIdGetRelation()
  -> RelationBuildDesc()
  -> pg_class / pg_attribute / pg_index / pg_rewrite / partition 信息
  -> RelationData in CacheMemoryContext
  -> DDL 更新 catalog
  -> CacheInvalidateRelcache*()
  -> CommandEndInvalidationMessages()
  -> AtEOXact_Inval()
  -> SendSharedInvalidMessages()
  -> AcceptInvalidationMessages()
  -> LocalExecuteInvalidationMessage()
  -> RelationCacheInvalidateEntry()
  -> RelationFlushRelation()
```
本节不把 catcache 展开成完整课程。
syscache/catcache 在这里主要是 relcache build 的 catalog tuple 入口，以及 shared invalidation 的相邻消费者。
本节也不讲 dependency 记录如何决定 DROP CASCADE。
dependency 是对象生命周期规则。
relcache invalidation 是“已经发生或即将可见的 catalog 变化如何让本地描述符过期”。
## 2. 核心矛盾与一句话运行模型
relcache 的 tension 可以压缩成一句话：
```text
Relation 指针必须像普通 C 结构一样便宜稳定；
但它代表的 catalog 语义必须跟随事务可见性和 DDL 变化刷新。
```
如果每次访问表都重新扫 `pg_class`、`pg_attribute`、`pg_index`，普通 DML 会被 catalog lookup 淹没。
如果只在首次打开时构造一次 `Relation`，后续 `ALTER TABLE`、`CREATE INDEX`、`DROP RULE`、`ATTACH PARTITION` 会让优化器和执行器读到旧结构。
PostgreSQL 的选择是四层协作。
| 层次 | 位置 | 作用 |
| --- | --- | --- |
| `RelationIdCache` | backend-local hash | relid 到 `RelationData *` 的本地索引。 |
| `RelationData` | `CacheMemoryContext` | 当前 backend 对某个 relation 的描述符。 |
| `rd_refcnt` + `ResourceOwner` | backend-local | 保证正在使用的 `Relation *` 不被释放，并在 ERROR/abort 时释放引用。 |
| shared invalidation | shared queue + local handlers | 传播“哪个 cache 语义过期”，不传播本地指针或完整结构。 |
这四层回答不同问题。
`RelationIdCache` 回答“我是否已经有这个 relid 的本地描述符”。
`RelationData` 回答“这张表/索引当前在本 backend 中被解释成什么结构”。
`rd_refcnt` 回答“是否还有代码正在持有这个指针”。
shared invalidation 回答“哪些描述符的语义不应继续被当成新鲜事实”。
不要把它们混成一个“缓存是否存在”的布尔值。
一个 entry 可以存在但 `rd_isvalid = false`。
一个 entry 可以 `rd_refcnt > 0`，同时已经收到失效消息。
一个 nailed entry 不能从 hash 里删除，但仍可能需要 reload `pg_class` 的部分字段。
一个 heap relation 的 `rd_indexlist` 可能还没构造，或者已经被 invalidation 标成需要重算。
本课的核心不变量是：
```text
指针安全由 refcount/ResourceOwner 保证；
语义新鲜由 invalidation/rebuild 保证；
catalog 一致性由 relation lock、command boundary 和事务提交顺序共同保证。
```
## 3. 核心文件分工与阅读顺序
| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/utils/rel.h` | `RelationData` 字段、状态边界、哪些子结构属于 lazy cache。 |
| 2 | `src/include/utils/relcache.h` | relcache 对外 API、init file 名称、startup/init/flush 接口。 |
| 3 | `src/backend/utils/cache/relcache.c` | `RelationIdGetRelation()`、`RelationBuildDesc()`、rebuild、flush、init file、EOX cleanup。 |
| 4 | `src/backend/utils/cache/inval.c` | catalog tuple 变化如何注册 relcache inval，command/transaction boundary 如何发送和本地处理。 |
| 5 | `src/include/storage/sinval.h` | `SharedInvalidationMessage` 类型和 relcache message 格式。 |
| 6 | `src/backend/utils/cache/syscache.c` | syscache lookup 与 `AcceptInvalidationMessages()` 的边界。 |
| 7 | `src/backend/utils/cache/partcache.c` | `RelationGetPartitionKey()`、partition qual 的 relcache 子缓存。 |
| 8 | `src/backend/partitioning/partdesc.c` | `RelationGetPartitionDesc()` 如何构造并缓存 partition descriptor。 |
| 9 | `src/backend/utils/cache/plancache.c` | plan cache 如何注册 relcache callback 并标记 cached plan 失效。 |
| 10 | `src/backend/catalog/index.c` | index 创建/删除为什么显式 invalidate heap relation 的 index list。 |
| 11 | `src/backend/rewrite/rewriteDefine.c`、`rewriteRemove.c` | rule 变化如何显式刷新 relation 的 `rd_rules`。 |
| 12 | `src/backend/commands/trigger.c`、`policy.c`、`publicationcmds.c` | trigger、RLS、publication 信息如何传播到 relcache。 |
| 13 | `src/backend/access/transam/xact.c` | `CommandCounterIncrement()`、commit/abort 顺序中 relcache 与 inval 的 cleanup。 |
推荐阅读顺序：
```text
rel.h 的 RelationData
  -> relcache.c 的 RelationIdGetRelation()
  -> relcache.c 的 RelationBuildDesc()
  -> relcache.c 的 RelationRebuildRelation() / RelationFlushRelation()
  -> inval.c 的 CacheInvalidateHeapTupleCommon() / CacheInvalidateRelcache()
  -> inval.c 的 CommandEndInvalidationMessages() / AtEOXact_Inval()
  -> partcache.c / partdesc.c 的 lazy partition cache
  -> plancache.c 的 callback 传播
```
不要从 `pg_internal.init` 文件格式开始读。
init file 是 startup optimization 和 bootstrap fallback。
主线仍然是运行期 `RelationData` 如何构造、持有、失效和重建。
## 4. 关键数据结构与状态
### 4.1 `RelationData` 不是 `pg_class` 的简单拷贝
`RelationData` 定义在 `src/include/utils/rel.h`。
它是一个 backend-local 结构。
核心字段可以按语义分组。
| 字段组 | 语义 |
| --- | --- |
| `rd_id`、`rd_rel`、`rd_att` | relation OID、`pg_class` 固定字段拷贝、tuple descriptor。 |
| `rd_locator`、`rd_smgr`、`rd_backend` | 物理文件定位、storage manager handle、temp relation backend identity。 |
| `rd_refcnt`、`rd_isvalid`、`rd_isnailed` | 引用计数、语义是否有效、是否 nailed in cache。 |
| `rd_createSubid`、`rd_newRelfilelocatorSubid`、`rd_firstRelfilelocatorSubid`、`rd_droppedSubid` | 当前事务/子事务中创建、换 relfilenode、drop 的 cleanup 状态。 |
| `rd_rules`、`trigdesc`、`rd_rsdesc` | rewrite rules、triggers、RLS policies。 |
| `rd_index`、`rd_indextuple`、`rd_indexcxt`、`rd_indam` | index relation 自己的 `pg_index` 和 index AM 信息。 |
| `rd_indexlist`、`rd_pkindex`、`rd_replidindex` | heap relation 上的 index 列表和主键/replica identity 缓存。 |
| `rd_partkey`、`rd_partdesc`、`rd_partdesc_nodetached` | partition key、partition descriptor 及 omit-detached 版本。 |
| `rd_options`、`rd_amcache`、`rd_fdwroutine`、`rd_pubdesc` | reloptions、AM/FDW/publication 等附属缓存。 |
`rd_rel` 只保存 `pg_class` 的 fixed-size 部分。
`pg_class.relacl`、原始 `reloptions` 这类 variable-length 字段不在 `rd_rel` 中可靠可读。
reloptions 会被解析后放入 `rd_options`。
这就是 `rel.h` 注释里强调的边界：
需要 ACL 时应走 syscache，而不是从 `rd_rel` 猜。
`rd_att` 是 `TupleDesc`。
它来自 `pg_attribute`，还会补上 default、missing value、check 和 not-null constraint 相关信息。
`rd_att` 的 refcount 与 relcache entry 的生命周期交织。
重建 open relation 时，如果 tupledesc 可能被外部复制引用，旧 tupledesc 可能被延迟到事务结束再释放。
`rd_refcnt` 不是锁。
它只表示当前 backend 中有多少打开的引用。
其它 backend 看不到这个计数。
DDL 并发控制仍依赖 heavyweight relation lock。
`rd_isvalid` 也不是“能否解引用”的判断。
一个 `rd_isvalid = false` 的 entry 仍可能因为 `rd_refcnt > 0` 被保留，等待下次打开或当前安全边界重建。
### 4.2 子结构有不同的构造时机
`RelationBuildDesc()` 不会一次性把所有可能信息都装满。
PostgreSQL 有意把部分信息做成 lazy cache。
| 子结构 | 构造时机 | 原因 |
| --- | --- | --- |
| `rd_rel` | build 时立即从 `pg_class` 拷贝 fixed fields | 所有 relation 都需要。 |
| `rd_att` | build 时立即从 `pg_attribute` 构造 | executor、rewriter、planner 经常需要 tuple layout。 |
| index relation 的 `rd_index*` | index relation build 时立即构造 | 打开 index 本身必须知道 `pg_index` 和 AM。 |
| heap relation 的 `rd_indexlist` | `RelationGetIndexList()` 第一次请求时构造 | 不是所有访问都需要 index 列表。 |
| `rd_rules`、`trigdesc`、`rd_rsdesc` | `relhasrules`、`relhastriggers`、`relrowsecurity` 为真时 build | 由 `pg_class` 标志决定是否需要加载。 |
| `rd_partkey` | `RelationGetPartitionKey()` 第一次请求时构造 | 只有 partitioned table 需要。 |
| `rd_partdesc` | `RelationGetPartitionDesc()` 第一次请求时构造 | 成本随 partition 数增长，且有 snapshot 边界。 |
| `rd_partcheck` | `RelationGetPartitionQual()` 第一次请求时构造 | 主要用于 partition constraint。 |
lazy cache 的关键风险是指针外借。
`RelationGetIndexList()` 返回 caller context 中的 list copy。
因为 caller 随后可能做 syscache lookup，期间可能处理 SI message 并刷新 relcache。
`RelationGetPartitionDesc()` 则返回 relcache 内部指针。
因此 partdesc 有额外约束：调用者必须保持 relation open，并持有足够强的 relation lock，relcache 也会延迟释放旧 partdesc 上下文直到 refcount 归零。
### 4.3 `RelationIdCache` 与 `CacheMemoryContext`
`RelationCacheInitialize()` 创建 `RelationIdCache`。
key 是 relation OID。
value 是 `RelIdCacheEnt`，里面保存 `Relation reldesc`。
这个 hash 是 backend-local。
它不需要 LWLock。
它也不直接代表 shared catalog state。
`AllocateRelationDesc()` 会切换到 `CacheMemoryContext`。
`RelationData`、`rd_rel`、`rd_att` 的长期部分都在这个上下文中。
规则、index、partition 等复杂子结构通常还有自己的 child memory context。
这样 invalidation 时可以针对子结构整体删除，避免逐个释放深树节点。
但是构造过程中的工作内存不都应该永久留在 `CacheMemoryContext`。
`RelationBuildDesc()` 在 `debug_discard_caches` 或 `RECOVER_RELATION_BUILD_MEMORY` 情况下，会创建临时 workspace context。
这样反复丢 cache 的测试不会把临时构造垃圾无限留到事务结束。
### 4.4 `SharedInvalidationMessage`
`src/include/storage/sinval.h` 中的 shared invalidation message 是一个 union。
relcache message 使用：
```text
SharedInvalRelcacheMsg:
  id   = SHAREDINVALRELCACHE_ID
  dbId = database OID, or InvalidOid for shared relation
  relId = relation OID, or InvalidOid for whole relcache
```
这个消息不包含新 schema。
不包含 tupledesc。
不包含本地指针。
也不告诉 receiver 这次变化来自 `ADD COLUMN`、`DROP INDEX` 还是 `ALTER POLICY`。
receiver 只知道：
```text
如果我在这个 database 中有 relid 对应的本地 relcache entry，
它的语义过期了。
```
具体怎么处理由 receiver 本地状态决定。
zero refcount 的普通 entry 可以删除。
open entry 要重建或标 invalid。
nailed entry 不能删除，只能特殊 reload。
in-progress build 要被打断并重试。
### 4.5 relcache init file 是启动优化，不是共享 cache
`relcache.h` 定义 init file 名称：
```text
pg_internal.init
```
当前版本有 shared init file 和 per-database local init file。
shared 文件在 `global/pg_internal.init`。
local 文件在 database path 下。
init file 存的是一批可在 backend startup 直接恢复的 relcache entries。
它用于打破 bootstrap 循环：
```text
要用 pg_attribute 的 index 扫 pg_attribute；
但要打开这个 index，又需要 pg_attribute 的 relcache entry。
```
如果 init file 缺失或损坏，backend 会退回硬编码 critical catalog descriptor 或 heap scan 路径。
这会慢，但必须正确。
catalog 更新可能让 init file 过期。
因此 `RegisterRelcacheInvalidation()` 会在 invalidated relation 属于 init file 集合时设置 `RelcacheInitFileInval`。
commit 发送 SI message 前后，`RelationCacheInitFilePreInvalidate()` / `PostInvalidate()` 会在 `RelCacheInitLock` 下先 unlink init file，再发送消息，避免新 backend 读到旧文件后错过失效消息。
## 5. 主流程源码 walkthrough
### 5.1 打开 relation 的入口
上层通常不直接调用 `RelationBuildDesc()`。
常见入口是 `relation_open()`、`table_open()`、`index_open()` 一类 helper。
这些 helper 负责拿 relation lock，再通过 `RelationIdGetRelation()` 获取 `Relation *`。
`RelationIdGetRelation()` 的注释要求 caller 至少持有 `AccessShareLock`。
原因不是保护本地 hash。
原因是 build 需要跨多个 catalog 读取信息。
如果没有 relation lock，`pg_class`、`pg_attribute`、`pg_index` 可能在 build 中途被并发 DDL 改掉，拼出一个互相不匹配的 descriptor。
主入口行为是：
```text
RelationIdGetRelation(relid)
  -> AssertCouldGetRelation()
  -> RelationIdCacheLookup(relid)
  -> if found and dropped: return NULL
  -> if found: RelationIncrementReferenceCount()
  -> if found but !rd_isvalid: RelationRebuildRelation()
  -> if not found: RelationBuildDesc(relid, true)
  -> if build ok: RelationIncrementReferenceCount()
```
注意顺序。
命中 invalid entry 时先增加 refcount，再 rebuild。
这样 rebuild 期间不会因为某个 reset 路径把 entry 释放到悬空。
新建 entry 则在 build 完成、插入 hash 后才对 caller 增加 refcount。
### 5.2 `RelationBuildDesc()` 的总体骨架
`RelationBuildDesc()` 的职责是从 catalog 构造一个完整可用的 `RelationData`。
核心骨架是：
```text
RelationBuildDesc(targetRelId, insertIt)
  -> 注册 in_progress_list[targetRelId]
  -> ScanPgRelation(targetRelId)
  -> AllocateRelationDesc(pg_class tuple)
  -> 设置 rd_id / refcnt / temp persistence / subid 状态
  -> RelationBuildTupleDesc()
  -> 初始化 table/index AM
  -> RelationParseRelOptions()
  -> RelationBuildRuleLock() / RelationBuildTriggers() / RelationBuildRowSecurity()
  -> RelationInitLockInfo()
  -> RelationInitPhysicalAddr()
  -> 如果中途收到同 relid invalidation: destroy and retry
  -> RelationCacheInsert()
  -> rd_isvalid = true
```
`insertIt = true` 表示最终插入 `RelationIdCache`。
`insertIt = false` 常用于 in-place rebuild 的临时 descriptor。
临时 descriptor 不进 hash。
它构造完成后与旧 descriptor 交换内容。
### 5.3 从 `pg_class` 开始
`ScanPgRelation()` 根据 `pg_class.oid` 取目标 tuple。
在 critical relcaches 尚未建立时，它会避免使用还不能安全打开的 index。
这就是 `criticalRelcachesBuilt` 的意义。
它不是性能开关，而是 bootstrap recursion 边界。
`ScanPgRelation()` 返回 copy。
`AllocateRelationDesc()` 只复制 `CLASS_TUPLE_SIZE`。
它还创建一个空的 `TupleDesc` 模板，属性数量来自 `pg_class.relnatts`。
此时 `rd_att` 还没有填入各列的 `pg_attribute` 细节。
接下来 `RelationBuildDesc()` 设置 persistence。
永久和 unlogged relation 的 `rd_backend` 是 `INVALID_PROC_NUMBER`。
本 session 的 temp relation 会设置 `rd_backend = ProcNumberForTempRelations()` 且 `rd_islocaltemp = true`。
其它 session 的 temp relation 要通过 temp namespace 推断 owning backend。
这里要区分 `rd_backend` 和 `rd_islocaltemp`。
因为崩溃遗留的 temp namespace 可能碰巧和当前 ProcNumber 相同，但不能因此把它当作“我的 temp table”。
### 5.4 从 `pg_attribute` 构造 tuple descriptor
`RelationBuildTupleDesc()` 扫 `pg_attribute`。
scan key 是：
```text
attrelid = relation relid
attnum > 0
```
它使用 `AttributeRelidNumIndexId`，但只有在 critical relcaches built 后才安全使用 index scan。
每条 `pg_attribute` 填入 `TupleDescAttr(rd_att, attnum - 1)`。
然后调用 `populate_compact_attribute()`。
构造过程还收集：
- `attnotnull`。
- generated column 状态。
- `atthasdef`。
- `atthasmissing` 和 missing value。
- relation 的 check constraint 计数。
如果 catalog 里缺少预期属性，build 会 ERROR：
```text
pg_attribute catalog is missing ... attribute(s)
```
这不是普通 cache miss。
`pg_class.relnatts` 和 `pg_attribute` 不一致意味着 descriptor 无法安全构造。
default 来自 `pg_attrdef`。
check 和 invalid not-null 信息来自 `pg_constraint`。
如果 default 或 check 数量和 `pg_class` 里的计数不匹配，部分路径会 WARNING，而不是立即 ERROR。
原因是 executor 后续真正使用时仍会做约束检查。
但 relcache 不能凭一个半成品 `TupleDesc` 假装完全正确。
### 5.5 index relation 的 build 与 heap relation 的 index list
打开一个 index relation 时，`RelationBuildDesc()` 会调用 `RelationInitIndexAccessInfo()`。
这条路径读取：
- `pg_index` tuple，保存到 `rd_indextuple` 和 `rd_index`。
- `pg_am`，取得 AM handler。
- index AM routine。
- `pg_index.indcollation`。
- `pg_index.indclass`。
- `pg_index.indoption`。
- opclass/opfamily/support procedure 信息。
这些内容放在 `rd_indexcxt`。
这个 context 让 index AM support 信息能整体释放。
但是 heap relation 上“有哪些 index”不是 build 时立即加载。
`RelationGetIndexList()` 第一次请求时扫描 `pg_index` 的 `indrelid`。
它忽略 `indislive = false` 的 index。
它按 OID 排序返回。
排序不是美观问题。
executor 更新多个 index 时需要所有 backend 以一致顺序拿锁，降低 deadlock 风险。
`RelationGetIndexList()` 返回 caller context 中的 copy。
同时把 copy 存到 `relation->rd_indexlist`，并设置：
- `rd_pkindex`。
- `rd_ispkdeferrable`。
- `rd_replidindex`。
- `rd_indexvalid = true`。
index 创建/删除时，仅 invalidating index 自己不够。
heap relation 的 `rd_indexlist`、HOT-safety bitmap、replica identity 判断都依赖这组 index。
因此 `src/backend/catalog/index.c` 在建/删 index 的关键点显式调用 `CacheInvalidateRelcache(heapRelation)`。
### 5.6 index attribute bitmap 的 retry
`RelationGetIndexAttrBitmap()` 是一个重要边界。
它需要打开每个 index，收集：
- foreign key 可引用列。
- primary key 列。
- replica identity 列。
- HOT blocking 列。
- summarizing index 需要更新的列。
打开 index 期间可能处理 SI message，导致 heap relation 的 index list 被刷新。
所以源码先保存 `rd_pkindex` 和 `rd_replidindex`。
遍历结束后再次调用 `RelationGetIndexList()`。
如果 index OID list 或主键/replica identity OID 变化，就释放已构造 bitmap 并 `goto restart`。
这体现了 relcache 的典型模式：
```text
先用本地 cache 快速推进；
一旦过程中可能消费 invalidation，就在提交结果前重新校验输入集合。
```
### 5.7 rule、trigger、RLS 的装载
`RelationBuildDesc()` 根据 `pg_class` 的标志决定是否加载 rule、trigger、RLS。
`relhasrules` 为真时调用 `RelationBuildRuleLock()`。
它扫描 `pg_rewrite`，把 `ev_action`、`ev_qual` 的 text 反序列化成 Node tree。
这些树可能很大。
所以 rule 信息放在独立 `rd_rulescxt`。
`RelationBuildRuleLock()` 还会根据 view 的 `security_invoker` reloption 设置 rule tree 中的 `checkAsUser`。
这个动作必须在 load rule 时做。
否则 `ALTER TABLE OWNER` 一类操作需要重写存储的 rule tree。
trigger 由 `RelationBuildTriggers()` 构造。
RLS 由 `RelationBuildRowSecurity()` 构造。
它们的 invalidation 并不都来自 `CacheInvalidateHeapTupleCommon()` 的自动 relcache 分支。
`pg_rewrite`、trigger、policy 的修改路径会显式调用 `CacheInvalidateRelcache*()`。
例如 `rewriteDefine.c`、`rewriteRemove.c`、`trigger.c`、`policy.c` 都有这种显式调用。
这点很重要。
“relcache 从哪些 catalog 读信息”和“哪些 catalog update 自动推导 relcache inval”不是同一个集合。
### 5.8 partition key 与 partition descriptor
partition 信息分两类。
`RelationGetPartitionKey()` 在 `partcache.c` 中读取 `pg_partitioned_table`。
它构造 `PartitionKeyData`：
- partition strategy。
- partition key attnums 或 expressions。
- opclass/opfamily。
- support function。
- collation。
- key type 信息。
构造过程先把 context 挂在 `CurTransactionContext` 下。
只有成功后才 reparent 到 `CacheMemoryContext` 并赋给 `rd_partkeycxt` / `rd_partkey`。
这样中途 ERROR 不会把半成品长久挂到 relcache。
`RelationGetPartitionDesc()` 在 `partdesc.c` 中读取 `pg_inherits` 和每个 child 的 `pg_class.relpartbound`。
它维护两个 descriptor：
- `rd_partdesc`：包含所有 partition，包括正在 detach 的。
- `rd_partdesc_nodetached`：在特定 snapshot 语义下排除 detached partition。
后者还保存 `rd_partdesc_nodetached_xmin`。
调用者请求 omit detached 时，能否复用 cached descriptor 要看这个 xmin 是否仍不在当前 active snapshot 中。
这说明 partition descriptor 的正确性不仅是 relid 集合问题，还绑定 active snapshot。
并发 `ATTACH PARTITION` 和 `DETACH CONCURRENTLY` 会制造中间状态。
如果 syscache 看不到刚加入 partition 的 `relpartbound`，`RelationBuildPartitionDesc()` 会直接扫 `pg_class`。
如果仍然拿不到，源码会 `AcceptInvalidationMessages()` 并重试一次。
再失败就是 ERROR。
这是 partition relcache 的异常路径，不是普通 miss。
### 5.9 invalid entry 的 in-place rebuild
`RelationIdGetRelation()` 命中 entry 后，如果 `rd_isvalid = false`，会调用 `RelationRebuildRelation()`。
open relation 不能简单释放再重新分配。
调用者已经持有旧 `Relation *`。
直接释放会制造悬挂指针。
因此普通 rebuild 的策略是：
```text
RelationRebuildRelation(old)
  -> RelationInvalidateRelation(old)
  -> if index and rd_indexcxt exists: RelationReloadIndexInfo(old)
  -> else if nailed: RelationReloadNailed(old)
  -> else:
       newrel = RelationBuildDesc(relid, false)
       compare tupledesc/rules/RLS
       swap most fields between old and newrel
       re-swap preserved fields: rd_refcnt, rd_smgr, subids, rd_rel pointer, toast oid, pgstat, partkey
       keep old partition descriptor contexts if external users may hold pointers
       RelationDestroyRelation(newrel)
```
这个 swap 逻辑看起来不优雅，但它服务一个具体错误边界。
如果 rebuild 过程中 ERROR，旧 entry 仍然存在，只是 invalid。
下一次访问可以再尝试。
最坏情况是临时 new entry 泄漏，而不是把正在使用的 `Relation *` 变成悬挂指针。
对于 index relation，很多 schema 信息不允许随意变化。
`RelationReloadIndexInfo()` 只 reload `pg_class` 的有限字段和 `pg_index` 中允许变化的简单字段。
对 nailed relation，结构不能变化。
`RelationReloadNailed()` 主要 reload `pg_class` 中像 relfrozenxid、physical address 这类仍需更新的字段。
## 6. shared invalidation 主流程
### 6.1 失效消息从哪里产生
`CacheInvalidateHeapTupleCommon()` 是自动推导的主入口。
它只关心 system catalog。
普通 user table 的 heap tuple 更新不会直接影响 catcache 或 relcache。
它也忽略 system catalog 的 toast table。
它先让 catcache 注册 tuple-level 或 catalog-level invalidation。
然后判断这个 catalog tuple 是否“定义了某个 relcache entry”。
当前源码里的自动 relcache 分支主要包括：
| catalog | 推导出的 relcache relid |
| --- | --- |
| `pg_class` | `pg_class.oid` 对应 relation。 |
| `pg_attribute` | `pg_attribute.attrelid` 对应 relation。 |
| `pg_index` | `pg_index.indexrelid` 对应 index relation。 |
| `pg_constraint` | foreign key 的 `conrelid`。 |
其它 relcache 子结构变化常由命令路径显式触发。
例子：
- index 创建/删除显式 invalidates heap relation。
- rule 创建、删除、启停显式 invalidates event relation。
- trigger 创建、删除、启停显式 invalidates target relation。
- RLS policy 改动显式 invalidates target table。
- publication 改动可能 invalidates 单表或 whole relcache。
- extended statistics 改动会 invalidate relation 的统计子缓存。
这就是为什么排查 stale relcache 不能只看 `CacheInvalidateHeapTupleCommon()`。
你必须回到具体 DDL 命令路径，确认它是否调用了 `CacheInvalidateRelcache()`、`CacheInvalidateRelcacheByRelid()` 或 `CacheInvalidateRelcacheAll()`。
### 6.2 注册消息：当前命令先本地可见
`RegisterRelcacheInvalidation()` 把 relcache message 加入当前命令的 invalidation group。
它还调用 `GetCurrentCommandId(true)`。
这是为了确保下一次 `CommandCounterIncrement()` 会调用 `CommandEndInvalidationMessages()`。
本 backend 在同一事务后续命令中必须看到自己的 catalog 改动。
因此 command boundary 需要本地执行 invalidation。
流程是：
```text
DDL modifies catalog tuple
  -> CacheInvalidateHeapTupleCommon() or explicit CacheInvalidateRelcache()
  -> AddRelcacheInvalidationMessage(CurrentCmdInvalidMsgs)
  -> CommandCounterIncrement()
  -> CommandEndInvalidationMessages()
  -> LocalExecuteInvalidationMessage()
  -> CurrentCmdInvalidMsgs moves to PriorCmdInvalidMsgs
```
这一步不发送给其它 backend。
因为事务还没 commit。
别的 backend 不应该看到未提交 DDL。
但当前 backend 自己已经推进了 command id，后续命令应该看见新 catalog state。
### 6.3 commit 时发送到 shared invalidation queue
主事务 commit 时，`AtEOXact_Inval(true)` 处理 `PriorCmdInvalidMsgs` 和剩余 `CurrentCmdInvalidMsgs`。
它把消息发送给 shared invalidation queue。
如果事务触碰了 relcache init file 中的 relation，还会在发送前后调用：
```text
RelationCacheInitFilePreInvalidate()
SendSharedInvalidMessages()
RelationCacheInitFilePostInvalidate()
```
`PreInvalidate()` 在 `RelCacheInitLock` 下 unlink shared 和 local init file。
然后发送 SI messages。
最后释放锁。
这个顺序解决启动竞态：
```text
新 backend 不能先读旧 pg_internal.init，
然后又因为尚未进入或刚进入 sinval 消费边界而错过对应失效消息。
```
abort 路径不同。
`AtEOXact_Inval(false)` 不会发送给其它 backend。
它只本地处理 `PriorCmdInvalidMsgs`。
原因是当前事务中之前命令可能已经本地刷新过 cache。
abort 后这些 catalog 改动不成立，本 backend 必须把已经加载的“未提交新状态”也冲掉。
### 6.4 receiver 何时消费消息
`AcceptInvalidationMessages()` 从 shared invalidation queue 读取并处理消息。
注释要求它是处理一个 transaction 的第一步。
源码中还有多个显式调用点。
例如：
- `xact.c` 在事务启动路径调用。
- lock manager 某些关系锁获取路径会调用。
- syscache 的 inplace update 相关路径会调用。
- partition descriptor retry 会调用。
- relcache init file 写入前会调用，确认没有漏掉刚到的 relcache inval。
消费函数是 `LocalExecuteInvalidationMessage()`。
对 relcache message，它执行：
```text
if dbId matches MyDatabaseId or InvalidOid:
  if relId == InvalidOid:
    RelationCacheInvalidate(false)
  else:
    RelationCacheInvalidateEntry(relId)
  Call relcache callbacks
```
这个 callback 是跨模块传播点。
plan cache、typcache、relfilenumber map 都可能注册 relcache callback。
relcache 自身先处理本地 `RelationData`，然后 callback 让相邻 cache 标记自己的状态过期。
### 6.5 单个 relid 的本地处理
`RelationCacheInvalidateEntry(relid)` 先查本地 `RelationIdCache`。
如果找到 entry，就增加 `relcacheInvalsReceived` 并调用 `RelationFlushRelation()`。
如果没找到，它还会扫描 `in_progress_list`。
如果当前 backend 正在 build 这个 relid，但还没插入 hash，invalidation 会设置该 build 的 `invalidated = true`。
`RelationBuildDesc()` 在 build 末尾检查这个标志。
若为真，销毁半成品并重试。
这避免了一个 race：
```text
backend A 正在从多个 catalog 构造 Relation；
backend B 提交 DDL 并发送 relcache inval；
A 的 Relation 还没进 hash，普通 hash lookup 找不到；
如果没有 in_progress_list，A 会把已经过期的半成品插入 cache。
```
`RelationFlushRelation()` 根据本地状态决定处理方式。
简化模型是：
```text
if relation created or relfilenode changed in current transaction:
  if in transaction and not dropped:
    temporarily refcount++ and rebuild
  else:
    mark invalid
else:
  if refcount == 0:
    clear and destroy
  else if not in transaction:
    mark invalid
  else if nailed but effectively unused:
    mark invalid
  else:
    rebuild in place
```
这段逻辑体现了本节最重要的边界。
invalidation 不能无条件释放 entry。
也不能无条件 rebuild。
能否读 catalog、是否有 active transaction、是否有人持有 `Relation *`、是否为当前事务新建 relation，都会改变处理方式。
### 6.6 whole relcache reset
如果 shared invalidation queue overflow，receiver 不知道丢了哪些消息。
这时 `ReceiveSharedInvalidMessages()` 会调用 reset function，最终进入 `InvalidateSystemCaches()` / `InvalidateSystemCachesExtended(false)`。
relcache 部分调用 `RelationCacheInvalidate(false)`。
它不是逐条 message 的简单循环。
它分两阶段处理。
第一阶段扫描 hash。
zero refcount 的普通 entry 直接 `RelationClearRelation()`。
有 refcount 的 entry 收集到 rebuild list。
mapped relation 会先更新 `rd_locator`。
第二阶段 rebuild 或 mark invalid。
`pg_class` 和 `pg_class_oid_index` 优先。
其它 nailed relation 再处理。
这样避免在 rebuild 其它 entry 时先依赖了旧的 critical catalog descriptor。
最后，如果不是 `debug_discard` 调用，所有 `in_progress_list` entry 都标 invalidated。
失效消息丢失后的正确策略不是猜哪个 relation 变了，而是把本地系统 cache 当成不可信。
## 7. 生命周期 / ownership / cleanup
### 7.1 谁创建
`RelationCacheInitialize()` 创建空的 `RelationIdCache`。
startup phase2/phase3 会从 init file 或硬编码 descriptor 创建 critical relcache entries。
运行期普通 relation 由 `RelationBuildDesc()` 创建。
`CREATE TABLE` 等路径会先通过 `RelationBuildLocalRelation()` 创建“即将存在”的本地 entry。
这个 entry 带有 `rd_createSubid`。
它不能像旧 relation 一样被随意 flush。
因为 abort 时需要知道它是在当前事务创建的。
### 7.2 谁持有
调用 `RelationIdGetRelation()` 成功返回时，refcount 会增加。
`RelationIncrementReferenceCount()` 先 `ResourceOwnerEnlarge(CurrentResourceOwner)`，再 `rd_refcnt++`，再把 relation ref 记到 `CurrentResourceOwner`。
bootstrap 模式除外。
这保证了普通 ERROR/abort 路径能通过 ResourceOwner 释放 relcache ref。
关闭时调用 `RelationClose()`。
它只递减 refcount，然后执行 `RelationCloseCleanup()`。
真正的 relation lock 是否释放由上层 `relation_close(rel, lockmode)` 或对应 helper 决定。
不要把 relcache close 和 heavyweight lock release 混在一起。
### 7.3 谁释放
zero refcount 且不属于当前事务特殊状态的普通 entry 可以被 `RelationClearRelation()` 销毁。
销毁顺序是：
```text
RelationInvalidateRelation()
  -> RelationCacheDelete()
  -> RelationDestroyRelation()
```
`RelationDestroyRelation()` 会：
- close smgr。
- unlink pgstat relation entry。
- free `rd_rel`。
- 处理 `rd_att` refcount。
- free trigger、fkey、index list、stat list、bitmap、publication、options。
- free `rd_indextuple`、`rd_amcache`、`rd_fdwroutine`。
- delete `rd_indexcxt`、`rd_rulescxt`、RLS context、partition contexts。
- free `RelationData`。
`RelationInvalidateRelation()` 本身只关闭 smgr、释放 `rd_amcache`、设置 `rd_isvalid = false`。
它不释放整个 descriptor。
这就是“标 invalid”和“destroy”的区别。
### 7.4 ERROR / abort 时谁兜底
relcache ref 由 ResourceOwner 在 release phase 中释放。
`relref_resowner_desc` 的 release phase 是 `RESOURCE_RELEASE_BEFORE_LOCKS`。
这意味着在释放 relation locks 之前，先释放 relation references。
如果某段代码打开 relation 后 ERROR，没有调用 close，ResourceOwner 会调用 release callback，最终 decrement refcount。
事务结束还会调用 `AtEOXact_RelationCache()`。
它不是主要释放 refcount 的机制。
它主要做 cross-check 和特殊 cleanup：
- 忘掉 `in_progress_list`，处理 `RelationBuildDesc()` 中途 ERROR 的情况。
- 清理当前事务创建或 drop 后保留的 relcache entry。
- 释放延迟到事务结束的 tupledesc。
- reset `eoxact_list`。
subtransaction 结束走 `AtEOSubXact_RelationCache()`。
subcommit 时把 `rd_createSubid` 等状态转移给父 subtransaction。
subabort 时，如果 relation 是当前 subtransaction 创建且 refcount 为零，就清掉 entry。
如果 refcount 非零，只能 WARNING 并把状态转移，避免 error during error recovery。
### 7.5 partition descriptor 的延迟释放
partition descriptor 是 relcache 中少数会把内部指针直接交给调用者的子结构。
因此 rebuild open partitioned relation 时不能直接删除旧 `rd_pdcxt`。
`RelationRebuildRelation()` 会把旧 partition descriptor context 重新挂到新 context 下。
`RelationCloseCleanup()` 在 relation refcount 归零时删除 `rd_pdcxt` / `rd_pddcxt` 的 child contexts。
这是一种指针安全 trade-off。
它可能短期保留更多内存。
但避免 PartitionDirectory 或 executor 仍持有旧 `PartitionDesc *` 时被释放。
### 7.6 relcache init file 的生命周期
init file 在 backend startup 尝试读取。
读取失败退回 bootstrap 构造。
phase3 结束时，如果需要且没有收到 relcache invalidation，会写新 init file。
写入使用临时文件，再 rename。
rename 前拿 `RelCacheInitLock`，调用 `AcceptInvalidationMessages()`，确认没有刚到的 relcache invalidation。
如果启动期间已收到 relcache invalidation，放弃写入。
commit invalidation 会在同一把锁下删除 init file 并发送 SI message。
postmaster startup 也会移除 init files，避免 crash recovery/PITR 场景使用不可靠旧文件。
## 8. 正确性机制层次
### 8.1 relation lock 保护跨 catalog 组合
`ScanPgRelation()` 的注释很直接。
构造 relcache entry 时，多次 catalog scan 未必共享同一个 snapshot。
如果没有 relation lock，可能先读到旧 `pg_class.relnatts`，再读到新 `pg_attribute`，或者反过来。
所以 `RelationBuildDesc()` 要求 caller 至少持有 `AccessShareLock`。
这个 lock 不是为了保护本地 `RelationData`。
它是为了阻止并发 DDL 让多个 catalog 片段互相不一致。
### 8.2 invalidation 不是互斥
shared invalidation 不阻塞 DDL。
它也不保证 receiver 立即处理消息。
一个 backend 可能直到下一次 transaction start 或显式 `AcceptInvalidationMessages()` 才消费。
正确性来自锁和 command/transaction boundary：
- DDL 修改 catalog 时持有足够强的 relation lock。
- 当前 backend 在 CCI 后本地处理失效。
- 其它 backend 在后续事务或安全点消费失效。
- 使用旧 descriptor 的 backend 通常仍持有能防止破坏性 DDL 的锁。
因此看到 stale metadata 时，先问：
```text
这个 backend 是否还在同一个已持锁的执行区间？
这个 stale 指针是否只是允许用到语句结束？
还是某个 invalidation 根本没有注册或没有在安全点消费？
```
### 8.3 refcount 只保证内存安全
`rd_refcnt > 0` 说明这个 backend 内有人打开 relation。
它不说明 descriptor 是新鲜的。
`RelationFlushRelation()` 对 refcount 大于零的 entry 可能 rebuild，也可能在不能读 catalog 时只 mark invalid。
调用者继续使用 `Relation *` 的安全性来自：
- 指针没有被释放。
- 当前持有的 relation lock 足以让关键 schema 不发生破坏性变化。
- 如果下一次重新打开，它会触发 rebuild。
这就是 refcount 和 invalidation 的边界。
### 8.4 rebuild in place 保护外借指针
很多代码会在持有 relation open 的同时保存 `Relation *`、`TupleDesc`、`RuleLock *` 或 partition 指针。
如果 invalidation 到来时简单 free/realloc，会制造难以诊断的 use-after-free。
所以 open entry 的 rebuild 保留物理 `RelationData` 地址。
能保持不变的子结构尽量保留。
不能保证不变的子结构通过 context 延迟释放或重建。
这让 relcache 代码看起来有很多 swap/re-swap hack。
这些 hack 是 C 指针 ABI 与 runtime invalidation 的代价。
### 8.5 plan cache race 的二次检查
`plancache.c` 注册 relcache callback。
`PlanCacheRelCallback()` 命中 `relationOids` 后把 `CachedPlanSource` 或 generic plan 标 invalid。
`GetCachedPlan()` 一类路径还会先拿 planner locks，然后再次检查 `plansource->is_valid`。
源码注释说明这是为了覆盖 race：
```text
invalidation 在拿锁前到达；
callback 已经把 plan 标 invalid；
拿锁后必须再看一次，不能继续使用旧 query tree。
```
这里的正确性层次是：
- relcache invalidation 告诉 plan cache “依赖对象变了”。
- plan cache 标记 invalid。
- 下一次使用时拿锁并重新 parse/rewrite/plan。
- 如果输出 tupledesc 变化，prepared statement caller 可能看到错误。
### 8.6 WAL 和 recovery 中的 invalidation
transactional invalidation messages 会被收集到 commit WAL record。
`xactGetCommittedInvalidationMessages()` 在正式 commit 前收集消息。
redo 路径通过 `ProcessCommittedInvalidationMessages()` 发送 shared invalidation，并按 `RelcacheInitFileInval` 删除 init file。
这不是 relcache 数据页 WAL。
relcache entry 本身是 backend-local 内存。
WAL 记录的是 catalog change 及其 invalidation side effect，让 standby/recovery 也能推进 cache invalidation。
## 9. 错误路径 / 异常路径 / fallback
### 9.1 `pg_class` 找不到
`RelationBuildDesc()` 找不到 `pg_class` row 时返回 NULL。
这通常表示尝试访问刚被删除的 relation。
调用者可能把它转成用户可见 ERROR，也可能把 object-not-found 作为合法并发结果处理。
但是 rebuild open entry 时，找不到新 `pg_class` 更严重。
`RelationRebuildRelation()` 中如果 `RelationBuildDesc(save_relid, false)` 返回 NULL：
- historic snapshot 活跃时，可以留下 invalid entry，等待后续 cache reset。
- 普通处理下会 ERROR：relation deleted while still in use。
因为正常 DROP 应该通过锁和 `CheckTableNotInUse()` 防止删除仍被引用的 relation。
### 9.2 build 中途收到 invalidation
`in_progress_list` 是专门为这个异常设计的。
它是一个 stack，记录正在 `RelationBuildDesc()` 的 relid。
如果 invalidation 到达时 hash 里还没有 entry，`RelationCacheInvalidateEntry()` 会把对应 in-progress entry 标 invalidated。
build 末尾检查后销毁半成品并 retry。
如果是 whole relcache reset，`RelationCacheInvalidate(false)` 会把所有 in-progress build 标 invalidated。
`debug_discard_caches` 触发的 reset 不这样做。
否则 `RelationBuildDesc()` 在测试模式下可能陷入无限重试。
### 9.3 sinval queue overflow
shared invalidation queue overflow 后，receiver 不能知道丢了哪条 message。
正确 fallback 是 whole cache reset。
`InvalidateSystemCaches()` 会清 catcache、relcache、smgr cache。
relcache 采用两阶段清理/重建，避免 hash scan 中递归删除导致崩溃。
这条路径的成本高。
但它用全量不信任换 correctness。
不要把它优化成“尽量猜测几个可能 relid”。
消息丢失后 guess 不构成正确性机制。
### 9.4 init file 缺失、损坏或写入失败
`load_relcache_init_file()` 失败会返回 false。
startup 会用硬编码 critical descriptors 或 heap scan 构造。
如果 nailed rel/index 数量不符合预期，也会放弃 init file。
`write_relcache_init_file()` 创建文件失败只 WARNING 并继续 startup。
写 magic、write item 或 close 失败可能 FATAL。
rename 失败则删除临时文件。
这些 fallback 的共同目标是：
```text
init file 只能让启动更快；
不能让 backend 使用可疑 relcache state。
```
### 9.5 open index 的有限 reload
index relation 的 invalidation 不一定走完整 rebuild。
`RelationReloadIndexInfo()` 只允许 reload 有限字段。
如果 `pg_index` 中出现不应该变化的结构性差异，源码通过 assert/error 防御。
这是因为 critical index 可能 nailed，且 index reload 自身依赖 catalog/index。
完整删除重建可能递归或破坏已有指针。
### 9.6 partition attach/detach 的 retry
`RelationBuildPartitionDesc()` 处理并发 `ATTACH` / `DETACH CONCURRENTLY`。
它先从 syscache 取 child 的 `relpartbound`。
拿不到时直接扫 `pg_class`。
仍拿不到并且还没 retry 时，调用 `AcceptInvalidationMessages()` 后重试 `find_inheritance_children_extended()`。
只重试一次。
避免 catalog corruption 或异常竞态导致无限循环。
### 9.7 ERROR during rebuild
普通 open entry rebuild 先构造 `newrel`，再 swap。
如果构造中 ERROR，旧 entry 仍在，只是 invalid。
如果 swap 后销毁 `newrel` 前 ERROR 的窗口被源码尽量压缩。
关键 swap 段要求没有 `CHECK_FOR_INTERRUPTS`。
这不是风格问题。
这是为了避免 `RelationData` 瞬间处于半交换状态时被异步中断打断。
## 10. 成本、资源与跨模块传播
### 10.1 hot path 成本
命中有效 relcache entry 时，成本主要是 backend-local hash lookup、refcount increment、ResourceOwner remember。
没有 shared memory lookup。
没有跨进程锁。
这就是 relcache 值得存在的原因。
但是每次 open/close 都会触碰 `ResourceOwner`。
高频短查询、打开大量 relation/index 的执行路径仍会看到这部分 CPU 和分配成本。
### 10.2 build/rebuild 成本
cache miss 或 invalid rebuild 成本明显更高。
它可能包括：
- 扫 `pg_class`。
- 扫 `pg_attribute`。
- 读取 `pg_attrdef`、`pg_constraint`。
- 读取 `pg_index`、`pg_am`、opclass/opfamily。
- 反序列化 rule tree、trigger、RLS policy。
- 解析 reloptions。
- 初始化 table/index AM。
- 初始化 physical locator 和 relmapper 查询。
成本随列数、constraint 数、rule tree 大小、index key 数增长。
这个成本通常不是单条 catalog lookup。
它是多个 catalog 和子缓存组合。
### 10.3 lazy 子缓存成本
`RelationGetIndexList()` 成本随 index 数增长。
`RelationGetIndexAttrBitmap()` 还会打开每个 index，并解析 expression/predicate。
它可能因为中途 invalidation 重试。
`RelationGetPartitionDesc()` 成本随 partition 数增长。
它读取 inheritance children、partition bounds，并构造 `PartitionBoundInfo`。
大分区表上，partition descriptor build 能成为 planner/executor 前置成本。
`RelationGetPartitionKey()` 还会做 opclass、support function、type 信息查找。
### 10.4 invalidation fan-out
relcache invalidation 的发送方只发一条 relid message。
但接收方数量是 backend 数。
每个 backend 本地可能：
- 删除或 rebuild relcache entry。
- 清 smgr handle。
- 执行 relcache callbacks。
- 扫 saved plan list，标记 cached plan invalid。
- 清 typcache 或 relfilenumber map 的相关状态。
所以成本可能随 `backends * affected-relations * local-dependent-state` 放大。
DDL 频繁、连接数很大、每个连接有大量 prepared statements 时，这种 fan-out 会变得可见。
### 10.5 plan cache 传播
`PlanCacheRelCallback()` 扫 `saved_plan_list` 和 cached expressions。
它不立即 replan。
它只标 invalid。
下一次使用 plan 时才重新 parse/rewrite/plan。
这把 commit 时延控制住，但把成本转移到下一次执行。
如果 workload 有大量长寿命 prepared statements，DDL 后第一次执行可能集中付出 revalidation/replan 成本。
### 10.6 init file 与启动成本
如果 init file 被频繁删除，新 backend startup 会更多地走 hard way。
这会增加连接建立延迟。
连接池可以缓解 startup 频率。
但如果系统本身有频繁 DDL，已有 backend 仍然要处理 invalidation。
init file 只影响 startup，不解决运行期 invalidation fan-out。
### 10.7 内存保留
`CacheMemoryContext` 是长寿命上下文。
正常情况下 relcache entry 和子结构会留到 invalidation 或 backend 结束。
open partitioned relation rebuild 时，旧 partition descriptor context 可能被挂到新 context 下，直到 refcount 归零。
debug 或压力测试中可用 `debug_discard_caches` 暴露 cache flush hazard。
但它会让系统极慢。
源码注释中提到 `debug_discard_caches = 1` 可让 regression tests 约慢两个数量级，递归模式更慢。
## 11. 观测与诊断入口
### 11.1 能直接看到什么
PostgreSQL 没有通用 `pg_stat_relcache` 视图。
你不能直接从 SQL 查询某个 backend 的 `RelationIdCache` 内容。
能直接看到的主要是间接现象：
- DDL 后 prepared statement 被重新分析或报输出 tupledesc 改变。
- `pg_stat_activity.wait_event` 显示 relation lock 等待，而不是 relcache 等待。
- server log 中出现 cache lookup failed、init file warning、relcache refcount warning。
- `pg_prepared_statements` 可看到 prepared statement 存在，但不显示 relcache entry 是否 valid。
- `pg_locks` 可确认 DDL/DML 是否被 relation lock 排序。
### 11.2 只能推断什么
这些通常只能推断：
- 某个 backend 是否已经消费特定 SI message。
- 某个 relcache entry 是刚 rebuild，还是之前已经 valid。
- `RelationGetIndexList()` 是否因为中途 invalidation retry。
- `PlanCacheRelCallback()` 是否刚把某个 plan 标 invalid。
需要 gdb、tracepoint、临时日志或 `perf` 才能确定。
SQL 视图只能给你外部结果。
### 11.3 gdb 断点
调试 relcache build 建议断点：
```gdb
break RelationIdGetRelation
break RelationBuildDesc
break RelationBuildTupleDesc
break RelationInitIndexAccessInfo
break RelationGetIndexList
break RelationGetPartitionDesc
break CacheInvalidateRelcache
break CacheInvalidateHeapTupleCommon
break CommandEndInvalidationMessages
break AtEOXact_Inval
break AcceptInvalidationMessages
break LocalExecuteInvalidationMessage
break RelationCacheInvalidateEntry
break RelationFlushRelation
break RelationRebuildRelation
break PlanCacheRelCallback
```
关键观察字段：
```gdb
p relation->rd_id
p relation->rd_refcnt
p relation->rd_isvalid
p relation->rd_isnailed
p relation->rd_indexvalid
p relation->rd_createSubid
p relation->rd_firstRelfilelocatorSubid
p relation->rd_droppedSubid
p relcacheInvalsReceived
p in_progress_list_len
```
### 11.4 perf / flamegraph
如果怀疑 relcache 成本，`perf` 比 SQL 视图更有用。
关注函数：
- `RelationBuildDesc`
- `RelationBuildTupleDesc`
- `RelationGetIndexList`
- `RelationGetIndexAttrBitmap`
- `RelationBuildPartitionDesc`
- `RelationGetPartitionKey`
- `stringToNode`
- `SearchSysCache*`
- `PlanCacheRelCallback`
火焰图里如果这些函数在 DDL 后或连接启动时集中出现，通常说明 cache miss/rebuild/startup init file fallback 正在放大成本。
如果 workload 是大量分区表或大量 prepared statements，成本来源可能不是单个 relcache entry，而是 partition/plan callback 的扩散。
### 11.5 日志与错误信息
有用的日志/错误包括：
- `cache lookup failed for relation ...`
- `pg_attribute catalog is missing ...`
- `relation ... deleted while still in use`
- `cannot remove relcache entry ... because it has nonzero refcount`
- `could not create relation-cache initialization file ...`
- init file nailed rel/index 数量不匹配 warning。
这些信息不是完整因果。
例如 `cache lookup failed` 可能是 catalog corruption、并发对象删除、错误 snapshot 使用或代码忘记持锁。
诊断时要回到调用点看是否持有 relation lock、是否在 transaction state 中、是否可能使用 historic snapshot。
### 11.6 一个可观测 runtime truth
本节锚定的 runtime truth 是：
```text
DDL 不会修改其它 backend 的 RelationData；
它只提交 catalog change 和 invalidation message。
其它 backend 在安全边界消费 message 后，旧 RelationData 才会被本地删除、重建或标 invalid。
```
能看到的外部现象是 prepared statement、cached plan、同事务 CCI 后的 schema 可见性变化。
能在源码中确认的是 `CommandEndInvalidationMessages()`、`AtEOXact_Inval()`、`AcceptInvalidationMessages()`、`RelationCacheInvalidateEntry()` 这条链。
不能直接从 SQL 看到的是某个 backend 内部 `rd_isvalid` 何时翻转。
## 12. 常见误区
### 12.1 把 relcache 当 shared cache
relcache entry 是 backend-local。
shared invalidation 共享的是失效消息，不是 `RelationData`。
任何把 `Relation *` 跨进程保存或传递的设计都是错误方向。
### 12.2 把 refcount 当 freshness
`rd_refcnt` 防释放。
`rd_isvalid` 和 invalidation 才表达语义过期。
一个被 refcount pin 住的 entry 可以已经 invalid。
### 12.3 以为 invalidation 会阻塞 DDL
invalidation 是通知。
并发互斥靠 relation lock。
如果需要阻止 schema 改动，必须持有正确 lock mode。
### 12.4 以为 `pg_class` 更新自动覆盖所有 relcache 子结构
不是。
rule、trigger、policy、publication、index list 等很多变化依赖命令路径显式 `CacheInvalidateRelcache*()`。
排查 stale metadata 时必须查具体 DDL 源码。
### 12.5 以为 open relation 可以直接 free/rebuild
open relation 的 `Relation *` 地址可能被调用栈保存。
rebuild 需要 in-place swap 或延迟 invalid。
直接释放会造成悬挂指针。
### 12.6 以为 relcache init file 是可靠持久状态
init file 是启动优化。
缺失、损坏、过期都必须安全 fallback。
它不是 catalog 的替代来源。
### 12.7 以为 plan cache invalidation 等于立即 replan
relcache callback 通常只标记 cached plan invalid。
真正 reparse/rewrite/replan 发生在下一次使用。
DDL 后的延迟抖动可能落在业务下一次执行上。
## 13. 课堂实验
### 实验 1：源码断点跟一次普通 build
准备一张新表，然后在一个 backend 中第一次打开它。
建议断点：
```gdb
break RelationIdGetRelation
break RelationBuildDesc
break ScanPgRelation
break RelationBuildTupleDesc
break RelationCacheInsert
```
SQL：
```sql
DROP TABLE IF EXISTS rc_demo;
CREATE TABLE rc_demo(id int primary key, v text);
SELECT * FROM rc_demo WHERE id = 1;
```
观察：
```gdb
p targetRelId
p relation->rd_id
p relation->rd_att->natts
p relation->rd_refcnt
p relation->rd_isvalid
p relation->rd_indexvalid
```
画出：
```text
pg_class row
  -> AllocateRelationDesc
  -> pg_attribute rows
  -> TupleDesc
  -> reloptions/rules/triggers/RLS
  -> physical locator
  -> RelationIdCache
```
重点问题：
`rd_indexvalid` 为什么可能还是 false？
因为 heap relation 的 index list 是 `RelationGetIndexList()` lazy 构造，不是 build 主体的一部分。
### 实验 2：两会话观察 DDL 到 plan/relcache invalidation
会话 A：
```sql
DROP TABLE IF EXISTS rc_plan_demo;
CREATE TABLE rc_plan_demo(id int);
PREPARE p AS SELECT * FROM rc_plan_demo;
EXECUTE p;
```
会话 B：
```sql
ALTER TABLE rc_plan_demo ADD COLUMN v text;
```
回到会话 A：
```sql
EXECUTE p;
```
预期现象：
prepared statement 会在 relcache/plan invalidation 后重新分析或发现输出 tupledesc 改变。
具体用户可见结果取决于调用协议和语句形态。
常见情况是报 cached plan result type 变化。
源码回看：
```text
ALTER TABLE updates pg_class/pg_attribute
  -> relcache invalidation registered
  -> commit sends SI message
  -> session A AcceptInvalidationMessages()
  -> RelationCacheInvalidateEntry(rc_plan_demo)
  -> PlanCacheRelCallback(rc_plan_demo)
  -> next EXECUTE revalidates/replans
```
断点建议：
```gdb
break CacheInvalidateHeapTupleCommon
break AtEOXact_Inval
break LocalExecuteInvalidationMessage
break RelationCacheInvalidateEntry
break PlanCacheRelCallback
```
### 实验 3：同一事务中 CCI 后本地 relcache 更新
单会话：
```sql
DROP TABLE IF EXISTS rc_cci_demo;
CREATE TABLE rc_cci_demo(a int);
BEGIN;
ALTER TABLE rc_cci_demo ADD COLUMN b int;
SELECT b FROM rc_cci_demo;
ROLLBACK;
```
观察点：
`ALTER TABLE` 后，当前 backend 不需要等 commit 才看到新列。
原因是 command boundary 会执行 `CommandEndInvalidationMessages()`。
这会本地刷新 relcache。
断点：
```gdb
break CommandEndInvalidationMessages
break LocalExecuteInvalidationMessage
break RelationRebuildRelation
```
思考：
为什么 abort 时仍要本地处理 `PriorCmdInvalidMsgs`？
因为当前事务前面命令可能已经让本 backend 的 cache 看到未提交 catalog 状态。
abort 后必须把这些状态冲掉。
### 实验 4：分区表 descriptor 的 lazy 构造
SQL：
```sql
DROP TABLE IF EXISTS rc_part_demo CASCADE;
CREATE TABLE rc_part_demo(id int, v text) PARTITION BY RANGE (id);
CREATE TABLE rc_part_demo_1 PARTITION OF rc_part_demo FOR VALUES FROM (0) TO (100);
CREATE TABLE rc_part_demo_2 PARTITION OF rc_part_demo FOR VALUES FROM (100) TO (200);
EXPLAIN SELECT * FROM rc_part_demo WHERE id = 42;
```
断点：
```gdb
break RelationGetPartitionKey
break RelationBuildPartitionKey
break RelationGetPartitionDesc
break RelationBuildPartitionDesc
```
观察：
```gdb
p rel->rd_partkey
p rel->rd_partdesc
p rel->rd_partdesc_nodetached
p rel->rd_partdesc_nodetached_xmin
```
扩展实验：
另一个会话执行 `ALTER TABLE ... ATTACH PARTITION` 或 `DETACH PARTITION CONCURRENTLY`。
观察 `RelationBuildPartitionDesc()` 中 syscache miss、direct `pg_class` scan、`AcceptInvalidationMessages()` retry 的路径。
## 14. 讨论题
1. 为什么 `RelationBuildDesc()` 要求 caller 持有 relation lock，而不是自己只靠 catalog snapshot 解决一致性？
2. `rd_refcnt > 0`、`rd_isvalid = false` 同时成立时，调用者还能安全做什么，不能安全做什么？
3. 为什么 heap relation 的 index list 是 lazy cache，而 index relation 自己的 `rd_index` 信息必须 build 时立即构造？
4. 为什么 `RelationGetIndexList()` 返回 list copy，而 `RelationGetPartitionDesc()` 返回 relcache 内部指针？
5. `CacheInvalidateHeapTupleCommon()` 没有自动处理 `pg_rewrite` 时，rule 变化如何让 `rd_rules` 失效？
6. 为什么 commit 删除 relcache init file 要在发送 SI messages 之前，并且要用 `RelCacheInitLock` 序列化？
7. sinval overflow 后为什么必须 reset whole system cache，而不是只处理已经收到的 relids？
8. plan cache 为什么只在 relcache callback 中标 invalid，而不立刻 replan？
9. 如果线上 DDL 后第一次业务查询很慢，你会如何区分 relcache rebuild、plan revalidation、lock wait 和 partition descriptor build？
10. 为什么 partition descriptor 的 omit-detached 版本要记录 `pg_inherits.xmin`？
## 15. 本节小结
本节核心链路：
```text
RelationIdGetRelation()
  -> local hash lookup
  -> refcount/ResourceOwner
  -> RelationBuildDesc() or RelationRebuildRelation()
  -> CacheMemoryContext RelationData
  -> catalog DDL registers relcache invalidation
  -> command boundary local flush
  -> commit shared invalidation
  -> receiver local flush/rebuild/callback
```
核心状态和边界：
- `RelationData` 是 backend-local 描述符，不是 shared state。
- `rd_refcnt` 保护本地指针生命周期。
- `rd_isvalid` 表示语义是否需要重建。
- `rd_rel`、`rd_att`、`rd_index*`、`rd_rules`、partition fields 来自不同 catalog 和不同构造时机。
- relcache init file 是启动优化，不是 catalog truth。
ownership / cleanup：
- 长寿命对象在 `CacheMemoryContext`。
- 复杂子结构用 child context。
- relation reference 记到 `ResourceOwner`。
- `RelationClose()` 只释放 ref，不等于释放 lock。
- open entry invalidation 走 rebuild in place 或 mark invalid。
- zero refcount entry 才能被 `RelationClearRelation()` 物理删除。
错误路径规则：
- `pg_class` 缺失可能返回 NULL，open rebuild 中通常 ERROR。
- build 中途收到 invalidation 通过 `in_progress_list` retry。
- sinval overflow 触发 whole cache reset。
- init file 失败必须 fallback 到 bootstrap/heap scan。
- partition attach/detach 可能 direct scan 并 retry。
观测边界：
- SQL 主要看到 prepared statement、DDL 后 schema 可见性、lock wait 和错误信息。
- `rd_isvalid`、in-progress build、callback 命中主要靠 gdb、临时日志或 profiling。
- `pg_stat_*` 不能直接告诉你某个 backend 的 relcache 状态。
可迁移 mental model：
```text
当一个本地长寿命指针代表可变的全局元数据时，
系统通常把问题拆成三层：
本地 refcount 保护内存，
全局 invalidation 传播语义过期，
下一次安全访问点重建本地事实。
```
relcache 是这个模型的典型实现。
它用 backend-local 结构换 hot path 速度。
用 relation lock 和 command/transaction boundary 保证 catalog 组合的一致性。
用 shared invalidation 把 DDL 的影响传播到每个 backend。
用 in-place rebuild、lazy child cache 和 init file fallback 接住 C 指针、启动性能和并发 DDL 的复杂边界。
