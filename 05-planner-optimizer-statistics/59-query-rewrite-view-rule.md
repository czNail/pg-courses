# PostgreSQL Query Rewrite / View / Rule

## 课程定位
前置知识：已经理解 parse analysis 会把 SQL 文本变成语义化 `Query`，也知道 planner 入口接收的是 rewrite 后的 `Query` 或 `Query` list。
本节唯一主问题：
```text
view/rule rewrite 如何把用户 Query 改写成等价 Query，哪些边界会影响权限和可更新视图？
```
核心矛盾：view 要像普通 relation 一样可读、可授权、可在简单场景中可更新；rule 又允许一条 DML 变成 0 条、1 条或多条语句。但 rewrite 不能只追求结果行等价，还必须保留原对象身份、权限身份、RLS/security barrier 顺序和 WITH CHECK OPTION。
学完后应能判断：
- view 的 `_RETURN` ON SELECT rule 如何把 `RTE_RELATION` 改成 `RTE_SUBQUERY`。
- 为什么 view 展开后仍保留 `relid`、`rellockmode`、`perminfoindex`。
- DML rule 为什么可能返回多个 `Query`。
- auto-updatable view 什么时候能改写到 base relation，什么时候必须报错或交给 INSTEAD OF trigger。
- `security_barrier`、`security_invoker`、RLS 和 WCO 在 rewrite 阶段分别留下什么状态。
本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e9`。用户建议核对的 `src/backend/catalog/pg_rewrite.c` 在这个源码树中不存在；本课使用真实路径 `src/backend/rewrite/rewriteDefine.c` 和 `src/include/catalog/pg_rewrite.h` 解释 rule catalog 边界。

## 1. 本节在总主线中的位置
本节补齐 planner 之前的最后一个语义阶段。
主线是：
```text
SQL text
  -> raw parse tree
  -> analyzed Query
  -> query rewrite
  -> rewritten Query list
  -> planner
  -> PlannedStmt
```
parser/analyzer 解决名字、类型、target relation、初始权限需求。
rewriter 解决 view/rule/RLS/WCO 的语义展开。
planner 只在这些边界已经稳定后，才开始 subquery pullup、qual 下推、path search。
本节只围绕 rewrite/view/rule 展开。
不讨论 rule system 的全部 SQL 语法。
不展开 planner 如何使用 `securityQuals` 决定 qual pushdown。
也不把 view 展开和 planner subquery pullup 混为一谈。
一个简化判断是：
```text
rewrite expansion 是语义必需动作；
planner pullup 是受安全和语义约束的优化动作。
```
这一区分能解释很多诊断误判。
例如 `EXPLAIN` 最终没有 `Subquery Scan`，不代表 view 没有在 rewrite 阶段展开。
它可能只是后续 planner 又把简单 subquery 拉平了。

## 2. 核心矛盾与一句话运行模型
一句话运行模型：
```text
QueryRewrite() 先调用 RewriteQuery() 处理非 SELECT rules，
可能生成 0 个、1 个或多个 Query；
然后对每个 Query 调用 fireRIRrules()，
把 view 的 _RETURN ON SELECT rule 展开成 RTE_SUBQUERY，
并把权限、锁、security barrier、RLS、WCO 和可更新 view 状态写回 Query。
```
核心 tension 是：
```text
把 view/rule 展开成 planner 能理解的 Query
  vs
