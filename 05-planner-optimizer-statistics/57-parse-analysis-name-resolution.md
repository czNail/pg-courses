# PostgreSQL parse analysis 中 range table、namespace 与 name resolution
## 课程定位
前置知识：已经知道 PostgreSQL parser 会把 SQL 文本转成 raw parse tree，
也知道 planner 接收的是语义化后的 `Query`，不是原始 `SelectStmt`。
本节唯一主问题：
```text
parse analysis 如何把 FROM、列名、别名、namespace、scope 和 ParseState，
解析成 Query.rtable、jointree、targetList 中稳定可消费的 Var 引用？
```
核心矛盾：SQL 名字解析看起来像简单的“从表里找列”，
但 PostgreSQL 必须同时支持别名隐藏、JOIN 输出列、子查询外层引用、
LATERAL 左到右可见性、CTE 作用域、权限标记、错误位置和后续 planner 合约。
如果只维护一个全局 `name -> column` 映射，
scope、歧义和错误诊断都会变得不可维护。
学完后应能判断：
- `p_rtable` 中有某个 RTE，为什么当前位置仍可能不能引用它。
- `p_namespace` 如何决定 unqualified / qualified column lookup。
- `ParseNamespaceColumn` 如何把列名落成 `Var.varno` / `varattno`。
- `varlevelsup` 如何表达外层 query 引用。
- `LATERAL` 为什么需要 `p_lateral_active`、`p_lateral_only`、`p_lateral_ok`。
- ambiguous column、missing FROM-clause entry、错误 cursor position 来自哪条源码路径。
本课基于本地 `/home/nail/postgres` 源码，
分支 `master`，提交 `0e1f1ed157e9`。
## 1. 本节在总主线中的位置
05 目录多数课程从 `Query` 进入 planner，
解释 `PlannerInfo`、`RelOptInfo`、Path、cost 和 createplan。
本节倒回 planner 输入之前的一步：
```text
SQL text
  -> raw parse tree
  -> parse analysis
  -> Query.rtable / jointree / targetList / quals
  -> rewrite
  -> planner
```
planner 看到的 `RangeTblEntry`、`Var`、`TargetEntry`、
`SortGroupClause` 和 `Query.jointree`，
都来自 parse analysis 的语义压缩。
本节只讲 name resolution。
不展开 grammar 如何生成 `SelectStmt`，
不展开 rewrite 如何展开 view 或 RLS，
不展开 planner 如何搜索 Path。
读本节时始终分清三层对象：
```text
raw parse tree: 用户写了什么语法形状
ParseState: 当前解析位置能看见什么名字
Query: 下游 rewrite/planner 要消费的稳定语义树
```
最常见的误判是把 `p_rtable` 当成 scope。
`RangeTblEntry` 已经进入 `p_rtable`，
只说明当前 query level 语义上包含这个 relation-like 对象。
当前表达式能不能引用它，
要看它是否以合适的 `ParseNamespaceItem` 进入当前 `p_namespace`，
还要看 relation name、column name 和 LATERAL flags。
另一个常见误判是把列名字符串当成 planner 输入。
`SELECT a FROM t` 经过 parse analysis 后，
`a` 不再只是字符串。
它变成一个 `Var`：
```text
varno      -> Query.rtable 的 1-based 下标
varattno   -> 该 RTE 内的 attribute number
varlevelsup -> 是否引用外层 query level
location   -> 原 SQL token 位置
```
后续 rewrite 和 planner 主要消费这些结构化引用，
而不是重新按字符串做名字解析。
## 2. 核心矛盾与一句话运行模型
一句话运行模型：
```text
parse_analyze_*() 创建 ParseState；
transformSelectStmt() 先把 FROM 左到右转成 RTE、joinlist 和 namespace；
transformExpr() 遇到 ColumnRef 时在当前 namespace 和 parent ParseState 链上查找；
查找成功则生成 Var，并标记权限、outer-join nulling、RETURNING type 和 location；
最后 Query 保存 rtable、jointree、targetList、quals，临时 namespace 随 ParseState 消失。
```
这个模型的 tension 是：
```text
稳定 Query 语义合约
  vs
解析过程中不断变化的作用域、别名、可见性和错误上下文
```
PostgreSQL 用三类状态拆开这个 tension。
`RangeTblEntry` 回答：
```text
这个 query level 语义上引用了哪些 relation-like 对象？
```
`ParseNamespaceItem` 回答：
```text
在当前 parse 位置，这个 RTE 的关系名和列名是否可见？
可见时，列名应该构造什么 Var？
```
`ParseState` 回答：
```text
当前 query level、父 query level、FROM 处理进度、表达式上下文、
LATERAL 状态、CTE 可见性和错误位置是什么？
```
这层拆分支撑了几个 SQL 规则。
写了 `FROM t AS x` 后，
qualified reference 应该用 `x.a`，
不是 `t.a`。
一个没有 alias 的 JOIN，
可以保留输入表名用于 qualified reference，
但 unqualified column reference 应该看 JOIN 输出列的规则。
一个没有 alias 的 subquery，
可以暴露输出列给 unqualified lookup，
但不应该暴露自动生成的 relation name。
一个 LATERAL 右侧 item，
可以看见左侧允许暴露的 namespace item。
一个普通非 LATERAL 子查询，
不能仅仅因为左侧 RTE 已加入 `p_rtable` 就引用它。
因此本节的 mental model 是：
```text
p_rtable 是语义账本；
p_namespace 是当前位置查找窗口；
ParseNamespaceColumn 是列名到 Var 的映射规则；
Var 是把查找结果固化到 Query 表达式树里的引用。
```
## 3. 核心文件分工与阅读顺序
| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/parser/parse_node.h` | `ParseState`、`ParseNamespaceItem`、`ParseNamespaceColumn`。 |
| 2 | `src/backend/parser/analyze.c` | `parse_analyze_*()`、`transformTopLevelStmt()`、`transformSelectStmt()`。 |
| 3 | `src/backend/parser/parse_clause.c` | `transformFromClause()`、`transformFromClauseItem()`。 |
| 4 | `src/backend/parser/parse_relation.c` | `refnameNamespaceItem()`、`colNameToVar()`、`scanNSItemForColumn()`、错误提示。 |
| 5 | `src/backend/parser/parse_expr.c` | `transformColumnRef()` 解释 `A`、`A.B`、`A.B.C`、`A.*`。 |
| 6 | `src/backend/parser/parse_target.c` | `transformTargetList()`、星号展开、`TargetEntry`。 |
| 7 | `src/backend/parser/parse_node.c` | `make_parsestate()`、`free_parsestate()`、error position callback。 |
| 8 | `src/include/nodes/parsenodes.h` | `Query`、`RangeTblEntry`、`RTEKind`。 |
| 9 | `src/include/nodes/primnodes.h` | `Var`、`TargetEntry`。 |
推荐跟读的最小链路：
```text
transformSelectStmt()
  -> transformFromClause()
  -> transformTargetList()
  -> transformExpr()
  -> transformColumnRef()
  -> colNameToVar() / refnameNamespaceItem()
  -> scanNSItemForColumn()
  -> makeVar()
