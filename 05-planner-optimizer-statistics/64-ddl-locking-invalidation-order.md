# PostgreSQL DDL locking / invalidation ordering

## 课程定位

前置知识：已经理解 `ProcessUtility()` 如何把 DDL 分派到 command 模块，也理解上一节中 catalog tuple、dependency、CommandCounterIncrement 和 cache invalidation 的基本角色。
本节唯一主问题：
```text
DDL 为什么必须先拿对象锁，再改 catalog，再发 invalidation，错误顺序会破坏什么？
```
本节核心矛盾：
```text
DDL 必须改变所有 backend 共同理解的对象定义；
但每个 backend 又有自己的 syscache、relcache、typcache、plancache 和正在运行的 query。
```
一句话运行模型：
```text
DDL 先用 heavyweight object lock 建立互斥和等待边界，
再把 catalog 当作事务性 heap tuple 修改，
CommandCounterIncrement 在本 backend 的命令边界处理本地 invalidation，
commit 记录成功后、释放对象锁前，把 shared invalidation 发给其他 backend。
```
学完后应能独立判断：
某个 DDL 必须拿多强的对象锁。
某个 catalog update 何时需要 `CommandCounterIncrement()`。
某个 invalidation 是本地命令边界生效，还是提交时跨 backend 传播。
为什么 invalidation 不能替代 lock。
为什么 lock 也不能替代 invalidation。
为什么 abort 时不能把未提交的 catalog invalidation 广播给其他 backend。
为什么等待 DDL 锁返回后还要先处理 invalidation，才能安全使用 relcache 或 syscache。
本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e90741e12a3715909e1b2d71ff9344`。
## 1. 本节在总主线中的位置

05 目录前面的课程主要追 SQL 如何进入 planner。
第 61 节把 `CMD_UTILITY` 语句带到 `ProcessUtility()`。
第 62 节解释 DDL 如何把 `pg_class`、`pg_attribute`、`pg_index`、`pg_depend` 等 catalog 当作事务性表更新。
本节继续缩小问题。
我们不再问“DDL 会改哪些 catalog”。
本节只问“这些动作为什么必须按固定顺序发生”。
DDL 的动作看似可以拆成三件事：
```text
拿对象锁
修改 catalog
通知缓存失效
```
真正的内核问题是：这三件事不能随意交换。
如果先改 catalog 再拿锁，正在使用旧定义的 backend 可能继续运行，新的 backend 又可能看到新定义。
如果先发 invalidation 再改 catalog，其他 backend 可能丢掉缓存后又重新装载旧 tuple。
如果 commit 后先释放锁再发 invalidation，等待锁的 backend 可能醒来，用旧 relcache 解释已经提交的新对象。
如果 abort 时把 invalidation 当成 commit 消息广播，其他 backend 会无意义地丢缓存，严重时还会把未提交对象当成可解释事件。
本节把这些错误顺序都压回一条主链路：
```text
对象锁
  -> catalog tuple change
  -> command boundary local invalidation
  -> commit record / ProcArray end
  -> shared invalidation
  -> release locks