不能丢失用户实际访问的是哪个对象、用谁的权限、哪些 qual 必须先执行、哪些行修改必须满足 view 条件
```
如果只考虑结果行，view 很像一个文本宏：
```sql
SELECT * FROM v WHERE id = 1;
```
可以想成：
```sql
SELECT * FROM (SELECT ... FROM base WHERE ...) v WHERE id = 1;
```
但 PostgreSQL 不能只做这种替换。
它还必须回答：
- 用户有没有 `SELECT` view 的权限？
- view owner 有没有读 base table 的权限？
- view 是否是 `security_invoker`，从而要求 caller 也有 base table 权限？
- view 的 WHERE qual 是否来自 `security_barrier` view？
- RLS policy 应按 caller 还是 `checkAsUser` 身份展开？
- INSERT/UPDATE 后的新行是否满足 view 的 `WITH CHECK OPTION`？
- DML target 是 view 时，目标列如何映射到底层表列？
所以 rewrite 输出的 `Query` 往往不是最“漂亮”的 query tree。
它更像一个携带安全和权限边界的语义中间态。
这也是本节要建立的 mental model。

## 3. 核心文件分工与阅读顺序
| 顺序 | 文件 | 作用 |
| --- | --- | --- |
| 1 | `src/backend/tcop/postgres.c` | `pg_analyze_and_rewrite_*()` 在 parse analysis 后调用 `QueryRewrite()`。 |
| 2 | `src/backend/rewrite/rewriteHandler.c` | 主体：`QueryRewrite()`、`RewriteQuery()`、`fireRIRrules()`、`ApplyRetrieveRule()`、`rewriteTargetView()`。 |
| 3 | `src/backend/rewrite/rewriteManip.c` | tree 改写工具：`CombineRangeTables()`、`ReplaceVarsFromTargetList()`、`ChangeVarNodes()`。 |
| 4 | `src/backend/rewrite/rowsecurity.c` | RLS policy 生成 `securityQuals` 和 `withCheckOptions`，并传播 `checkAsUser`。 |
| 5 | `src/backend/rewrite/rewriteDefine.c` | `DefineQueryRewrite()`、`setRuleCheckAsUser()`，创建和维护 rewrite rule。 |
| 6 | `src/backend/commands/view.c` | `CREATE VIEW` 通过 `DefineQueryRewrite()` 创建 `_RETURN` rule。 |
| 7 | `src/backend/parser/analyze.c` | parse analysis 建立 target relation、`rteperminfos` 和 view DML 的初始边界。 |
| 8 | `src/include/nodes/parsenodes.h` | `Query`、`RangeTblEntry`、`RTEPermissionInfo`、`WithCheckOption`。 |
| 9 | `src/include/rewrite/rewriteHandler.h` | rewriter 对外接口和 updatable view 判断接口。 |
| 10 | `src/include/catalog/pg_rewrite.h` | `pg_rewrite` catalog 声明。 |
推荐阅读路径：
```text
QueryRewrite()
  -> RewriteQuery()
  -> fireRIRrules()
  -> ApplyRetrieveRule()
  -> rewriteTargetView()
  -> get_row_security_policies()
  -> DefineQueryRewrite()
```
`rewriteHandler.c` 里有历史术语：
```text
RIR = Retrieve-Instead-Retrieve
```
今天可以把它理解为 view 的 `ON SELECT DO INSTEAD SELECT` rule。
普通 view 的 rule 名是 `_RETURN`，在 `rewriteSupport.h` 中由 `ViewSelectRuleName` 定义。

## 4. 关键数据结构与状态

### `Query`
`Query` 是 rewrite 的输入和输出单位。
本节重点字段：
```text
rtable
rteperminfos
jointree
targetList
resultRelation
withCheckOptions
hasRowSecurity
querySource
canSetTag
```
`QueryRewrite()` 返回 `List *`。
这说明一条用户语句可能被 rule 改写成多条 `Query`。
后续 tcop/planner 处理的是这个 list，而不是原 SQL。

### `RangeTblEntry`
view 展开的核心状态在 `RangeTblEntry`。
parse analysis 后，引用 view 的 RTE 通常是：
```text
rtekind = RTE_RELATION
relid = view oid
perminfoindex = index into Query.rteperminfos
```
`ApplyRetrieveRule()` 展开 view 时原位修改：
```text
rtekind = RTE_SUBQUERY
subquery = copied view query
security_barrier = RelationIsSecurityView(view)
```
但它故意保留：
```text
relid
relkind
rellockmode
perminfoindex
```
`parsenodes.h` 明确说明这是 view expansion 的特殊情况。
原因是 view relation 虽然不再作为扫描节点直接使用，但仍要在执行启动前被锁住并做权限检查。
因此 view 展开后的 RTE 是混合状态：
```text
subquery semantics + original view identity
```
这是本节最重要的不变量。

### `RTEPermissionInfo`
`RTEPermissionInfo` 表示运行期权限检查契约。
关键字段：
```text
relid
inh
requiredPerms
checkAsUser
selectedCols
insertedCols
updatedCols
```
`requiredPerms` 是 ACL bitmask。
`checkAsUser` 非 0 时，权限检查使用该 role，而不是当前 effective user。
rule 和 view 依赖它实现 owner 权限边界。
它不是注释，也不是 planner hint。
executor startup 会消费这些信息做权限检查。

### `RewriteRule`
运行时通过 relcache 的 `Relation.rd_rules` 看到 rules。
本节关心这些语义：
```text
event: CMD_SELECT / CMD_INSERT / CMD_UPDATE / CMD_DELETE / CMD_MERGE
isInstead: DO INSTEAD vs DO ALSO
qual: rule qualification
actions: analyzed Query actions
```
view 的 `_RETURN` rule 是特殊的 `CMD_SELECT` rule。
`DefineQueryRewrite()` 强制它：
- 只能定义在 view 或 materialized view 上。
- 不能是 INSTEAD NOTHING。
- 不能有多个 action。
- action 必须是 `INSTEAD SELECT`。
- 不能有 data-modifying WITH。
- 不能有 rule qual。
- targetlist 必须匹配 view rowtype。

### `WithCheckOption`
`WithCheckOption` 表示对新 tuple 的检查。
它既服务 auto-updatable view，也服务 RLS WITH CHECK policy。
本地源码中的 `WCOKind` 包括：
```text
WCO_VIEW_CHECK
WCO_RLS_INSERT_CHECK
WCO_RLS_UPDATE_CHECK
WCO_RLS_CONFLICT_CHECK
WCO_RLS_MERGE_UPDATE_CHECK
WCO_RLS_MERGE_DELETE_CHECK
```
view 的 `WITH CHECK OPTION` 在 rewrite 阶段进入 `Query.withCheckOptions`。
这不是优化器推断出来的 filter。
它是 executor 必须检查的语义条件。

### `securityQuals`
`RangeTblEntry.securityQuals` 在 parser 输出中为空。
rewriter 会为 security-barrier view 和 RLS 添加它。
语义是：
```text
这些 qual 必须在 relation 返回一行前按顺序测试。
```
后续 planner 可以做优化，但不能破坏这个安全顺序。

### 状态判断规则
不要把 raw field 当语义。
看到 `RTE_SUBQUERY`，不能立刻断言“不需要 relation 权限”。
view 展开的 `RTE_SUBQUERY` 仍可能有 `relid` 和 `perminfoindex`。
看到 `checkAsUser`，也不能单独判断权限边界。
必须同时看：
```text
requiredPerms
checkAsUser
RelationHasSecurityInvoker(view)
rule action owner
RLS policy expansion
securityQuals / withCheckOptions
```

## 5. 主流程源码 walkthrough
主调用链：
```text
pg_analyze_and_rewrite_*()
  -> QueryRewrite()
     -> RewriteQuery()
        -> rewrite DML targetlist
        -> matchLocks()
        -> fireRules()
        -> rewriteTargetView() if needed
        -> recursively rewrite product queries
     -> fireRIRrules()
        -> ApplyRetrieveRule() for view _RETURN rule
     -> choose canSetTag query