```
读源码时先找状态变化，
不要先背 `addRangeTableEntry*()` 的全部变体。
本课引用的本地 `master` 已包含 `RTE_GRAPH_TABLE`、
`GraphTableParseState`、`VarReturningType` 等新字段。
这些字段不改变本节核心模型：
```text
ParseState 维护过程状态；
namespace 决定名字可见性；
Query 保存稳定语义；
Var 固化列引用。
```
## 4. 关键数据结构与状态
### 4.1 `ParseState`
`ParseState` 是 parse analysis 的 backend-local 临时状态。
它不是 shared memory，
不跨 backend 可见，
也不会原样进入 planner。
顶层入口大致是：
```c
ParseState *pstate = make_parsestate(NULL);
pstate->p_sourcetext = sourceText;
pstate->p_queryEnv = queryEnv;
query = transformTopLevelStmt(pstate, parseTree);
free_parsestate(pstate);
```
`ParseState` 的字段可以按用途分成几组。
Query level 链：
| 字段 | 语义 |
| --- | --- |
| `parentParseState` | 子查询解析时指向外层 query level。 |
| `p_sourcetext` | 原 SQL 文本，用于错误 cursor position。 |
| `p_queryEnv` | 查询环境，例如外部命名 tuplestore。 |
长期 Query 语义的构建材料：
| 字段 | 语义 |
| --- | --- |
| `p_rtable` | 当前 query level 已创建的 `RangeTblEntry` 列表。 |
| `p_rteperminfos` | relation RTE 的权限检查信息。 |
| `p_joinexprs` | 与 `RTE_JOIN` 对应的 `JoinExpr`。 |
| `p_nullingrels` | outer join 可能把哪些 base rel 置 null。 |
| `p_joinlist` | 将成为 `Query.jointree.fromlist` 的 join items。 |
当前位置名字解析窗口：
| 字段 | 语义 |
| --- | --- |
| `p_namespace` | 当前可引用的 `ParseNamespaceItem` 列表。 |
| `p_lateral_active` | 当前是否正在解析 LATERAL 子表达式。 |
| `p_ctenamespace` | 当前可见的 CTE 名字；它们还不是 RTE。 |
| `p_future_ctes` | 尚未进入作用域的 CTE，用于错误提示。 |
target relation 和特殊 namespace：
| 字段 | 语义 |
| --- | --- |
| `p_target_relation` | DML target relation 的 relcache 句柄。 |
| `p_target_nsitem` | target relation 对应的 namespace item。 |
| `p_grouping_nsitem` | grouping step 的 namespace item。 |
表达式上下文和 hook：
| 字段 | 语义 |
| --- | --- |
| `p_expr_kind` | 当前表达式位置，用于上下文限制和错误消息。 |
| `p_next_resno` | 下一个 `TargetEntry.resno`。 |
| `p_windowdefs` | 收集窗口定义，稍后统一 transform。 |
| `p_pre_columnref_hook` | 扩展优先解释 `ColumnRef` 的 hook。 |
| `p_post_columnref_hook` | 标准解释后扩展再检查或替换的 hook。 |
| `p_paramref_hook` | 解析 `$n` 参数引用。 |
边界要点：
```text
p_rtable 最终进入 Query；
p_namespace 只是当前查找窗口；
p_expr_kind 只约束当前表达式；
p_sourcetext 只服务错误位置；
ParseState 释放后不能再依赖这些过程状态。
```
### 4.2 `RangeTblEntry`
`RangeTblEntry` 是 `Query.rtable` 的元素。
它表示一个 relation-like 语义对象，
包括普通 relation、FROM 子查询、显式 JOIN、FROM 函数、VALUES、CTE、
named tuplestore、graph table 和 grouping step。
源码注释里有一句关键提醒：
```text
range table 中的 relname 或 refname 不保证唯一；
按名字搜索 rtable 是坏主意。
```
RTE 是长期语义账本，
不是当前位置的 scope index。
本节需要记住的字段组合：
| 字段 | 语义 |
| --- | --- |
| `alias` | 用户写的 `AS` 别名。 |
| `eref` | 展开后的 reference name 和列名。 |
| `rtekind` | RTE 类型。 |
| `relid` / `relkind` | relation RTE 对应的 catalog 身份。 |
| `rellockmode` | 该 RTE 需要的 relation lock mode。 |
| `perminfoindex` | 指向 `RTEPermissionInfo`。 |
| `subquery` | 子查询 RTE 的 `Query`。 |
| `joinaliasvars` | JOIN 输出列如何展开成输入列或表达式。 |
| `lateral` | 用户是否写了 LATERAL。 |
| `inFromCl` | 是否来自用户 FROM clause。 |
| `securityQuals` | parser 输出通常为空，rewrite 阶段可能加入。 |
RTE 不直接回答当前名字是否可见。
这个问题由 namespace 回答。
### 4.3 `ParseNamespaceItem`
`ParseNamespaceItem` 是 name resolution 的中心结构。
它把一个 RTE 包成当前位置可见的名字集合。
| 字段 | 语义 |
| --- | --- |
| `p_names` | 对外暴露的 relation name 和 column names。 |
| `p_rte` | 底层 RTE。 |
| `p_rtindex` | RTE 在 `p_rtable` 中的 1-based 下标。 |
| `p_perminfo` | 权限信息。 |
| `p_nscolumns` | 每个列名如何构造 `Var`。 |
| `p_rel_visible` | relation name 是否可用于 qualified reference。 |
| `p_cols_visible` | column names 是否可用于 unqualified reference。 |
| `p_lateral_only` | 是否只对 LATERAL 子表达式可见。 |
| `p_lateral_ok` | 如果 lateral-only，被引用是否允许。 |
| `p_returning_type` | `RETURNING` 中 OLD / NEW 的特殊标记。 |
关键不变量：
```text
RTE 存在不等于 nsitem 可见；
nsitem 可见还要区分 relation-name 可见和 column-name 可见；
LATERAL 可见还要检查 lateral_ok。
```
JOIN 和 subquery 最能体现这个拆分。
没有 alias 的 JOIN 不隐藏成员表名，
所以成员表仍可用于 qualified reference。
但 unqualified column reference 应该按 JOIN 输出列规则解析，
不能直接让左右输入列同时参与并制造错误歧义。
没有 alias 的 subquery 可以暴露输出列，
但不暴露一个用户可引用的 relation name。
这就是为什么需要 `p_rel_visible` 和 `p_cols_visible` 两个 flag。
### 4.4 `ParseNamespaceColumn`
`ParseNamespaceColumn` 是列级映射。
它回答：
```text
如果 nsitem 的第 N 个列名被引用，应该生成哪个 Var？
```
核心字段：
| 字段 | 语义 |
| --- | --- |
| `p_varno` | 语义 referent 的 range table index。 |
| `p_varattno` | 语义 referent 的 attribute number。 |
| `p_vartype` / `p_vartypmod` / `p_varcollid` | 类型信息。 |
| `p_varreturningtype` | OLD / NEW returning 语义。 |
| `p_varnosyn` / `p_varattnosyn` | 语法上出现的 referent。 |
| `p_dontexpand` | star expansion 是否跳过。 |
`p_varno` 和 `p_varnosyn` 不一定相同。
语义 referent 可能是底层表列，
但用户语法上写的是 JOIN alias 或其他暴露名。
因此 parser 同时保存：
```text
semantic identity: p_varno / p_varattno
syntactic identity: p_varnosyn / p_varattnosyn
```
`Var` equality 主要看语义身份，
但 deparse、错误消息和 ruleutils 仍需要语法身份。
### 4.5 `Var`
`Var` 是名字解析落到表达式树里的稳定引用。
最小语义组合：
| 字段 | 语义 |
| --- | --- |
| `varno` | range table index，或后续阶段特殊 varno。 |
| `varattno` | attribute number；0 表示 whole-row Var。 |
| `vartype` / `vartypmod` / `varcollid` | 类型、typmod、collation。 |
| `varnullingrels` | 哪些 outer join 可能把该 Var 置 null。 |
| `varlevelsup` | 引用外层 query level 的层数。 |
| `varreturningtype` | RETURNING OLD / NEW。 |
| `varnosyn` / `varattnosyn` | 语法来源。 |
| `location` | raw parse tree token 位置。 |
`varlevelsup` 是 scope 的长期残留。
子查询引用外层列时，
`colNameToVar()` 会沿 `parentParseState` 向外找。
每上升一层，
`sublevels_up` 增加一。
最终 `makeVar()` 把它写进 `Var.varlevelsup`。
诊断 correlated subquery 时，
不能只看 `varno`；
必须同时看 `varlevelsup`。
### 4.6 `TargetEntry`
`transformTargetList()` 把 raw `ResTarget` 转成 `TargetEntry`。
本节只需要记住：`expr` 保存已 transform 的表达式，里面可能含 `Var`；
`resname` 是顶层 SELECT 非 resjunk 输出名；
`ressortgroupref` 连接 ORDER BY / GROUP BY / DISTINCT；
`resjunk` 表示工作列，不进入最终用户输出。
名字解析产生 `Var`，target list 决定它是用户输出、工作列，
还是 DML 目标表达式的一部分。
## 5. 主流程源码 walkthrough
### 5.1 顶层入口
入口在 `analyze.c`：
```text
parse_analyze_fixedparams()
parse_analyze_varparams()
parse_analyze_withcb()
```
三者主要区别是 `$n` 参数的解析策略。
name resolution 主流程相同：
```text
make_parsestate(NULL)
  -> 设置 p_sourcetext / p_queryEnv / 参数 hook
  -> transformTopLevelStmt()
  -> 可选 JumbleQuery()
  -> post_parse_analyze_hook()
  -> free_parsestate()