```
这条链路连接了五个模块。
`commands/*cmds.c` 决定对象语义和锁强度。
`storage/lmgr` 负责 heavyweight lock 和等待。
`catalog` 负责 catalog tuple、dependency 和 object address。
`utils/cache/inval.c` 负责本地和共享 invalidation。
`access/transam/xact.c` 把 CCI、commit、abort、resource owner 和 lock release 串成事务收尾顺序。
## 2. 核心矛盾与一句话运行模型

DDL 改的不是普通用户数据。
DDL 改的是 PostgreSQL 用来解释所有用户数据的元数据。
`ALTER TABLE t ADD COLUMN b int` 改变 tuple descriptor。
`DROP INDEX i` 改变 planner 和 executor 对 index set 的理解。
`ALTER TABLE t ALTER COLUMN a TYPE bigint` 可能改变 tuple rewrite、表达式类型和 cached plan。
`CREATE INDEX CONCURRENTLY` 甚至要让 index 在几个事务阶段逐步从不可用变成可插入再变成可查询。
这些行为共同面对一个 tension：
```text
对象定义需要全局一致
  vs
PostgreSQL 的缓存、snapshot、query 执行和事务状态都是 backend-local 或时间分层的
```
PostgreSQL 没有一个“全局元数据大锁”让所有 backend 停下来重读 catalog。
它选择更细的组合：
对象锁阻止不兼容的并发 DDL/DML 穿过同一语义边界。
MVCC catalog tuple 让 DDL 事务可提交、可回滚、可按 command id 逐步可见。
`CommandCounterIncrement()` 让当前 backend 在下一条命令看到自己刚写的 catalog 版本。
本地 invalidation 让当前 backend 的 syscache、relcache、typcache 和 plancache 不再解释旧语义。
shared invalidation 让其他 backend 在合适边界丢弃旧缓存。
lock release 让等待者在已经可以看到新 catalog 且已经收到 invalidation 后继续。
这就是本节的一句话模型：
```text
lock 决定谁可以同时穿过对象语义边界；
catalog tuple 决定语义的事务性事实；
invalidation 决定 backend-local cache 何时不再可信。
```
任何一个机制都不能替代另外两个。
lock 不会自动改掉别人的 relcache。
invalidation 不会阻塞已经打开 relation 的 executor。
MVCC 不会告诉 backend-local cached plan 需要重写。
错误顺序破坏的不是单个字段。
错误顺序破坏的是“等待者醒来时应当看到哪个对象定义”这个跨模块不变量。
## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/commands/tablecmds.c` | `RemoveRelations()`、`AlterTableGetLockLevel()`、`AlterTable()`、`ATController()` 展示 DDL 先定锁、再改 catalog、锁持有到 commit。 |
| 2 | `src/backend/commands/indexcmds.c` | `DefineIndex()` 和 concurrent index build 展示弱锁、多事务阶段和额外等待如何补足 ordering。 |
| 3 | `src/backend/commands/dropcmds.c` | generic DROP 入口如何把名字解析成 `ObjectAddress`，再进入 dependency deletion。 |
| 4 | `src/backend/catalog/dependency.c` | `performDeletion()`、`AcquireDeletionLock()`、`findDependentObjects()`、`deleteObjectsInList()` 展示 DROP 图和删除锁。 |
| 5 | `src/backend/storage/lmgr/lmgr.c` | `LockRelationOid()`、`LockDatabaseObject()`、`LockSharedObject()` 在拿锁后处理 invalidation。 |
| 6 | `src/backend/storage/lmgr/lock.c` | lock conflict matrix、`LockAcquireExtended()`、`LockReleaseAll()` 是 heavyweight lock 的共享状态核心。 |
| 7 | `src/backend/utils/cache/inval.c` | `CacheInvalidateHeapTuple()`、`CommandEndInvalidationMessages()`、`AtEOXact_Inval()` 定义命令和事务边界。 |
| 8 | `src/backend/utils/cache/relcache.c` | `RelationCacheInvalidateEntry()`、`RelationCacheInvalidate()` 展示 relcache 被 flush、重建或标记重试。 |
| 9 | `src/backend/utils/cache/plancache.c` | `PlanCacheRelCallback()`、`PlanCacheObjectCallback()` 把 relcache/syscache inval 转成 cached plan 失效。 |
| 10 | `src/backend/access/transam/xact.c` | `CommandCounterIncrement()`、`CommitTransaction()`、`AbortTransaction()` 给出 CCI、SI 发送和 lock release 的最终顺序。 |
推荐阅读顺序不是从 `lock.c` 顶部开始背 conflict matrix。
更好的顺序是从一个具体 DDL 开始。
先看 `ALTER TABLE`：
```text
ProcessUtilitySlow()
  -> AlterTableGetLockLevel()
  -> RangeVarGetRelidExtended(..., lockmode, RangeVarCallbackForAlterRelation)
  -> AlterTable()
  -> relation_open(..., NoLock)
  -> ATController()
  -> ATRewriteCatalogs()
  -> ATRewriteTables()
```
再看 `DROP TABLE`：
```text
RemoveRelations()
  -> AcceptInvalidationMessages()
  -> RangeVarGetRelidExtended(..., lockmode, RangeVarCallbackForDropRelation)
  -> ObjectAddress
  -> performMultipleDeletions()
  -> performDeletion()
  -> AcquireDeletionLock()
  -> deleteObjectsInList()
```
最后看事务边界：
```text
CommandCounterIncrement()
  -> AtCCI_LocalCache()
  -> CommandEndInvalidationMessages()
CommitTransaction()
  -> RecordTransactionCommit()
  -> ProcArrayEndTransaction()
  -> AtEOXact_Inval(true)
  -> ResourceOwnerRelease(... RESOURCE_RELEASE_LOCKS ...)
AbortTransaction()
  -> RecordTransactionAbort()
  -> ProcArrayEndTransaction()
  -> AtEOXact_Inval(false)
  -> ResourceOwnerRelease(... RESOURCE_RELEASE_LOCKS ...)
```
这三段合起来，才是本节要讲的 ordering。
## 4. 关键数据结构与状态

### 4.1. `LOCKTAG` 与 object identity
heavyweight lock 的对象身份不是 C 指针。
它是 `LOCKTAG`。
relation lock 通过 database OID 和 relation OID 定位。
database-local object lock 通过当前 database、classid、objid、objsubid 定位。
shared object lock 使用 `InvalidOid` 作为 database id。
这解释了一个容易误解的边界：
```text
LockRelationOid(relid, mode)
  和
LockDatabaseObject(RelationRelationId, relid, 0, mode)
```
不是同一个 lock namespace。
`lmgr.c` 明确提醒普通 object lock 不要用于 relation，因为它不会和 `LockRelation()` 一类 relation lock 冲突。
DDL 必须使用能和相关读写路径冲突的 lock tag。
否则你以为拿了“对象锁”，实际没有阻止 executor 或另一条 DDL 穿过 relation 语义边界。
### 4.2. `LOCALLOCK`、ResourceOwner 与锁生命周期
`LockAcquireExtended()` 会在 shared lock table 中建立或找到 lock/proclock 状态。
backend 本地还会有 `LOCALLOCK`。
`LOCALLOCK` 不是语义事实本身。
它记录当前 backend 对某个 lock tag/mode 的持有计数、owner 关系和 fast-path 状态。
事务级锁通常由 `CurrentResourceOwner` 记住。
正常 commit 或 abort 都会走 `ResourceOwnerRelease(... RESOURCE_RELEASE_LOCKS ...)`。
因此 DDL command 函数常见模式是：
```text
relation_open(relid, lockmode)
relation_close(rel, NoLock)
```
`NoLock` close 的意思不是锁已经释放。
它表示关闭 relcache ref，但保留事务级 heavyweight lock 到事务结束。
`ATController()` 中“Close the relation, but keep lock until commit”就是这个设计。
### 4.3. catalog tuple 与 command id
catalog tuple 是普通 heap tuple。
它有 xmin、xmax、command id 和 MVCC 可见性。
DDL 修改 catalog 时，本命令内旧 tuple 仍可能按照同一 command 的可见性规则被认为有效。
`inval.c` 顶部注释强调：不能在 `heap_update()` 或 `heap_delete()` 时立刻 flush cache。
原因是旧 tuple 在同一个 command 内仍有效。
即使 flush 了，同一 command 后续 lookup 也可能重新把旧 tuple 装回 cache。
所以 catalog invalidation 要先排队。
真正的本地缓存切换发生在 command boundary。
这个 boundary 由 `CommandCounterIncrement()` 推进。
### 4.4. invalidation 消息组
`inval.c` 把 pending request 存成 `SharedInvalidationMessage` 数组。
事务性消息分两层：
`CurrentCmdInvalidMsgs` 表示当前 command 产生但还没在本地处理的消息。
`PriorCmdInvalidMsgs` 表示前面 command 已在本地处理，事务结束时还需要 commit/abort 收尾的消息。
命令结束时：
```text
CurrentCmdInvalidMsgs
  -> LocalExecuteInvalidationMessage()
  -> append 到 PriorCmdInvalidMsgs
```
事务提交时：
```text
PriorCmdInvalidMsgs + CurrentCmdInvalidMsgs
  -> SendSharedInvalidMessages()
```
事务 abort 时：
```text
PriorCmdInvalidMsgs
  -> LocalExecuteInvalidationMessage()
CurrentCmdInvalidMsgs
  -> 丢弃
```
这个状态划分回答一个关键问题：
本 backend 需要在命令边界忘掉旧定义。
其他 backend 只能在事务真的提交后被通知。
### 4.5. relcache 与 plancache 的下游状态
relcache entry 是 backend-local 的 `RelationData`。
它有 refcount、rd_isvalid、rd_createSubid、rd_firstRelfilelocatorSubid 等生命周期状态。
被 invalidation 命中的 relcache entry 可能直接清除，也可能因 refcount 非零而延迟重建。
`RelationCacheInvalidateEntry()` 如果找不到 entry，还会标记正在 build 的 entry 被 invalidated。
`RelationCacheInvalidate()` 在 SI overflow 或全量失效时两阶段清理和重建。
plancache 也不是直接读 catalog。
`CachedPlanSource` 保存 `relationOids`、`invalItems`、`is_valid`、`generation`。
`CachedPlan` 保存 `stmt_list`、`is_valid`、`generation`、refcount。
relcache callback 会把依赖某 relation 的 query tree 或 generic plan 标成 invalid。
syscache callback 会把依赖某 type/function/operator 等对象的 plan 标成 invalid。
所以 DDL ordering 的影响会传播到 planner。
旧计划不是因为 lock 释放而自动消失。
旧计划是因为 invalidation callback 把 `is_valid` 改掉，下一次使用时才被重写或重规划。
## 5. 主流程源码 walkthrough

### 5.1. `ALTER TABLE`：先决定最大锁级别
`ALTER TABLE` 的第一步不是打开 catalog 改 tuple。
它先根据 subcommand 列表决定需要的最大 lock mode。
`AlterTableGetLockLevel()` 的注释很直白。
这个函数在拿表锁之前运行。
因此它不能依赖表的当前 metadata 来决定锁强度。
它必须仅根据 parse 后的 subcommand 类型给出保守答案。
如果一个 statement 包含多个 subcommand，锁强度取最大值。
例如会影响 SELECT 可见定义、可能重写 heap、可能改变 rule/constraint/type 语义的 subcommand，通常需要 `AccessExclusiveLock`。
一些只影响维护策略、统计目标、validate constraint 等路径，可以使用 `ShareUpdateExclusiveLock`。
这一段的重点不是背每个 subcommand 的锁级别。
重点是 ordering：
```text
先在不知道更多 catalog 细节的情况下决定足够强的锁；
再用这个锁做 name lookup 和对象检查；
最后才允许修改 catalog。
```
如果反过来，先读旧 catalog 判断“可能只要弱锁”，再在弱锁下改出更强语义变化，就会制造并发窗口。
### 5.2. `RangeVarGetRelidExtended()`：name lookup 和 lock 是一组动作
DDL 常通过 `RangeVarGetRelidExtended()` 把 schema/name 解析成 OID。
这个函数不仅返回 OID。
它还负责按传入 lock mode 获取 relation lock，并运行 callback 做权限、relkind、系统表限制等检查。
`tablecmds.c` 中 `RemoveRelations()` 的注释说得很直接：
先 identify all relations，再一次性删除，避免多个 DROP 对象之间的 restrict/cascade 误报。
每个对象在进入 deletion list 前都通过 `RangeVarGetRelidExtended()` 拿锁和验证。
同时它会在 name lookup 前调用 `AcceptInvalidationMessages()`。
这是为了覆盖名字对应的 relation 自事务开始后被 drop/recreate 的情况。
如果不先吸收已有 invalidation，backend 可能用旧 syscache entry latch 到旧 OID，然后后面才报错。
这里已经能看到本节主问题的半个答案：
```text
DDL 拿锁不是单独的“等待动作”；
拿锁后必须先让本 backend 的对象缓存追上已经提交的事实。
```
### 5.3. `LockRelationOid()`：锁返回后处理 invalidation
`lmgr.c` 的 `LockRelationOid()` 是本节最关键的小函数之一。
它调用 `SetLocktagRelationOid()` 设置 relation lock tag。
然后调用 `LockAcquireExtended()`。
拿到锁以后，如果结果不是 `LOCKACQUIRE_ALREADY_CLEAR`，就调用：
```text
AcceptInvalidationMessages();
MarkLockClear(locallock);
```
注释解释了原因。
拿到锁以后要检查 invalidation messages。
这样在尝试使用 relcache entry 前，会 update 或 flush 旧 relcache。
`RangeVarGetRelid()` 依赖这个行为。
如果当前事务自己修改了 relation，relcache update 通过 `CommandCounterIncrement()` 发生，不在这里完成。
这给出一个非常具体的不变量：
```text
等待对象锁成功返回
  并不等价于
本 backend 的对象缓存已经是新的。
```
PostgreSQL 把这两个动作粘在 `LockRelationOid()` 一起做。
所以 DDL 或 DML 路径一般不会在等待锁后继续使用 stale relcache。
`LockDatabaseObject()` 和 `LockSharedObject()` 也有同类处理。
它们拿到 general object lock 后调用 `AcceptInvalidationMessages()`，让 syscache 跟上等待期间的变化。
### 5.4. `ALTER TABLE`：持锁改 catalog，close rel 不放锁
`AlterTable()` 的入口注释说明 caller 必须已经提供足够 lock。
然后它用 `relation_open(context->relid, NoLock)` 打开 relation。
这里 `NoLock` 是因为锁已经在外层拿过。
`CheckAlterTableIsSafe()` 之后进入 `ATController()`。
`ATController()` 分三阶段：
```text
Phase 1: ATPrepCmd() 初步检查和 work queue
Phase 2: ATRewriteCatalogs() 更新系统 catalog
Phase 3: ATRewriteTables() 必要时扫描或 rewrite heap，并执行 after statements
```
阶段 1 之后有一行关键注释：
```text
Close the relation, but keep lock until commit
```
也就是：
```text
relation_close(rel, NoLock)
```
relcache ref 可以放掉。
heavyweight lock 继续由事务持有。
这保证 Phase 2 和 Phase 3 改 catalog 或 rewrite heap 时，不兼容的并发 backend 不能穿过同一 relation 语义边界。
如果 close relation 时释放锁，Phase 2 catalog 已变而 Phase 3 heap 还没 rewrite 的中间状态就可能被别人看见。
### 5.5. catalog update：只排队 invalidation，不立即全局通知
catalog 修改最终会走 heap insert/update/delete。
对有 catcache 或 relcache 影响的 catalog tuple，`CacheInvalidateHeapTuple()` 会注册 invalidation。
它不会立即向其他 backend 发送 SI。
它也不会立刻把本 command 内的旧 tuple 全部 flush 掉。
原因还是 command id。
同一个 command 中旧 tuple 仍然可以有效。
只有 `CommandCounterIncrement()` 后，本事务的后续命令才应该看到新 catalog 版本。
`CommandCounterIncrement()` 的关键步骤是：
```text
currentCommandId += 1
SnapshotSetCommandId(currentCommandId)
AtCCI_LocalCache()
```
`AtCCI_LocalCache()` 又调用：
```text
AtCCI_RelationMap()
CommandEndInvalidationMessages()
```
`CommandEndInvalidationMessages()` 本地执行当前 command 的 invalidation，并把消息移到 prior command list。
这就是“先改 catalog，再本地 invalidation”的精确含义。
不是所有 invalidation 都等到 commit。
当前 backend 必须在命令边界看到自己的 catalog 改动。
但其他 backend 不应在 commit 之前得到这组消息。
### 5.6. `DROP TABLE`：先锁所有目标，再按 dependency 图删除
`RemoveRelations()` 处理 `DROP TABLE`、`DROP INDEX`、`DROP SEQUENCE`、`DROP VIEW` 等 relation 类对象。
它先决定 lock mode。
默认是 `AccessExclusiveLock`。
`DROP INDEX CONCURRENTLY` 使用 `ShareUpdateExclusiveLock`，但有多项限制。
它不支持一次 drop 多个对象。
它不支持 `CASCADE`。
它对 partitioned index 还有额外限制。
普通 DROP 的主线是：
```text
AcceptInvalidationMessages()
RangeVarGetRelidExtended(..., lockmode, RangeVarCallbackForDropRelation)
检查 relkind / owner / persistence / partition 情况
构造 ObjectAddress list
performMultipleDeletions()
```
先 lock and validate all relation。
然后一次性删除。
这避免一个 DROP 列表中对象相互依赖时出现错误的 `DROP RESTRICT`。
进入 `dependency.c` 后，`performDeletion()` 会打开 `pg_depend`。
然后调用 `AcquireDeletionLock(object, 0)`。
注释说“理想情况下 caller 已经做了，但很多地方并不严格”。
再通过 `findDependentObjects()` 构造级联删除对象列表。
再由 `reportDependentObjects()` 判断 restrict/cascade。
最后 `deleteObjectsInList()` 执行具体删除。
这里的 ordering 是：
```text
先锁目标对象和需要的相关对象；
再计算 dependency 图；
再删除 catalog / storage / dependency 边；
最后在事务边界 invalidation 和释放锁。
```
如果先删 catalog 再计算 dependency 图，DROP 可能漏掉依赖对象。
如果先发 invalidation 再删除 catalog，其他 backend 可能重载到仍存在的对象。
如果不持锁算图，另一个 DDL 可以同时改变图结构。
### 5.7. `CREATE INDEX`：常规路径和 concurrently 路径对比
`DefineIndex()` 展示了一个很好的边界案例。
常规 `CREATE INDEX` 对 heap relation 使用 `ShareLock`。
源码注释说：常规 build 期间只允许 `SELECT FOR UPDATE/SHARE` 一类操作，`INSERT/UPDATE/DELETE` 被阻止。
`CREATE INDEX CONCURRENTLY` 使用 `ShareUpdateExclusiveLock`。
这显然比常规路径更弱。
但它不是简单“少拿锁也可以”。
它用多事务阶段、session-level lock、`WaitForLockers()`、`WaitForOlderSnapshots()` 和 `pg_index` 状态位补足正确性。
concurrent index build 的核心阶段包括：
```text
事务 1: 建 catalog entry，index 尚未 ready/valid
commit
等待可能用旧 index list 写表的事务
事务 2: build index，使其 ready for inserts
commit
再次等待旧状态事务
事务 3: validate missing entries，等待 older snapshots
更新 pg_index 标记 valid
CacheInvalidateRelcacheByRelid(parent table)
commit
```
这说明弱锁 DDL 不是违反 ordering。
它把“一个强锁保护的原子变化”拆成多个可观察状态，并在每个状态间加入等待和 invalidation。
如果只把锁降级，而不增加这些状态和等待，就会让 planner 或 executor 在 index 半可用时误用它。
### 5.8. commit：记录事务成功后、释放锁前，发送 shared invalidation
`CommitTransaction()` 中有一段注释直接回答本节主问题。
post-commit cleanup 的总体原则是：
先释放其他 backend 可见资源。
再释放 locks。
再释放 backend-local resources。
具体到 catalog change：
```text
AtEOXact_RelationCache(true)
AtEOXact_TypeCache()
AtEOXact_Inval(true)
ResourceOwnerRelease(... RESOURCE_RELEASE_LOCKS ...)
```
注释说 `AtEOXact_Inval(true)` 必须发生在 locks released 之前。
原因是如果有人在等我们修改过的 relation lock，希望它在开始使用 relation 前已经知道 catalog change。
`inval.c` 的顶部注释还给出另一半顺序。
commit 记录必须先于发送 SI messages。
否则其他 backend 收到 invalidation 后重读 catalog，也看不到更新 tuple 是 committed。
所以实际顺序是：
```text
RecordTransactionCommit()
ProcArrayEndTransaction()
AtEOXact_Inval(true)
ResourceOwnerRelease(... LOCKS ...)
```
这组顺序同时保证两件事。
其他 backend 收到消息后能看到 committed catalog tuple。
等待锁的 backend 醒来前已经有机会收到并处理 invalidation。
### 5.9. abort：不广播未提交 catalog 变化
`AbortTransaction()` 的 post-abort cleanup 也调用：
```text
AtEOXact_Inval(false)
ResourceOwnerRelease(... RESOURCE_RELEASE_LOCKS ...)
```
`AtEOXact_Inval(false)` 不发送 shared invalidation。
它只本地处理 `PriorCmdInvalidMsgs`。
`CurrentCmdInvalidMsgs` 可以忘掉。
原因很清楚：
未提交 catalog 变化其他 backend 根本不应该看见。
如果 abort 时广播，其他 backend 会因为一个不存在的事务事实丢 cache。
这通常只是成本问题，但也会让诊断变得混乱。
更重要的是当前 backend 曾经在 earlier command 中本地看见过自己的 catalog 改动。
abort 后它必须把这些本地缓存也清掉，不能在回滚后继续相信半成品对象。
### 5.10. 等待者醒来：锁和 invalidation 的闭环
现在把读者 backend B 加进来。
backend A 执行：
```sql
BEGIN;
ALTER TABLE t ADD COLUMN b int;
COMMIT;
```
backend B 在 A commit 前尝试：
```sql
SELECT * FROM t;
```
如果 A 持有 `AccessExclusiveLock`，B 会等待 relation lock。
A commit 时先记录事务成功。
再把 relcache/syscache invalidation 放入 SI queue。
再释放 lock。
B 被唤醒后，从 `LockRelationOid()` 或相关 relation open 路径继续，会处理 invalidation。
然后才用 relcache 解释 `t`。
这个闭环才是“DDL 对并发读者正确”的实际机制。
不是单纯因为 `AccessExclusiveLock`。
也不是单纯因为 relcache invalidation。
而是：
```text
锁让读者停在语义边界之外；
commit 让新 catalog tuple 成为事实；
invalidation 让读者丢弃旧解释；
lock release 让读者继续进入新语义。
```
## 6. 生命周期 / ownership / cleanup

### 6.1. 谁创建对象锁
DDL command 模块决定锁需求。
`AlterTableGetLockLevel()` 决定 ALTER TABLE 的最大锁级别。
`RemoveRelations()` 决定 DROP relation 的锁级别。
`DefineIndex()` 决定 CREATE INDEX 常规和 concurrent 路径的 lock mode。
general object 的 DROP 路径通过 `AcquireDeletionLock()` 兜底。
这些函数创建的不是普通内存对象。
它们在 shared lock table 中建立 `LOCKTAG` 对应的 lock/proclock 状态。
backend-local 的 `LOCALLOCK` 记录本 backend 的持有情况。
### 6.2. 谁持有对象锁
事务级 DDL lock 由当前 transaction 的 `ResourceOwner` 持有。
subtransaction 失败时，subtransaction owner 会释放或转移相应资源。
top-level commit/abort 时，`ResourceOwnerRelease()` 分阶段释放。
DDL 中常见 `relation_close(rel, NoLock)` 不释放 lock。
只有事务结束或显式 unlock 路径才释放。
session-level lock 是例外。
`CREATE INDEX CONCURRENTLY` 会用 `LockRelationIdForSession()` 持有 parent table lock，跨越内部事务 commit。
最后再 `UnlockRelationIdForSession()`。
它这样做是为了防止表或 index 在 concurrent build 多阶段之间被 drop。
### 6.3. 谁创建 catalog 事实
command 模块调用 catalog helper。
`tablecmds.c` 通过 `heap_create_with_catalog()`、`AddNewRelationTuple()`、`AddNewAttributeTuples()` 等路径创建 relation catalog。
`indexcmds.c` 调用 `index_create()` 和 `index_set_state_flags()` 等路径创建并推进 index catalog 状态。
DROP 路径通过 dependency deletion 调用 object-specific `Remove*()` 函数删除对应 catalog tuple。
这些修改都是事务性 heap 修改。
它们的 commit/abort 语义来自事务系统，不来自 DDL 自己的私有日志。
### 6.4. 谁创建 invalidation
对 catcache 相关 catalog tuple 的 insert/update/delete 会调用 `CacheInvalidateHeapTuple()`。
`pg_class`、`pg_attribute`、`pg_index`、某些 `pg_constraint` 变化还会推导出 relcache invalidation。
有些操作不是通过可自动推导的 catalog tuple 修改表达。
例如 dropping an index 需要强制 parent relation relcache rebuild。
这类路径会显式调用 `CacheInvalidateRelcache()` 或 `CacheInvalidateRelcacheByRelid()`。
plan cache 不直接由 DDL 函数手写遍历。
它注册 relcache/syscache callbacks。
当 invalidation 被本地执行时，callback 把相关 `CachedPlanSource` 或 `CachedPlan` 标成 invalid。
### 6.5. command boundary ownership
当前 command 的 invalidation 消息属于 `CurrentCmdInvalidMsgs`。
`CommandCounterIncrement()` 后，它们被本地执行并转入 `PriorCmdInvalidMsgs`。
这是一种“已对本 backend 生效，但还不能对外广播”的状态。
所以 CCI 是 DDL 生命周期中的核心边界。
没有 CCI，本事务后续命令可能看不到刚创建的对象。
过早 CCI，则会让当前 command 内部对旧 tuple 的合法使用变得不稳定。
### 6.6. transaction boundary ownership
commit 时，`AtEOXact_Inval(true)` 发送 prior 和 current invalidation。
发送后，TopTransactionContext 即将清空，不需要手工释放数组。
abort 时，`AtEOXact_Inval(false)` 只做本地 cleanup，不发送共享消息。
lock release 总是在 invalidation 收尾之后。
这个顺序的 owner 不是某个 command 模块。
它属于 `xact.c`。
这也是为什么不要在单个 DDL 函数里手动模拟“提交时通知其他 backend”。
正确位置是事务收尾。
## 7. 正确性机制层次

### 7.1. heavyweight lock 保证什么
heavyweight lock 保证对象级并发互斥。
`AccessExclusiveLock` 会和 `AccessShareLock` 冲突。
这意味着普通 SELECT 如果需要打开 relation，会被阻塞在 DDL 外。
`ShareUpdateExclusiveLock` 不会阻塞普通 SELECT，也允许部分写路径继续。
所以使用它的 DDL 必须额外保证读者看到的状态可解释。
lock 不能保证 cache 更新。
一个 backend 拿到锁之前可能已经有旧 relcache。
所以 `LockRelationOid()` 拿锁后还要 `AcceptInvalidationMessages()`。
lock 也不能保证 catalog tuple 可见。
commit record 和 ProcArray 状态才决定其他 backend 的 visibility 事实。
### 7.2. MVCC catalog 保证什么
MVCC catalog tuple 保证事务性。
未提交 DDL 对其他事务不可见。
abort 后 catalog tuple 变化回滚。
同一事务内通过 command id 区分前后命令。
MVCC catalog 不能保证正在执行的 query 停下来。
也不能保证 cached plan 自动重写。
更不能阻止两个 DDL 同时尝试修改同一对象定义。
所以它必须和 lock、CCI、invalidation 一起使用。
### 7.3. CommandCounterIncrement 保证什么
`CommandCounterIncrement()` 让当前事务的后续命令看到当前命令的写入。
它推进 `currentCommandId`。
它更新 static snapshots 的 command id。
它调用 `AtCCI_LocalCache()`。
它不是 commit。
它不会向其他 backend 广播 shared invalidation。
它只让当前 backend 从“当前命令内部语义”进入“下一命令语义”。
DDL 中缺失 CCI 的典型现象是：同一事务内后续命令查不到刚创建或刚修改的 catalog 状态。
过多 CCI 则会增加 command id 消耗和本地 cache churn。
### 7.4. invalidation 保证什么
invalidation 保证 cache 语义过期通知。
它让 syscache 丢旧 catalog tuple。
它让 relcache flush 或 rebuild relation descriptor。
它让 plan cache 标记依赖 relation/type/function 的 cached plan 无效。
invalidation 不保证互斥。
它不能让一个已经持有 relation refcount 的 executor 立刻停下。
它也不能阻止另一个 DDL 进入同一对象。
所以 `inval.c` 注释说：很多上层 cache 依赖 DDL 对“不能错过的 catalog changes”拿 `AccessExclusiveLock`。
如果用较弱锁修改 catalog，某些 backend 可能永远不会读到变化，除非相关 cache 像 relcache 对 concurrent index 那样支持 build 重试和额外边界。
### 7.5. relcache refcount 保证什么
relcache refcount 保证当前 backend 正在使用某个 relation descriptor。
它保护内存对象不被立即释放。
它不保证 descriptor 语义仍然最新。
收到 invalidation 时，refcount 为零的 entry 可以清除。
refcount 非零的 entry 可能被标记重建或延后处理。
这就是为什么 lock 仍然重要。
DDL 不能只发 invalidation 然后指望所有运行中的 query 立刻切到新 tuple descriptor。
对影响 SELECT 语义的 DDL，强锁会等待旧 query 离开对象边界。
### 7.6. plan cache 保证什么
plan cache 保证 prepared statement 或 generic plan 可以跨执行复用。
它不是 catalog 的真相来源。
它只根据 dependency list 和 invalidation callback 判断是否仍可用。
`CachedPlanSource.is_valid` 失效后，后续执行会重新 analyze/rewrite/plan。
已经开始执行的 plan 不会因为 invalidation 在中途被替换。
所以影响运行中 plan 安全性的 DDL 必须靠锁边界避免并发。
### 7.7. WAL 与 crash safety 的位置
本节重点不是 WAL-before-data。
但 catalog update 作为 heap update 仍然会产生 WAL。
commit record 确立事务成功。
`inval.c` 要求发送 SI messages 之前先记录 commit。
logical decoding 需要在 command end 记录 logical invalidations。
crash recovery 后，catalog 的持久事实来自 WAL redo。
shared invalidation queue 是运行期 cache 通知，不是持久事实。
不要把 SI message 当成 redo 机制。
### 7.8. ordering 不变量
本节最重要的不变量可以写成：
```text
任何 backend 在越过对象锁等待边界后，
如果相关 DDL 已经提交，
它必须先有机会处理对应 invalidation，
再使用该对象的 syscache/relcache/plancache 解释。
```
这个不变量分解为四个源码事实：
`LockRelationOid()` 拿锁后处理 incoming invalidation。
`CommandCounterIncrement()` 在本地命令边界处理 current command invalidation。
`AtEOXact_Inval(true)` 在 commit 后发送 shared invalidation。
`CommitTransaction()` 在释放 locks 前调用 `AtEOXact_Inval(true)`。
## 8. 错误路径 / 异常路径 / fallback

### 8.1. lock wait 和死锁
DDL 拿 heavyweight lock 可能等待。
等待不是异常。
这是正确性机制的一部分。
等待过程中，另一个 backend 可能提交 DDL 并发送 invalidation。
等待结束后，本 backend 必须吸收这些 invalidation。
`LockRelationOid()`、`LockDatabaseObject()`、`LockSharedObject()` 都体现了这个边界。
如果等待中形成 cycle，deadlock detector 会报错。
ERROR 后进入 abort cleanup。
已拿到的事务级锁通过 ResourceOwner 释放。
未提交 catalog 修改不会广播给其他 backend。
### 8.2. DDL 在改 catalog 后 ERROR
例如 `ALTER TABLE` Phase 2 改了部分 catalog，Phase 3 rewrite 或约束验证失败。
这些 catalog tuple 修改属于当前事务。
ERROR 会让事务进入 abort。
heap tuple 变化由事务 abort 回滚。
smgr pending delete/pending sync 按事务结局处理。
`AtEOXact_Inval(false)` 处理本地已生效 invalidation，不向其他 backend 发送 commit 消息。
对象锁在 abort 收尾后释放。
等待者醒来后不会看到未提交 catalog 成为事实。
### 8.3. subtransaction abort
DDL 通常不建议在复杂 subtransaction 中混用，但源码必须支持。
`inval.c` 对 subtransaction 的策略是：
subtransaction abort 时处理并丢弃它排队的事件。
subtransaction commit 时把事件并入 parent transaction。
这确保内层失败不会把 half-created catalog cache 状态泄漏到外层。
同样，ResourceOwner 树会处理 subtransaction 持有的资源。
### 8.4. `DROP INDEX CONCURRENTLY`
`DROP INDEX CONCURRENTLY` 是异常路径的好例子。
它使用弱锁。
它限制一次只能 drop 一个 index。
它不允许 `CASCADE`。
它通过 `PERFORM_DELETION_CONCURRENTLY` 进入特殊 deletion 行为。
这些限制不是语法保守。
它们是 ordering 成本。
弱锁意味着不能在一个强互斥窗口里同时解决所有 dependency、relcache、planner 和 executor 问题。
因此必须缩小语义范围。
### 8.5. `CREATE INDEX CONCURRENTLY`
`CREATE INDEX CONCURRENTLY` 不是普通 `CREATE INDEX` 的 lock mode 参数。
它跨多个事务。
它必须在中间 commit 后继续保护目标 table 不被 drop。
所以它使用 session-level lock。
它还必须等待旧写事务、等待旧 snapshot，并在最后显式 invalidate parent relcache。
如果某一步失败，index catalog 可能留下 invalid index。
后续 DROP 或 REINDEX 需要清理。
这是 concurrent DDL 用可恢复中间状态换取低阻塞的成本。
### 8.6. SI message overflow
shared invalidation queue 可能 overflow。
backend 处理 overflow 时不能只丢一个对象 entry。
`RelationCacheInvalidate()` 会做更大范围 flush。
这牺牲局部性，换正确性。
relcache 全量失效成本高，但比继续使用无法证明正确的缓存安全。
这也是 invalidation 设计中的 fallback：
消息可以不精确。
不能漏掉必须失效的语义。
### 8.7. relcache build 中收到 invalidation
`relcache.c` 对正在 build 的 entry 有特殊处理。
`RelationCacheInvalidateEntry()` 找不到正式 entry 时，会检查 `in_progress_list`。
如果正在 build 的 relation 被 invalidated，会标记 `invalidated`。
后续 build 需要重试。
这解决“新 cache entry 刚出生就过期”的问题。
很多上层 cache 没有这种保护。
它们依赖 DDL 的强锁避免错过不能错过的变化。
### 8.8. hot standby 的额外约束
`AlterTableGetLockLevel()` 注释提醒：Hot Standby 只知道 primary 上的 `AccessExclusiveLock`。
如果某些变化会影响 standby 上正在运行的 SELECT，就不能轻易降低锁级别。
这不是本节主线，但它说明 lock strength 不是只服务 primary 本地 executor。
它也服务 WAL/recovery/standby 的语义边界。
## 9. 成本、资源与跨模块传播

### 9.1. lock contention 成本
DDL 的第一类成本是 lock wait。
`AccessExclusiveLock` 会阻塞普通 `AccessShareLock`。
长查询、idle in transaction、prepared transaction 都可能延迟 DDL。
DDL 等待时又可能阻塞排在它后面的普通查询。
这会形成 operational 上常见的锁队列放大。
成本随 backend 数、长事务数量、目标 relation 热度和 lock mode 强度扩张。
### 9.2. invalidation fan-out 成本
commit 时发送 shared invalidation 不是免费。
每个 backend 在合适边界都需要接收和处理消息。
relcache flush 可能导致后续 catalog lookup、tuple descriptor rebuild、index list reload。
plancache invalidation 可能导致 prepared statement 下一次执行重新 rewrite/plan。
成本随 backend 数、被修改对象数、依赖该对象的 cached plan 数和 catalog cache 热度扩张。
### 9.3. CCI 成本
DDL command 内显式 `CommandCounterIncrement()` 会让本地 cache boundary 前移。
它可能触发 `CommandEndInvalidationMessages()`。
它也消耗 command id。
单个事务内大量 DDL 或自动生成大量对象，会出现 CCI 和 local cache churn。
这通常不是普通 OLTP 的主瓶颈，但对 migration、extension install、partition maintenance 可能明显。
### 9.4. dependency 图成本
DROP 不只是删除目标对象。
它要打开 `pg_depend`，递归查找依赖对象，报告 restrict/cascade，再逐个删除。
依赖图规模会随对象数量、分区层级、constraint、index、view、rule、type、function 依赖扩张。
DDL 锁持有时间会包含这段 dependency 分析和删除成本。
### 9.5. relcache rebuild 成本
relcache rebuild 会读取多个 catalog。
普通表要读 `pg_class`、`pg_attribute`、`pg_attrdef`、`pg_constraint`、`pg_index` 等。
分区表还要构建 partition descriptor。
索引还要构建 opclass、operator、collation、expression、predicate 等状态。
一次 DDL 对 hot relation 的 relcache invalidation，可能让多个 backend 在下次使用时同时付 rebuild 成本。
### 9.6. plan cache 和 planner 成本
DDL invalidation 可能让 saved plan 失效。
下一次执行 prepared statement 时，parse/analyze/rewrite/plan 成本会重新出现。
如果应用大量使用 server-side prepared statements，DDL 后的第一波请求可能出现延迟尖刺。
这个尖刺不是 DDL commit 时完全支付的。
它被推迟到各 backend 下一次使用相关 plan 时支付。
### 9.7. concurrent DDL 成本
`CREATE INDEX CONCURRENTLY` 降低 lock 阻塞，但增加总成本。
它多次启动事务。
它等待 lockers。
它等待 older snapshots。
它要 validate index。
它可能产生更多 WAL 和 IO。
它还可能留下 invalid index 需要人工处理。
这是典型的：
```text
降低前台互斥窗口
  换取更长生命周期、更多状态和更多 cleanup 复杂度
```
### 9.8. 跨模块传播
DDL ordering 影响 parser/analyzer。
name lookup 和 type lookup 依赖 syscache。
DDL ordering 影响 planner。
relation stats、index list、constraint、partition descriptor、RLS/policy 都可能来自 catalog/relcache。
DDL ordering 影响 executor。
tuple descriptor、trigger、rule、index insert decision、HOT safety 都依赖 relation descriptor。
DDL ordering 影响 storage。
table rewrite、relfilenode change、smgr pending delete 要在事务结局后清理。
DDL ordering 影响 replication/recovery。
commit WAL、logical invalidations、standby conflict、AccessExclusiveLock WAL 记录都参与最终可见语义。
## 10. 观测与诊断入口

### 10.1. 观测锁等待
最直接入口是 `pg_locks` 和 `pg_stat_activity`。
```sql
SELECT pid, locktype, database, relation::regclass, mode, granted
FROM pg_locks
WHERE relation = 'public.t'::regclass
ORDER BY granted, mode;
```
查看阻塞链：
```sql
SELECT pid, pg_blocking_pids(pid), wait_event_type, wait_event, query
FROM pg_stat_activity
WHERE wait_event_type = 'Lock';
```
能看到的是 lock wait。
看不到的是某个 backend 的 relcache 是否已经被 flush。
### 10.2. 观测 DDL 对 prepared statement 的影响
可以用 prepared statement 做现象实验。
```sql
PREPARE q AS SELECT * FROM t;
EXECUTE q;
ALTER TABLE t ADD COLUMN b int;
EXECUTE q;
```
第二次 `EXECUTE` 可能触发重新分析或报 result type 变化相关错误，具体取决于语句、返回描述符和客户端协议。
诊断时要区分：
plan 被 invalidated 是 expected。
客户端是否允许 result descriptor 改变是另一层协议问题。
### 10.3. 观测 `CREATE INDEX CONCURRENTLY`
查看进度：
```sql
SELECT pid, phase, lockers_total, lockers_done, current_locker_pid
FROM pg_stat_progress_create_index;
```
结合锁等待：
```sql
SELECT pid, wait_event_type, wait_event, query
FROM pg_stat_activity
WHERE query LIKE '%CREATE INDEX CONCURRENTLY%';
```
如果停在 waiting for old snapshots，问题常常不是锁，而是长 snapshot。
这时需要看 long transaction、replication slot、idle in transaction。
### 10.4. 观测 invalidation 的间接现象
PostgreSQL 没有一个普通 SQL 视图直接列出每个 backend 待处理的 SI message。
常用方法是间接观察。
DDL 后 prepared statement 首次执行变慢。
DDL 后某个 backend 重新读取 catalog。
DDL 后 relcache init file 被重建。
debug 构建中可以在这些函数断点：
```text
AcceptInvalidationMessages
LocalExecuteInvalidationMessage
CommandEndInvalidationMessages
AtEOXact_Inval
RelationCacheInvalidateEntry
PlanCacheRelCallback
```
这些断点能把 runtime 现象接回源码。
### 10.5. 观测 abort cleanup
可以在一个事务中制造 DDL 失败。
```sql
BEGIN;
CREATE TABLE ddl_abort_demo(a int);
ALTER TABLE ddl_abort_demo ADD CONSTRAINT bad CHECK (a > 0) NOT VALID;
SELECT oid FROM pg_class WHERE relname = 'ddl_abort_demo';
ROLLBACK;
SELECT oid FROM pg_class WHERE relname = 'ddl_abort_demo';
```
如果中间命令可见，说明 CCI 和当前事务 catalog visibility 正常。
ROLLBACK 后对象消失，说明 catalog tuple 仍然是事务性事实。
但普通 SQL 视图无法告诉你 `AtEOXact_Inval(false)` 处理了哪些本地消息。
那需要断点或临时日志。
### 10.6. 源码断点建议
`ALTER TABLE` 主链路：
```text
AlterTableGetLockLevel
RangeVarGetRelidExtended
LockRelationOid
AlterTable
ATController
ATRewriteCatalogs
CommandCounterIncrement
CommandEndInvalidationMessages
AtEOXact_Inval
```
`DROP TABLE` 主链路：
```text
RemoveRelations
RangeVarCallbackForDropRelation
performDeletion
AcquireDeletionLock
deleteObjectsInList
CacheInvalidateHeapTuple
AtEOXact_Inval
```
`CREATE INDEX CONCURRENTLY` 主链路：
```text
DefineIndex
LockRelationIdForSession
WaitForLockers
index_concurrently_build
validate_index
WaitForOlderSnapshots
index_set_state_flags
CacheInvalidateRelcacheByRelid
```
### 10.7. 能看到与看不到
能直接看到：
lock mode、granted 状态、wait event、blocking pid。
`pg_stat_progress_create_index` 的 concurrent index 阶段。
catalog tuple 的提交后状态。
prepared statement 下一次执行是否重新规划或失败。
只能推断：
某个 backend 是否刚处理过某条 relcache invalidation。
某个 cached plan 是否因为哪个具体 catalog tuple 失效。
几乎不可见：
SI queue 中每个 backend 待消费消息的精确列表。
relcache build 中途被 invalidated 后重试的内部瞬间。
## 11. 常见误区

误区一：DDL 只要改 catalog 就完成了。
实际还需要对象锁、dependency、CCI、local invalidation、commit invalidation、lock release 和 cleanup 顺序。
误区二：invalidation 是一种 lock。
invalidation 只告诉 cache 语义过期。
它不阻塞并发，不等待 executor，不提供互斥。
误区三：lock 能让所有 cache 自动变新。
lock 只建立等待边界。
拿锁后仍要处理 invalidation。
这就是 `LockRelationOid()` 拿锁后调用 `AcceptInvalidationMessages()` 的意义。
误区四：catalog tuple 一更新就可以立刻 flush cache。
同一个 command 内旧 tuple 仍可能有效。
所以 invalidation 排队到 command boundary。
误区五：commit 时先释放锁再通知也可以。
不可以。
等待者可能在收到 invalidation 前用旧 relcache 进入新语义。
`CommitTransaction()` 明确把 `AtEOXact_Inval(true)` 放在 lock release 前。
误区六：abort 不需要处理 invalidation。
对其他 backend 不需要广播。
但本 backend 之前 command 可能已经本地看过自己的 catalog 变化。
abort 必须清理这些本地缓存语义。
误区七：`CREATE INDEX CONCURRENTLY` 说明弱锁 DDL 都是安全的。
不是。
它安全是因为多阶段 catalog 状态、等待旧事务、等待旧 snapshot、session lock 和额外 relcache invalidation 共同补足。
误区八：`pg_locks` 能解释所有 DDL 现象。
`pg_locks` 只能解释等待和冲突。
它不能直接解释 relcache/plancache 是否已经失效，也不能显示 CCI 边界。
## 12. 课堂实验

### 实验 1：确认 commit 前锁阻塞，commit 后读者看到新定义
准备：
```sql
CREATE TABLE ddl_order_demo(a int);
INSERT INTO ddl_order_demo VALUES (1);
```
会话 A：
```sql
BEGIN;
ALTER TABLE ddl_order_demo ADD COLUMN b int DEFAULT 10;
SELECT pg_backend_pid();
```
会话 B：
```sql
SELECT * FROM ddl_order_demo;
```
观察 B 是否等待。
会话 C：
```sql
SELECT pid, pg_blocking_pids(pid), wait_event_type, wait_event, query
FROM pg_stat_activity
WHERE query LIKE '%ddl_order_demo%';
```
会话 A：
```sql
COMMIT;
```
会话 B 继续后，结果应按提交后的定义解释。
源码回扣：
```text
AlterTableGetLockLevel
LockRelationOid
ATController
AtEOXact_Inval(true)
ResourceOwnerRelease(... LOCKS ...)
```
### 实验 2：验证 rollback 后 catalog 事实消失
会话 A：
```sql
BEGIN;
CREATE TABLE ddl_rollback_demo(a int);
SELECT oid FROM pg_class WHERE relname = 'ddl_rollback_demo';
ROLLBACK;
SELECT oid FROM pg_class WHERE relname = 'ddl_rollback_demo';
```
第一个 `SELECT` 在事务内能看到对象。
`ROLLBACK` 后看不到。
这说明 DDL catalog update 是事务性 heap 修改。
源码回扣：
```text
CommandCounterIncrement
CommandEndInvalidationMessages
AbortTransaction
AtEOXact_Inval(false)
```
### 实验 3：prepared statement 和 relcache/plancache invalidation
准备：
```sql
DROP TABLE IF EXISTS ddl_plan_demo;
CREATE TABLE ddl_plan_demo(a int);
INSERT INTO ddl_plan_demo VALUES (1);
PREPARE p AS SELECT * FROM ddl_plan_demo;
EXECUTE p;
```
然后：
```sql
ALTER TABLE ddl_plan_demo ADD COLUMN b int;
EXECUTE p;
```
观察第二次执行的行为。
如果使用固定 result descriptor 的客户端协议，可能看到返回类型变化相关错误。
如果路径允许重新描述，可能看到重新规划后的结果。
源码回扣：
```text
CacheInvalidateRelcacheByRelid
PlanCacheRelCallback
CachedPlanSource.is_valid
```
实验目标不是记住某个客户端表现。
目标是确认 DDL invalidation 会传播到 plan cache，而不是只影响 relcache。
### 实验 4：`CREATE INDEX CONCURRENTLY` 的多阶段等待
准备一张较大的表：
```sql
CREATE TABLE ddl_cic_demo(a int, b text);
INSERT INTO ddl_cic_demo
SELECT g, md5(g::text)
FROM generate_series(1, 200000) g;
```
会话 A 开长事务：
```sql
BEGIN;
SELECT count(*) FROM ddl_cic_demo;
```
会话 B：
```sql
CREATE INDEX CONCURRENTLY ddl_cic_demo_a_idx ON ddl_cic_demo(a);
```
会话 C：
```sql
SELECT pid, phase, lockers_total, lockers_done, current_locker_pid
FROM pg_stat_progress_create_index;
```
再看：
```sql
SELECT pid, state, backend_xmin, xact_start, query
FROM pg_stat_activity
WHERE backend_xmin IS NOT NULL
ORDER BY xact_start;
```
源码回扣：
```text
DefineIndex
WaitForLockers
WaitForOlderSnapshots
CacheInvalidateRelcacheByRelid
```
实验目标是看到弱锁 DDL 把一次强互斥改成多个可等待阶段。
### 实验 5：gdb 观察 lock 后 invalidation
在调试构建上设置断点：
```text
break LockRelationOid
break AcceptInvalidationMessages
break AtEOXact_Inval
break RelationCacheInvalidateEntry
```
让会话 A 执行 DDL 并停在 commit 前。
让会话 B 尝试访问同一 relation 并等待。
继续会话 A commit。
观察会话 B 从等待返回后是否进入 `AcceptInvalidationMessages()`。
这个实验直接验证：
```text
wait for lock
  -> accept invalidation
  -> use relation metadata
```
### 实验 6：源码改动 thought experiment
不要真的提交修改。
只在本地临时阅读或加日志。
尝试思考三个错误 patch：
```text
1. 在 CommitTransaction() 中把 AtEOXact_Inval(true) 移到 lock release 之后。
2. 在 CacheInvalidateHeapTuple() 中立即 LocalExecuteInvalidationMessage()。
3. 在 LockRelationOid() 中删掉 AcceptInvalidationMessages()。
```
分别预测现象：
第一个 patch 可能让等待者醒来后先用旧 relcache。
第二个 patch 可能让同一 command 内重新装载旧 tuple 或破坏 command visibility。
第三个 patch 可能让等待锁的 backend 锁已拿到但缓存还停在等待前。
把预测和 `inval.c`、`lmgr.c`、`xact.c` 注释对应起来。
## 13. 讨论题

1. 为什么 `LockRelationOid()` 在拿锁后还要 `AcceptInvalidationMessages()`，而不是只依赖 shared invalidation 在事务开始时处理？
2. 为什么 `AtEOXact_Inval(true)` 必须在释放 locks 之前，而不是事务完全清理后再发？
3. 为什么 `CacheInvalidateHeapTuple()` 不在 heap update/delete 当场 flush cache？
4. 如果 `ALTER TABLE` 用较弱锁修改会影响 SELECT 的定义，哪些 backend-local 状态可能错过变化？
5. `CREATE INDEX CONCURRENTLY` 为什么需要 `WaitForLockers()` 和 `WaitForOlderSnapshots()`，而不只是创建 catalog tuple 后发 invalidation？
6. `DROP TABLE` 为什么要先收集对象并做 dependency 删除，而不是每解析到一个名字就立即删除？
7. plan cache invalidation 为什么通常只让下一次执行重规划，而不是中断正在执行的 plan？
8. 如果一个扩展新增 DDL-like command，应该在哪些边界上考虑 lock、catalog update、dependency 和 invalidation？
## 14. 本节小结

本节唯一主问题是：
```text
DDL 为什么必须先拿对象锁，再改 catalog，再发 invalidation？
```
答案不是“源码刚好这么写”。
答案是三种状态分别服务不同正确性边界。
对象锁让不兼容的 backend 停在同一对象语义边界之外。
catalog tuple 让对象定义成为事务性事实。
invalidation 让 backend-local cache 不再解释旧事实。
正确主链路是：
```text
先决定 lock mode
  -> 拿对象锁并处理已有 invalidation
  -> 修改 catalog / dependency / storage state
  -> CCI 在本 backend 命令边界处理本地 invalidation
  -> commit record 让 catalog tuple 成为 committed fact
  -> AtEOXact_Inval(true) 广播给其他 backend
  -> 释放对象锁
```
abort 路径不广播未提交变化。
但它必须清理本 backend 已经在 prior command 中看过的本地缓存语义。
错误顺序破坏的是等待者醒来后的解释一致性。
先改 catalog 再拿锁，会让并发 backend 穿过旧/新定义夹缝。
先 invalidation 再改 catalog，会让别人重载旧 tuple。
先释放锁再发 invalidation，会让等待者用旧 relcache 解释新 catalog。
只靠 lock，不会更新 syscache、relcache、plancache。
只靠 invalidation，不能阻止正在运行的 executor 或另一个 DDL。
可迁移规律：
```text
当一个系统用事务性 metadata 定义运行时对象语义，
并允许每个 worker 缓存这套 metadata 时，
正确性通常依赖三段式 ordering：
先用 lock 建立互斥边界，
再提交 metadata 事实，
最后在释放等待者前传播 cache invalidation。
```
这些判断仍然依赖 DDL 类型、lock mode、是否 concurrent、backend 数、长事务、standby、prepared statement 使用方式和 PostgreSQL 版本。
诊断时先看 `pg_locks` 和 `pg_stat_activity` 找等待，再回到 `inval.c`、`lmgr.c`、`xact.c` 判断缓存和事务边界。