```

### 5.1. rewrite 入口
`QueryRewrite()` 只处理 top-level original query。
源码断言：
```text
querySource == QSRC_ORIGINAL
canSetTag == true
```
它保存输入 `queryId`。
rule expansion 可能复制和替换 query。
最终每个输出 query 会重新带上原 query id。

### 5.2. `RewriteQuery()` 处理非 SELECT rules
第一阶段：
```text
querylist = RewriteQuery(parsetree, NIL, 0, 0)
```
它主要处理 INSERT/UPDATE/DELETE/MERGE rules。
目标 relation 流程简化为：
```text
relation_open(target, NoLock)
rewriteTargetListIU(...)
matchLocks(...)
fireRules(...)
maybe rewriteTargetView(...)
table_close(target, NoLock)
```
这里的 `NoLock` 不是无保护。
parser 或 `AcquireRewriteLocks()` 已经根据 RTE 的 `rellockmode` 拿过锁。
`RewriteQuery()` 只是打开 relcache 状态读取 rules。

### 5.3. rule 为什么可能产生多个 Query
DML rule 有三种基本效果：
```text
DO ALSO: 原 query 保留，追加 action query。
DO INSTEAD NOTHING: 原 query 消失。
DO INSTEAD action: 用 action query 替代原 query。
```
qualified INSTEAD rule 还可能产生 `qual_product`。
所以 `QueryRewrite()` 的输出是 list。
这会影响后续 planner 调用次数和 command tag。
`QueryRewrite()` 最后会决定哪条 query 可以 `canSetTag`。
如果原 query 还在，它设置 tag。
否则，最后一个同 command type 的 INSTEAD query 可以设置 tag。

### 5.4. DML rules 的异常边界
rule expansion 会和一些 SQL 语义冲突。
典型边界：
```text
data-modifying WITH 只能接受 unconditional single-statement DO INSTEAD
WITH query 如果被 rules 改成多 query，会破坏 CTE single-evaluation 语义
INSERT ... ON CONFLICT 不能用于带 INSERT/UPDATE rules 的普通表
```
对应错误不是执行期偶然失败。
它们在 rewrite 阶段拒绝，因为此时系统已经知道等价改写无法保持原 SQL 语义。

### 5.5. `fireRIRrules()` 展开 view
第二阶段对每个 query 调用：
```text
query = fireRIRrules(query, NIL)
```
核心循环按 rtable index 扫描。
它不能简单 foreach，因为 view expansion 可能追加新的 RTE。
简化流程：
```text
for each rtable entry:
  if RTE_SUBQUERY:
    recurse into subquery
  if not RTE_RELATION:
    continue
  if materialized view:
    continue
  if ON CONFLICT EXCLUDED pseudo relation:
    continue
  if RTE not referenced:
    continue
  open relation
  collect CMD_SELECT rules
  detect activeRIR recursion
  ApplyRetrieveRule()
  close relation