```
`post_parse_analyze_hook` 发生在 `Query` 形成后、`ParseState` 释放前。
扩展可以观察结果，
但不能假设 `ParseState` 会活到 planner。
### 5.2 `transformSelectStmt()` 的顺序
普通 SELECT 的关键时间线：
```text
process WITH clause
set locking clause and window defs
transformFromClause()
transformTargetList()
markTargetListOrigins()
transform WHERE
transform HAVING
transform ORDER BY
transform GROUP BY
transform DISTINCT
construct Query.jointree
```
`FROM` 必须先处理，
因为 target list、WHERE、HAVING、GROUP BY、ORDER BY 中的列名，
都要从当前 namespace 查找。
WITH clause 先独立处理，
因为 CTE 的名字可见性和 RTE 构造是两步：
```text
CTE name visible in p_ctenamespace
  -> FROM 引用 CTE 时创建 RTE_CTE
```
ORDER BY 在 GROUP BY / DISTINCT 之前处理，
因为后两者可能复用 target list 中已经生成或补充的条目。
### 5.3 `transformFromClause()` 建立查找窗口
`transformFromClause()` 对 FROM list 左到右处理：
```text
foreach FROM item:
  transformFromClauseItem()
  checkNameSpaceConflicts()
  setNamespaceLateralState(namespace, true, true)
  append transformed item to p_joinlist
  append namespace to p_namespace
