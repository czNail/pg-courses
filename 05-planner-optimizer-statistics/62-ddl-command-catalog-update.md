# PostgreSQL DDL command / catalog update
## 课程定位
前置知识：已经理解 PostgreSQL 一条 SQL 会经过 parser、parse analysis、rewrite、planner 和 executor。 也应该知道 catalog 本身是一组普通 heap relation，只是访问路径被 syscache、relcache 和 dependency 机制包裹起来。 本节唯一主问题：
```text
CREATE/ALTER/DROP 如何修改 catalog、记录 dependency、发出 invalidation，并在事务 abort 时回滚？
```
本节核心矛盾：
```text
DDL 必须立刻改变当前 backend 的对象语义，方便同一事务后续命令继续使用；
但这个改变又必须对其他事务按 MVCC、lock、dependency 和 invalidation 边界可见，
并且在 ERROR 或事务 abort 时像普通数据修改一样被撤销。
```
一句话运行模型：
```text
utility command 在持有对象锁后，把 pg_class、pg_attribute、pg_type、pg_constraint、pg_depend 等 catalog 当作事务性 heap 表更新；
CommandCounterIncrement 让本事务后续命令看到新 catalog 版本；
catcache/relcache invalidation 先在本地命令边界生效，提交时再通过 shared invalidation 通知其他 backend；
abort 时事务性 catalog tuple、物理文件 pending delete、pending inval 和 lock 都按各自 owner 回滚或丢弃。
```
学完后应能判断：
- 一个 DDL 失败后为什么 catalog 不留下半成品。
- 一个 DDL 成功但未提交时，为什么同事务能看见而其他事务通常看不见。
- 为什么有些 DDL 需要显式 `CommandCounterIncrement()`。
- 为什么 dependency 是 DROP 判断的图，而不是执行时的锁。
- 为什么 invalidation 不是并发互斥机制。
- 如何从 SQL 现象回到 `tablecmds.c`、`heap.c`、`dependency.c`、`inval.c` 和 `xact.c`。
本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e90741e12a3715909e1b2d71ff9344`。
## 1. 本节在总主线中的位置
05 目录前面的课程已经从 raw parser、parse analysis、rewrite、RLS 和 planner 入口讲到优化器。 那条主线主要关注一条可优化查询如何进入 planner。 DDL 的路径不同。
它通常不会进入普通 planner/executor 的 row pipeline。 它在 `ProcessUtility()` 一侧执行。 但是 DDL 产生的 catalog 状态会反过来决定后续 parse analysis、rewrite、planner 和 executor 的行为。
例如：
```sql
BEGIN;
CREATE TABLE t(a int DEFAULT 1);
INSERT INTO t DEFAULT VALUES;
ROLLBACK;
```
这段 SQL 中，`INSERT` 能在同一个事务里看到刚创建的表和默认值。 `ROLLBACK` 后，表又完全消失。 这个现象不是由一个“DDL 私有事务系统”完成的。
它依赖普通 heap tuple 的 MVCC、catalog index、command id、syscache、relcache、dependency、smgr pending delete 和 shared invalidation 的组合。 本节把视角放在 utility command 的 catalog side effect 上。 主线是：
```text
raw DDL parse node
  -> ProcessUtility 分派
  -> command module 持锁和语义检查
  -> catalog helper 写系统表
  -> dependency helper 写 pg_depend
  -> invalidation helper 排队缓存失效
  -> CCI 让本事务继续使用新对象
  -> commit 发送 sinval / abort 丢弃或回滚
```
这里不展开每一种 DDL 的语法细节。 也不把 `CREATE TABLE`、`CREATE INDEX`、`CREATE TYPE`、`CREATE FUNCTION` 各讲成一节源码百科。 本节只追问一个问题：
```text
DDL side effect 如何在“立即可用”和“事务可回滚”之间保持一致？
```
后续阅读 dependency、relcache、syscache、plan cache 和 storage cleanup 时，都可以把本节当作一个连接点。
## 2. 核心矛盾与一句话运行模型
DDL 和 DML 的共同点是：都在事务中修改 heap tuple。 区别是：DDL 修改的是 PostgreSQL 用来理解数据库自身的 heap tuple。 `pg_class` 的一行决定 relation 是否存在。
`pg_attribute` 的多行决定 tuple descriptor。 `pg_type` 的一行决定 rowtype 或用户类型。 `pg_constraint`、`pg_attrdef`、`pg_index` 决定约束、默认值和 index 语义。
`pg_depend` 决定 DROP 图。 这些 catalog tuple 一旦改变，本 backend 的 syscache、relcache、typcache 和 plan cache 都可能变成旧语义。 所以 DDL 的核心 tension 是：
```text
catalog 也是普通 MVCC 数据
  vs