recurse into CTE queries
```
递归保护使用 `activeRIRs`。
如果 view 递归引用自身，rewrite 抛出：
```text
infinite recursion detected in rules for relation ...
```

### 5.6. `ApplyRetrieveRule()` 的检查
`ApplyRetrieveRule()` 处理 view `_RETURN` rule。
入口要求：
```text
rule actions length == 1
rule qual == NULL
```
普通 SQL 路径不会创建违反这些约束的 view rule。
如果 catalog 被破坏或扩展错误制造了非法 rule，会在这里 `elog(ERROR)`。

### 5.7. view query copy、lock、递归展开
`ApplyRetrieveRule()` 先复制 rule action：
```text
rule_action = copyObject(linitial(rule->actions))
```
然后：
```text
AcquireRewriteLocks(rule_action, true, rc != NULL)
```
这保证 view definition 中引用的 relations 在 rewrite、planning、execution 期间 schema 稳定。
如果外层 `FOR UPDATE` / `FOR SHARE` 影响 view，`markQueryForLocking()` 会把锁语义推进 view 内部 relations。
随后：
```text
rule_action = fireRIRrules(rule_action, activeRIRs)
```
也就是 view 内部的 view 继续递归展开。

### 5.8. view RTE 原位变成 `RTE_SUBQUERY`
核心状态变化：
```text
rte = rt_fetch(rt_index, parsetree->rtable)
rte->rtekind = RTE_SUBQUERY
rte->subquery = rule_action
rte->security_barrier = RelationIsSecurityView(relation)
```
接着清掉不适用于 subquery RTE 的字段：
```text
rte->tablesample = NULL
rte->inh = false
```
但保留：
```text
relid
relkind
rellockmode
perminfoindex
```
所以等价改写后，RTE 同时表达两件事：
```text
用 subquery 表示 view definition；
用保留的 relation 字段表示用户访问的 view 对象。
```

### 5.9. view 的 `_RETURN` rule 如何创建
`CREATE VIEW` 在 `view.c` 中调用 `DefineViewRules()`。
核心是：
```text
DefineQueryRewrite(ViewSelectRuleName,
                   viewOid,
                   NULL,
                   CMD_SELECT,
                   true,
                   replace,
                   list_make1(viewParse))
```
`ViewSelectRuleName` 是 `_RETURN`。
因此普通 view 的持久形态可以理解为：
```text
pg_class relkind = view
  + pg_rewrite _RETURN rule
  + stored analyzed Query action
```
`CREATE OR REPLACE VIEW` 不能删除已有列，也不能改已有列名字、类型、typmod、collation。
原因是其它 stored query tree 可能已经用 `Var` 引用了这些列。

## 6. 生命周期 / ownership / cleanup

### 创建者
`CREATE VIEW` 创建 view relation 后，`DefineQueryRewrite()` 创建 `_RETURN` rule。
普通 user-defined rules 也通过 `DefineQueryRewrite()` 进入 `pg_rewrite`。
这些是 catalog 状态。

### 持有者
运行时 relcache 通过 `Relation.rd_rules` 暴露 rules。
rewriter 读取 rule action 后用 `copyObject()` 复制。
后续 `ChangeVarNodes()`、`ReplaceVarsFromTargetList()` 等都改写当前语句副本。
不会修改 relcache 中缓存的 rule 原件。

### 锁
`AcquireRewriteLocks()` 按 RTE 的 `rellockmode` 获取 relation locks。
这些 locks 由事务和 resource owner 生命周期管理。
rewrite 阶段不自己实现 refcount 或 WAL。
它依赖 relation lock 保证 schema 不在 rewrite/planning/execution 中途变化。

### 权限状态
权限状态在 `Query.rteperminfos`。
RTE 通过 `perminfoindex` 引用。
view SELECT 展开后：
```text
外层 view RTE 变成 RTE_SUBQUERY
但保留 view 的 perminfoindex
view definition 内部 base relations 使用 rule action 中的 perminfos
```
可更新 view 额外添加 base relation RTE 和新的 `RTEPermissionInfo`。
原 view perminfo 仍保留，用来检查 caller 对 view 的权限。

### cleanup
rewrite 产生的 query tree 是 backend-local memory。
正常路径由 query/portal 相关 memory context 清理。
ERROR/abort 通过错误栈展开和 memory context reset 清理。
relation locks 随事务结束或 abort 释放。
catalog/relcache 状态通过 invalidation 保持跨语句一致。

## 7. 正确性机制层次
| 层次 | 机制 | 保证 |
| --- | --- | --- |
| catalog | `pg_rewrite`、view `_RETURN` rule | 持久 view/rule definition。 |
| relcache | `Relation.rd_rules` | 当前 backend 读取 rule metadata 和 action。 |
| lock | `AcquireRewriteLocks()`、`rellockmode` | rewrite 到 execution 期间 relation schema 稳定。 |
| copy | `copyObject()` | 当前语句可改写 rule action，不污染 relcache 原件。 |
| identity | 保留 `relid` / `perminfoindex` | view 展开后仍检查 view 对象权限和锁。 |
| permission | `requiredPerms` / `checkAsUser` | startup 按正确 role 检查 ACL。 |
| RLS | `securityQuals` / `withCheckOptions` / `hasRowSecurity` | 行安全 qual、WCO 和 plan cache 依赖。 |
| barrier | `security_barrier` / `securityQuals` | 安全 qual 顺序不能被可泄漏表达式跨越。 |
| recursion | `activeRIRs` / `rewrite_events` | 防止 view/rule 无限展开。 |
锁不是权限。
权限不是 RLS。
RLS 不是普通 WHERE。
security barrier 不是性能 hint。
这四句话是读本模块源码时最有用的边界。
`AcquireRewriteLocks()` 只保证 relation definition 稳定。
ACL 权限由 `RTEPermissionInfo` 表示。
RLS policy 在 `rowsecurity.c` 里变成 `securityQuals` 或 `withCheckOptions`。
security barrier 约束后续 planner 对 qual 的重排。

## 8. 错误路径 / 异常路径 / fallback

### view 递归
`fireRIRrules()` 用 `activeRIRs` 记录正在展开的 view OID。
重复出现时抛出 infinite recursion。
这是 rewrite 阶段错误，不等到 planner。

### DML rule 递归
`RewriteQuery()` 用 `rewrite_events` 记录 relation OID 和 event。
同一 relation 上同一 event 递归触发时拒绝。
它和 view 的 `activeRIRs` 分开，因为 DML rule 和 SELECT view expansion 的递归维度不同。

### CTE 与多 query rules
如果带 CTE 的 query 被 rule 改成多个 non-utility query，rewriter 报错。
原因是 CTE 要求单次求值。
把 CTE list 复制到多个 query 会改变语义。

### materialized view
`fireRIRrules()` 跳过 `RELKIND_MATVIEW`。
普通查询引用 matview 时，它像 relation 被扫描。
刷新 matview 走专门路径。

### `EXCLUDED` pseudo relation
`INSERT ... ON CONFLICT` 的 `EXCLUDED` RTE 不展开。
即使它指向 view，也必须保持 `RTE_RELATION` 形态。
否则 ON CONFLICT 后续处理没有稳定语义。

### 非可更新 view
DML target 是 view 时 fallback 顺序是：
```text
unconditional INSTEAD rule
  -> INSTEAD OF trigger
  -> auto-updatable view rewrite
  -> ERROR