after FROM list:
  setNamespaceLateralState(p_namespace, false, true)
```
左到右顺序是 LATERAL 的基础。
右侧 item 被 transform 时，
左侧 item 可能临时加入 namespace，
并带着 lateral-only 状态。
整个 FROM list 完成后，
这些 namespace item 才变成普通可见。
这解释了：
```sql
SELECT *
FROM t1, LATERAL (SELECT t1.a) s;
```
右侧 LATERAL 子查询能看见 `t1`。
而普通非 LATERAL 子查询不能仅凭 `t1` 已经在 `p_rtable` 中就引用它。
### 5.4 `transformFromClauseItem()` 创建 RTE 和 nsitem
该函数按 raw node 类型分派：
```text
RangeVar          -> table / CTE / tuplestore nsitem -> RangeTblRef
RangeSubselect    -> subquery RTE -> RangeTblRef
RangeFunction     -> function RTE -> RangeTblRef
RangeTableFunc    -> table function RTE
RangeGraphTable   -> graph table RTE
RangeTableSample  -> transform contained relation, attach tablesample
JoinExpr          -> recursively transform left and right, build join RTE
```
普通 relation 大致产生：
```text
p_rtable   += RTE_RELATION
p_namespace += ParseNamespaceItem
p_joinlist += RangeTblRef(rtindex)
```
显式 JOIN 会产生 `RTE_JOIN`。
隐式逗号 join 不产生 JOIN RTE；
它只是 `p_joinlist` 中多个 from items。
显式 JOIN 需要 RTE，
是因为 outer join、JOIN USING、join alias 和 join output columns
需要稳定的语义载体。
### 5.5 JOIN 右侧与 LATERAL
处理 `JoinExpr` 时，
源码先 transform 左子树，
再把左侧 namespace 临时并入 `pstate->p_namespace`，
然后 transform 右子树。
关键状态：
```text
lateral_ok = (jointype == JOIN_INNER || jointype == JOIN_LEFT)
setNamespaceLateralState(l_namespace, true, lateral_ok)
```
如果 join type 不允许 LATERAL 引用，
左侧名字仍可能被暴露，
但 `p_lateral_ok` 为 false。
这样错误可以表达为“这个位置不能引用”，
而不是误报“名字不存在”。
这是 SQL 标准和兼容性带来的 awkwardness。
### 5.6 `transformTargetList()` 处理 `*`
FROM 完成后，
`transformTargetList()` 处理 SELECT list。
它先识别星号展开：
```text
ColumnRef ending with A_Star
  -> ExpandColumnRefStar()
A_Indirection ending with A_Star
  -> ExpandIndirectionStar()
otherwise
  -> transformTargetEntry()
```
裸 `*` 只在 SELECT list 顶层由 grammar 接受，
并由 target list 变换路径处理。
`transformColumnRef()` 不负责普通裸 `*`。
这也是为什么 star expansion 要看 namespace，
但最终会生成多个 target entries，
不是生成一个名为 `*` 的列引用。
### 5.7 `transformColumnRef()` 解释列引用形状
表达式 transform 遇到 `ColumnRef`，
进入 `parse_expr.c` 的 `transformColumnRef()`。
它先检查 `p_expr_kind`。
大多数表达式位置允许列引用。
DEFAULT expression、partition bound expression、
`FOR PORTION OF` 等位置会报上下文错误。
随后 pre hook 有第一次机会：
```text
p_pre_columnref_hook
```
如果返回非 NULL，
标准解析跳过。
标准解析按 fields 长度解释：
| 形式 | 解析策略 |
| --- | --- |
| `A` | 先尝试 unqualified column，再尝试 unqualified relation whole-row。 |
| `A.B` | `A` 是 relation refname，`B` 是 column、`*` 或 whole-row 函数调用。 |
| `A.B.C` | `A` 是 schema，`B` 是 relation，`C` 是 column 或 `*`。 |
| `A.B.C.D` | `A` 必须是当前 database，随后 schema / relation / column。 |
标准解析后，
post hook 有第二次机会：
```text
p_post_columnref_hook
```
如果标准解析已有结果，
post hook 又返回结果，
会报 ambiguous column。
这保护了一个不变量：
```text
一个 ColumnRef 在一个 parse context 中只能落成一个语义节点。
```
### 5.8 unqualified column: `colNameToVar()`
`SELECT a FROM t` 中的 `a` 走：
```text
transformColumnRef()
  -> colNameToVar(pstate, "a", false, location)
```
`colNameToVar()` 从当前 query level 开始：
```text
while pstate != NULL:
  foreach nsitem in pstate->p_namespace:
    skip if !p_cols_visible
    skip if p_lateral_only && !p_lateral_active
    newresult = scanNSItemForColumn()
    if result already exists and newresult exists:
      ERROR ambiguous column
  if found or localonly:
    break
  pstate = parentParseState
  sublevels_up++
