# PostgreSQL dependency recording / DROP CASCADE
## 课程定位
前置知识：已经理解 `MemoryContext`、`ResourceOwner`、catcache/syscache、relcache、shared invalidation、catalog snapshot 和 heavyweight lock 的基本边界。
本节唯一主问题：
```text
dependency 记录如何支撑 DROP RESTRICT/CASCADE、extension ownership 和对象生命周期？
```
核心矛盾：
```text
DDL 想把对象当成独立实体创建和删除；
但真实对象之间存在表达式引用、内部实现、extension membership、
分区成员、constraint/index、sequence ownership 和 shared owner 等生命周期耦合。
```
一句话运行模型：
```text
对象创建时把生命周期边写入 pg_depend / pg_shdepend；
DROP 时从目标对象反向遍历 pg_depend 构造删除闭包；
report 阶段用 DropBehavior 和 DependencyType 判断 RESTRICT 失败还是 CASCADE 通过；
delete 阶段按闭包顺序调用对象类型自己的删除函数并清理 dependency 边。
```
学完后应能判断：
- `pg_depend` 的方向。
- `DependencyType` 和 `DropBehavior` 的边界。
- 为什么 `DROP RESTRICT` 也可能隐式删除 auto/internal 对象。
- 为什么 `DROP CASCADE` 不是忽略依赖，而是依赖依赖图。
- 为什么 extension member 不能脱离 extension 单独生命周期。
- 为什么删除对象不能只删 `pg_depend` row。
- 如何用 SQL、`pg_locks`、server log 和 gdb 定位一个 cascade 问题。
本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e`。
## 1. 本节在总主线中的位置
第 56 到 60 节已经讲过 catalog metadata 如何被读取、缓存、失效和重新读取。
那些课程回答的是：
```text
一个 backend 如何用 syscache/relcache 低成本读取元数据；
另一个 backend 做 DDL 后，旧 cache 如何失效。
```
本节回答另一个问题：
```text
当一个对象要被删除时，系统如何知道哪些对象必须一起删除，哪些对象必须阻止删除？
```
这个问题不能只靠 syscache 或 relcache。
syscache 返回 catalog tuple。
relcache 返回 `RelationData`。
dependency infrastructure 维护的是对象生命周期图。
本节只追一条主线：
```text
CREATE object
  -> ObjectAddress
  -> recordDependencyOn*
  -> pg_depend / pg_shdepend
  -> DROP target
  -> performDeletion
  -> findDependentObjects
  -> reportDependentObjects
  -> deleteOneObject
  -> catalog invalidation