```
如果 auto-updatable 检查失败，`error_view_not_updatable()` 给出更具体原因。

### RETURNING
如果原 query 带 RETURNING，而 INSTEAD rule 接管后没有可用 RETURNING，rewriter 报错。
auto-updatable view 路径会重写 RETURNING list，因此不会触发同类错误。

## 9. 可更新视图边界

### 9.1. 入口
`RewriteQuery()` 处理 DML target 后检查：
```text
no unqualified INSTEAD rule
target relation is RELKIND_VIEW
no matching INSTEAD OF trigger
```
满足这些条件时调用：
```text
rewriteTargetView(parsetree, view_relation)
```
成功后，改写后的 query 被加入 product_queries，并递归 rewrite。

### 9.2. auto-updatable 结构条件
`view_query_is_auto_updatable()` 要求 view definition 足够简单。
典型拒绝条件：
```text
DISTINCT
GROUP BY / grouping sets
HAVING
set operations
WITH
LIMIT / OFFSET
aggregate functions
window functions
targetlist SRF
FROM 不只一个 table/view
TABLESAMPLE
```
允许的 base RTE relkind：
```text
ordinary table
foreign table
view
partitioned table
```
函数只检查当前 view definition。
如果 base relation 又是 view，后续递归处理。

### 9.3. 为什么这些限制存在
auto-updatable view 依赖一个假设：
```text
每个 view row 可以稳定映射到一个底层 base relation row。
```
聚合、DISTINCT、GROUP BY、set operation、window、LIMIT、多 base relation 都破坏这个映射。
rewrite 阶段不能凭成本或猜测决定要更新哪张表、哪一行。
所以它拒绝。

### 9.4. column-level updatability
PostgreSQL 允许一个 view 中既有可更新列，也有不可更新列。
只要 INSERT/UPDATE 不写不可更新列，仍可自动改写。
`view_col_is_auto_updatable()` 要求该列最终是 base relation 的普通 `Var`。
表达式列、junk 列、system column、whole-row Var 都不可更新。
例子：
```sql
CREATE VIEW v AS
  SELECT id, lower(name) AS lname FROM t;