```
这里同时表达三件事。
未限定列名只看 `p_cols_visible`。
非 LATERAL 位置跳过 lateral-only nsitem。
当前层找不到时，
可以继续找外层 `parentParseState`，
并把层数写进最终 `Var.varlevelsup`。
### 5.9 qualified name: `refnameNamespaceItem()`
`SELECT x.a FROM t AS x` 中的 `x` 走：
```text
refnameNamespaceItem(pstate, schemaname, refname, location, &levels_up)
```
无 schema 的 refname 匹配可见 alias，
或无 alias relation item 的 unqualified relname。
带 schema 的 refname 更特殊：
```text
只能匹配没有 alias 的 relation item；
先把 schema.refname 转成 relation OID；
再按 relid 扫 namespace。
```
因此：
```sql
SELECT t.a FROM t AS x;
```
会失败。
写了 alias 后，
用户可见的 qualified refname 是 `x`，
不是真实表名 `t`。
`errorMissingRTE()` 会在这种场景给出 alias hint。
### 5.10 `scanNSItemForColumn()` 构造 `Var`
找到 nsitem 后，
列名还要在该 nsitem 内查找：
```text
scanNSItemForColumn()
  -> scanRTEForColumn()
  -> makeVar()
  -> markNullableIfNeeded()
  -> markVarForSelectPriv()