catalog 又定义“如何解释普通数据和后续命令”的元数据
```
如果只按普通 heap 修改处理，当前事务内刚创建的表可能无法立即打开。 如果绕过事务直接改全局内存，abort 就无法撤销。 如果只靠 lock 保证一致，其他 backend 已经缓存的 relcache 不会自动重建。
如果只靠 invalidation，DROP 的 cascade/restrict 图又无法回答。 PostgreSQL 的选择是分层组合：
```text
MVCC tuple 负责事务可见性和 abort 回滚。
CommandId/CCI 负责同一事务内命令间可见性。
lock 负责并发 DDL/DML 的逻辑互斥。
dependency 负责对象生命周期图。
invalidation 负责缓存语义过期传播。
smgr pending delete 负责物理文件在事务结局后的清理。
```
这一组机制没有一个能单独替代其他机制。 因此本节读源码时不要问“DDL 到底靠哪个机制保证正确”。 更准确的问题是：
```text
每一层分别保证什么，又在哪个边界把责任交给下一层？
```
## 3. 核心文件分工与阅读顺序
| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/tcop/utility.c` | utility statement 分派；本节只把它作为入口边界。 |
| 2 | `src/backend/commands/tablecmds.c` | `DefineRelation()`、`RemoveRelations()`、`ALTER TABLE` 分阶段执行和 relation 级锁。 |
| 3 | `src/backend/catalog/heap.c` | `heap_create_with_catalog()`、`AddNewRelationTuple()`、`AddNewAttributeTuples()`、`StoreConstraints()`、`AddRelationNewConstraints()`。 |
| 4 | `src/backend/commands/indexcmds.c` | `DefineIndex()`，展示另一个 DDL command module 如何调用 catalog 层。 |
| 5 | `src/backend/catalog/index.c` | `index_create()`、`index_constraint_create()` 写 `pg_class`、`pg_index`、dependency 和 relcache invalidation。 |
| 6 | `src/backend/commands/typecmds.c` | `GenerateTypeDependencies()`、`RemoveTypeById()`，展示非 relation 对象的 catalog/dependency 更新。 |
| 7 | `src/backend/commands/functioncmds.c` / `src/backend/catalog/pg_proc.c` | SQL 侧解析 `CREATE FUNCTION`，`ProcedureCreate()` 写 `pg_proc`，`RemoveFunctionById()` 处理 DROP。 |
| 8 | `src/backend/commands/dropcmds.c` | `RemoveObjects()` 把 parsed DROP 转成 `ObjectAddress` 并进入 dependency 删除。 |
| 9 | `src/backend/catalog/dependency.c` | `performDeletion()`、`findDependentObjects()`、`recordDependencyOnExpr()`、`deleteObjectsInList()`。 |
| 10 | `src/backend/utils/cache/inval.c` | `CacheInvalidateHeapTuple()`、`CacheInvalidateRelcache()`、`CommandEndInvalidationMessages()`。 |
| 11 | `src/backend/utils/cache/relcache.c` | `AtEOXact_RelationCache()`，说明 relcache 引用在事务结尾如何清理。 |
| 12 | `src/backend/storage/smgr/smgr.c` | `smgrDoPendingDeletes()`，说明 CREATE/DROP 的物理文件副作用如何按 commit/abort 收尾。 |
| 13 | `src/backend/access/transam/xact.c` | `CommandCounterIncrement()`、`AtEOXact_*` 顺序、invalidation 与 resource cleanup 的入口。 |
| 14 | `src/backend/catalog/namespace.c` | `RangeVarGetAndCheckCreationNamespace()`、对象名解析和 namespace 锁。 |
| 15 | `src/include/catalog/objectaddress.h` | `ObjectAddress` 是 dependency 和 generic DROP 的对象身份。 |
| 16 | `src/include/catalog/dependency.h` | `DependencyType` 的语义边界。 |
| 17 | `src/include/utils/inval.h` | cache invalidation 对外 API。 |
推荐阅读顺序不是从 `ProcessUtility()` 的巨大 switch 开始背。 更好的顺序是先看一条具体路径：
```text
CREATE TABLE
  -> DefineRelation()
  -> heap_create_with_catalog()
  -> catalog tuple inserts
  -> dependency records
  -> CommandCounterIncrement()
  -> relation_open()
  -> AddRelationNewConstraints()
  -> CommandCounterIncrement()
```
然后看 DROP 如何走另一条路径：
```text
DROP TABLE
  -> RemoveRelations()
  -> ObjectAddress
  -> performDeletion()
  -> findDependentObjects()
  -> reportDependentObjects()
  -> deleteObjectsInList()
  -> object-specific Remove*()
```
最后把两条路径和 invalidation、abort 回滚接起来。 这样读能避免两个常见误判。 第一，`pg_depend` 不是创建对象时的注释信息。
它是后续 DROP 的图结构输入。 第二，`CacheInvalidateRelcache()` 不是把别的 backend 立刻停住。 它只是把“你的缓存语义过期了”放到正确的命令/事务边界传播。
## 4. 关键数据结构与状态
### 4.1. catalog tuple
DDL 最直接改变的是 catalog tuple。 对 `CREATE TABLE t(a int)`，至少会涉及：
| catalog | 典型状态 |
| --- | --- |
| `pg_class` | `t` 的 relation row，包含 relkind、relnamespace、relowner、reltype、relfilenode/relfilenumber、relam、relpersistence 等。 |
| `pg_attribute` | 每一列一行，包括 attrelid、attname、atttypid、attnum、attnotnull、attgenerated 等。 |
| `pg_type` | relation rowtype 及其 array type。 |
| `pg_depend` | relation 对 namespace、owner、access method、extension、rowtype、default expression 等依赖。 |
| `pg_constraint` | CHECK、NOT NULL、PK/FK/UNIQUE 等约束记录。 |
| `pg_attrdef` | column default 或 generated expression。 |
| `pg_index` | index relation 与 table 的语义连接。 |
| `pg_inherits` | inheritance 或 partition 关系。 |
这些 catalog 都是 heap relation。 写入方式不是“修改 C struct 后刷新全局状态”。 而是通过 `CatalogTupleInsert()`、`CatalogTupleUpdate()`、`CatalogTupleDelete()` 之类路径产生事务性 heap 版本。
raw field 不是语义。 例如 `pg_class.relfilenode` 或当前版本代码里的 `RelFileNumber` 只描述物理存储身份的一部分。 它必须和 tablespace、database、relmapper、persistence、SMgrRelation、pending delete 状态一起解释。
又例如 `pg_attribute.attisdropped` 表示列被逻辑删除。 它不是立即重写所有历史 tuple 的物理布局。
### 4.2. `ObjectAddress`
dependency 和 generic DROP 不直接拿 SQL 名称工作。 它们使用 `ObjectAddress`：
```c
typedef struct ObjectAddress
{
    Oid classId;
    Oid objectId;
    int32 objectSubId;
} ObjectAddress;
```
语义是：
```text
classId    -> 哪个 catalog 描述这个对象，例如 pg_class 或 pg_proc
objectId   -> 该 catalog row 的 OID
objectSubId -> 子对象编号，例如 table column 的 attnum；0 表示整个对象
```
这个三元组让 `pg_depend` 可以同时表达：
- 表依赖 schema。
- 列默认值依赖函数。
- constraint 依赖 table 或 column。
- index constraint 依赖 index relation。
- partition index 依赖父 index 和分区。
不要把 `objectId` 单独当对象身份。 同一个 OID 值在不同 catalog 中没有同一语义。
### 4.3. `pg_depend` 与 `DependencyType`
`pg_depend` 的关键字段组合是：
```text
classid,objid,objsubid
  -> refclassid,refobjid,refobjsubid
  -> deptype
```
前半段是 depender。 后半段是 referenced object。 `deptype` 决定 DROP 时如何解释这条边。
本节只需要掌握常见类型：
| 类型 | 语义 |
| --- | --- |
| `DEPENDENCY_NORMAL` | 普通依赖；被引用对象不能在 `RESTRICT` 下删除，在 `CASCADE` 下会带着依赖者删除。 |
| `DEPENDENCY_AUTO` | 依赖者可随被引用对象自动删除，常用于实现细节对象。 |
| `DEPENDENCY_INTERNAL` | 依赖者是被引用对象的内部组成部分，用户不能单独删除。 |
| `DEPENDENCY_EXTENSION` | 对象属于 extension，DROP extension 统一管理。 |
| `DEPENDENCY_PIN` | pinned object 不应被普通 DROP 删除。 |
| `DEPENDENCY_PARTITION_PRI` / `DEPENDENCY_PARTITION_SEC` | partition 相关对象的主/次依赖边，支持分区对象生命周期。 |
dependency 不是 runtime refcount。 它不表示某个 backend 正在使用对象。 它也不是锁。
它是持久化在 catalog 里的生命周期图。
### 4.4. command id 与 CCI
同一事务内，PostgreSQL 用 command id 区分“前一个命令写的版本”和“当前命令正在写的版本”。 `CommandCounterIncrement()` 的关键动作包括：
```text
currentCommandId += 1
SnapshotSetCommandId(currentCommandId)
CommandEndInvalidationMessages()
```
只有当当前 command id 被使用过时，CCI 才真正递增。 这避免只读命令耗尽 command id 空间。 DDL 中 CCI 的典型用途是：
```text
刚写入 pg_class/pg_attribute/pg_type
  -> CCI
  -> relation_open(newOid) 能通过 syscache/relcache 看见新 catalog tuple
```
如果漏掉 CCI，当前事务后续步骤可能像“表还不存在”一样失败。 如果过早 CCI，又可能把尚未自洽的 catalog 半成品暴露给本 backend 的后续步骤。
### 4.5. syscache、relcache、typcache
catalog tuple 不会每次都从 heap 重新扫。 PostgreSQL 有多层缓存：
| 缓存 | 典型内容 | 本节关注点 |
| --- | --- | --- |
| catcache/syscache | 按 catalog key 查到的 tuple | catalog tuple 变更后要丢弃旧条目。 |
| relcache | `RelationData`，包括 `pg_class`、tupledesc、index list、partition info 等派生状态 | DDL 常常需要 relcache invalidation。 |
| typcache | 类型、比较函数、opclass 等类型派生信息 | `CREATE/ALTER TYPE`、函数/operator 变化会影响。 |
| plan cache | 解析/重写/规划后的可复用计划 | catalog invalidation 会让 cached plan 失效。 |
这些缓存的 owner 多数是 backend-local memory。 其他 backend 不能直接释放你的缓存条目。 所以 invalidation 的语义是消息驱动：
```text
我修改了某个 catalog/object。
你在合适边界处理消息。
处理后你的本地缓存语义过期，需要重新查 catalog 或 rebuild。
```
### 4.6. invalidation message
`inval.c` 中有当前事务的 invalidation state。 核心分组是：
```text
CurrentCmdInvalidMsgs
PriorCmdInvalidMsgs
```
当前命令修改 catalog 时，invalidation 先进入 current command 列表。 `CommandEndInvalidationMessages()` 在 CCI 时本地处理 current 列表。 随后把消息移到 prior command 列表。
事务提交时，prior messages 才会发送到 shared invalidation queue。 事务 abort 时，这些尚未提交的消息不会作为 committed change 广播给其他 backend。 这解释了一个重要边界：
```text
本 backend 需要立即忘掉自己刚改旧了的缓存；
其他 backend 只有在提交后才需要知道这个已提交变更。
```
### 4.7. physical storage 与 pending delete
有存储的 relation 创建时，不只写 catalog。 `heap_create()` 还会创建 relcache entry 和物理文件。 源码注释强调：如果后续失败，移除磁盘文件是 smgr 的责任。
这不是说文件创建立即不可回滚。 而是说物理文件 cleanup 通过 storage manager 的 pending delete / transaction callback 完成。 事务提交后，DROP 产生的旧文件可以真正移除。
事务 abort 后，CREATE 过程中创建的新文件需要移除。 这层和 catalog tuple 的 MVCC 回滚不同。 一个负责 heap tuple visibility。
一个负责文件系统副作用。
### 4.8. lock state
DDL 不是只靠 MVCC。 `heap_create_with_catalog()` 会对新 relation OID 拿 `AccessExclusiveLock`。 `DefineRelation()` 对继承父表或分区父表拿相应锁。
DROP 路径通过 `AcquireDeletionLock()` 给 relation 或 database object 加删除锁。 ALTER TABLE 常见路径会拿目标 relation 的强锁，并把子命令分阶段执行。 这些锁的作用是：
```text
阻止并发命令在对象结构改变时观察到不可接受的中间语义。
```
但锁不负责：
- catalog tuple 的事务回滚。
- backend-local cache 的主动刷新。
- dependency 图遍历。
- 物理文件 pending delete。
## 5. 主流程源码 walkthrough
### 5.1. CREATE TABLE 的入口
`CREATE TABLE` 的主链路从 utility 分派进入 `DefineRelation()`。 本节不展开 `ProcessUtility()` 的所有 case。 只把它作为从 parse tree 到 command module 的边界。
核心调用链是：
```text
ProcessUtility()
  -> standard_ProcessUtility()
  -> DefineRelation()
```
`DefineRelation()` 收到的是已经 parse/analyze 到 utility 层可用的 `CreateStmt`。 它首先处理对象名、namespace、tablespace、owner、reloptions、inheritance、partition 信息。 这一步还没有写 catalog。
它在建立一个“可以安全写 catalog 的计划”。 例如 `RangeVarGetAndCheckCreationNamespace()` 会解析创建 namespace、检查权限、锁住 namespace 防止并发 drop。 继承和分区父表会通过 `RangeVarGetRelid()` 拿父表锁。
这说明 DDL 的第一步是建立并发边界。 不是直接插入 `pg_class`。
### 5.2. descriptor 和 raw defaults
`DefineRelation()` 接着构造 tuple descriptor。 `MergeAttributes()` 处理继承列。 `BuildDescForRelation()` 把 `ColumnDef` 转为 `TupleDesc`。
默认值分两类：
```text
cooked default
  -> 已经是表达式树，可随 heap_create_with_catalog() 早期存储
raw default
  -> 还需要 transformExpr()，必须等 relation catalog row 可见后处理
```
这个差异很关键。 raw default 需要 parser/analyzer 的表达式转换。 而表达式转换可能要查新 relation 的 column、类型和 namespace。
所以它必须等 catalog 先写入并 CCI 后才能做。 这就是 DDL 中 CCI 的一个真实来源。 不是“写完 catalog 就习惯性 CCI”。
而是后续步骤需要通过正常 catalog lookup 看见前一步结果。
### 5.3. `heap_create_with_catalog()`
`DefineRelation()` 调用：
```text
heap_create_with_catalog()
```
这个函数是 relation cataloging 的核心。 它会打开 `pg_class`：
```text
table_open(RelationRelationId, RowExclusiveLock)
```
然后做 sanity check、重复名检查、同名 type 冲突处理、OID/relfilenumber 分配。 随后它拿新 relation 的 `AccessExclusiveLock`：
```text
LockRelationOid(relid, AccessExclusiveLock)
```
源码注释点明：其他 session 在提交前看不到这个新 relation。 但持锁仍然有价值。 它让后续调用者不必各自决定锁模式。
也让当前事务里围绕这个 relation 的操作在 lock manager 视角保持一致。 之后进入物理和 relcache 创建：
```text
heap_create()
```
这里会创建 mostly dummy 的 relcache entry，并按 relkind/persistence 创建需要的磁盘文件。 如果后续失败，磁盘文件 cleanup 不是 catalog MVCC 的职责。 它由 storage manager 的事务收尾负责。
### 5.4. 写 `pg_type`
普通 table、view、foreign table、partitioned table 等会有 relation rowtype。 `heap_create_with_catalog()` 会调用 `AddNewRelationType()` 创建 composite type。 然后创建 rowtype 的 array type。
这解释了为什么创建表不只是 `pg_class` 一行。 表的存在会被类型系统引用。 例如函数参数、表达式类型推断、composite value 都可能引用这个 rowtype。
所以 rowtype 必须有自己的 catalog row 和 dependency。
### 5.5. 写 `pg_class` 和 `pg_attribute`
随后是 relation 的核心 catalog tuple：
```text
AddNewRelationTuple()
AddNewAttributeTuples()
```
`AddNewRelationTuple()` 写 `pg_class`。 `AddNewAttributeTuples()` 给每个 user/system attribute 写 `pg_attribute`。 这些写入使用 catalog heap insert。
它们不是直接修改 relcache。 relcache 是这些 tuple 的派生缓存。 所以后续读 relation metadata 时，必须经过 command id 和 invalidation 边界，确保缓存不会用旧语义。
### 5.6. 记录 relation dependency
`heap_create_with_catalog()` 接着记录 relation 的生命周期边。 典型动作包括：
```text
recordDependencyOnOwner()
recordDependencyOnNewAcl()
recordDependencyOnCurrentExtension()
record_object_address_dependencies()
```
普通 relation 会依赖 namespace。 有 table access method 的 relation 会依赖 access method。 typed table 会依赖 type。
extension 创建中的对象会依赖当前 extension。 这些 dependency 的对象身份都是 `ObjectAddress`。 重点是不只“被谁引用”需要记录。
还要记录“删除谁时我应该被阻止、级联或自动删除”。
### 5.7. post create hook 与 constraints
catalog tuple 和 dependency 记录后，会调用 object post create hook。 然后处理 cooked constraints：
```text
StoreConstraints()
```
`StoreConstraints()` 可能调用 `CommandCounterIncrement()` 并 rebuild relcache。 源码注释要求：到这里 relation 必须已经是 valid and self-consistent。 因为 constraints/defaults 的存储可能需要通过 relcache 重新解释 relation。
这也是 DDL 设计中的一个不变量：
```text
每次 CCI 前，已经暴露给本 backend 的 catalog 状态必须自洽。
```
不能指望 CCI 后再补关键字段把半成品修好。
### 5.8. `DefineRelation()` 的第一次 CCI
`heap_create_with_catalog()` 返回 relation OID 后，`DefineRelation()` 立即：
```text
CommandCounterIncrement();
```
源码注释非常直白：
```text
make the newly-created relation tuple visible for opening
```
然后当前 backend 才能：
```text
relation_open(relationId, AccessExclusiveLock)
```
这一步不是为了让其他 backend 看见。 未提交事务写入的 catalog tuple 对其他事务仍不可见。 它是为了让当前事务内后续命令使用正常 catalog lookup 路径。
### 5.9. raw defaults 和 generated expressions
打开新 relation 后，`DefineRelation()` 处理 raw defaults：
```text
AddRelationNewConstraints(rel, rawDefaults, NIL, ...)
```
`AddRelationNewConstraints()` 会 transform default expression。 然后 `StoreAttrDefault()` 或 `StoreRelCheck()` 写入 `pg_attrdef` 或 `pg_constraint`。 表达式里的对象引用会通过 dependency 记录下来。
典型路径是：
```text
recordDependencyOnSingleRelExpr()
recordDependencyOnExpr()
```
例如：
```sql
CREATE TABLE t(a int DEFAULT nextval('s'::regclass));
```
这个 default expression 需要记录对 sequence `s` 的依赖。 否则 DROP sequence 时无法知道 table default 仍然引用它。
### 5.10. 第二次 CCI
处理 raw defaults 和 generated expressions 后，`DefineRelation()` 再次：
```text
CommandCounterIncrement();
```
源码注释说明：让 column generation expressions 对 partitioning 可见。 这一点体现了 DDL 内部也有阶段化可见性。 同一条 `CREATE TABLE` 不是一个不可分割的 C 函数黑盒。
它内部可能多次写 catalog，多次需要后续步骤通过 syscache/relcache 看见前一步结果。
### 5.11. 分区、继承和 index 的后续步骤
如果有 partition bound、partition key、LIKE INCLUDING INDEXES、constraints 或自动创建 index，`DefineRelation()` 还会进入更多步骤。 本节不逐行展开。 只抓住同一个模式：
```text
写 catalog tuple
  -> 记录 dependency
  -> 必要时 CCI
  -> 让下一阶段通过正常 lookup 看见
  -> 必要时 invalidation relcache
```
分区对象会额外记录 `DEPENDENCY_PARTITION_PRI` 和 `DEPENDENCY_PARTITION_SEC`。 父表 partition descriptor 改变时，需要 relcache invalidation。 这让其他 backend 在处理 sinval 后重新构造 partition metadata。
### 5.12. CREATE INDEX 对照
`CREATE INDEX` 的 command module 是 `indexcmds.c`。 入口是：
```text
DefineIndex()
```
它做权限、并发选项、access method、operator class、expression/predicate 解析。 然后调用 catalog 层：
```text
index_create()
```
`index_create()` 位于 `catalog/index.c`。 它会创建 index relation 的 `pg_class` / `pg_attribute` 状态。 也会写 `pg_index`。
如果 index 支持 constraint，还会走：
```text
index_constraint_create()
```
并记录 index、constraint、table、operator class、collation、expression、predicate 等依赖。 这条路径说明 command module 和 catalog helper 的分工：
```text
commands/*.c 负责 SQL 语义、权限、锁和选项。
catalog/*.c 负责把对象持久化为 catalog tuple 和 dependency。
```
这个分工不是绝对干净。 源码中有历史耦合。 但阅读时保持这个 mental model 能帮助定位责任。
### 5.13. ALTER TABLE 的多阶段特点
`ALTER TABLE` 不像 `CREATE TABLE` 那样只创建新对象。 它可能：
- 改 `pg_class` 的 reloptions、owner、tablespace、persistence。
- 改 `pg_attribute` 的 type、not null、default、attisdropped。
- 新增或删除 `pg_constraint`。
- 创建、复用或删除 index。
- 触发表 rewrite。
- 递归处理 inheritance children 或 partitions。
`tablecmds.c` 对 ALTER TABLE 使用多 pass 设计。 典型形态是：
```text
ATPrepCmd()
  -> ATRewriteCatalogs()
  -> ATRewriteTables()
```
准备阶段检查语义、收集 work queue、决定锁。 catalog rewrite 阶段修改系统表。 table rewrite 阶段在必要时重写用户表数据。
为什么需要多阶段？ 因为 catalog 变化和 heap 数据变化的事务边界不同。 例如 `ALTER TABLE ... ADD COLUMN DEFAULT ...` 可能只改 catalog，也可能需要重写或验证。
`ALTER TABLE ... ALTER COLUMN TYPE` 可能需要创建新 heap、转换 tuple、替换 relfilenumber。 但无论路径多复杂，catalog 更新仍然遵循本节主模型：
```text
持锁
  -> 写 catalog
  -> 记录/删除 dependency
  -> CCI 让本事务后续步骤看见
  -> invalidation 让缓存过期
  -> abort 时按事务回滚
```
### 5.14. ALTER 中的 explicit invalidation
很多 catalog tuple 更新会通过 `CacheInvalidateHeapTuple()` 自动引发 catcache/relcache 失效。 但不是所有 relcache 语义变化都对应某个被自动识别的 tuple 变化。 因此源码中会直接调用：
```text
CacheInvalidateRelcache(rel)
```
例如某些 index、constraint、partition、trigger、rewrite 或 publication 相关变化会影响 relation 派生状态。 此时必须告诉 relcache：
```text
即使你没看到一个普通 pg_class tuple key 的变化，也要重建这个 relation 的派生描述。
```
这类显式 invalidation 是读 DDL 源码时必须找的边界。 否则你会以为 catalog tuple update 已经覆盖所有缓存状态。
### 5.15. DROP TABLE 的入口
DROP 的入口不直接“删除 `pg_class` 一行”。 对普通 DROP statement，`dropcmds.c` 的：
```text
RemoveObjects()
```
或 relation 专门路径 `tablecmds.c` 的：
```text
RemoveRelations()
```
会把用户指定的名字解析成 `ObjectAddress`。 然后进入 dependency 删除：
```text
performDeletion()
```
这一步是 DROP 与 CREATE/ALTER 最大的不同。 CREATE/ALTER 主要是构造或修改对象及其边。 DROP 必须先用已有边回答：
```text
删除这个对象会牵连哪些对象？
RESTRICT 能不能允许？
CASCADE 要删除哪些？
哪些对象是 internal/auto，不能由用户单独控制？
```
### 5.16. `performDeletion()`
`performDeletion()` 的主流程是：
```text
table_open(DependRelationId, RowExclusiveLock)
AcquireDeletionLock(object, flags)
targetObjects = new_object_addresses()
findDependentObjects()
reportDependentObjects()
deleteObjectsInList()
```
它先打开 `pg_depend`。 然后给目标对象拿删除锁。 随后递归查找依赖对象。
`findDependentObjects()` 会沿 `pg_depend` 图扩展 target list。 `reportDependentObjects()` 根据 `DROP_RESTRICT` 或 `DROP_CASCADE` 决定报错或 NOTICE。 最后 `deleteObjectsInList()` 执行真正删除。
这就是为什么 DROP 错误通常能列出“某对象依赖它”。 错误信息不是临时扫描 SQL 文本得出的。 而是来自持久化 dependency 图。
### 5.17. object-specific 删除
`deleteObjectsInList()` 最终会调用 `doDeletion()`。 `doDeletion()` 根据 `ObjectAddress.classId` 分派。 典型分支包括：
```text
RelationRelationId    -> 删除 relation 或 index
ProcedureRelationId   -> RemoveFunctionById()
TypeRelationId        -> RemoveTypeById()
ConstraintRelationId  -> RemoveConstraintById()
AttrDefaultRelationId -> RemoveAttrDefaultById()
RewriteRelationId     -> RemoveRewriteRuleById()
TriggerRelationId     -> RemoveTriggerById()
```
也就是说 generic DROP 只负责“图和顺序”。 具体 catalog tuple 怎么删，仍然由对象所属模块负责。 这保持了一个边界：
```text
dependency.c 知道对象图。
commands/catalog 模块知道对象内部 catalog 结构。
```
### 5.18. 删除 catalog tuple
删除 relation 时会调用类似：
```text
DeleteRelationTuple()
DeleteAttributeTuples()
DeleteSystemAttributeTuples()
DeleteInheritsTuple()
```
以及各对象自己的 `Remove*ById()`。 这些函数会对对应 catalog 做 `CatalogTupleDelete()`。 删除 dependency 本身也是 catalog delete。
所以 DROP 同样是事务性 heap 修改。 未提交前，其他事务不会把这些 catalog 删除当成已提交事实。 当前事务内通过 command id 能看到自己的删除效果。
abort 后删除 tuple 版本被回滚。
### 5.19. DROP 与 physical file
DROP relation 还要处理物理存储。 catalog tuple 删除只是让 relation 不再可见。 物理文件不能在语句中间随意 unlink。
原因是：
- 当前事务可能 abort。
- 其他事务可能仍持有旧 snapshot 或 relcache 引用。
- storage cleanup 必须与 WAL、checkpoint、smgr 状态协调。
因此 relation drop 会把文件删除登记到事务结局路径。 提交后才真正删除旧文件。 abort 时保留文件。
这个设计让 catalog MVCC 和文件系统副作用在事务边界重新对齐。
### 5.20. invalidation 的生成
catalog tuple insert/update/delete 会调用 invalidation 相关逻辑。 典型入口包括：
```text
CacheInvalidateHeapTuple()
CacheInvalidateRelcache()
CacheInvalidateRelcacheByRelid()
CacheInvalidateCatalog()
```
`CacheInvalidateHeapTuple()` 根据被修改的 catalog relation 和 tuple，决定需要 invalid 的 catcache 或 relcache。 `CacheInvalidateRelcache()` 用于那些“没有被普通 catalog tuple change 自动覆盖”的 relation 派生状态。 这些消息先进入当前事务的 invalidation state。
不是立即广播给全体 backend。
### 5.21. CCI 时的本地 invalidation
`CommandCounterIncrement()` 在命令边界调用：
```text
CommandEndInvalidationMessages()
```
`CommandEndInvalidationMessages()` 做两件事：
```text
ProcessInvalidationMessages(CurrentCmdInvalidMsgs, LocalExecuteInvalidationMessage)
AppendInvalidationMessages(PriorCmdInvalidMsgs, CurrentCmdInvalidMsgs)
```
第一步让本 backend 立刻处理自己刚造成的缓存过期。 第二步把消息保存到事务 prior list，等 commit 再广播。 这个顺序回答了一个常见问题：
```text
为什么同一事务内 DDL 后再查询，通常能看到正确的新结构？
```
因为 CCI 不只推进 command id。 它也处理本地 cache invalidation。
### 5.22. commit 时的 shared invalidation
事务提交时，prior invalidation messages 会通过 shared invalidation 机制发给其他 backend。 其他 backend 不会同步执行 DDL。 它们只是在处理消息时丢弃本地过期缓存。
下一次需要这个对象时，再从已提交 catalog tuple 重建。 这保持了两个性质：
```text
DDL 提交不需要扫描并直接修改每个 backend 的 private cache。
其他 backend 也不会在变更未提交时看到未提交 catalog 语义。
```
代价是 cache invalidation 是异步消费的。 backend 在安全边界会处理 sinval。 它不是一个“中断当前 C 函数立即清空所有状态”的机制。
### 5.23. abort 时发生什么
如果 DDL 过程中 ERROR 或用户 ROLLBACK，几个层次分别收尾：
```text
catalog heap tuple
  -> 普通事务 abort 让插入/更新/删除不可见或回滚
command id
  -> 当前事务结束，后续事务重新开始 command id 语义
local memory
  -> transaction context reset
locks
  -> transaction lock release
pending invalidations
  -> AtEOXact_Inval(false) 本地处理 prior 消息，但不向其他 backend 广播未提交 DDL
physical files
  -> smgr pending delete 根据 create/drop 的事务结局执行或取消
```
这不是一个单一 cleanup 函数完成的。 PostgreSQL 的事务收尾按资源类型调用多个 `AtEOXact_*` 路径。 本节要形成的判断是：
```text
abort 回滚的是“事务性状态”和“登记过事务收尾的外部副作用”；
不是靠 dependency 或 invalidation 把 catalog 手动改回去。
```
## 6. 生命周期 / ownership / cleanup
### 6.1. 谁创建
DDL command module 创建 catalog 变化。 例如：
```text
DefineRelation()
  -> heap_create_with_catalog()
DefineIndex()
  -> index_create()
ProcedureCreate()
  -> pg_proc catalog insert
TypeCreate()
  -> pg_type catalog insert
```
这些函数的 owner 不是某个长期 C 对象。 长期 owner 是 catalog tuple 本身。 一旦 commit，catalog 成为 durable metadata。
### 6.2. 谁持有
运行时持有分几类：
| 状态 | 持有者 |
| --- | --- |
| catalog tuple version | heap storage 与事务系统。 |
| relation lock | lock manager，以事务或会话为 release 边界。 |
| relcache/syscache entry | 单个 backend 的 memory context。 |
| dependency edge | `pg_depend` catalog tuple。 |
| invalidation message | 当前事务的 invalidation state，提交后进入 shared invalidation queue。 |
| physical relation file | storage manager 和 pending delete 事务状态。 |
不要把这些 owner 混在一起。 MemoryContext 释放 relcache 内存，不会撤销 catalog tuple。 ResourceOwner/lock release 不会删除 dependency。
invalidation 让缓存过期，不会删除物理文件。
### 6.3. 谁释放
释放路径取决于状态类型：
```text
catalog tuple
  -> 后续 DROP/ALTER 产生新的 tuple version 或 delete version
backend-local cache
  -> invalidation 后丢弃，或 backend exit 时释放
lock
  -> transaction end / explicit release
dependency edge
  -> owning object 删除时从 pg_depend 删除
physical file
  -> smgr pending delete 在 commit/abort 边界处理
temporary object
  -> backend/session/transaction cleanup 路径清理
```
这就是为什么内核课程不能写“事务结束时释放”。 事务结束只是很多 cleanup 的同步点。 真正释放动作由各资源 owner 决定。
### 6.4. ERROR 期间的半成品
如果 `heap_create_with_catalog()` 已经创建物理文件、写了部分 catalog tuple，然后后续 catalog insert 失败，会发生什么？ catalog tuple insert 属于当前事务。 abort 会让它们不可见。
物理文件由 smgr pending delete 处理。 locks 会释放。 local relcache 中可能出现过半成品 entry。
transaction abort 和 invalidation cleanup 会避免后续命令继续把它当成有效 committed state。 核心点是：
```text
DDL 源码允许在函数内部出现阶段性 side effect；
但每个 side effect 必须要么是事务性 heap change，
要么登记到事务 cleanup，
要么只存在于会随 ERROR 清理的 backend-local memory。
```
### 6.5. long-lived object 如何失效
commit 后的 DDL 不会主动改写其他 backend 的 C 指针。 其他 backend 的 relcache entry 可能仍在内存中。 shared invalidation message 让它们标记过期。
下一次使用时重建。 有些 relcache entry 不能立刻释放，因为当前执行栈可能还持有 `Relation` 引用。 因此 invalidation 的结果经常是：
```text
mark invalid now
rebuild later at safe point
```
这和 lock 的语义不同。 lock 阻止不合法并发。 invalidation 管缓存语义过期。
## 7. 正确性机制层次
### 7.1. MVCC visibility
catalog 是普通 heap relation。 因此 catalog tuple 的插入、更新、删除都带事务 ID 和 command ID。 其他事务按 snapshot 判断能否看到。
当前事务通过 command id 和 CCI 判断能否看到前一命令的修改。 MVCC 保证：
```text
未提交 DDL 不会成为其他事务的已提交 catalog reality。
abort 后 catalog 修改不可见。
```
MVCC 不保证：
```text
并发 DDL 不冲突。
缓存自动刷新。
DROP cascade 图正确。
物理文件立即删除。
```
### 7.2. heavyweight locks
DDL 使用 heavyweight lock 建立对象级并发边界。 常见例子：
- 新 relation OID 上的 `AccessExclusiveLock`。
- 父表 inheritance/partition lock。
- DROP 的 `AcquireDeletionLock()`。
- ALTER TABLE 的目标 relation lock。
- catalog relation 自身的 `RowExclusiveLock`。
lock 保证逻辑互斥和死锁检测。 它不负责缓存失效。 一个 backend 可以在 lock release 后仍有旧 relcache entry。
它必须通过 sinval 处理来知道旧 entry 不可用了。
### 7.3. CCI ordering
CCI 是 DDL 内部阶段化的核心。 正确性要求：
```text
CCI 前 catalog 状态必须自洽到足以被后续 lookup 使用。
CCI 后本 backend 的 syscache/relcache 必须处理当前命令 invalidation。
```
如果把所有 catalog 写入都拖到函数末尾，再做一次 CCI，很多 DDL 无法用正常 lookup 处理表达式和约束。 如果每插一行 catalog 都 CCI，会扩大中间状态暴露面，也增加 invalidation 成本。 源码选择的是按语义阶段 CCI。
### 7.4. dependency graph
dependency graph 保证对象生命周期的引用关系。 它能回答：
```text
DROP schema 会删除哪些 table？
DROP type 是否被 table column 阻止？
DROP function 是否被 default expression 或 check constraint 引用？
DROP extension 应该带走哪些 member object？
```
它不能回答：
```text
当前是否有事务正在读这个 table？
某个 backend 是否还持有 relcache entry？
某个 SQL 文本未来是否会引用这个对象？
```
这些分别属于 locks/snapshots、invalidation 和 parser/analyzer 的范围。
### 7.5. invalidation ordering
invalidation 的 ordering 分两层：
```text
command boundary
  -> 本地处理当前命令 invalidation
transaction commit
  -> 对其他 backend 发布已提交 invalidation
```
这个顺序避免两个错误。 第一，本事务继续用旧 cache 解释自己刚改过的 catalog。 第二，其他事务在 DDL abort 后错误丢弃或重建未提交语义。
注意第二点不是说 abort 时完全不需要本地 cleanup。 abort 当前 backend 当然要清理自己的事务状态。 但未提交 DDL 不需要变成全局 committed sinval。
### 7.6. WAL 和 crash safety
catalog heap/index 修改会写 WAL。 relation storage 创建、删除、rewrite 也会进入 WAL/smgr 的 crash safety 规则。 本节不展开 WAL record 格式。
只强调边界：
```text
事务提交后，catalog change 和相关 storage change 必须能在 crash recovery 后重放为一致状态。
事务 abort 或未提交 crash 后，恢复不能留下可见 catalog 半成品。
```
这也是为什么 DDL 不能只改 shared memory。 catalog 是 durable truth。 cache 只是可丢弃派生状态。
### 7.7. snapshot 与 relcache 的边界
普通 query 的 snapshot 决定能看到哪些 user table tuple。 catalog lookup 也有 catalog snapshot 和 syscache 行为。 但 relcache 不是简单受一个 user snapshot 控制的 tuple。
它是 catalog tuple、index list、rules、triggers、partition descriptor 等派生结构。 因此 relation metadata 变化要靠 invalidation 维护。 不要用“我的事务 snapshot 还旧，所以 relcache 可以永远旧”解释 DDL。
PostgreSQL 在安全边界处理 sinval，确保已提交 DDL 最终进入本 backend 的 metadata reality。
## 8. 错误路径 / 异常路径 / fallback
### 8.1. duplicate object race
`heap_create_with_catalog()` 会先检查同名 relation 或 type 是否存在。 但源码注释明确：并发创建可能在后续 unique index insert 时才失败。 原因是另一个事务可能也在创建同名对象，但尚未提交，早期 lookup 看不到。
因此 DDL 不能只依赖前置检查。 catalog unique index 是最后防线。 失败后当前事务进入 ERROR/abort 路径，已经产生的 side effect 按事务和 pending cleanup 回收。
### 8.2. CCI 过多导致 command id 上限
`CommandCounterIncrement()` 有 command id 上限。 源码中如果超过 `2^32-2` commands，会报 `PROGRAM_LIMIT_EXCEEDED`。 通常 DDL 不会接近这个上限。
但这解释了为什么 CCI 不是免费无限资源。 源码会在 `currentCommandIdUsed` 为 false 时跳过递增。 DDL 作者也不应该把 CCI 当成普通函数调用随意散落。
### 8.3. parallel mode 限制
`CommandCounterIncrement()` 在 parallel mode 或 parallel worker 中不能开始新 command。 原因是 parallel workers 在并行操作开始时同步事务状态。 并行执行中再推进 command id，会破坏 leader/worker 对事务状态的共同理解。
这也是为什么 utility DDL 不属于普通 parallel worker 内部随意执行的逻辑。
### 8.4. DROP RESTRICT 报错
`performDeletion()` 先构造 target object list。 如果 `DROP_RESTRICT` 发现仍有 normal dependent object，`reportDependentObjects()` 报错。 此时报错发生在真正 delete 之前。
也就是说，DROP 的 restrict 检查不是“删到一半发现不行再回滚”。 它先用 dependency graph 判断。 然后才进入 `deleteObjectsInList()`。
当然，即使 delete 阶段后续失败，事务 abort 仍然会回滚 catalog 删除。
### 8.5. DROP CASCADE 的 NOTICE 与删除顺序
`DROP ... CASCADE` 会删除依赖对象。 删除顺序不是用户 SQL 文本顺序。 它由 dependency graph 和对象类型决定。
`findDependentObjects()` 会避免重复对象，并处理 internal/extension/partition 等特殊边。 这防止同一个 object 被重复删除。 也避免用户单独删除 internal object 破坏 owning object。
### 8.6. invalidation 丢失或滞后
shared invalidation queue 有自己的容量和恢复策略。 如果 backend 滞后太多，可能需要更粗粒度地 reset cache 状态。 本节不展开 sinval queue 的内部实现。
这里只要记住：
```text
invalidation 是缓存一致性的通知系统；
它允许用粗粒度失效兜底，而不是要求每条消息永不丢失且精确处理。
```
这也是为什么 invalidation 不能承载业务语义。 业务 truth 必须仍在 catalog 中。
### 8.7. extension dependency 边界
在 extension script 中创建对象时，`recordDependencyOnCurrentExtension()` 会把对象归入 extension。 后续 DROP extension 会带走这些 member object。 但这也带来限制：
用户通常不能把 extension member 当普通对象随意 DROP 或 ALTER 到破坏 extension 一致性的状态。 这里的 fallback 不是运行时修复。 而是 dependency 和 permissions 在 DDL 边界阻止不合法操作。
### 8.8. temporary object cleanup
临时表、临时 schema 和 ON COMMIT 行为会进入额外 cleanup 路径。 `heap_create_with_catalog()` 会登记 on commit action。 临时对象的物理和 catalog 生命周期与 session/transaction 相关。
本节不展开 temp namespace 复用。 但要记住：临时对象仍然通过 catalog 和 dependency 表达语义，只是可见性和 cleanup 边界更窄。
## 9. 成本、资源与跨模块传播
### 9.1. catalog write amplification
一个看似简单的 DDL 会写多张 catalog。 `CREATE TABLE` 不是一行 `pg_class`。 它可能写：
```text
pg_class
pg_attribute
pg_type
pg_depend
pg_attrdef
pg_constraint
pg_index
pg_inherits
```
列数越多，`pg_attribute` 写入越多。 约束、默认值、expression index、partition 层级越复杂，dependency 和 invalidation 越多。 这就是 DDL 在 schema migration 中可能放大 WAL、catalog bloat 和 lock 持有时间的原因。
### 9.2. syscache/relcache churn
DDL 会让 cache 失效。 高频 DDL 或大量 partition attach/detach 会造成 relcache rebuild。 成本随以下变量扩张：
- relation 数量。
- partition 数量。
- index 数量。
- constraint/trigger/rule 数量。
- backend 数量。
- 每个 backend 的 cached plan 和 relcache 使用情况。
其他 backend 不会在 commit 时被逐个同步修改内存。 但它们后续处理 invalidation 和重建 cache 的成本是真实存在的。
### 9.3. lock contention
DDL 往往拿强锁。 `AccessExclusiveLock` 会阻塞普通 DML 的很多路径。 `ALTER TABLE` 还可能递归锁 partitions 或 inheritance children。
DROP 会通过 `AcquireDeletionLock()` 对目标对象拿删除锁。 因此 DDL 慢不一定是 catalog insert 慢。 实际瓶颈常常是：
```text
等待旧事务释放 relation lock
等待长查询结束
递归锁大量 partitions
table rewrite 扫描/写入大量 tuple
commit 时 WAL flush
其他 backend 后续 cache rebuild
```
### 9.4. dependency graph traversal
DROP 的成本与 dependency graph 大小有关。 schema、extension、partition tree、foreign key、view/rule、default expression 都会增加边。 `CASCADE` 可能把一个用户对象扩展成大量 target objects。
`RESTRICT` 也需要查图才能报错。 所以不要把 DROP 成本简化为删除单行 `pg_class`。
### 9.5. WAL 与 checkpoint 压力
catalog 修改和 storage 创建/删除都会写 WAL。 大量 DDL 会增加 WAL 量。 如果伴随 table rewrite、index build 或 partition maintenance，WAL 压力会扩散到：
- foreground commit latency。
- replication lag。
- checkpoint IO。
- archive backlog。
- crash recovery 时间。
观测时要区分：
```text
DDL 等锁慢
DDL 写 catalog 慢
DDL 重写数据慢
DDL commit/WAL flush 慢
DDL 后其他 backend cache rebuild 慢
```
### 9.6. 跨模块连接
本节至少连接这些模块：
| 模块 | 边界 |
| --- | --- |
| parser/analyzer | utility parse node 提供 DDL 结构；default/check expression 需要后续 transform。 |
| lock manager | relation/object/catalog locks 保护并发 DDL/DML 边界。 |
| heap/catalog access | catalog tuple 是事务性 heap 修改。 |
| dependency | `pg_depend` 持久化对象生命周期图。 |
| cache invalidation | syscache/relcache/plan cache 通过 sinval 处理过期。 |
| transaction manager | command id、commit/abort、resource cleanup 统一收尾。 |
| storage manager | relation physical file 创建/删除通过 pending delete 对齐事务结局。 |
| WAL/recovery | catalog 和 storage side effect 在 crash 后恢复一致。 |
## 10. 观测与诊断入口
### 10.1. 可直接观测的 runtime truth
本节锚定的 runtime 现象是：
```text
同一事务内，DDL 后的后续命令能看到新 catalog 状态；
事务 abort 后，外部看不到这个 DDL；
DDL commit 后，其他 backend 通过 invalidation 放弃旧缓存并重建。
```
最小实验：
```sql
BEGIN;
CREATE TABLE ddl_demo(a int DEFAULT 42);
SELECT relname FROM pg_class WHERE relname = 'ddl_demo';
INSERT INTO ddl_demo DEFAULT VALUES RETURNING *;
ROLLBACK;
SELECT relname FROM pg_class WHERE relname = 'ddl_demo';
```
第一条 `SELECT` 和 `INSERT` 依赖 CCI 后的当前事务可见性。 最后一条 `SELECT` 证明 catalog insert 被 abort 回滚。
### 10.2. 看 catalog tuple
可以观察：
```sql
SELECT oid, relname, relnamespace::regnamespace, relkind, reltype
FROM pg_class
WHERE relname = 'ddl_demo';
SELECT attnum, attname, atttypid::regtype, attnotnull
FROM pg_attribute
WHERE attrelid = 'ddl_demo'::regclass
ORDER BY attnum;
SELECT classid::regclass, objid, objsubid,
       refclassid::regclass, refobjid, refobjsubid, deptype
FROM pg_depend
WHERE objid = 'ddl_demo'::regclass;
```
注意这些查询本身也经过 syscache/catalog lookup。 它们看到的是 SQL 层可见的 catalog state。 不是所有 backend-local cache 状态。
### 10.3. 看锁
DDL 等待时看：
```sql
SELECT pid, locktype, relation::regclass, mode, granted
FROM pg_locks
WHERE relation IS NOT NULL
ORDER BY granted, pid;
```
`pg_locks` 能看到 heavyweight lock。 它看不到：
- dependency graph traversal 的完整中间 target list。
- invalidation message 队列内容。
- relcache entry 是否已经 rebuild。
因此 `pg_locks` 只能解释阻塞，不等于 DDL 全部成本。
### 10.4. 看 WAL 和 IO
大量 DDL 或 table rewrite 可以结合：
```sql
SELECT * FROM pg_stat_wal;
SELECT * FROM pg_stat_io;
```
粒度是 instance/database/backend 类型的统计，不是单条 DDL 的完整因果。 单条 SQL 可用 `EXPLAIN (ANALYZE, WAL)` 的场景主要是 DML/SELECT。 DDL 的 WAL 通常要通过前后差值、日志、profiling 或源码断点推断。
### 10.5. 看 invalidation
普通 SQL 层没有直接显示“这一条 relcache invalidation message”的视图。 可用方法是源码断点或临时日志。 推荐断点：
```text
CommandCounterIncrement
CommandEndInvalidationMessages
CacheInvalidateHeapTuple
CacheInvalidateRelcache
LocalExecuteInvalidationMessage
```
观察目标：
```text
哪一步产生 invalidation？
哪一次 CCI 本地处理？
commit 前是否只在事务 state 中排队？
另一个 backend 什么时候处理 sinval？
```
### 10.6. 看 dependency 删除
DROP 诊断推荐断点：
```text
RemoveObjects
RemoveRelations
performDeletion
findDependentObjects
reportDependentObjects
deleteObjectsInList
doDeletion
```
配合 SQL：
```sql
CREATE TABLE dep_parent(id int primary key);
CREATE TABLE dep_child(id int references dep_parent(id));
DROP TABLE dep_parent;
DROP TABLE dep_parent CASCADE;
```
第一条 DROP 应该在 restrict 路径报错。 第二条 DROP 会沿 dependency 图扩展删除对象。
### 10.7. 能看到、只能推断、几乎不可见
| 类型 | 例子 |
| --- | --- |
| 能直接看到 | `pg_class`、`pg_attribute`、`pg_depend`、`pg_locks`、错误信息、server log。 |
| 需要推断 | 某次 CCI 是否因为后续 lookup 需要、其他 backend 何时重建 relcache、DDL 后 plan cache 失效成本。 |
| 几乎不可见 | 单个 invalidation message 的用户态视图、relcache entry 内部字段生命周期、smgr pending delete list。 |
真实诊断经常要组合 SQL、日志、gdb、perf 和源码阅读。 不要把 `pg_stat_*` 视为完整 runtime reality。
## 11. 常见误区
### 11.1. 把 DDL 当成非事务性操作
PostgreSQL 中多数 DDL 是事务性的。 catalog tuple 通过普通事务回滚。 但“多数 DDL 事务性”不等于所有外部副作用都由 heap MVCC 自动处理。
物理文件、cache、locks、pending invalidation 都有自己的 cleanup 路径。
### 11.2. 认为 `pg_depend` 是注释
`pg_depend` 是 DROP 语义的输入。 如果创建对象时漏记 dependency，后果通常不是立即崩溃。 而是未来 DROP/ALTER 时允许破坏引用关系，或 cascade 不完整。
这类 bug 常常延迟暴露。
### 11.3. 把 invalidation 当 lock
invalidation 不阻止并发修改。 它只通知缓存过期。 阻止并发 DDL/DML 的是 locks 和 snapshot/visibility 边界。
如果一个设计需要互斥，不能用 invalidation 替代 lock。
### 11.4. 以为 CCI 是 commit
CCI 只推进同一事务内的 command visibility。 它不会让其他事务看到未提交 catalog tuple。 它也不会把 invalidation 广播为 committed change。
所以：
```sql
BEGIN;
CREATE TABLE t(a int);
-- 当前事务可见
-- 其他事务仍不可见
```
这个现象不是 isolation bug。
### 11.5. 只看 `pg_class` 判断对象完整性
一个 relation 的语义不在 `pg_class` 单行里。 必须结合 `pg_attribute`、`pg_type`、`pg_constraint`、`pg_index`、`pg_depend`、`pg_inherits`、relcache 派生状态和 locks。 看到 `pg_class` 行存在，不等于所有后续语义都已经可以安全使用。
要看 CCI 阶段和 catalog 自洽边界。
### 11.6. 把 abort cleanup 想成手写反向 DDL
abort 不会调用一套“反向 CREATE TABLE SQL”。 它让事务性 heap change 回滚，让 pending storage cleanup 执行相反结局，让 locks 和 memory context 释放，让 invalidation state 丢弃或本地清理。 这是资源 owner 分层收尾。
不是语义层拼接反向命令。
## 12. 课堂实验
### 实验 1：观察同一事务内 DDL 可见性
目标：把 CCI 与事务 commit 区分开。 步骤：
```sql
BEGIN;
CREATE TABLE cci_demo(a int DEFAULT 7);
SELECT 'cci_demo'::regclass;
INSERT INTO cci_demo DEFAULT VALUES RETURNING a;
ROLLBACK;
SELECT to_regclass('cci_demo');
```
预期：
- `CREATE TABLE` 后，同事务能把名称解析成 regclass。
- `INSERT` 能使用 default。
- `ROLLBACK` 后 `to_regclass` 返回 null。
源码跟读：
```text
DefineRelation()
  -> heap_create_with_catalog()
  -> CommandCounterIncrement()
  -> relation_open()
  -> AddRelationNewConstraints()
  -> CommandCounterIncrement()
```
问题：
```text
第一次 CCI 解决什么 lookup？
第二次 CCI 让哪些状态对后续步骤可见？
```
### 实验 2：dependency 如何阻止 DROP
目标：把 dependency 与 lock 区分开。 步骤：
```sql
CREATE TABLE dep_a(id int primary key);
CREATE TABLE dep_b(id int references dep_a(id));
DROP TABLE dep_a;
DROP TABLE dep_a CASCADE;
```
第一条 DROP 应报依赖错误。 第二条 DROP 会 cascade。 观察：
```sql
SELECT classid::regclass, objid, objsubid,
       refclassid::regclass, refobjid, refobjsubid, deptype
FROM pg_depend
WHERE refobjid = 'dep_a'::regclass
   OR objid = 'dep_a'::regclass;
```
源码断点：
```text
performDeletion
findDependentObjects
reportDependentObjects
```
问题：
```text
DROP RESTRICT 是在哪个阶段报错？
报错前有没有真正 delete catalog tuple？
```
### 实验 3：观察 relcache invalidation 触发点
目标：从 DDL 后查询行为回到 invalidation。 步骤： 在一个 backend 执行：
```sql
CREATE TABLE inval_demo(a int);
SELECT * FROM inval_demo;
```
另一个 backend 执行：
```sql
ALTER TABLE inval_demo ADD COLUMN b int;
```
回到第一个 backend：
```sql
SELECT * FROM inval_demo;
```
源码断点：
```text
CacheInvalidateHeapTuple
CacheInvalidateRelcache
CommandEndInvalidationMessages
LocalExecuteInvalidationMessage
```
观察：
- `ALTER TABLE` 修改了哪些 catalog。
- 哪些 invalidation 在 CCI 本地处理。
- 第一个 backend 在什么时候放弃旧 tuple descriptor。
注意： SQL 层看不到单条 invalidation message。 需要断点或临时日志。
### 实验 4：abort 后物理和 catalog 状态
目标：区分 catalog rollback 与 storage cleanup。 步骤：
```sql
BEGIN;
CREATE TABLE abort_demo(a int);
SELECT pg_relation_filepath('abort_demo');
ROLLBACK;
SELECT to_regclass('abort_demo');
```
源码跟读：
```text
heap_create_with_catalog()
  -> heap_create()
  -> catalog tuple inserts
  -> abort transaction cleanup
```
讨论：
```text
catalog tuple 为什么不可见？
物理文件是谁负责清理？
如果 crash 发生在中间，为什么 recovery 后不能留下可见半成品？
```
### 实验 5：ALTER TABLE 多阶段
目标：理解 ALTER TABLE 不是单点 catalog update。 步骤：
```sql
CREATE TABLE at_demo(a int);
ALTER TABLE at_demo ADD COLUMN b int DEFAULT 10;
ALTER TABLE at_demo ADD CONSTRAINT at_demo_check CHECK (b > 0);
```
源码跟读：
```text
ATPrepCmd
ATRewriteCatalogs
AddRelationNewConstraints
StoreAttrDefault
StoreRelCheck
CommandCounterIncrement
CacheInvalidateRelcache
```
问题：
```text
哪些步骤只改 catalog？
哪些步骤可能需要验证或重写数据？
哪些 catalog change 会影响 relcache tuple descriptor？
```
## 13. 讨论题
1. 为什么 `CREATE TABLE` 不能等函数最后一次性让所有 catalog tuple 对当前事务可见？
2. `CommandCounterIncrement()` 为什么不是 commit？它推进了哪些本地状态？
3. 如果创建 default expression 时漏掉 `recordDependencyOnExpr()`，最可能在哪类后续操作中暴露问题？
4. 为什么 DROP 需要先构造 dependency target list，再真正 delete catalog tuple？
5. `CacheInvalidateRelcache()` 为什么不能替代 relation lock？
6. 其他 backend 为什么不能在 DDL 修改瞬间被直接改写 relcache 内存？
7. 一个 DDL abort 后，哪些状态靠 MVCC 回滚，哪些状态靠 pending cleanup，哪些状态只是 backend-local memory 被释放？
8. 大量 partition DDL 的成本为什么会扩散到 relcache rebuild、sinval、WAL 和 lock wait，而不只是 catalog insert？
## 14. 本节小结
DDL 的长期 truth 是 catalog tuple。 `CREATE/ALTER/DROP` 并不是绕过事务系统去修改全局元数据。 它们把系统表当作事务性 heap relation 更新。
`CREATE TABLE` 的主链路是：
```text
DefineRelation()
  -> heap_create_with_catalog()
  -> pg_type / pg_class / pg_attribute / pg_depend
  -> CommandCounterIncrement()
  -> relation_open()
  -> AddRelationNewConstraints()
  -> CommandCounterIncrement()
```
DROP 的主链路是：
```text
RemoveObjects() / RemoveRelations()
  -> ObjectAddress
  -> performDeletion()
  -> findDependentObjects()
  -> reportDependentObjects()
  -> deleteObjectsInList()
  -> object-specific Remove*()
```
本节核心状态包括：
- catalog tuple version。
- `ObjectAddress`。
- `pg_depend` edge。
- command id。
- backend-local syscache/relcache。
- invalidation message。
- relation lock。
- smgr pending delete。
正确性来自分层组合：
```text
MVCC 负责事务可见性。
CCI 负责同事务命令间可见性。
lock 负责并发互斥。
dependency 负责对象生命周期图。
invalidation 负责缓存语义过期。
WAL/smgr 负责 crash safety 和物理文件收尾。
```
abort 时，不会执行一条反向 DDL。 事务系统回滚 catalog tuple。 storage manager 处理登记过的物理文件副作用。
lock manager 释放对象锁。 invalidation state 丢弃未提交的全局消息。 memory context 和 resource owner 清理 backend-local 状态。
可观测入口包括 `pg_class`、`pg_attribute`、`pg_depend`、`pg_locks`、server log、gdb 断点和 WAL/IO 统计。 不可直接观测的是单个 relcache entry 的完整生命周期、单条 sinval message 的 SQL 视图、以及 smgr pending delete list。 可迁移的系统规律是：
```text
当一个系统把 metadata 本身也做成事务性数据时，
必须额外提供“同事务阶段可见性”和“缓存语义失效”两层机制；
MVCC 解决提交/回滚，不自动解决本地缓存和对象生命周期图。
```
判断具体 DDL 问题时，先问四个问题：
```text
catalog truth 是否已经按 MVCC 可见？
当前 backend 是否经过必要 CCI 和本地 invalidation？
其他 backend 是否只是在等待 lock，还是还没处理 sinval？
dependency 图是否完整表达了对象生命周期？
```
这些判断仍然依赖版本、DDL 类型、schema 规模、partition 数量、backend 数量、WAL/IO 能力和并发 workload。 不要把单个源码函数或单个统计指标解释成完整因果。