```
`id` 可能可更新。
`lname` 不可更新。

### 9.5. `rewriteTargetView()` 做什么
成功路径简化为：
```text
copy view query
validate view structure and target columns
open base relation with RowExclusiveLock
append base relation RTE to outer query
create new RTEPermissionInfo for base relation
rewrite Vars from view columns to base columns
change resultRelation to new base RTE
remap targetlist resno to base attno
move/pull view quals
add WithCheckOption if needed
```
这段逻辑很像 planner 的 pullup。
但它必须在 rewrite 阶段发生。
原因是 result relation、权限、RLS、WCO 和 targetlist resno 都是 planner 之前必须稳定的语义状态。

### 9.6. 权限如何分裂
auto-updatable view 不会丢弃 view 权限检查。
它会新增 base relation 权限检查。
规则是：
```text
caller still needs privilege on the view
base relation privilege is checked as caller if security_invoker
otherwise base relation privilege is checked as view owner
```
源码中可更新 view 路径是：
```text
if RelationHasSecurityInvoker(view)
  new_perminfo->checkAsUser = InvalidOid
else
  new_perminfo->checkAsUser = view owner
```
`InvalidOid` 表示后续使用当前用户。
这就是 `security_invoker` view 的核心影响。

### 9.7. view qual 和 WCO
UPDATE/DELETE/MERGE 会把 view WHERE qual 拉到外层。
非 security-barrier view：
```text
AddQual(parsetree, viewqual)
```
security-barrier view：
```text
new_rte->securityQuals = lcons(viewqual, new_rte->securityQuals)
```
INSERT 不把 view qual 加入主 WHERE。
新行是否能进入 view，由 `WITH CHECK OPTION` 决定。
`rewriteTargetView()` 会为 INSERT/UPDATE/MERGE 添加 `WCO_VIEW_CHECK`。
新的 WCO 加到 list 前端，使 inner view 的检查先于 outer view。
CASCADED check 还会影响嵌套 view。

### 9.8. MERGE 和 RETURNING
MERGE 不能在同一个 view 上混用部分 INSTEAD OF trigger 和部分 auto-update action。
源码要求要么提供完整 INSTEAD OF trigger 集合，要么删除已有 trigger。
RETURNING 也要被重写到 base relation。
否则 INSTEAD rule 接管原 query 时，必须提供可用 RETURNING，rewriter 才能保留原语句契约。

## 10. 权限、RLS 与 security barrier

### `checkAsUser`
`RTEPermissionInfo.checkAsUser` 是权限身份边界。
非 0 时按该 role 检查权限。
普通 view 默认表现为 owner 权限 gateway：
```text
caller must have privilege on view
view owner must have privilege on base relations
```
`security_invoker` 改变 base relation 权限检查身份，让 caller 也需要 base relation 权限。

### RLS 使用同一身份
`rowsecurity.c` 中 `get_row_security_policies()` 读取 RTE 的 perminfo。
简化逻辑：
```text
user_id = checkAsUser ? checkAsUser : GetUserId()
rls_status = check_enable_rls(relid, checkAsUser, false)
```
所以 RLS policy 展开和 view/rule 权限身份一致。
RLS 生成的 `securityQuals` 和 `withCheckOptions` 中如果包含子查询，还会调用：
```text
setRuleCheckAsUser(securityQuals, checkAsUser)
setRuleCheckAsUser(withCheckOptions, checkAsUser)
```
否则 policy 子查询可能用错权限身份。

### `hasRowSecurity`
RLS 不只是添加 qual。
它还设置 `hasRowSecurity`。
即使当前环境下没有实际添加 qual，`RLS_NONE_ENV` 也可能设置它。
原因是 role 或 GUC 改变后，RLS 行为可能变化。
plan cache 需要这个标记决定是否重新计划。

### security barrier
security-barrier view 的 qual 进入 `securityQuals`。
后续 planner 不能把用户提供的可泄漏 qual 移到 barrier qual 之前。
这不是性能保守。
它是信息泄漏边界。
本节只要求知道 rewrite 如何留下状态：
```text
rte->security_barrier = true
rte->securityQuals contains barrier/RLS quals
```
更细的 leakproof pushdown 在下一节继续。

## 11. 成本、资源与跨模块传播
rewrite 的主要成本是 backend-local CPU 和内存分配。
放大因素：
```text
view nesting depth
rule action count
range table size
targetlist width
expression tree size
CTE / SubLink 数量
RLS policy 数量
```
典型操作包括：
```text
copyObject()
query_tree_walker()
rangeTableEntry_used()
ReplaceVarsFromTargetList()
ChangeVarNodes()
AcquireRewriteLocks()
relcache lookup
RLS policy expansion
```
DML rules 还会放大后续阶段。
一条输入语句可能变成多条 query。
每条 query 都要 planning 和 execution。
这会传播到 command tag、command counter、CTE 语义和 `ON CONFLICT` 限制。
跨模块边界：
- parser 负责初始 `Query`、RTE、namespace、target relation 和权限 bitmap。
- rewrite 负责 rule/view/RLS/WCO 展开。
- planner 可以 pull up subquery，但不能删除权限和安全边界。
- executor 消费 `PlannedStmt` 中的权限、WCO、row security 和 target relation 契约。
- catalog/relcache/invalidation 保证 view/rule DDL 影响后续语句。
没有 shared memory 状态。
没有 WAL 写入主路径。
正确性主要靠 catalog、lock、copy、permission list 和 invalidation。

## 12. 观测与诊断入口

### 12.1. `debug_print_rewritten`
最直接入口：
```sql
SET client_min_messages = log;
SET debug_print_rewritten = on;
```
执行：
```sql
SELECT * FROM v WHERE id = 1;
```
日志会打印 rewritten parse tree。
重点找：
```text
RTE_SUBQUERY
subquery
security_barrier
securityQuals
rteperminfos
withCheckOptions
hasRowSecurity
```
粒度是单 query。
不要在生产长期打开。

### 12.2. `EXPLAIN (VERBOSE)`
`EXPLAIN` 看的是 planner 之后的 plan。
它能帮助判断 view subquery 是否被 pull up。
但它不能替代 rewrite dump。
如果最终直接扫描 base table，可能是：
```text
rewrite 展开 view
planner 又 pull up subquery
```
不是 rewrite 没发生。

### 12.3. catalog 入口
查看 view rule：
```sql
SELECT c.relname, r.rulename, r.ev_type, r.is_instead
FROM pg_class c
JOIN pg_rewrite r ON r.ev_class = c.oid
WHERE c.relname = 'v';
```
普通 view 应看到 `_RETURN`。
`pg_get_viewdef('v'::regclass)` 是把 stored query tree 反编译成人可读 SQL。
它不是运行时重新解析原始 CREATE VIEW 文本。

### 12.4. gdb 入口
推荐断点：
```text
QueryRewrite
RewriteQuery
fireRIRrules
ApplyRetrieveRule
rewriteTargetView
get_row_security_policies
setRuleCheckAsUser
```
建议观察：
```text
parsetree->rtable
rte->rtekind
rte->relid
rte->perminfoindex
parsetree->rteperminfos
parsetree->withCheckOptions
parsetree->hasRowSecurity
```
如果调试 view DML，还要看：
```text
parsetree->resultRelation
targetList resno
new base RTE index
```

### 12.5. 权限诊断
权限问题不要只看最终 plan 扫描哪张表。
要回到 rewrite 后的：
```text
RTEPermissionInfo.requiredPerms
RTEPermissionInfo.checkAsUser
view security_invoker option
```
普通 view 和 security invoker view 的差异，最终体现在 base relation perminfo 的检查身份。

### 12.6. WCO 诊断
`WITH CHECK OPTION` 失败不是 base table constraint 失败。
它来自 rewrite 生成的 `Query.withCheckOptions`。
打开 `debug_print_rewritten` 后，可以看到 `WCO_VIEW_CHECK`。

## 13. 常见误区

### 误区 1：view 是 planner 才展开
view 的 `_RETURN` rule 在 rewrite 阶段展开。
planner 只是可能继续 pull up subquery。

### 误区 2：`RTE_SUBQUERY` 没有 relation 权限
view 展开的 `RTE_SUBQUERY` 保留 `relid` 和 `perminfoindex`。
所以仍要检查 view relation 的权限和锁。

### 误区 3：rule 和 trigger 是同一层
rule 在 planner 前改写 `Query`。
trigger 在 executor 执行时触发。
INSTEAD OF trigger 是 view DML fallback 之一，但不是 rewrite rule。

### 误区 4：security barrier 是性能选项
security barrier 是信息泄漏边界。
它影响 qual 顺序和 leakproof pushdown。

### 误区 5：view update 失败都是权限问题
很多失败来自 auto-updatable 结构限制或 column-level updatability。
权限错误和结构错误要分开定位。

### 误区 6：`debug_print_rewritten` 能解释最终执行计划
它只显示 rewrite 后 query tree。
最终 path、join order、Subquery Scan 是否存在，要看 planner 和 `EXPLAIN`。

## 14. 课堂实验

### 实验一：普通 view 展开
目标：观察 `RTE_RELATION` view 变成保留 relation identity 的 `RTE_SUBQUERY`。
```sql
SET client_min_messages = log;
SET debug_print_rewritten = on;
CREATE TABLE t_rewrite(id int, val text);
CREATE VIEW v_rewrite AS
  SELECT id FROM t_rewrite WHERE id > 0;