```
本节不展开所有 DDL 语法。
本节也不展开每种对象的完整 drop 实现。
对象类型自己的删除函数只作为 `doDeletion()` 的下游边界出现。
一个可复现现象：
```sql
CREATE TABLE t(a int);
CREATE VIEW v AS SELECT a FROM t;
DROP TABLE t RESTRICT;
```
会失败。
原因不是 relcache stale。
原因是 view/rewrite rule 对 table/column 有 normal dependency。
改成：
```sql
DROP TABLE t CASCADE;
```
系统会先删除 dependent objects，再删除 table。
这就是本节的主线边界：
```text
dependency 不是 metadata cache；
dependency 是对象生命周期规则。
```
## 2. 核心矛盾与运行模型
PostgreSQL 的模型是分层的。
| 层次 | 状态 | 作用 |
| --- | --- | --- |
| object identity | `ObjectAddress` | 统一表示 relation、function、type、column 等对象。 |
| durable edge | `pg_depend` | database-local 对象生命周期边。 |
| shared edge | `pg_shdepend` | role、ACL、tablespace 等 shared dependency。 |
| DROP closure | `ObjectAddresses` | 本次 DROP 要删除的对象列表。 |
| recursion guard | `ObjectAddressStack` | 防止递归环。 |
| object lock | heavyweight lock | 防止并发 DDL 改写对象生死。 |
| object deletion | `doDeletion()` | 按对象类型执行实际删除。 |
这个模型有一个重要结论：
```text
RESTRICT 和 CASCADE 不需要两套图搜索。
```
二者都先用 `findDependentObjects()` 找出删除闭包。
差异主要在 `reportDependentObjects()`：
```text
DROP_RESTRICT 遇到 normal dependent 报错；
DROP_CASCADE 允许 normal dependent 被报告并删除。
```
但 `CASCADE` 不是万能通行证。
`DEPENDENCY_INTERNAL`、`DEPENDENCY_EXTENSION` 和 partition dependency 有 ownership 语义。
直接删除 member object 时，系统会把它看成 owner 的一部分。
这类规则在 traversal 阶段就生效，不是简单的 report 文案。
## 3. 核心源码文件与阅读顺序
优先按生命周期阅读。
| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/catalog/objectaddress.h` | `ObjectAddress` 的三字段对象身份。 |
| 2 | `src/include/catalog/pg_depend.h` | `pg_depend` schema 和两个方向索引。 |
| 3 | `src/include/catalog/dependency.h` | `DependencyType`、`performDeletion()` flags 和 public API。 |
| 4 | `src/backend/catalog/pg_depend.c` | `recordDependencyOn()`、`recordMultipleDependencies()`、extension lookup、dependency row 修改。 |
| 5 | `src/backend/catalog/dependency.c` | `performDeletion()`、`findDependentObjects()`、`reportDependentObjects()`、`deleteOneObject()`。 |
| 6 | `src/backend/catalog/objectaddress.c` | SQL object spec 到 `ObjectAddress`，以及 object description。 |
| 7 | `src/backend/commands/extension.c` | extension row 创建、member 添加/移除、`RemoveExtensionById()`。 |
| 8 | `src/backend/commands/tablecmds.c` | column、constraint、partition 等 table DDL dependency 调用点。 |
| 9 | `src/backend/commands/indexcmds.c` / `src/backend/catalog/index.c` | index、constraint、table 之间的 dependency 和 drop path。 |
| 10 | `src/backend/commands/functioncmds.c` | function 对 language、type、support object 的 dependency。 |
| 11 | `src/backend/catalog/pg_shdepend.c` | owner、ACL、tablespace 等 shared dependency。 |
最小阅读链：
```text
pg_depend.h
  -> dependency.h 的 DependencyType
  -> pg_depend.c 的 recordMultipleDependencies()
  -> dependency.c 的 performDeletion()
  -> dependency.c 的 findDependentObjects()
  -> dependency.c 的 reportDependentObjects()
  -> dependency.c 的 deleteOneObject()
  -> extension.c 的 InsertExtensionTuple / RemoveExtensionById
```
如果问题是“为什么不能 drop 某对象”，先读：
```text
findDependentObjects()
  first scan: DependDependerIndexId
  deptype: INTERNAL / EXTENSION / PARTITION
```
如果问题是“为什么 CASCADE 没带走某对象”，先读：
```text
创建路径有没有 recordDependencyOn*
pg_depend 是否有 incoming edge
findDependentObjects() second scan: DependReferenceIndexId
```
如果问题是“extension member 为什么被绑定”，先读：
```text
recordDependencyOnCurrentExtension()
getExtensionOfObject()
ExecAlterExtensionContentsRecurse()
RemoveExtensionById()
```
## 4. 关键结构与状态
### 4.1 `ObjectAddress`
`ObjectAddress` 定义在 `objectaddress.h`。
它有三个字段：
```text
classId
objectId
objectSubId
```
`classId` 是保存对象的 catalog relation OID。
`objectId` 是该 catalog 中的对象 OID。
`objectSubId` 表示 sub-object，最常见是 column attnum。
`objectSubId = 0` 表示 whole object。
因此：
```text
(pg_class, relid, 0)      是 table。
(pg_class, relid, attnum) 是 column。
```
dependency traversal 必须保留 sub-object。
因为 view 可能只依赖某一列。
删除列和删除整张表不是同一个动作。
当 drop whole object 时，`findDependentObjects()` 会考虑它的 sub-object dependencies。
源码体现为：
```text
objectSubId == 0 时，扫描 pg_depend 少一个 objsubid/refobjsubid key。
```
### 4.2 `pg_depend`
`pg_depend` 的一行表示：
```text
dependent object -> referenced object
```
字段分两端。
| 端点 | 字段 | 含义 |
| --- | --- | --- |
| dependent | `classid` / `objid` / `objsubid` | 生命周期受别人影响的对象。 |
| referenced | `refclassid` / `refobjid` / `refobjsubid` | 被依赖的对象。 |
| type | `deptype` | 这条边的 drop / ownership 语义。 |
两个索引服务两个问题。
`DependDependerIndexId` 回答：
```text
当前对象依赖谁？
```
`DependReferenceIndexId` 回答：
```text
谁依赖当前对象？
```
`findDependentObjects()` 两个方向都用。
第一轮 outgoing scan 判断当前对象是否是 internal object 或 extension member。
第二轮 incoming scan 找所有依赖当前对象的 objects。
诊断 `pg_depend` 时最常见错误是把方向读反。
### 4.3 `DependencyType`
`DependencyType` 定义在 `dependency.h`。
本地源码包含：
```text
DEPENDENCY_NORMAL         'n'
DEPENDENCY_AUTO           'a'
DEPENDENCY_INTERNAL       'i'
DEPENDENCY_PARTITION_PRI  'P'
DEPENDENCY_PARTITION_SEC  'S'
DEPENDENCY_EXTENSION      'e'
DEPENDENCY_AUTO_EXTENSION 'x'
```
本节只记行为分组。
`NORMAL`：
```text
RESTRICT 下阻止删除；
CASCADE 下允许 dependent 一起删除。
```
`AUTO` / `AUTO_EXTENSION`：
```text
referenced object 被删除时，dependent 自动删除；
通常不要求用户显式 CASCADE。
```
`INTERNAL`：
```text
dependent 是 referenced 的内部实现；
不能脱离 owner 单独删除。
```
`EXTENSION`：
```text
dependent 是 extension member；
行为接近 internal ownership，但有 extension script 例外。
```
`PARTITION_PRI` / `PARTITION_SEC`：
```text
表达 partition hierarchy 相关 ownership；
删除必须从正确 partition owner 路径进入。
```
`DropBehavior` 是用户请求。
`DependencyType` 是 catalog edge 语义。
最终行为是二者和 traversal direction 的组合。
### 4.4 `ObjectAddresses` 与 `ObjectAddressExtra`
`ObjectAddresses` 是一次 DROP 的 backend-local 删除闭包。
它不是共享状态。
它不是持久 catalog。
它保存：
```text
refs[]
extras[]
numrefs
maxrefs
```
`refs[]` 是要删除的对象。
`extras[]` 记录为什么这个对象进入闭包。
典型 flag 包括：
```text
DEPFLAG_ORIGINAL
DEPFLAG_NORMAL
DEPFLAG_AUTO
DEPFLAG_INTERNAL
DEPFLAG_EXTENSION
DEPFLAG_PARTITION
DEPFLAG_REVERSE
```
`findDependentObjects()` 保证：
```text
dependent objects 先加入列表；
referenced / owner object 后加入列表。
```
因此 `deleteObjectsInList()` 可以按 list 顺序删除。
`reportDependentObjects()` 常反向遍历，是为了让错误信息更接近用户看到的依赖方向。
### 4.5 `ObjectAddressStack`
递归 traversal 需要防环。
`ObjectAddressStack` 表示当前递归路径。
它和 `targetObjects` 不同。
`targetObjects` 表示已经完成处理并计划删除的对象。
`ObjectAddressStack` 表示正在访问的对象链。
源码中先检查 stack，再检查 targetObjects。
这避免 dependency loop 导致无限递归。
但普通 DFS 还不够。
如果环里有 internal ownership，必须把 traversal 改写到 owner。
所以 dependency traversal 是带 ownership 语义的 graph walk。
### 4.6 `pg_shdepend`
`pg_depend` 主要处理 database-local objects。
role、tablespace、ACL 这类 shared state 用 `pg_shdepend`。
常见 shared dependency：
```text
SHARED_DEPENDENCY_OWNER
SHARED_DEPENDENCY_ACL
SHARED_DEPENDENCY_INITACL
SHARED_DEPENDENCY_POLICY
SHARED_DEPENDENCY_TABLESPACE
```
普通对象 drop 时，`deleteOneObject()` 会调用：
```text
deleteSharedDependencyRecordsFor()
```
这会清理 role owner、ACL、initial privilege、tablespace 等 shared dependency。
extension membership 不在 `pg_shdepend`。
它是普通 `pg_depend` row：
```text
member object --DEPENDENCY_EXTENSION--> pg_extension row
```
## 5. 主流程源码 walkthrough
### 5.1 记录 dependency：`recordMultipleDependencies()`
最普通入口：
```text
recordDependencyOn(depender, referenced, behavior)
  -> recordMultipleDependencies(depender, referenced, 1, behavior)
```
批量入口：
```text
recordMultipleDependencies()
```
表达式入口：
```text
recordDependencyOnExpr()
recordDependencyOnSingleRelExpr()
collectDependenciesOfExpr()
```
`recordMultipleDependencies()` 的关键步骤：
```text
if nreferenced <= 0: return
if bootstrap mode: return
open pg_depend with RowExclusiveLock
for each referenced object:
    skip pinned referenced object
    dependencyLockAndCheckObject(referenced)
    fill pg_depend tuple
    multi-insert with catalog indexes
close pg_depend
```
两个细节很重要。
第一，pinned referenced object 不记录 dependency。
源码注释说这样节省大量 `pg_depend` 空间。
因此：
```text
没有 pg_depend row 不等于没有生命周期约束。
```
第二，记录 dependency 前会锁住并检查 referenced object。
`dependencyLockAndCheckObject()` 是 backstop。
如果 caller 没有提前持有防 drop 的锁，它会获取 `AccessShareLock` 级别的对象锁。
然后用 syscache 或 `SnapshotSelf` catalog scan 检查对象还存在。
这避免永久写入指向已被并发删除对象的 bogus dependency。
### 5.2 extension 创建和 member 记录
`InsertExtensionTuple()` 插入 `pg_extension` row。
它还记录 extension 自己的依赖：
```text
extension --OWNER(shared)--> role
extension --NORMAL--> schema
extension --NORMAL--> required extensions
```
随后执行 extension script 时，`extension.c` 设置：
```text
creating_extension = true
CurrentExtensionObject = extensionOid
```
脚本中创建的 user-definable object 调用：
```text
recordDependencyOnCurrentExtension(object, isReplace)
```
如果不在 extension script 中，这个函数不记录 member edge。
如果正在创建 extension，它记录：
```text
object --DEPENDENCY_EXTENSION--> CurrentExtensionObject
```
`isReplace = true` 时还有重要检查。
如果对象已经属于当前 extension，允许。
如果对象属于其它 extension，报错。
如果对象是 free-standing object，也报错。
原因是 extension 不能用 `CREATE OR REPLACE` 静默吸收一个自己不拥有的对象。
这既是生命周期边界，也是安全边界。
`ALTER EXTENSION ADD/DROP` 走：
```text
ExecAlterExtensionContentsRecurse()
```
ADD 时插入 `DEPENDENCY_EXTENSION`。
DROP 时用 `deleteDependencyRecordsForClass(..., ExtensionRelationId, DEPENDENCY_EXTENSION)` 删除 membership edge。
同时处理 extension initial privileges 和 `extconfig`。
### 5.3 `performDeletion()`：DROP 总入口
多数参与 dependency 的 DROP 会进入：
```text
performDeletion(object, behavior, flags)
```
主链路：
```text
performDeletion()
  -> table_open(DependRelationId, RowExclusiveLock)
  -> AcquireDeletionLock(object, 0)
  -> new_object_addresses()
  -> findDependentObjects(object, DEPFLAG_ORIGINAL, ...)
  -> reportDependentObjects(targetObjects, behavior, flags, object)
  -> deleteObjectsInList(targetObjects, &depRel, flags)
  -> free_object_addresses()
  -> table_close(depRel)
```
这里先构造闭包，再决定是否报错，再实际删除。
所以 `DROP ... RESTRICT` 因 normal dependent 失败时，通常还没有开始删除对象。
`performMultipleDeletions()` 处理多个原始对象。
它不是简单循环调用 `performDeletion()`。
它把所有原始对象作为 `pendingObjects` 传给 traversal。
这样一个对象是另一个对象的 internal member 时，不会在批量删除中提前报错。
### 5.4 deletion lock：`AcquireDeletionLock()`
`performDeletion()` 和 traversal 会获取 deletion lock。
relation object 通常：
```text
LockRelationOid(objectId, AccessExclusiveLock)
```
`DROP INDEX CONCURRENTLY` 例外，开始用更弱的 `ShareUpdateExclusiveLock`。
非 relation database-local object：
```text
LockDatabaseObject(classId, objectId, 0, AccessExclusiveLock)
```
shared object：
```text
LockSharedObject(...)
```
这个 lock 不保护 C 指针。
它保护对象级 DDL 生死顺序。
### 5.5 `findDependentObjects()` 第一轮：当前对象是否被 owner 拥有
第一轮扫描：
```text
systable_beginscan(pg_depend, DependDependerIndexId, ...)
```
问题是：
```text
当前 object 自己依赖谁？
```
普通 `NORMAL`、`AUTO`、`AUTO_EXTENSION` outgoing edge 不阻止当前对象作为 drop target 继续处理。
`INTERNAL` 表示当前对象是 referenced object 的内部实现。
直接 drop internal object 时，最外层会报错：
```text
cannot drop X because Y requires it
You can drop Y instead.
```
如果 traversal 是从 owner 递归过来的，就允许继续。
如果 owner 还没有被访问，源码会把请求改写到 owner：
```text
ReleaseDeletionLock(current)
AcquireDeletionLock(owner)
systable_recheck_tuple(scan, tup)
findDependentObjects(owner, DEPFLAG_REVERSE, ...)
```
释放当前锁是为了避免和并发删除 owner 死锁。
拿到 owner 锁后 recheck 是为了确认原 dependency row 还 live。
`EXTENSION` 在普通路径上按 internal-like ownership 处理。
两个例外：
```text
PERFORM_DELETION_SKIP_EXTENSIONS
creating_extension && otherObject == CurrentExtensionObject
```
前者主要用于 temporary-object cleanup。
后者允许 extension script 内部清理 transient/member objects。
partition dependency 在第一轮记录 `DEPFLAG_IS_PART` 和用于报错的 partition owner。
它要求对象从正确 partition owner 路径被删除。
### 5.6 `findDependentObjects()` 第二轮：谁依赖当前对象
第二轮扫描：
```text
systable_beginscan(pg_depend, DependReferenceIndexId, ...)
```
问题是：
```text
哪些 otherObject 依赖当前 object？
```
每个 dependent object 处理顺序：
```text
construct otherObject from classid/objid/objsubid
skip self sub-object rows when dropping whole object
reject unsupported shared-catalog dependents
AcquireDeletionLock(otherObject)
if !systable_recheck_tuple(scan, tup):
    ReleaseDeletionLock(otherObject)
    continue
map deptype to subflags
append to dependentObjects[]
```
`systable_recheck_tuple()` 是并发正确性关键。
等待 dependent object lock 期间，对象或 dependency row 可能已经被别的事务删除。
recheck 失败时，本次 traversal 跳过这条旧边。
源码不会立刻递归。
它先收集到 `dependentObjects[]`。
然后排序：
```text
qsort(..., object_address_comparator)
```
排序主要保证稳定删除顺序和稳定回归测试输出。
最后递归到每个 dependent object。
递归完成后才把当前 object 加入 `targetObjects`。
因此 list 的自然顺序就是：
```text
dependent first
referenced later
```
### 5.7 `reportDependentObjects()`：RESTRICT/CASCADE 分叉
`findDependentObjects()` 只回答“如果允许删除，需要删除什么”。
`reportDependentObjects()` 决定“是否允许”。
核心规则：
```text
original object 不报告。
sub-object 通常不单独报告。
AUTO / INTERNAL / PARTITION / EXTENSION 路径进入的对象，在 RESTRICT 下也可隐式处理。
NORMAL dependent 在 RESTRICT 下报错。
NORMAL dependent 在 CASCADE 下输出 drop cascades to ...
```
伪逻辑：
```text
for object in reverse(targetObjects):
    if original or subobject: skip
    if flags has AUTO/INTERNAL/PARTITION/EXTENSION:
        DEBUG2 auto-cascade
    else if behavior == DROP_RESTRICT:
        collect error detail
        ok = false
    else:
        collect cascade notice
if !ok:
    ERROR with hint "Use DROP ... CASCADE"
```
客户端 detail 最多报告 `MAX_REPORTED_DEPS = 100`。
server log 会拿到更完整 detail。
大型 cascade 诊断不要只看 psql 显示。
### 5.8 `deleteObjectsInList()` 与 event trigger
报告通过后进入：
```text
deleteObjectsInList()
```
如果需要 SQL drop event trigger，它会先调用：
```text
EventTriggerSQLDropAddObject()
```
这里使用 `ObjectAddressExtra.flags` 判断 original / normal 等语义。
随后按 `targetObjects` 顺序调用：
```text
deleteOneObject()
```
如果设置了：
```text
PERFORM_DELETION_SKIP_ORIGINAL
```
原始对象会跳过，只删除依赖它的对象。
### 5.9 `deleteOneObject()`：单对象删除和清理
`deleteOneObject()` 主链路：
```text
InvokeObjectDropHookArg()
maybe close pg_depend for concurrent drop
doDeletion(object, flags)
maybe reopen pg_depend
delete pg_depend rows where object is depender
deleteSharedDependencyRecordsFor()
DeleteComments()
DeleteSecurityLabel()
DeleteInitPrivs()
CommandCounterIncrement()
```
注意顺序。
实际对象删除先发生。
然后删除这个对象作为 depender 的 outgoing `pg_depend` rows。
源码注释解释了 concurrent drop 的约束。
`DROP INDEX CONCURRENTLY` 的 object-specific deletion 可能提交当前事务。
所以在这种路径下，不能先做会被一起提交的 transactional dependency cleanup。
`doDeletion()` 按 `classId` 分发。
常见分支：
```text
RelationRelationId -> index_drop() / heap_drop_with_catalog() / RemoveAttributeById()
ProcedureRelationId -> RemoveFunctionById()
TypeRelationId -> RemoveTypeById()
ConstraintRelationId -> RemoveConstraintById()
RewriteRelationId -> RemoveRewriteRuleById()
TriggerRelationId -> RemoveTriggerById()
ExtensionRelationId -> RemoveExtensionById()
```
不需要特殊逻辑的 catalog object 走：
```text
DropObjectById()
```
global object 如 database、tablespace、subscription 不由这里删除。
删除后调用 `CommandCounterIncrement()`。
这保证前一步 catalog changes 对下一步 deletion scan 可见。
这和 catalog snapshot / invalidation 课程的边界一致。
### 5.10 `RemoveExtensionById()`：extension 自身删除
`RemoveExtensionById()` 的职责很小。
源码注释概括为：
```text
只删除 pg_extension tuple；
其它一切由 dependency infrastructure 处理。
```
也就是说：
```text
DROP EXTENSION ext
  -> dependency traversal 找 member objects
  -> member objects 先删
  -> RemoveExtensionById(ext) 删除 pg_extension row
```
它还禁止删除当前正在创建或修改的 extension：
```text
extId == CurrentExtensionObject
```
否则后续 `recordDependencyOnCurrentExtension()` 可能写入指向已删除 extension OID 的悬空 dependency。
## 6. 生命周期 / ownership / cleanup
### 6.1 谁创建
dependency row 在对象创建或对象定义更新时创建。
常见创建入口：
```text
recordDependencyOn()
recordMultipleDependencies()
recordDependencyOnExpr()
recordDependencyOnSingleRelExpr()
recordDependencyOnCurrentExtension()
recordDependencyOnOwner()
recordDependencyOnTablespace()
updateAclDependencies()
```
它们写的是 catalog tuple。
因此事务 abort 会回滚这些 rows。
dependency row 不需要像 buffer pin 那样由 ResourceOwner 单独释放。
### 6.2 谁持有
持久状态由 catalog 持有：
```text
pg_depend
pg_shdepend
```
一次 DROP 的临时状态由当前 backend memory context 持有：
```text
ObjectAddresses
ObjectAddressStack
ObjectAddressAndFlags[]
```
对象锁由 lock manager / ResourceOwner 持有，通常事务结束释放。
三者语义不同。
catalog row 决定未来 DROP 图。
backend-local list 只决定本次 DROP。
lock 决定并发 DDL 的排队。
### 6.3 谁释放
正常 drop 成功时：
```text
deleteOneObject() 删除 outgoing pg_depend rows
deleteSharedDependencyRecordsFor() 删除 shared dependency rows
DeleteComments() 删除注释
DeleteSecurityLabel() 删除安全标签
DeleteInitPrivs() 删除 initial privileges
free_object_addresses() 释放本次闭包
```
如果中途 ERROR：
```text
memory context reset 清 backend-local traversal state
ResourceOwner release locks/scans/pins
transaction abort 回滚未提交 catalog changes
```
如果错误发生在 `reportDependentObjects()`，通常还没删除对象。
如果错误发生在对象类型删除函数内部，已经做过的 catalog changes 依靠事务 abort 回滚。
已发送的 NOTICE 不会回滚。
### 6.4 extension ownership
extension lifecycle：
```text
InsertExtensionTuple()
  -> pg_extension row
  -> extension normal dependencies
  -> shared owner dependency
execute script with CurrentExtensionObject
  -> member object --DEPENDENCY_EXTENSION--> extension
DROP EXTENSION
  -> find member objects through incoming DEPENDENCY_EXTENSION
  -> delete members
  -> RemoveExtensionById()
```
`ALTER EXTENSION ADD` 是添加 member edge。
`ALTER EXTENSION DROP` 是删除 member edge。
它不是删除对象本身。
这也是 extension ownership 和 role ownership 的差异。
role ownership 是 shared dependency。
extension membership 是 ordinary dependency。
## 7. 正确性机制层次
dependency drop 的正确性由多层组成。
| 层次 | 机制 | 保证 |
| --- | --- | --- |
| identity | `ObjectAddress` | 统一定位 object/sub-object。 |
| graph fact | `pg_depend` / `pg_shdepend` | lifecycle edge 可事务性记录和回滚。 |
| object lock | `AcquireDeletionLock()` | traversal 和删除期间对象级 DDL 排序。 |
| recheck | `systable_recheck_tuple()` | 等锁后确认 dependency row 仍 live。 |
| traversal order | recursive closure | dependent 在 referenced 之前删除。 |
| behavior check | `reportDependentObjects()` | normal dependent 在 RESTRICT 下报错。 |
| object deletion | `doDeletion()` | 每类对象删除自己的 catalog/storage state。 |
| command boundary | `CommandCounterIncrement()` | 当前事务后续删除步骤看到前序 catalog changes。 |
| invalidation | catalog update path | 其它 backend 的 cache 在安全点过期。 |
重要不变量：
```text
dependency graph 只决定生命周期闭包；
不替代 object lock；
不替代 MVCC；
不替代 object-specific deletion；
不替代 cache invalidation。
```
`DROP CASCADE` 不能删除 pinned system object。
`findDependentObjects()` 一开始会检查：
```text
IsPinnedObject(object->classId, object->objectId)
```
pinned object 通常没有必要在 `pg_depend` 中展开所有边。
因此诊断系统对象时，不能只看 `pg_depend`。
`DROP CASCADE` 也不能绕过 object-specific 限制。
例如某些 global objects 不由 `doDeletion()` 支持。
某些 concurrent drop 还有专门事务边界。
## 8. 错误路径 / 异常路径 / fallback
### 8.1 RESTRICT 失败
路径：
```text
performDeletion()
  -> findDependentObjects()
  -> targetObjects contains normal dependent
  -> reportDependentObjects(DROP_RESTRICT)
  -> ERROR
```
这个错误发生在实际删除前。
用户通常看到：
```text
cannot drop ... because other objects depend on it
Use DROP ... CASCADE to drop the dependent objects too.
```
### 8.2 直接删除 internal / extension member
路径：
```text
DROP member_object
  -> first scan sees INTERNAL or EXTENSION outgoing edge
  -> stack == NULL
  -> remember owningObject
  -> ERROR after scan
```
错误会提示 drop owner。
extension member 通常提示 drop extension。
如果同时存在 partition owner 信息，源码可能优先报告 partition 相关对象。
### 8.3 lock wait 后 dependency row 失效
第二轮 incoming scan 找到 dependent 后，会先锁 dependent object。
等待期间其它事务可能删除该 dependent 或 dependency row。
所以拿到锁后要：
```text
systable_recheck_tuple(scan, tup)
```
失败则释放锁并跳过。
internal owner 改写路径也有类似 recheck。
这是在并发 DDL 下避免使用过期 dependency edge 的关键 fallback。
### 8.4 referenced object 被并发删除时记录 dependency
创建 dependency 时，`dependencyLockAndCheckObject()` 会尝试锁住 referenced object。
如果锁后发现对象不存在，报：
```text
referenced ... was concurrently dropped
```
这是 recording 侧的并发保护。
DROP 侧靠 deletion lock 和 recheck。
### 8.5 extension script 例外
extension script 内部允许删除当前 extension 的 member dependency。
条件是：
```text
creating_extension
otherObject.classId == ExtensionRelationId
otherObject.objectId == CurrentExtensionObject
```
这允许脚本管理 transient object。
但 `RemoveExtensionById()` 禁止删除当前 extension row。
两者共同防止 dangling membership。
### 8.6 `DROP INDEX CONCURRENTLY`
`PERFORM_DELETION_CONCURRENTLY` 只适用于窄场景。
它影响 lock mode 和 `deleteOneObject()` 中 `pg_depend` relation 的开关。
不要把它当成普通 cascade 的事务模型。
普通 DROP 通常在一个事务里按闭包删除。
## 9. 成本、资源与跨模块传播
dependency traversal 是低频 DDL 成本，不是普通查询 hot path。
主要成本变量：
| 变量 | 影响 |
| --- | --- |
| dependency edge 数 | `pg_depend` index scan 命中更多 tuple。 |
| cascade closure 大小 | 递归、lock、object description、delete 调用增加。 |
| closure 深度 | 递归深度增加，可能触发 `check_stack_depth()`。 |
| 并发 DDL | deletion lock wait 和 recheck 增加。 |
| 分区对象数量 | partition dependency 和 relation locks 放大。 |
| extension member 数 | `DROP EXTENSION` 闭包扩大。 |
| object-specific deletion | table/index 删除可能引入 storage、WAL、relcache invalidation 成本。 |
| report detail | 客户端限制 100 条，server log 可能很大。 |
`pg_depend` 有两个方向索引。
所以不是全表扫描图。
但大型 `DROP SCHEMA ... CASCADE` 仍可能慢。
瓶颈经常在：
```text
对象锁等待
大量 object-specific deletion
WAL 和 catalog tuple delete
storage unlink
relcache/plancache invalidation 后重建
server log detail 构造
```
跨模块传播：
```text
lock manager      排队 DDL/DML
syscache/catcache 描述对象和扫描 catalog
relcache          relation/index metadata invalidation
storage manager   table/index 文件生命周期
WAL               catalog/storage changes
event trigger     SQL DROP 观察点
extension         membership ownership
autovacuum        未来清理 dead catalog tuples
startup process   recovery 重放 catalog changes
```
没有后台进程维护 dependency graph。
graph traversal 由执行 DROP 的 foreground backend 同步完成。
可迁移规律：
```text
低频、高正确性的生命周期图适合放在事务性 catalog 中；
一次操作的 closure 用 backend-local memory 临时构造；
高频读取另用 cache/invalidation 降成本。
```
## 10. 观测与诊断入口
### 10.1 查询 incoming dependencies
看谁依赖某个对象：
```sql
SELECT
  d.deptype,
  pg_describe_object(d.classid, d.objid, d.objsubid) AS dependent,
  pg_describe_object(d.refclassid, d.refobjid, d.refobjsubid) AS referenced
FROM pg_depend AS d
WHERE d.refclassid = 'pg_class'::regclass
  AND d.refobjid = 'public.t'::regclass
ORDER BY 1, 2;
```
这对应 `findDependentObjects()` 第二轮 incoming scan。
### 10.2 查询 outgoing dependencies
看某对象依赖谁：
```sql
SELECT
  d.deptype,
  pg_describe_object(d.classid, d.objid, d.objsubid) AS dependent,
  pg_describe_object(d.refclassid, d.refobjid, d.refobjsubid) AS referenced
FROM pg_depend AS d
WHERE d.classid = 'pg_class'::regclass
  AND d.objid = 'public.v'::regclass
ORDER BY 1, 3;
```
这对应 `findDependentObjects()` 第一轮 outgoing scan。
诊断 internal / extension membership 时不要只查 incoming edge。
### 10.3 查询 extension membership
extension members：
```sql
SELECT
  e.extname,
  d.deptype,
  pg_describe_object(d.classid, d.objid, d.objsubid) AS member
FROM pg_depend AS d
JOIN pg_extension AS e
  ON d.refclassid = 'pg_extension'::regclass
 AND d.refobjid = e.oid
WHERE d.deptype = 'e'
ORDER BY e.extname, member;
```
如果对象不能直接 drop，先查它是否有：
```text
object --DEPENDENCY_EXTENSION--> pg_extension
```
### 10.4 观察 locks
DROP 等锁时：
```sql
SELECT locktype, relation::regclass, classid::regclass, objid, objsubid,
       mode, granted, pid
FROM pg_locks
WHERE locktype IN ('relation', 'object')
ORDER BY granted, pid;
```
能看到对象锁。
看不到 `targetObjects`、`ObjectAddressExtra.flags` 和 recheck 是否跳过某条 row。
### 10.5 server log 与客户端 detail
`reportDependentObjects()` 限制客户端 detail。
大量依赖时，server log 更完整。
如果只看 psql 输出，可能误以为 cascade closure 很小。
### 10.6 gdb 断点
建议断点：
```text
recordMultipleDependencies
recordDependencyOnCurrentExtension
performDeletion
findDependentObjects
reportDependentObjects
deleteObjectsInList
deleteOneObject
doDeletion
AcquireDeletionLock
RemoveExtensionById
```
重点变量：
```text
*object
foundDep->deptype
otherObject
objflags
subflags
targetObjects->numrefs
targetObjects->refs[i]
targetObjects->extras[i].flags
behavior
flags
creating_extension
CurrentExtensionObject
```
可直接观测：
```text
pg_depend rows
pg_shdepend rows
pg_extension membership
pg_locks
DROP ERROR / NOTICE
server log detail
```
只能推断或断点观察：
```text
targetObjects 完整闭包
ObjectAddressExtra.flags
递归 stack
systable_recheck_tuple 是否跳过 row
event trigger 记录前的 internal classification
```
## 11. 常见误区
误区一：
```text
pg_depend 是引用计数。
```
不是。
它是生命周期边。
backend 是否正在使用对象由 lock、pin、refcount、snapshot 等机制决定。
误区二：
```text
CASCADE 是忽略依赖。
```
相反，CASCADE 依赖 `pg_depend` 才知道要删除什么。
漏记 dependency 时，CASCADE 也找不到对象。
误区三：
```text
RESTRICT 绝不删除任何附属对象。
```
RESTRICT 阻止 normal dependent 被级联。
auto/internal/extension/partition 语义仍可能让附属对象被隐式处理。
误区四：
```text
只查 refobjid 就能解释所有 drop 错误。
```
不能。
直接删除 extension/internal member 时，关键 edge 在 outgoing side。
误区五：
```text
extension owner role 和 extension member ownership 是同一种依赖。
```
不是。
role owner 在 `pg_shdepend`。
member ownership 在 `pg_depend`，deptype 是 `DEPENDENCY_EXTENSION`。
误区六：
```text
删除对象只需要删除 pg_depend rows。
```
不对。
必须调用 object-specific deletion，删除对象 catalog tuple、storage、comments、security labels、shared dependencies，并触发 invalidation。
误区七：
```text
没有 pg_depend row 就能 drop。
```
不一定。
pinned system object、shared dependency 和 object-specific restrictions 仍可能阻止删除。
## 12. 课堂实验
### 实验 1：view 依赖 table
目标：观察 normal dependency 如何影响 RESTRICT/CASCADE。
SQL：
```sql
DROP SCHEMA IF EXISTS depdemo CASCADE;
CREATE SCHEMA depdemo;
CREATE TABLE depdemo.t(id int primary key, v int);
CREATE VIEW depdemo.v AS SELECT id FROM depdemo.t;
SELECT d.deptype,
       pg_describe_object(d.classid, d.objid, d.objsubid) AS dependent,
       pg_describe_object(d.refclassid, d.refobjid, d.refobjsubid) AS referenced
FROM pg_depend d
WHERE d.refobjid = 'depdemo.t'::regclass
ORDER BY 1, 2;
DROP TABLE depdemo.t RESTRICT;
DROP TABLE depdemo.t CASCADE;
```
断点：
```text
performDeletion
findDependentObjects
reportDependentObjects
deleteOneObject
```
观察：
```text
DEPENDENCY_NORMAL 如何进入 DEPFLAG_NORMAL；
RESTRICT 在 reportDependentObjects() 失败；
CASCADE 进入 deleteObjectsInList()。
```
### 实验 2：extension membership
目标：观察 `DEPENDENCY_EXTENSION`。
SQL：
```sql
CREATE EXTENSION IF NOT EXISTS hstore;
SELECT e.extname, d.deptype,
       pg_describe_object(d.classid, d.objid, d.objsubid) AS member
FROM pg_depend d
JOIN pg_extension e
  ON d.refclassid = 'pg_extension'::regclass
 AND d.refobjid = e.oid
WHERE e.extname = 'hstore'
  AND d.deptype = 'e'
ORDER BY member;
```
挑一个 member object 直接 drop，再比较：
```sql
DROP EXTENSION hstore;
```
断点：
```text
findDependentObjects
recordDependencyOnCurrentExtension
RemoveExtensionById
```
观察：
```text
直接 drop member 走 outgoing EXTENSION edge；
DROP EXTENSION 从 pg_extension row incoming edge 找 members；
RemoveExtensionById() 只删除 pg_extension row。
```
## 13. 讨论题
1. 为什么 `DROP RESTRICT` 和 `DROP CASCADE` 共享 `findDependentObjects()`？
2. 如果创建 view 时漏记 column dependency，未来 `DROP COLUMN ... CASCADE` 会出现什么风险？
3. 为什么 internal / extension ownership 必须检查 outgoing dependency？
4. 为什么 pinned object 可以没有普通 `pg_depend` rows？
5. `dependencyLockAndCheckObject()` 和 `systable_recheck_tuple()` 分别保护哪个并发窗口？
6. extension member、extension required extension、extension owner role 分别存在哪里？
7. 为什么 `deleteOneObject()` 不能只删除 object catalog tuple？
8. 大型 `DROP SCHEMA ... CASCADE` 慢时，如何区分 dependency traversal、lock wait、object-specific deletion 和 invalidation 成本？
## 14. 本节小结
本节的核心链路：
```text
recordDependencyOn*()
  -> pg_depend / pg_shdepend
  -> performDeletion()
  -> findDependentObjects()
  -> reportDependentObjects()
  -> deleteObjectsInList()
  -> deleteOneObject()
  -> object-specific deletion
  -> dependency cleanup
  -> catalog invalidation
```
核心状态：
```text
ObjectAddress 标识对象或 sub-object；
pg_depend 记录 database-local lifecycle edge；
pg_shdepend 记录 role、ACL、tablespace 等 shared edge；
ObjectAddresses 保存一次 DROP 的删除闭包；
ObjectAddressExtra.flags 保存对象进入闭包的原因。
```
RESTRICT/CASCADE 的边界：
```text
NORMAL dependent 在 RESTRICT 下阻止删除，在 CASCADE 下进入级联删除；
AUTO / INTERNAL / PARTITION / EXTENSION 有更强的隐式或 ownership 语义；
CASCADE 不会绕过 pinned object、object locks、object-specific deletion 限制。
```
extension ownership：
```text
member object --DEPENDENCY_EXTENSION--> pg_extension row；
DROP EXTENSION 通过 dependency traversal 删除 members；
ALTER EXTENSION ADD/DROP 改的是 membership edge；
extension owner role 是 pg_shdepend，不是 DEPENDENCY_EXTENSION。
```
正确性依赖：
```text
dependency row 提供图事实；
deletion lock 排序并发 DDL；
recheck 处理锁等待后的边失效；
递归顺序保证 dependent 先删；
CCI 推进同事务内 catalog 可见性；
catalog invalidation 让其它 backend 的 metadata cache 过期；
transaction abort 回滚未提交 catalog changes。
```
诊断时先问四个问题：
```text
这条 dependency edge 是否存在？
方向是否读对？
deptype 是 cascade policy 还是 ownership policy？
失败发生在 recording、closure traversal、RESTRICT report、object-specific deletion，还是 commit/invalidation 后？
```
这四个问题比直接猜某个 DDL 函数更可靠。