```
`scanRTEForColumn()` 扫 `eref->colnames`。
如果同一个 RTE 暴露的列名内部重复，
会报 ambiguous column。
普通用户列会从 `p_nscolumns[attnum - 1]` 取语义信息，
再构造 `Var`。
system column 使用 `SystemAttributeDefinition()` 的类型信息。
构造后会设置：
```text
var->location = location
var->varreturningtype = nsitem->p_returning_type
```
并执行两个重要副作用：
```text
markNullableIfNeeded(pstate, var)
markVarForSelectPriv(pstate, var)
```
因此名字解析不是纯查表。
它还把 outer join nulling 语义和列权限需求带进 Query。
手工绕过这条路径造 `Var`，
很容易漏掉权限、nullingrels、RETURNING 或 location 语义。
### 5.11 Query 组装
`transformSelectStmt()` 最终把 `ParseState` 中的结果组装到 `Query`：
```text
Query.rtable      <- pstate->p_rtable
Query.jointree    <- FromExpr(p_joinlist, qual)
Query.targetList  <- transformTargetList() 结果
Query.sortClause  <- transformSortClause() 结果
Query.groupClause <- transformGroupClause() 结果
Query.cteList     <- transformWithClause() 结果
```
`Query` 保存：
```text
RTE 列表、join tree、target entries、quals、
sort/group/distinct metadata、hasAggs/hasWindowFuncs/hasSubLinks 等 flags。
```
`Query` 不保存：
```text
p_namespace、p_future_ctes、当前 p_expr_kind、
p_lateral_active、parser error callback stack。
```
planner 不重新解析用户列名。
它消费已经落好的 `Var`、RTE、target list 和 jointree。
## 6. 生命周期 / ownership / cleanup
### 6.1 创建与持有
`ParseState` 由 `make_parsestate(parentParseState)` 创建。
顶层 parent 是 NULL。
子查询解析会创建新的 `ParseState`，
并把 `parentParseState` 指向外层。
RTE 由 `addRangeTableEntry*()` 系列路径创建。
Namespace item 通常随 RTE 创建或 JOIN 构造同步建立。
`Var` 由 `scanNSItemForColumn()`、whole-row reference 或少数特殊路径构造。
`TargetEntry` 由 `transformTargetEntry()` 构造。
这些对象都在当前 backend 的 memory context 中分配。
没有 shared ownership，
没有 refcount，
没有 pin。
本节 ownership 可以压缩成：
```text
MemoryContext 管内存生命周期；
ParseState 管解析过程可达性；
Query 管下游语义结果。
```
### 6.2 释放与 ERROR
`free_parsestate(pstate)` 释放 `ParseState` 自身持有的过程状态。
大量 node 内存依赖上层 memory context reset。
parse analysis 中途 `ereport(ERROR)` 时，
不会返回半成品 `Query`。
错误跳转到上层 statement / transaction 错误处理，
相关临时内存由 memory context reset 收尾。
本节没有 WAL、shared memory、ResourceOwner pin cleanup。
不过 relation RTE 创建期间可能打开 relation、读取 catalog、取得 lock。
这些由 relcache、syscache、lock manager 和事务资源管理负责，
不是 namespace list 自己管理。
### 6.3 失效边界
`ParseState` 没有 invalidation 协议，
因为它不跨 statement 生命周期存在。
parse analysis 产生的 `Query` 可能进入 prepared statement 或 plan cache。
这时依赖失效由 relcache/syscache invalidation、plan cache dependency 等机制处理。
要区分：
```text
namespace 不需要 invalidation，因为它是短命过程状态；
Query / plan cache 需要依赖跟踪，因为它们可能被复用。
```
## 7. 正确性机制层次
parse analysis 的正确性主要来自状态分层和时序边界，
不是 WAL 或 shared-memory 并发协议。
| 层次 | 机制 | 保证 |
| --- | --- | --- |
| 作用域链 | `parentParseState` | 子查询可按 SQL 规则引用外层。 |
| 当前可见性 | `p_namespace` | 当前 parse 位置能看见哪些 relation / column names。 |
| RTE 稳定性 | `p_rtable` 下标 | `Var.varno` 可长期指向 RTE。 |
| 列映射 | `ParseNamespaceColumn` | 从列名构造正确 semantic / syntactic `Var`。 |
| LATERAL | `p_lateral_active` / `p_lateral_only` / `p_lateral_ok` | 左到右可见性和 join type 限制。 |
| 错误定位 | `p_sourcetext` / `location` / `parser_errposition()` | 报错指向 SQL 文本位置。 |
| 权限准备 | `RTEPermissionInfo` / `markVarForSelectPriv()` | 后续权限检查知道读了哪些列。 |
| outer join | `p_nullingrels` / `markNullableIfNeeded()` | Var 带上可能被 outer join 置 null 的信息。 |
`p_rtable` 下标是 Query 内部引用不变量。
一旦 `Var.varno` 写入表达式树，
当前 query level 的 rtable 顺序就不能随意重排。
后续 planner 可以构造自己的 final rtable，
但那是另一个阶段的替换协议。
namespace 冲突不等于 rtable 冲突。
`checkNameSpaceConflicts()` 检查的是当前 namespace 层面的冲突，
不是整个 `p_rtable` 是否没有重复名字。
PostgreSQL 因历史兼容允许某些重复 alias 出现在复杂 JOIN 场景中，
并把真正的 ambiguous reference 推迟到实际引用时报告。
LATERAL 是三段式正确性：
```text
p_lateral_active: 当前是否在 LATERAL 子表达式里
p_lateral_only: 这个 nsitem 是否只对 LATERAL 可见
p_lateral_ok: 当前 join type 是否允许真正引用
```
错误位置也是语义工程的一部分。
`parser_errposition(pstate, location)` 把 raw node 的 byte location
转换成用户看到的 cursor position。
深层 transform 需要时会用：
```text
setup_parser_errposition_callback()
cancel_parser_errposition_callback()
```
如果新 parser 代码丢掉 location，
功能可能还能跑，
但错误诊断会退化。
权限标记也和名字解析绑定。
`scanNSItemForColumn()` 成功后调用 `markVarForSelectPriv()`，
把 relation / column read 需求记录到 RTE 权限信息。
所以 name resolution 的结果直接影响权限检查。
## 8. 错误路径 / 异常路径 / fallback
### 8.1 ambiguous column
复现：
```sql
CREATE TEMP TABLE t1(a int, b int);
CREATE TEMP TABLE t2(a int, c int);
SELECT a FROM t1, t2;
```
`a` 走 `colNameToVar()`。
当前 `p_namespace` 中 `t1` 和 `t2` 都 `p_cols_visible`，
且都能通过 `scanNSItemForColumn()` 返回 `Var`。
第二个 match 出现时，
`colNameToVar()` 报：
```text
column reference "a" is ambiguous
```
没有引用 `a` 时，
两个表都有同名列完全合法。
错误发生在 `ColumnRef` 必须落成唯一语义节点的时刻。
### 8.2 ambiguous alias
qualified relation name lookup 走 `scanNameSpaceForRefname()`。
它只看 `p_rel_visible` 的 namespace items。
如果多个可见 nsitem 的 aliasname 匹配同一个 refname，
会报：
```text
table reference "x" is ambiguous
```
这同样是实际引用时判断，
不是简单的 FROM list 全局去重。
### 8.3 missing FROM-clause entry 与 alias hint
复现：
```sql
CREATE TEMP TABLE t(a int);
SELECT t.a FROM t AS x;
```
`transformColumnRef()` 处理 `t.a` 时，
`refnameNamespaceItem()` 找不到 `t`。
`errorMissingRTE()` 会再扫描 range table，
发现确实有来自表 `t` 的 RTE，
但它的可见 alias 是 `x`。
于是错误会提示：
```text
Perhaps you meant to reference the table alias "x".
```
这条路径说明：
```text
真实解析只看 namespace；
错误诊断可以扫描 rtable 寻找更好的 hint；
诊断搜索不能改变语义。
```
### 8.4 LATERAL scope 错误
复现：
```sql
CREATE TEMP TABLE t1(a int);
SELECT * FROM t1, (SELECT t1.a) s;
```
右侧 subquery 没有写 LATERAL。
即使 `t1` 已经是左侧 FROM item，
其 namespace item 对普通右侧子查询也不可用。
错误可能提示该表存在，
但不能从这个 query part 引用，
并建议使用 LATERAL。
改成：
```sql
SELECT * FROM t1, LATERAL (SELECT t1.a) s;
```
`p_lateral_active` 让 lateral-only nsitem 可见，
解析成功。
### 8.5 illegal lateral reference
某些 join type 下，
左侧 namespace 暴露给右侧，
但 `p_lateral_ok` 为 false。
实际引用会被 `check_lateral_ref_ok()` 拒绝。
这不是多余复杂性。
如果直接隐藏名字，
错误会退化成“找不到表”，
而不是“这个位置不能引用”。
### 8.6 missing column 与 fuzzy hint
`errorMissingColumn()` 会搜索 range table 中可能提供近似列名的 RTE，
用于生成 hint。
这条路径可能看见真实解析路径看不见的对象。
因此诊断时要分清：
```text
lookup path: 只决定 SQL 语义
diagnostic path: 只改善错误消息
```
### 8.7 hook 冲突
扩展可以设置 column ref hooks。
pre hook 返回结果时，
标准解析跳过。
post hook 在标准解析后运行。
如果标准解析已有结果，
post hook 也返回结果，
会报 ambiguous column。
核心边界是：
```text
扩展能参与解释名字；
但同一个 ColumnRef 不能同时拥有两套冲突解释。
```
## 9. 成本、资源与跨模块传播
name resolution 通常不是执行期热点，
但会随 SQL 形状扩张。
主要成本变量：
| 变量 | 成本来源 |
| --- | --- |
| FROM item 数 | `p_namespace` 线性扫描更长。 |
| 暴露列数 | `scanRTEForColumn()` 扫 `eref->colnames`。 |
| query level 深度 | 外层引用沿 `parentParseState` 上升。 |
| SELECT / WHERE / GROUP / ORDER 表达式数 | `transformExpr()` 次数增加。 |
| JOIN USING / NATURAL JOIN 列数 | 需要构造 join output column 和 alias vars。 |
| 自动生成 SQL 宽度 | target list、sort/group resjunk、错误 hint 都可能放大。 |
PostgreSQL 没有为每个 scope 构造复杂全局 hash，
而是用 list + flags。
原因是 SQL scope 不是单纯键值查找。
它要表达：
- relation name visible 但 column names 不 visible。
- column names visible 但 relation name 不 visible。
- lateral-only 但当前不 active。
- visible 但 not lateral_ok。
- 当前 level 找不到再查 parent level。
- JOIN USING alias 只暴露一部分列。
- 错误时区分 alias mistake、scope mistake、真正缺表。
对普通 SQL，
parse analysis 每条语句一次，
SQL 名字数量通常远小于 executor 数据量。
因此 PostgreSQL 更偏向正确性和诊断质量，
而不是为少数超宽自动生成 SQL 引入复杂索引结构。
跨模块边界：
| 模块 | 边界 |
| --- | --- |
| relcache / syscache | parser 查 relation、列类型、system column；Query 不保存 relcache 指针。 |
| lock manager | RTE 记录 `rellockmode`；namespace 不负责并发 DDL 保护。 |
| rewrite | parser 输出 `securityQuals` 通常为空；view / RLS 后续加入。 |
| planner | planner 消费 `Query` 中的 RTE、Var、jointree，不重做字符串名字解析。 |
| plan cache | `ParseState` 不失效；缓存的 Query / Plan 依赖 invalidation。 |
## 10. 观测与诊断入口
### 10.1 错误消息是最直接入口
ambiguous column：
```sql
CREATE TEMP TABLE t1(a int);
CREATE TEMP TABLE t2(a int);
SELECT a FROM t1, t2;
```
源码链路：
```text
transformColumnRef()
  -> colNameToVar()
  -> scanNSItemForColumn()
  -> second match triggers ERRCODE_AMBIGUOUS_COLUMN