SELECT * FROM v_rewrite WHERE id < 10;
```
源码对照：
```text
QueryRewrite()
fireRIRrules()
ApplyRetrieveRule()
```
观察点：
```text
RTE_SUBQUERY
subquery targetList / jointree
view RTE relid / perminfoindex
```

### 实验二：可更新 view 与 WCO
目标：观察 `rewriteTargetView()` 和 `WCO_VIEW_CHECK`。
```sql
CREATE TABLE t_wco(id int primary key, active bool not null);
CREATE VIEW v_wco AS
  SELECT id, active FROM t_wco
  WHERE active
  WITH CHECK OPTION;
INSERT INTO v_wco VALUES (1, true);
INSERT INTO v_wco VALUES (2, false);
```
第二条应失败。
源码对照：
```text
RewriteQuery()
rewriteTargetView()
view_query_is_auto_updatable()
Query.withCheckOptions
```

### 实验三：不可更新列
目标：区分 view 可更新和列可更新。
```sql
CREATE TABLE t_expr(id int primary key, name text);
CREATE VIEW v_expr AS
  SELECT id, lower(name) AS lname FROM t_expr;
UPDATE v_expr SET id = 10 WHERE id = 1;
UPDATE v_expr SET lname = 'a' WHERE id = 1;
```
表达式列 `lname` 不可更新。
源码对照：
```text
view_col_is_auto_updatable()
view_cols_are_auto_updatable()
error_view_not_updatable()
```

### 实验四：security invoker 权限
目标：观察 view owner 权限和 caller 权限边界。
概念步骤：
```sql
CREATE ROLE view_owner LOGIN;
CREATE ROLE view_caller LOGIN;
CREATE TABLE t_sec(id int);
ALTER TABLE t_sec OWNER TO view_owner;
CREATE VIEW v_sec AS SELECT id FROM t_sec;
GRANT SELECT ON v_sec TO view_caller;
REVOKE ALL ON t_sec FROM view_caller;
```
普通 view 下 caller 可能通过 view 读数据。
创建 security invoker view：
```sql
CREATE VIEW v_sec_invoker
WITH (security_invoker = true) AS
SELECT id FROM t_sec;
```
此时 caller 还需要 base table 权限。
源码对照：
```text
RelationHasSecurityInvoker()
RTEPermissionInfo.checkAsUser
setRuleCheckAsUser()
```

### 实验五：security barrier 状态
目标：观察 barrier qual 进入安全边界。
```sql
CREATE VIEW v_barrier
WITH (security_barrier = true) AS
SELECT * FROM t_sec WHERE id > 0;
SELECT * FROM v_barrier WHERE id = 1;
```
在 rewritten tree 中找：
```text
security_barrier = true
securityQuals
```
后续 planner 是否下推外层 qual，取决于 security level 和 leakproof 判断。

## 15. 讨论题
1. 为什么 view expansion 不能删除 view RTE 后只保留 base table RTE？
2. `RTE_SUBQUERY` 为什么还要保留 `relid`、`rellockmode`、`perminfoindex`？
3. `security_invoker` 改变的是 view 权限检查，还是 base relation 权限检查？
4. 为什么 rule 把带 CTE 的 query 改成多 query 时会破坏 CTE 语义？
5. auto-updatable view 为什么拒绝 DISTINCT、GROUP BY、LIMIT 和多 base relation？
6. `WITH CHECK OPTION` 为什么要在 rewrite 阶段进入 `withCheckOptions`？
7. 为什么 `EXPLAIN` 看不到 Subquery Scan 仍不能说明 view 没有被 rewrite？
8. RLS 的 `hasRowSecurity` 为什么即使没添加 qual 也可能为 true？

## 16. 本节小结
本节核心链路：
```text
QueryRewrite()
  -> RewriteQuery()
  -> fireRIRrules()
  -> ApplyRetrieveRule()
  -> rewriteTargetView()