```
qualified reference 对照：
```sql
SELECT t1.a FROM t1, t2;
```
这次 `refnameNamespaceItem("t1")` 先定位唯一 nsitem，
再在该 nsitem 中查列。
### 10.2 `debug_print_parse` 看最终 Query
示例：
```sql
SET client_min_messages = log;
SET debug_print_parse = on;
SELECT t1.a FROM t1;
```
server log 中可以看到 parse analysis 后的 Query 树，
包括 RTE、TargetEntry、Var 等 node。
它适合验证：
- `Var.varno` / `varattno` / `varlevelsup`。
- target list 是否出现额外工作列。
- rtable 中有哪些 RTE。
它看不见：
- `p_namespace` 中间变化。
- LATERAL 处理中临时加入又移除的 namespace。
- `p_future_ctes`。
- error callback stack。
所以它验证“最终落成什么 Query”，
不解释完整“为什么这个位置可见或不可见”。
### 10.3 `EXPLAIN (VERBOSE)` 看后果
`EXPLAIN (VERBOSE)` 显示的是 planner 之后的 plan output，
不是 parser 内部状态。
它能间接验证名字已经解析成具体 relation column。
示例：
```sql
EXPLAIN (VERBOSE)
SELECT t1.a
FROM t1
JOIN t2 ON t1.a = t2.a;
```
看到 join qual 和 output 后，
可以回到 parser：
```text
qualified ColumnRef -> refnameNamespaceItem -> scanNSItemForColumn -> Var
```
注意：
`EXPLAIN` 不能显示 `p_namespace`。
scope 类问题仍要靠错误消息、debug print 或 gdb。
### 10.4 gdb 断点
推荐断点：
```gdb
break transformSelectStmt
break transformFromClause
break transformColumnRef
break colNameToVar
break refnameNamespaceItem
break scanNSItemForColumn
break errorMissingRTE
```
常用观察：
```gdb
print list_length(pstate->p_rtable)
print list_length(pstate->p_namespace)
print pstate->p_lateral_active
print ((ColumnRef *) cref)->fields
```
对 ambiguous column，
看 `colNameToVar()` 中 `result` 第一次被设置，
第二次 match 如何触发 ERROR。
对 alias hiding，
看 `refnameNamespaceItem()` 找不到真实表名，
再看 `errorMissingRTE()` 如何扫描 rtable 生成 hint。
### 10.5 观测粒度
| 入口 | 能看见 | 看不见 |
| --- | --- | --- |
| ERROR 消息 | 失败类型、cursor position、hint。 | namespace 完整列表。 |
| `debug_print_parse` | 最终 Query 树。 | 过程中的 ParseState。 |
| `EXPLAIN VERBOSE` | planner 后的输出表达式。 | parser scope 时间线。 |
| gdb | 完整 pstate 和调用链。 | 生产环境长期趋势。 |
| perf | parser CPU 是否显著。 | 语义错误原因。 |
诊断顺序：
```text
错误消息或 Query 输出
  -> ColumnRef 形状
  -> 当前 p_namespace
  -> refname / colname lookup 路径
  -> Var 或 ERROR