```
`RewriteQuery()` 处理非 SELECT rules。
它能把一条 DML 改写成 0 条、1 条或多条 `Query`，并处理 command tag、CTE、RETURNING、ON CONFLICT 和 auto-updatable view 边界。
`fireRIRrules()` 处理 view `_RETURN` ON SELECT rule。
`ApplyRetrieveRule()` 把 view RTE 原位改成 `RTE_SUBQUERY`，但保留 `relid`、`relkind`、`rellockmode`、`perminfoindex`，从而保住 view 对象的锁和权限检查。
可更新 view 的核心是稳定行映射。
只有简单 view 能通过 `rewriteTargetView()` 改写到底层 base relation。
改写时还要重写 Var、targetlist resno、resultRelation、view qual、WCO 和 base relation perminfo。
权限边界由 `RTEPermissionInfo.requiredPerms`、`checkAsUser`、`security_invoker`、view owner 和 caller 共同决定。
RLS 和 security barrier 进一步把安全条件放入 `securityQuals` / `withCheckOptions`，并通过 `hasRowSecurity` 影响计划缓存失效。
诊断时要分清三层：
```text
debug_print_rewritten: rewrite 后 Query
EXPLAIN (VERBOSE): planner 后 Plan
runtime error / stats: executor 现象
```
本节可迁移规律：
```text
等价改写不能只保持结果等价；
还必须携带原对象身份、权限身份、安全 qual 顺序、可更新映射和失效边界。
```
下一节继续看 RLS、security barrier view 和 leakproof qual 如何限制 planner 搜索空间。