```
## 11. 常见误区
1. 把 `p_rtable` 当 scope。
   RTE 存在只说明语义账本中有条目；
   当前表达式能否引用要看 namespace flags。
2. 按名字搜索 rtable。
   源码注释明确提醒 rtable 中 refname 不保证唯一；
   正确入口是 namespace lookup。
3. 忽略 `p_rel_visible` 和 `p_cols_visible` 的差异。
   JOIN、subquery、OLD/NEW 都可能只打开其中一部分。
4. 把 LATERAL 当普通外层引用。
   LATERAL 是 FROM 内部左到右可见性；
   外层 query 引用靠 `parentParseState` 和 `varlevelsup`。
5. 只看 `Var.varno`。
   `varlevelsup > 0` 时，
   `varno` 指向外层 query level 的 rtable。
6. 绕过标准路径手工造 `Var`。
   可能漏掉权限标记、outer join nulling、RETURNING type 和 location。
7. 把错误提示路径当语义路径。
   `errorMissingColumn()` / `errorMissingRTE()` 可以扫描更多对象生成 hint，
   但不能让不可见对象变成可见。
8. 忽略 location。
   `location = -1` 可能让功能仍运行，
   但错误定位会退化。
## 12. 课堂实验
### 实验 1：未限定列名变成 ambiguous column
SQL：
```sql
CREATE TEMP TABLE t1(a int, b int);
CREATE TEMP TABLE t2(a int, c int);
SELECT a FROM t1, t2;
```
断点：
```gdb
break colNameToVar
break scanNSItemForColumn
```
观察：
```text
第一次 scanNSItemForColumn 返回 t1.a 的 Var；
第二次 scanNSItemForColumn 返回 t2.a 的 Var；
colNameToVar 发现 result 已存在，报 ambiguous column。
```
改写：
```sql
SELECT t1.a FROM t1, t2;
```
预期：
```text
transformColumnRef 走 refnameNamespaceItem("t1")；
scanNSItemForColumn 只扫描 t1 的列；
最终得到唯一 Var。
```
要画出的状态：
```text
ColumnRef("a")
  -> p_namespace: [t1, t2]
  -> two matches
  -> ERROR
ColumnRef("t1.a")
  -> refnameNamespaceItem("t1")
  -> one nsitem
  -> Var(varno=t1_rtindex, varattno=a_attnum)
```
### 实验 2：alias hiding 与 `errorMissingRTE()`
SQL：
```sql
CREATE TEMP TABLE t(a int);
SELECT t.a FROM t AS x;
```
断点：
```gdb
break refnameNamespaceItem
break errorMissingRTE
```
观察：
```text
namespace 中可见 aliasname 是 x；
refnameNamespaceItem("t") 找不到；
errorMissingRTE 扫描 rtable 找到原始 relation t；
发现 alias x 可见；
生成 alias hint。
```
改写：
```sql
SELECT x.a FROM t AS x;
```
确认 `refnameNamespaceItem("x")` 成功。
讨论：
```text
为什么写了 alias 后真实表名不能继续作为 qualified refname？
如果真实名和 alias 同时可见，self join 和 deparse 会遇到什么问题？
```
### 实验 3：LATERAL 的三段状态
SQL：
```sql
CREATE TEMP TABLE t1(a int);
SELECT * FROM t1, (SELECT t1.a) s;
```
观察 scope 错误。
再改写：
```sql
SELECT * FROM t1, LATERAL (SELECT t1.a) s;
```
断点：
```gdb
break transformFromClause
break transformFromClauseItem
break colNameToVar
```
观察：
```gdb
print pstate->p_lateral_active
```
并检查左侧 nsitem：
```text
p_lateral_only
p_lateral_ok
p_rel_visible
p_cols_visible
```
时间线：
```text
left FROM item transformed
  -> left namespace appended as lateral-only while RHS is transformed
  -> RHS LATERAL activates p_lateral_active
  -> ColumnRef resolves left Var
  -> after FROM list, namespace items become normally visible
```
## 13. 讨论题
1. 为什么 `p_rtable` 中允许名字不唯一，而 `p_namespace` 要维护冲突规则？
2. JOIN 不带 alias 时，为什么 relation name 和 unqualified column name 的可见性要分开？
3. `SELECT a FROM t1, t2` 报 ambiguous column 的准确源码触发点在哪里？
4. 为什么 `SELECT t.a FROM t AS x` 应该提示 alias，而不是直接说 relation 不存在？
5. LATERAL 为什么需要 `p_lateral_only` 和 `p_lateral_ok` 两个标志？
6. 子查询引用外层列时，`varno` 和 `varlevelsup` 如何共同定位语义对象？
7. `markVarForSelectPriv()` 为什么放在名字解析成功的位置？
8. `debug_print_parse` 能验证哪些结论？哪些结论必须靠 gdb？
## 14. 本节小结
本节主链路：
```text
parse_analyze_*()
  -> ParseState
  -> transformSelectStmt()
  -> transformFromClause()
  -> RTE + joinlist + namespace
  -> transformTargetList() / transformExpr()
  -> transformColumnRef()
  -> namespace lookup
  -> Var / TargetEntry / Query
```
核心状态边界：
```text
p_rtable: 当前 query level 的 RTE 语义账本
p_namespace: 当前 parse 位置的名字查找窗口
ParseNamespaceColumn: 列名到 Var 的构造规则
Var: 名字解析后的稳定语义引用
Query: 下游 rewrite/planner 的输入合约
```
`ParseState` 是短命 backend-local 状态，
由 `make_parsestate()` 创建，
由 `free_parsestate()` 和 memory context reset 收尾。
`Query` 及其 node 树返回给调用者，
继续进入 rewrite 和 planner。
本节没有 shared memory、WAL、pin、refcount。
正确性来自作用域链、namespace flags、RTE 下标稳定性、
列级映射、error position、权限标记和 outer-join nulling 标记。
错误路径不是附录。
ambiguous column、ambiguous alias、missing FROM-clause entry、
illegal LATERAL reference，
都是 namespace 规则在运行时暴露出来的可观察现象。
诊断时按这个顺序还原：
```text
错误消息或 Query 输出
  -> ColumnRef 形状
  -> 当前 p_namespace
  -> refname / colname lookup 路径
  -> Var 或 ERROR
```
可迁移规律：
```text
当一个模块既要输出稳定语义对象，又要在构造过程中遵守复杂作用域规则时，
不要把长期语义表和临时可见性窗口合并。
用短命 context 表达过程规则，
用稳定 node 表达下游合约。
```
这也是后续 planner 中 `PlannerInfo` 与 `PlannedStmt` 分层、
executor 中 `PlanState` 与 `TupleTableSlot` 分层的同类工程模式。
