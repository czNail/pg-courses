# PostgreSQL prepared statement 与 plan cache
## 课程定位
前置知识：已经理解 frontend/backend protocol 的 simple query 与 extended query 分派。 前置知识：已经理解
Portal 最终会把 `PlannedStmt` 交给 executor。 本节唯一主问题：
```text
prepared statement 如何保存 raw/analyzed query、generic/custom plan 和参数类型？
```
核心矛盾：prepared statement 希望把 parse/analyze/rewrite/plan 的成本跨执行复用，但 SQL
语义又依赖参数类型、参数值、search_path、RLS、role、schema invalidation 和执行时资源 owner。 学完后应能判断：一个慢 SQL
或错误到底来自 prepared statement 的参数类型固定、generic/custom plan 选择、cached query tree 失效，还是
portal/executor 执行期状态。 学完后还应能解释：为什么 PostgreSQL 不把 prepared statement 简化成“SQL 字符串 ->
PlannedStmt 指针”的 hash table。 本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交
`0e1f1ed157e9`。 本节只讲 prepared statement 与 plan cache 的保存、选择和使用边界。 完整 invalidation
传播会在后续 `61-cached-plan-invalidation.md` 中继续展开。 Portal 暂停、holdable cursor 和输出协议也留给后续课程。
## 1. 本节在总主线中的位置
04 目录前面已经从 executor 内部一路讲到协议层。 第 57 节说明 backend 如何在 `PostgresMain()` 消息循环中分派 simple
query 和 extended query。 待补第 59 节会说明 extended query 的 `Parse` / `Bind` / `Execute`
如何拆分生命周期。 本节接在这里，回答 `Parse` 和 SQL `PREPARE` 留下的核心问题：
```text
一个 prepared statement 到底保存了什么？
```
直觉上，prepared statement 好像只是缓存了一份 plan。 源码里不是这样。 它先保存能够重新分析的源信息。 它保存已经分析和 rewrite 后的
query tree。 它按需保存一个 generic plan。 它按执行参数临时构造 custom plan。
它还保存参数类型数组、结果描述、依赖对象、search_path 和 role/RLS 相关状态。 这些状态分布在两个层次：
```text
PreparedStatement
  -> backend-local name table entry

CachedPlanSource
  -> raw/analyzed/rewrite state
  -> param type contract
  -> dependency and invalidation state
  -> optional generic CachedPlan

CachedPlan
  -> planned statement list
  -> refcounted execution plan object
```
本节要把这三层拆开。 这样后面读 `PortalStart()` 和 `ExecutorStart()` 时，才不会把 plan cache 的问题误判成 executor
问题。 一个重要的上下文边界是：
```text
PostgreSQL 的 prepared statement 是 backend-local 的。
```
同一个 session 内可以复用。 不同 backend 之间不能共享指针。 这不是 shared memory plan cache。 也不是全局 plan
cache。 所以本节的核心资源问题不是跨 backend 并发访问。 核心资源问题是单 backend
内长生命周期对象、refcount、ResourceOwner、invalidation 和错误路径收尾。
## 2. 核心矛盾与一句话运行模型
一句话运行模型：
```text
PREPARE 或 Parse 先创建 CachedPlanSource，保存 raw tree、query string、param_types 和 rewrite 后 query_list；
EXECUTE 或 Bind 调用 GetCachedPlan()，按参数和历史成本选择复用 generic CachedPlan 或构造 custom CachedPlan；
随后 PortalDefineQuery() 把 refcounted CachedPlan 交给 portal，portal 再进入 executor。
```
这个模型的关键不是“缓存 plan”。 关键是把不同生命周期的状态分开。 `CachedPlanSource` 是语义来源。 `CachedPlan` 是某一次规划结果。
`PreparedStatement` 是名字到 `CachedPlanSource` 的 backend-local 索引。 `Portal` 是某次绑定和执行的容器。
核心矛盾可以压缩为：
```text
复用越多，parse/plan 成本越低
  vs
参数值、schema、search_path、role/RLS 和结果类型变化越容易让旧语义不再适用
```
PostgreSQL 没有选择“永远复用第一次计划”。 它也没有选择“每次都从 SQL 字符串重新开始”。 它把 reusable 的语义状态和 volatile
的执行状态拆成多层。 这样可以做到：
- 参数类型一旦确定，就作为 prepared statement 的契约保存。
- rewrite 后的 query tree 在未失效时可以复用。
- generic plan 可以 refcount 共享给多次 portal 执行。
- custom plan 可以用当前参数值重新规划。
- invalidation 只标记过期，下一次需要 plan 时再重建。
- ERROR 路径由 memory context、ResourceOwner 和 portal cleanup 共同兜底。
本节阅读源码时，要反复问同一个问题：
```text
这个状态是 query 语义，plan 结果，执行引用，还是名字索引？
```
如果把这些状态混在一起，就会得出错误结论。 例如： `CachedPlanSource->query_string` 不是可执行 plan。
`CachedPlanSource->gplan` 可能为空，也可能无效。 `CachedPlan->refcount` 只说明有人正在使用该 plan，不说明 plan
语义永远有效。 `PreparedStatement->stmt_name` 只在当前 backend 的 hash table 里有意义。 `param_types`
说明参数类型契约，不说明每次执行的参数值。
## 3. 核心文件分工与阅读顺序
| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/utils/plancache.h` | `CachedPlanSource`、`CachedPlan` 字段边界和公开 API。 |
| 2 | `src/backend/utils/cache/plancache.c` | `CreateCachedPlan()`、`CompleteCachedPlan()`、`GetCachedPlan()`、generic/custom 选择、refcount 和 invalidation 标记。 |
| 3 | `src/include/commands/prepare.h` | `PreparedStatement` 名字索引结构和 SQL PREPARE/EXECUTE 入口声明。 |
| 4 | `src/backend/commands/prepare.c` | SQL `PREPARE` / `EXECUTE` / `DEALLOCATE` 如何调用 plan cache。 |
| 5 | `src/backend/tcop/postgres.c` | extended protocol `Parse` / `Bind` / `Execute` 如何创建 statement、绑定 portal 并取 plan。 |
| 6 | `src/backend/tcop/pquery.c` | `PortalStart()` 如何把 portal 中的 `PlannedStmt` 交给 utility 或 executor。 |
| 7 | `src/backend/utils/mmgr/portalmem.c` | `PortalDefineQuery()` 如何接管 `CachedPlan` refcount 并在 portal cleanup 中释放。 |
| 8 | `src/backend/parser/analyze.c` | 参数类型推断和 parse analysis 的边界。 |
| 9 | `src/backend/rewrite/rewriteHandler.c` | query rewrite 后 `query_list` 可能从一条 SQL 变成 0、1 或多条 `Query`。 |
| 10 | `src/backend/optimizer/plan/planner.c` | `CachedPlanSource` 最终如何变成 `PlannedStmt`。 |
建议阅读顺序不是按文件名。 先读 `plancache.h`。 因为结构体字段已经暴露了 PostgreSQL 认为必须长期保存的状态。 再读 `plancache.c`
的创建、完成和取 plan 三段。 然后对照 SQL `PREPARE/EXECUTE` 和 extended protocol 两条入口。 最后再看 portal
cleanup。 不要一开始就从 `standard_planner()` 向下追。 本节的主题不是优化器如何生成 plan。 本节的主题是 prepared
statement 为什么要保存足够的信息，让优化器可以在以后某个时间点重新生成 plan。 本节源码核对基线：
```text
cd /home/nail/postgres
git rev-parse --abbrev-ref HEAD
git rev-parse --short=12 HEAD
```
本地输出：
```text
master
0e1f1ed157e9
```
## 4. 关键数据结构与状态
状态不是字段清单。 一个字段只有和创建时机、访问者、生命周期、错误路径一起看，才有诊断语义。 本节所有核心状态都是 backend-local。 它们不在 shared
memory 中。 其他 backend 不能直接读取当前 backend 的 `CachedPlanSource` 指针。 这也解释了为什么 PostgreSQL 的
prepared statement 不会自动跨连接复用。
### 4.1. `PreparedStatement`
`PreparedStatement` 定义在 `src/include/commands/prepare.h`。 它是 SQL 层和 extended protocol
层都可以使用的名字索引条目。 关键字段：
| 字段 | 语义 |
| --- | --- |
| `stmt_name[NAMEDATALEN]` | 当前 backend 内的 prepared statement 名称，dynahash key。 |
| `plansource` | 指向实际 plan cache 源对象。 |
| `from_sql` | 是否来自 SQL `PREPARE`，而不是 protocol `Parse`。 |
| `prepare_time` | 创建时间，用于 `pg_prepared_statements` 输出。 |
`PreparedStatement` 本身不是 plan。 它也不保存参数值。 它只是名字到 `CachedPlanSource` 的入口。 源码注释明确说
prepared query hash table 是 per-backend 的。 这意味着：
```text
session A PREPARE q AS ...
session B EXECUTE q ...
```
不会因为 SQL 文本相同而命中同一个 `CachedPlanSource`。 在 SQL `PREPARE` 路径中，空名字被禁止。 原因是空名字保留给
protocol-level unnamed statement。 这是一条非常重要的诊断边界。 SQL 层 named prepared statement 和
extended protocol unnamed prepared statement 使用同一批 plancache 结构，但名字生命周期不同。
### 4.2. `CachedPlanSource`
`CachedPlanSource` 是本节最重要的结构。 它不是 plan。 它是“能够反复生成 plan 的语义来源”。 核心字段可以分成七组。 第一组，源输入：
| 字段 | 语义 |
| --- | --- |
| `raw_parse_tree` | `raw_parser()` 输出或 SQL `PREPARE` 包装后的 `RawStmt`。 |
| `analyzed_parse_tree` | 某些调用者用已经分析好的 `Query` 创建 plansource 时使用。 |
| `query_string` | 原始 SQL 文本，错误上下文、重分析和 portal 展示都需要。 |
| `commandTag` | command completion 和 portal 默认 tag 的来源。 |
`raw_parse_tree` 和 `query_string` 是重建语义的基础。 如果只保存 `PlannedStmt`，一旦 schema 或
search_path 变化，就没有可靠方式重新分析。 第二组，参数类型契约：
| 字段 | 语义 |
| --- | --- |
| `param_types` | 参数类型 OID 数组。 |
| `num_params` | 参数数量。 |
| `parserSetup` | 替代的参数解析 hook。 |
| `parserSetupArg` | hook 参数。 |
prepared statement 的参数类型不是每次执行随便改变的。 SQL `PREPARE` 可以显式写类型。 如果没有显式写，parse analysis
会从上下文推断。 extended protocol `Parse` 可以传入部分或全部参数类型。 未指定类型也可能从表达式上下文推断。 一旦
`CompleteCachedPlan()` 完成，`param_types` 成为该 statement 的契约。 后续 `Bind` 或 `EXECUTE`
只是在这个契约下提供参数值。 第三组，结果与 cursor 约束：
| 字段 | 语义 |
| --- | --- |
| `cursor_options` | 影响规划的 cursor 标志，也可强制 generic/custom。 |
| `fixed_result` | 是否禁止 replan 后结果 tuple descriptor 改变。 |
| `resultDesc` | 结果类型描述，NULL 表示不返回 tuples。 |
SQL `EXECUTE` 要求 fixed result。 extended query 的 describe/bind 也需要结果描述稳定性。 这就是为什么
cached plan 不能只关心运行速度。 它还必须维护客户端协议能理解的结果形状。 第四组，query tree 与依赖：
| 字段 | 语义 |
| --- | --- |
| `query_list` | parse analysis 和 rewrite 后的 `Query` 列表。 |
| `relationOids` | query 依赖的 relation OID。 |
| `invalItems` | 函数、类型等依赖项。 |
| `search_path` | 上一次分析/规划使用的 search_path 匹配器。 |
| `query_context` | 保存 `query_list` 和依赖信息的 memory context。 |
`query_list` 不是 raw parse tree。 它已经过 parse analysis 和 rewrite。 规则系统可能让一条 SQL 变成多条
query。 也可能变成空列表。 所以 plan cache 必须保存 list，而不是假设一条 SQL 对应一个 `Query`。 第五组，role 与 RLS：
| 字段 | 语义 |
| --- | --- |
| `rewriteRoleId` | rewrite 时使用的 role。 |
| `rewriteRowSecurity` | rewrite 时的 row_security 设置。 |
| `dependsOnRLS` | query 是否依赖 RLS 环境。 |
如果 query 依赖 RLS，那么 role 或 row security 环境变化可能使 query tree 过期。 这不是执行器能修复的问题。 必须回到
parse/rewrite 语义层。 第六组，generic plan：
| 字段 | 语义 |
| --- | --- |
| `gplan` | 当前 generic `CachedPlan`，可能为 NULL 或无效。 |
`gplan` 是 optional。 存在 generic plan 不代表一定会用它。 `GetCachedPlan()` 每次仍要按参数、GUC、cursor
option、成本历史和有效性做选择。 第七组，生命周期与统计：
| 字段 | 语义 |
| --- | --- |
| `is_oneshot` | one-shot plansource，不做保存和 invalidation 检查。 |
| `is_complete` | 是否已经 `CompleteCachedPlan()`。 |
| `is_saved` | 是否已经进入长期 context 和 saved list。 |
| `is_valid` | 当前 `query_list` 是否有效。 |
| `generation` | 每次生成 plan 时递增，用于关联 plan/source 代际。 |
| `node` | saved plan list 链接。 |
| `generic_cost` | generic plan 估算成本，未知时为 -1。 |
| `total_custom_cost` | 历史 custom plan 成本累计。 |
| `num_custom_plans` | 已构造 custom plan 次数。 |
| `num_generic_plans` | 已使用 generic plan 次数。 |
这里最容易误读的是 `is_valid`。 它只说明 `query_list` 当前是否有效。 它不是“整个 prepared statement 一定可以执行”的证明。
重分析可能因为列被删除而报错。 重规划可能因为权限、RLS、依赖对象变化而走不同路径。
### 4.3. `CachedPlan`
`CachedPlan` 是真正的 planned statement 容器。 它从 `CachedPlanSource` 派生而来。 关键字段：
| 字段 | 语义 |
| --- | --- |
| `stmt_list` | `PlannedStmt` 列表。 |
| `is_oneshot` | 是否 one-shot。 |
| `is_saved` | 是否在长期 context 中。 |
| `is_valid` | plan 是否仍然有效。 |
| `planRoleId` | plan 创建时的 role。 |
| `dependsOnRole` | plan 是否依赖 role。 |
| `saved_xmin` | 如果需要，`TransactionXmin` 变化后必须 replan。 |
| `generation` | 对应 parent plansource 的代际。 |
| `refcount` | 当前活跃引用数。 |
| `context` | 保存 plan 和附属数据的 memory context。 |
`CachedPlan` 的 refcount 包括两类引用。 第一类是 `CachedPlanSource->gplan` 持有的引用。 第二类是正在执行的 portal
或调用者持有的引用。 这就产生一个重要不变量：
```text
plan invalidation 可以把 plan 标记为无效，
但正在执行的 CachedPlan 不能因为失效立刻释放。
```
refcount 管内存安全。 invalidation 管语义过期。 两者不能互相替代。
### 4.4. `ParamListInfo`
`ParamListInfo` 不在 `plancache.h` 中定义，但它是 generic/custom 选择的关键输入。 它表示本次执行绑定的参数值。
`GetCachedPlan()` 接收 `boundParams`。 如果需要 custom plan，planner 可以看见这些参数值。 如果使用 generic
plan，规划时不使用具体参数值。 所以 parameter 相关状态分两层：
```text
param_types
  -> prepared statement 的长期类型契约

ParamListInfo
  -> 某次 Bind/EXECUTE 的参数值
```
不要把这两者混成“prepared statement 的参数”。 类型决定 parse analysis 能不能成立。 值决定 custom plan 是否更合适。
### 4.5. named 与 unnamed prepared statement
PostgreSQL 有两种常见入口。 SQL 入口：
```sql
PREPARE q(int) AS SELECT * FROM t WHERE id = $1;
EXECUTE q(42);
DEALLOCATE q;
```
protocol 入口：
```text
Parse(statement_name, query, param_type_oids)
Bind(portal_name, statement_name, param_values)
Execute(portal_name, max_rows)
Sync
```
SQL `PREPARE` 禁止空名字。 protocol `Parse` 可以使用空 statement name。 空名字表示 unnamed prepared
statement。 新的 unnamed statement 会替换旧的 unnamed statement。 类似地，unnamed portal 也有特殊替换语义。
诊断 prepared statement 问题时必须先确认：
- 是 SQL `PREPARE` 还是 extended protocol `Parse`。
- 是 named statement 还是 unnamed statement。
- 是 named portal 还是 unnamed portal。
- plan 是 generic 还是 custom。
- 当前问题发生在 parse/analyze/rewrite、plan、bind、portal start，还是 executor run。
## 5. 主流程源码 walkthrough
本节用两条入口串同一个内部模型。 第一条是 SQL `PREPARE` / `EXECUTE`。 第二条是 extended protocol `Parse` /
`Bind` / `Execute`。 两条入口最终都围绕 `CachedPlanSource` 和 `GetCachedPlan()` 转。
### 5.1. SQL `PREPARE`：创建语义来源
SQL `PREPARE` 的入口是 `PrepareQuery()`。 主链路：
```text
PrepareQuery()
  -> wrap contained statement as RawStmt
  -> CreateCachedPlan(rawstmt, source_text, commandTag)
  -> transform explicit TypeName list into Oid param_types
  -> pg_analyze_and_rewrite_varparams(...)
  -> CompleteCachedPlan(...)
  -> StorePreparedStatement(name, plansource, from_sql=true)
```
第一步，`PrepareStmt` 里的实际 SQL 被包装成 `RawStmt`。 这一步看起来啰嗦，但很关键。 `CachedPlanSource` 需要保存 raw
parse tree。 后续 invalidation 后可能要重新分析。 第二步，`CreateCachedPlan()` 在当前 context 下创建一个独立
`CachedPlanSource` context。 它复制 raw parse tree 和 query string。 源码注释强调，它应该在 parse
analysis 前调用。 原因是 parser/analyzer 可能会修改 raw parse tree。 如果晚创建，就可能保存到已经被改写过的树。 第三步，SQL
`PREPARE` 的显式类型列表会被转成 OID 数组。 例如：
```sql
PREPARE q(int, text) AS SELECT $1, $2;
```
这里保存的不是字符串 `"int"` 和 `"text"`。 保存的是类型 OID。 第四步，`pg_analyze_and_rewrite_varparams()` 进行
parse analysis 和 rewrite。 它允许从上下文推断未知参数类型。 这一步输出 `query_list`。
第五步，`CompleteCachedPlan()` 把
`query_list`、`param_types`、`cursor_options`、`fixed_result` 写回 `CachedPlanSource`。 SQL
`PREPARE` 路径里 `fixed_result` 为 true。 第六步，`StorePreparedStatement()` 把 plansource 存入当前
backend 的 `prepared_queries` hash table。 从这一步开始，它不再只是临时对象。 它成为 session 内可通过名字找到的
prepared statement。
### 5.2. `CreateCachedPlan()` 与 `CompleteCachedPlan()` 为什么分两步
这两个函数分开不是 API 偶然。 它们分别对应两个时间点。 `CreateCachedPlan()` 的时间点：
```text
raw_parser 已经产出 raw tree
parse analysis 还没有开始或还没有破坏 raw tree
```
`CompleteCachedPlan()` 的时间点：
```text
parse analysis 和 rewrite 已经完成
参数类型、query_list 和结果描述可以确定
```
这样设计的好处是：
- raw tree 可以在被 analyzer 修改前复制。
- query tree 可以放在独立 `query_context` 中，失效时整体丢弃。
- source context 可以长期保存 query string、raw tree、param types。
- query context 可以随 invalidation 重建。
这一点直接回答本节主问题的一半：
```text
raw query 和 analyzed/rewrite query 不在同一个生命周期里。
```
raw query 是重新分析的源。 analyzed/rewrite query 是当前有效语义的缓存。
### 5.3. SQL `EXECUTE`：从名字到 portal
SQL `EXECUTE` 的入口是 `ExecuteQuery()`。 主链路：
```text
ExecuteQuery()
  -> FetchPreparedStatement(name, true)
  -> EvaluateParams(...) if num_params > 0
  -> CreateNewPortal()
  -> copy plansource->query_string into portal context
  -> GetCachedPlan(plansource, paramLI, NULL, NULL)
  -> PortalDefineQuery(portal, ..., cplan)
  -> PortalStart(portal, paramLI, eflags, GetActiveSnapshot())
  -> PortalRun(portal, count, ...)
  -> PortalDrop(portal, false)
```
第一步，`FetchPreparedStatement()` 在当前 backend hash table 中按名字查找。 查不到就是当前 session 没有该
statement。 它不会去其他 backend 找。 第二步，如果有参数，`EvaluateParams()` 会创建表达式执行所需的 `EState`。
注释里说不能过早删除 `EState`。 原因是参数可能是 pass-by-reference。 参数值的生命周期必须覆盖本次 query 执行。
第三步，`CreateNewPortal()` 创建一个内部 portal。 SQL `EXECUTE` 用的 portal 对用户不可见。
第四步，`GetCachedPlan()` 是本节最核心的分叉点。 它可能返回 generic plan。 它也可能构造 custom plan。
调用者不应该假定是哪一种。 第五步，`PortalDefineQuery()` 接管 `CachedPlan` 引用。 `prepare.c` 源码有一条非常强的注释：
```text
DO NOT add any logic that could possibly throw an error between
GetCachedPlan and PortalDefineQuery
```
原因是 `GetCachedPlan()` 已经增加了 plan refcount。 如果在 `PortalDefineQuery()` 之前抛错，而没有 owner
记录这次引用，就会泄漏 refcount。 这是本节最值得记住的 ownership 边界之一。 第六步，`PortalStart()` 把 portal
推进到可执行状态。 第七步，`PortalRun()` 进入 utility 或 executor 执行。 第八步，`PortalDrop()` 释放
portal，并最终释放它持有的 cached plan 引用。
### 5.4. extended protocol `Parse`：创建 protocol statement
extended query 的 `Parse` 消息入口是 `exec_parse_message()`。 它接收：
```text
statement name
query string
parameter type OID array
```
主链路可以压缩为：
```text
exec_parse_message(query_string, stmt_name, paramTypes, numParams)
  -> raw_parser(query_string)
  -> CreateCachedPlan(raw_parse_tree, query_string, commandTag)
  -> analyze and rewrite using supplied/inferred param types
  -> CompleteCachedPlan(psrc, query_list, ..., param_types, ...)
  -> if named: StorePreparedStatement(stmt_name, psrc, from_sql=false)
  -> if unnamed: SaveCachedPlan(psrc); unnamed_stmt_psrc = psrc
```
这条路径和 SQL `PREPARE` 的结构相似。 差别在输入来源和名字语义。 protocol `Parse` 可以创建 unnamed statement。 如果
`stmt_name` 是空字符串，后续新的 unnamed statement 会替换它。 SQL `PREPARE` 则禁止空名字。
`exec_parse_message()` 还禁止一个 prepared statement 中包含多条用户命令，避免协议层同时处理多个结果 tuple
descriptor。 另一个差别是参数类型来自 wire protocol。 客户端可以给出 0 表示未知，或给出明确 OID。 服务端会在 parse
analysis 中继续推断。 最终写入 `CachedPlanSource->param_types` 的，仍是确定后的类型契约。
### 5.5. extended protocol `Bind`：从 statement 到 portal
`Bind` 消息入口是 `exec_bind_message()`。 `Bind` 不重新 parse SQL。 它做的是：
```text
statement name
  -> PreparedStatement
  -> parameter values and formats
  -> ParamListInfo
  -> Portal
  -> GetCachedPlan()
  -> PortalDefineQuery()
  -> PortalStart()
```
主链路：
```text
exec_bind_message(input_message)
  -> read portal_name and stmt_name
  -> FetchPreparedStatement(stmt_name, true)
  -> parse input/output format codes
  -> convert supplied parameter bytes to Datum/null
  -> CreatePortal(portal_name, ...)
  -> GetCachedPlan(psrc, params, NULL, NULL)
  -> PortalDefineQuery(portal, ..., cplan)
  -> PortalStart(portal, params, 0, InvalidSnapshot)
```
`Bind` 会按 `param_types` 调用文本或二进制输入函数，把 wire bytes 转成 Datum/null，并把每个参数标记为
`PARAM_FLAG_CONST`，让 custom plan 能充分使用参数值。 它还会在 `GetCachedPlan()` 前把 query string 和 statement
name 复制到 portal context，因为这类分配可能 OOM，不能放在 refcount 已经增加之后。 `Bind` 的关键产物是 portal。
一个 statement 可以被多次 bind 成不同 portal。 每个 portal 可以有不同参数值。 所以 generic/custom 决策发生在
`Bind`/`EXECUTE` 这类“有参数值”的时刻，而不是 `Parse` 时刻。 这也是 prepared statement 的核心设计点：
```text
Parse 阶段固定参数类型和语义来源。
Bind 阶段提供参数值并决定本次 plan。
Execute 阶段推进 portal。
```
### 5.6. extended protocol `Execute`：运行已有 portal
`Execute` 消息入口是 `exec_execute_message()`。 它不再调用 `GetCachedPlan()`。 因为 `Bind` 阶段已经把
plan 放进 portal。 `Execute` 只是按 `max_rows` 推进 portal。 简化链路：
```text
exec_execute_message(portal_name, max_rows)
  -> GetPortalByName(portal_name)
  -> PortalRun(portal, max_rows, ...)
  -> maybe suspend portal
  -> maybe finish and drop unnamed portal later
```
这条边界非常适合排查问题。 如果 `EXPLAIN (ANALYZE)` 或 `pg_stat_activity` 显示 executor 正在慢，问题可能在 plan
选择之前已经决定。 如果慢点发生在 `Bind`，通常是参数转换、`GetCachedPlan()` revalidate/replan 或 portal start。
如果慢点发生在 `Execute`，通常已经进入 portal/executor。
### 5.7. `GetCachedPlan()`：核心分叉
`GetCachedPlan()` 的主流程：
```text
GetCachedPlan(plansource, boundParams, owner, queryEnv)
  -> assert plansource is complete
  -> RevalidateCachedQuery(plansource, queryEnv)
  -> choose_custom_plan(plansource, boundParams)
  -> if generic wanted and cached generic plan is valid
       use plansource->gplan
  -> else if generic wanted
       BuildCachedPlan(..., boundParams=NULL)
       attach as plansource->gplan
       compute generic_cost
       maybe re-check and switch to custom
  -> if custom wanted
       BuildCachedPlan(..., boundParams)
       accumulate custom cost history
  -> increment plan refcount
  -> optionally remember ref in ResourceOwner
  -> stamp PlannedStmt.planOrigin
  -> return CachedPlan
```
第一步，`RevalidateCachedQuery()` 确保 `query_list` 有效。 如果 search_path、RLS、role 或 dependency
invalidation 让 query tree 过期，它会重新分析和 rewrite。 这一步可能报错。 例如列被删除后，旧 SQL 无法重新分析。
第二步，`choose_custom_plan()` 决定是否尝试 custom plan。 这个函数不构造 plan。 它只是根据状态做选择。 第三步，如果选择
generic plan，`CheckCachedPlan()` 会检查现有 `gplan` 是否可用。 可用就复用。 不可用就 `BuildCachedPlan()`
生成一个新的 generic plan。 generic plan 构造时不传具体 `boundParams`。 第四步，第一次尝试 generic plan
后，源码会重新调用 `choose_custom_plan()`。 这是一个容易漏掉的细节。 原因是 `generic_cost` 之前可能未知。 生成 generic
plan 后才知道它的成本。 如果根据新成本发现 custom 仍明显更合适，就丢弃本次 generic 执行选择，重新生成 custom plan。 第五步，如果选择
custom plan，`BuildCachedPlan()` 会带着当前 `boundParams` 规划。 然后把该 plan 的成本计入
`total_custom_cost`。 第六步，函数返回前增加 `CachedPlan->refcount`。 如果调用者传入 `ResourceOwner`，还会记录到
owner 中。 SQL `EXECUTE` 和 extended `Bind` 在本地路径里通常把 plan 交给 portal，`owner` 参数为 NULL。
Portal 负责后续释放。
### 5.8. `choose_custom_plan()` 的选择规则
`choose_custom_plan()` 的规则有明确顺序。 简化为：
```text
oneshot
  -> always custom

no boundParams
  -> generic

statement does not require planning
  -> generic

plan_cache_mode = force_generic_plan
  -> generic

plan_cache_mode = force_custom_plan
  -> custom

cursor_options force generic/custom
  -> obey cursor option

num_custom_plans < 5
  -> custom

generic_cost < average custom cost
  -> generic

otherwise
  -> custom
```
这个顺序解释了很多线上现象。 前几次执行 prepared statement 时，经常看到 custom plan。 执行几次后，可能切到 generic plan。
如果 `plan_cache_mode` 被强制，行为会改变。 如果没有参数，custom plan 没有意义。 如果 statement 不需要重新规划，custom
plan 也没有意义。 `5` 是当前源码里的经验阈值。 它不是 SQL 标准语义。 它也不是绝对性能保证。 它只是 PostgreSQL
在规划成本和执行成本之间做的一条工程折中。
### 5.9. generic 与 custom 的真正差别
generic plan：
```text
使用参数类型和通用估计规划
不使用当前参数值
可以挂在 CachedPlanSource->gplan 下复用
refcount 可能同时包含 source 引用和 portal 引用
```
custom plan：
```text
使用当前 ParamListInfo 规划
可以根据参数值选择不同索引、join order 或分区路径
通常不挂在 CachedPlanSource->gplan 下长期复用
本次 portal 使用完即可释放
```
generic plan 省 planning cost。 custom plan 可能省 execution cost。 哪一个更快取决于 workload。
一个典型例子是 skewed data：
```sql
PREPARE q(int) AS SELECT * FROM orders WHERE customer_id = $1;
```
如果大多数 customer_id 很少出现，但某个 customer_id 占据大量行，custom plan 可能对不同参数值选择不同路径。 generic plan
只能选择一个平均意义上的路径。 如果查询本身很简单，planning cost 占比高，generic plan 可能更合适。 如果查询参数决定选择性、分区剪枝或 join
order，custom plan 可能更合适。
### 5.10. `PortalDefineQuery()`：refcount ownership 交接点
`PortalDefineQuery()` 在 `src/backend/utils/mmgr/portalmem.c`。 它的注释要求调用者已经完成
`GetCachedPlan()`，因此 `CachedPlan` refcount 已经增加。 它把这些信息放入 portal：
- source text。
- command tag。
- `PlannedStmt` list。
- `CachedPlan` 指针。
从这一步开始，portal 是 plan 引用的 owner。 如果 portal 正常结束，portal cleanup 会 release cached plan。
如果 portal 因 ERROR 被清理，portal cleanup 仍应释放引用。 所以 `GetCachedPlan()` 和
`PortalDefineQuery()` 之间的无 ERROR 区间非常重要。 这不是代码风格问题。 这是 refcount ownership 不能丢的问题。
### 5.11. `PlannedStmt.planOrigin`
`GetCachedPlan()` 返回前会给每个 `PlannedStmt` 标记 `planOrigin`。 它区分：
```text
PLAN_STMT_CACHE_GENERIC
PLAN_STMT_CACHE_CUSTOM
```
这个字段对诊断很有价值。 如果在 gdb 中停在 executor 或 planner/executor 边界，可以看当前 plan 来源。 SQL 层用户通常通过
`EXPLAIN EXECUTE` 的计划行为和 `pg_prepared_statements` 的计数间接判断。 不要把 `pg_stat_statements` 的
queryid 聚合误读成 plan origin。 那是不同层次的观测。
### 5.12. query tree revalidation 的时间点
plan cache 的 invalidation 通常不是“DDL 发生时立刻重建所有计划”。 更常见的是：
```text
DDL or syscache/relcache invalidation
  -> mark matching CachedPlanSource or CachedPlan invalid
  -> later GetCachedPlan()
     -> RevalidateCachedQuery()
     -> maybe rebuild query_list
     -> maybe rebuild generic plan
```
这就是 lazy revalidation。 它减少了无用工作。 但也带来诊断上的延迟： DDL 后第一个再次执行 prepared statement 的
session，可能承担重分析和重规划成本。 如果重分析失败，错误也在下一次执行时暴露。
## 6. 生命周期 / ownership / cleanup
本节必须把四类 lifetime 分开。
### 6.1. message lifetime
`postgres.c` 中每轮 frontend message 使用 `MessageContext`。 Parse/Bind
消息体里的字符串和参数字节不能直接长期保存。 需要跨消息存活的内容必须复制到更长生命周期的 context。 例如：
- statement name 进入 prepared statement hash table。
- query string 进入 `CachedPlanSource->context`。
- 参数值进入 portal 或执行所需 context。
- plan 引用交给 portal。
### 6.2. prepared statement lifetime
`StorePreparedStatement()` 把 `PreparedStatement` 放入当前 backend 的 hash table。 SQL named
prepared statement 通常存活到：
- `DEALLOCATE name`。
- `DEALLOCATE ALL`。
- session end。
- backend ERROR cleanup 之外的显式 drop 路径。
protocol unnamed statement 通常更短。 新的 unnamed `Parse` 会替换旧 unnamed statement。 session
结束时，整个 backend-local 状态消失。
### 6.3. plansource lifetime
`CachedPlanSource` 初始 context 是 caller context 的 child。 这样如果构造中途 ERROR，它会随 caller
context 清理。 完成后如果要长期保存，会调用 `SaveCachedPlan()` 或通过 `StorePreparedStatement()` 保存。
保存后，它通常进入 `CacheMemoryContext` 或长期 context。 `query_context` 可以独立释放。 这允许 invalidation
后丢弃 rewrite 后的 query tree，但保留 raw tree、query string 和 param type 契约。
### 6.4. cached plan lifetime
`CachedPlan` 有自己的 memory context。 generic plan 可以挂在 `CachedPlanSource->gplan` 下。
custom plan 通常只被本次执行引用。 `GetCachedPlan()` 返回前会增加 refcount。 之后必须有对应的 release。 有两种
release 机制。 第一，显式 `ReleaseCachedPlan(plan, owner)`。 第二，ResourceOwner callback 在 owner
release 阶段释放。 Portal 持有 plan 时，portal cleanup 负责释放。
### 6.5. ResourceOwner 与 refcount
`plancache.c` 注册了 ResourceOwner 描述符：
```text
name: plancache reference
release_phase: RESOURCE_RELEASE_AFTER_LOCKS
priority: RELEASE_PRIO_PLANCACHE_REFS
```
当 `GetCachedPlan()` 的 `owner` 不为 NULL 时，它会：
```text
ResourceOwnerEnlarge(owner)
plan->refcount++
ResourceOwnerRememberPlanCacheRef(owner, plan)
```
这样 ERROR 或 transaction abort 时，ResourceOwner 可以释放 plan ref。 但这只支持 saved
`CachedPlanSource`。 源码中如果 owner 非 NULL 但 plansource 未 saved，会报错。 这是为了避免 owner 管不住临时
memory context 里的对象。
### 6.6. Portal ownership
`PortalDefineQuery()` 是 portal 接管 plan 引用的边界。 在它之前，调用者必须确保没有 ERROR 泄漏 refcount。
在它之后，portal cleanup 会成为兜底。 这就是为什么 `prepare.c` 和 `postgres.c` 都有相似注释：
```text
do not throw error between GetCachedPlan and PortalDefineQuery
```
这个约束比普通代码局部性更重要。 它是 plan cache 正确性的资源边界。
### 6.7. session end cleanup
session 结束时，backend-local prepared statement hash table 和相关 contexts 都会随 backend 退出释放。
没有跨 backend 的 global plan state 要清理。 这减少了共享内存复杂度。 代价是每个 backend 都可能维护自己的 prepared
statement 和 generic plan。 连接数越多、每个连接 prepared statement 越多，内存占用越分散且越难从单个全局指标看清。
## 7. 正确性机制层次
plan cache 的正确性不是靠一个机制保证。 它是多层机制叠加。
| 层次 | 机制 | 保证什么 | 不保证什么 |
| --- | --- | --- | --- |
| 语义来源 | `raw_parse_tree` + `query_string` | 可以重新分析 | 当前 plan 一定最优 |
| 类型契约 | `param_types` + `num_params` | 后续参数值按固定类型解释 | 参数值选择性稳定 |
| query tree 有效性 | `is_valid` + `query_context` | rewrite 后 query tree 当前可用 | generic plan 当前可用 |
| plan 有效性 | `CachedPlan->is_valid` | planned statements 未被标记过期 | 正在执行时不会看到旧语义影响 |
| refcount | `CachedPlan->refcount` | plan 内存不会被提前释放 | plan 语义不会过期 |
| ResourceOwner | owner callback | ERROR/abort 能释放引用 | 所有调用路径都自动安全 |
| MemoryContext | source/query/plan/portal contexts | 批量释放内存 | 外部资源自动释放 |
| invalidation | relcache/syscache callbacks | 标记依赖对象变化 | 阻止 DDL 发生 |
| lock | parse/planner/executor locks | 保护对象访问期间的并发语义 | 缓存对象跨 backend 共享 |
### 7.1. refcount 不是 validity
这是最重要的不变量。 一个 plan 可以 refcount 大于 0 且已经被标记 invalid。 因为当前 executor 可能还在使用它。 这时不能释放内存。
但下一次 `GetCachedPlan()` 不应盲目复用它。 所以：
```text
refcount
  -> lifetime safety

is_valid / generation / invalidation
  -> semantic freshness
```
这两个维度必须分开理解。
### 7.2. invalidation 不是 lock
relcache/syscache invalidation 不阻塞 DDL。 它通知 backend：你的缓存语义可能过期。 backend 在安全点接收
invalidation，并在下一次使用缓存时 revalidate。 如果需要并发互斥，依赖其他 lock 层。 把 invalidation 当 lock，会误解很多
DDL 后 prepared statement 的行为。
### 7.3. parameter type 不是 parameter value
prepared statement 固定的是类型契约。 每次执行提供的是值。 generic/custom 选择看的是值是否值得进入 planner。
类型变化通常意味着不是同一个 prepared statement 语义。 值变化则是同一个 statement 的不同执行。
### 7.4. resultDesc 不是 SELECT list 文本
`resultDesc` 表示客户端可见 tuple descriptor。 `SELECT *` 的输出形状可能随表结构变化。 如果 `fixed_result` 为
true，replan 后结果形状变化可能报错。 这解释了为什么 prepared statement 对 DDL 敏感。 不是 executor 不会返回新列。
而是协议和 statement 契约要求结果形状稳定。
### 7.5. search_path 是语义的一部分
同一条 SQL 文本在不同 search_path 下可能解析到不同对象。 plan cache 保存 `search_path` matcher。 如果当前
search_path 不匹配，`RevalidateCachedQuery()` 可能需要重新分析。 所以“SQL 文本相同”不等于“语义相同”。
### 7.6. RLS 与 role 是语义的一部分
如果 query 依赖 RLS，rewrite 结果可能随 role 或 row security setting 变化。 plan cache 通过
`rewriteRoleId`、`rewriteRowSecurity` 和 `dependsOnRLS` 记录这一点。 这再次说明 prepared statement
不是简单文本缓存。
## 8. 错误路径 / 异常路径 / fallback
### 8.1. 构造中途 ERROR
`CreateCachedPlan()` 把 source context 建在 caller context 下面。 如果 parse analysis、rewrite
或参数类型解析中途 ERROR，caller context 会被清理。 尚未保存的 `CachedPlanSource` 不会留在长期 hash table 中。
这就是先构造、再完成、最后保存的安全性。 如果先把半成品放入 hash table，再慢慢填字段，ERROR 路径会复杂得多。
### 8.2. 参数类型不能推断
SQL 或 protocol 都可能留下未知参数。 如果上下文无法推断类型，parse analysis 会报错。 错误发生在 prepared statement
创建阶段，而不是 executor 阶段。 例如：
```sql
PREPARE q AS SELECT $1;
```
具体行为取决于上下文和版本。 诊断时要看错误是在 `PREPARE` / `Parse` 发生，还是在 `EXECUTE` / `Bind` 参数转换时发生。
### 8.3. 参数值转换失败
extended `Bind` 收到的是 wire protocol 参数。 它要按 `param_types` 把文本或二进制输入转换成 Datum。
如果客户端传入不能转换的值，错误发生在 `Bind`。 这时 statement 已经存在。 portal 可能尚未成功创建或尚未 start。 不要把这类错误误判成
plan cache invalidation。
### 8.4. `GetCachedPlan()` 到 `PortalDefineQuery()` 之间不能抛错
这是本节最具体的异常安全边界。 `GetCachedPlan()` 已经增加 refcount。 如果后续没有交给 portal，也没有 ResourceOwner
记录，ERROR 会泄漏引用。 所以源码在两处都留下警告。 工程上评审这段代码时，任何新增分支都要问：
```text
这里会不会 ereport(ERROR)？
这里会不会 palloc 失败？
这里会不会调用可能 ERROR 的函数？
```
如果答案是会，就必须调整 ownership。
### 8.5. generic plan 失效
relcache/syscache invalidation 可能把 `CachedPlanSource` 和 `gplan` 标记 invalid。
这不会立即释放正在执行的 plan。 下一次 `GetCachedPlan()` 会检查并重建。 如果重建成功，调用者几乎只看到一次额外 planning latency。
如果重建失败，错误在下一次执行时暴露。
### 8.6. generic plan 成本不理想的 fallback
当 `num_custom_plans >= 5` 后，系统可能尝试 generic plan。 但第一次生成 generic plan 后，如果发现
`generic_cost` 并不优于平均 custom cost，`GetCachedPlan()` 会改回 custom。 这是性能 fallback。 它不是
correctness fallback。 它避免因为第一次知道 generic cost 就立刻执行明显不划算的 generic plan。
### 8.7. unnamed statement 替换
protocol unnamed statement 是临时槽位。 新的 unnamed `Parse` 会覆盖旧 unnamed statement。
如果客户端错误地以为 unnamed statement 可长期保存，就会出现“前一次 Parse 被覆盖”的现象。 这是协议层生命周期问题。 不是 plan cache
随机丢失。
### 8.8. DEALLOCATE 与正在执行的 plan
`DEALLOCATE` 删除 prepared statement 名字入口并释放 plansource。 如果某个 portal 已经拿到了 `CachedPlan`
引用，refcount 会保护内存。 名字消失不等于正在执行的 plan 立即被释放。 这再次体现：
```text
name lifetime != plan execution lifetime
```
## 9. 成本、资源与跨模块传播
### 9.1. CPU 成本
prepared statement 节省的是部分 parse/analyze/rewrite/plan 成本。 但每次执行仍可能有：
- 参数值解析和转换。
- `RevalidateCachedQuery()` 检查。
- generic/custom 决策。
- custom planning。
- portal 创建和启动。
- executor 本身成本。
如果 custom plan 被频繁选择，planning CPU 仍会出现在每次执行中。 这时 prepared statement 不等于免 planning。
### 9.2. planning cost 与 relation 数
`cached_plan_cost()` 在估算 custom plan 成本时，会把 planner effort 加进去。 当前实现使用一个粗略模型：
```text
1000.0 * cpu_operator_cost * (nrelations + 1)
```
源码注释也承认 join planning 的真实曲线更复杂。 这说明 generic/custom 的成本比较是工程近似。 不要把它当成精确预测器。 当 relation
数、join 数、partition 数增加时，custom planning 成本可能明显放大。 但执行收益也可能放大。 最终结果 workload-dependent。
### 9.3. 内存成本
每个 saved `CachedPlanSource` 至少会占用：
- source context。
- query string。
- raw parse tree。
- param type array。
- result descriptor。
- query context 中的 rewritten query list。
- dependency lists。
- optional generic `CachedPlan` context。
如果每个连接都准备大量 statement，内存按连接数放大。 这不是共享 plan cache。 连接池模式会显著影响 prepared statement 内存形态。
transaction pooling 场景下，客户端以为复用了 session-level statement，实际上可能换了 backend。
### 9.4. invalidation 成本
plan cache invalidation 的直接成本包括：
- callback 遍历 saved plan list。
- 标记 matching plansource 或 plan invalid。
- 下一次执行时重新 analyze/rewrite/plan。
如果一个 backend 保存了大量 prepared statement，invalidation callback 的扫描成本会增加。 如果 DDL
频繁，延迟会扩散到下一次执行 prepared statement 的请求。 但 PostgreSQL 仍选择 lazy rebuild。 原因是很多
invalidated statement 可能再也不会执行。
### 9.5. lock 与 dependency 边界
parse/rewrite/planning 会记录 relation OID 和 inval items。 执行前还需要足够的 locks。
`GetCachedPlan()` 返回时，注释说明 plan valid 且已有足够 locks 可以开始执行。 这里的 lock 保护对象访问期间的并发语义。 它不把
cached plan 变成全局同步对象。
### 9.6. 与 parser/analyzer 的边界
parser 输出 raw tree。 analyzer 解析名称、类型、函数、操作符和参数。 prepared statement 必须保存足够信息让 analyzer
可以重跑。 参数类型契约就在这个边界形成。 如果类型推断失败，不会进入 planner。
### 9.7. 与 rewrite/RLS 的边界
rewrite 可能展开规则、视图和 RLS。 所以 cached query tree 依赖 role、RLS setting 和相关 catalog。 这就是
`CachedPlanSource` 有 `rewriteRoleId`、`rewriteRowSecurity` 和 `dependsOnRLS` 的原因。
### 9.8. 与 optimizer 的边界
optimizer 接收 rewritten query tree 和可选参数值。 generic plan 不看具体参数值。 custom plan 可以看
`ParamListInfo`。 优化器输出的是 `PlannedStmt`。 plan cache 不负责执行这个 plan。 它只负责在恰当时间把
`PlannedStmt` list 交给 portal。
### 9.9. 与 portal/executor 的边界
portal 接管 `CachedPlan` 引用。 `PortalStart()` 根据 `PlannedStmt` 类型决定进入 utility 还是
executor。 executor 看到的是已经规划好的 plan tree。 如果 plan 选择不理想，executor 只是在执行已给定的路径。 不要在
executor 节点里寻找 generic/custom 决策逻辑。
### 9.10. 与 pg_stat 和 EXPLAIN 的边界
`pg_prepared_statements` 可以看到 prepared statement 级别的信息。 `EXPLAIN EXECUTE` 可以观察当前执行选择的
plan 形态。 `pg_stat_statements` 可以聚合同类语句的执行统计。 这些视图不是同一层。 一个 queryid 下可能混合 generic 与
custom 的执行效果。 一个 prepared statement 的 generic/custom 计数也不等同于每次执行的耗时因果。
## 10. 观测与诊断入口
本节锚定的 runtime truth 是：
```text
同一个 prepared statement 前几次可能生成 custom plan，
执行几次后可能切换到 generic plan；
DDL、search_path 或 RLS 变化可能让下一次执行承担 revalidation/replan 或报错。
```
### 10.1. 能直接观测的状态
`pg_prepared_statements` 是最直接入口。 常见字段包括：
- statement name。
- statement text。
- prepare time。
- parameter types。
- from_sql。
- generic plans count。
- custom plans count。
示例：
```sql
SELECT name,
       parameter_types,
       from_sql,
       generic_plans,
       custom_plans
FROM pg_prepared_statements
ORDER BY name;
```
这些计数来自 `CachedPlanSource` 的统计字段。 它们能告诉你历史上用了多少次 generic/custom。 它们不能告诉你下一次一定用哪一种。
因为下一次还会受 invalidation、GUC、参数值和成本比较影响。
### 10.2. 用 `EXPLAIN EXECUTE` 看当前计划形态
示例：
```sql
PREPARE q(int) AS SELECT * FROM t WHERE id = $1;
EXPLAIN (ANALYZE, BUFFERS) EXECUTE q(1);
EXPLAIN (ANALYZE, BUFFERS) EXECUTE q(100000);
```
如果两个参数值选择不同路径，通常说明 custom plan 正在起作用。 如果多次后路径稳定且不随参数值变化，可能已经切到 generic plan。 但要注意：
`EXPLAIN EXECUTE` 本身也是一次执行入口。 它会影响 custom/generic 计数。 它不是无扰动观察。
### 10.3. 用 GUC 强制对照
`plan_cache_mode` 可以帮助构造对照实验。 示例：
```sql
SET plan_cache_mode = force_custom_plan;
EXPLAIN (ANALYZE, BUFFERS) EXECUTE q(1);

SET plan_cache_mode = force_generic_plan;
EXPLAIN (ANALYZE, BUFFERS) EXECUTE q(1);

RESET plan_cache_mode;
```
这个实验能分离：
- generic plan 的执行形态。
- custom plan 的执行形态。
- auto 模式的选择。
不要长期在线上随意强制。 它是诊断工具，不是普遍优化开关。
### 10.4. 用 gdb 观察关键字段
可设断点：
```text
break GetCachedPlan
break choose_custom_plan
break BuildCachedPlan
break PortalDefineQuery
```
典型观察：
```text
p plansource->num_params
p plansource->param_types[0]
p plansource->num_custom_plans
p plansource->num_generic_plans
p plansource->generic_cost
p customplan
p plan->refcount
```
如果停在 executor 边界，也可以看：
```text
p ((PlannedStmt *) linitial(plan->stmt_list))->planOrigin
```
这比从 SQL 层猜测更直接。
### 10.5. 用日志观察 replan 近似现象
PostgreSQL 没有把每次 `GetCachedPlan()` 的决策都直接作为普通日志输出。 可以用：
- `auto_explain` 观察执行计划形态。
- `log_min_duration_statement` 观察某次执行延迟。
- `debug_print_plan` 辅助确认计划输出。
- 源码临时加 `elog(LOG, ...)` 观察 `choose_custom_plan()`。
这些手段粒度不同。 `auto_explain` 观察 executor 执行后的 plan。 `debug_print_plan` 更接近规划输出。
临时日志能看到内部选择，但需要修改源码或调试环境。
### 10.6. 能推断但不直接观测的状态
普通 SQL 很难直接看到：
- `CachedPlanSource->is_valid`。
- `CachedPlan->refcount`。
- `query_context` 是否刚刚重建。
- `search_path` matcher 是否触发 reanalysis。
- `gplan` 是否存在但本次没有使用。
这些需要 gdb、源码日志或间接现象推断。 例如 DDL 后第一次执行变慢，可能是 revalidation/replan。 但也可能是 buffer cache、lock
wait 或统计变化。 必须结合 wait event、EXPLAIN、日志和源码断点判断。
### 10.7. 几乎不可见的状态
正在执行中的 portal 对 `CachedPlan` 的 refcount，在 SQL 视图中基本不可见。 你可以看到 session、query、wait
event。 但很难直接从 SQL 看到某个 plan refcount。 这类状态属于内核调试层。 不要让课程或排障手册假装 `pg_stat_*` 能覆盖所有
runtime reality。
## 11. 课堂实验
### 实验 1：观察 custom 到 generic 的切换
目标：把 `choose_custom_plan()` 的阈值和 `pg_prepared_statements` 计数连起来。 步骤：
```sql
DROP TABLE IF EXISTS pc_demo;
CREATE TABLE pc_demo(id int, payload text);
INSERT INTO pc_demo
SELECT g, repeat('x', 20)
FROM generate_series(1, 10000) g;
CREATE INDEX ON pc_demo(id);
ANALYZE pc_demo;

PREPARE q(int) AS SELECT * FROM pc_demo WHERE id = $1;

SELECT generic_plans, custom_plans
FROM pg_prepared_statements
WHERE name = 'q';

EXECUTE q(1);
EXECUTE q(2);
EXECUTE q(3);
EXECUTE q(4);
EXECUTE q(5);
EXECUTE q(6);

SELECT generic_plans, custom_plans
FROM pg_prepared_statements
WHERE name = 'q';
```
观察点：
- 前几次一般会累计 custom。
- 后续是否 generic 取决于成本比较。
- 表很小或查询太简单时，结果可能和预期不同。
回到源码：
```text
choose_custom_plan()
  -> num_custom_plans < 5
  -> generic_cost vs avg_custom_cost
```
### 实验 2：强制 generic/custom 对照 plan 形态
目标：确认 `plan_cache_mode` 影响 `choose_custom_plan()`。 步骤：
```sql
SET plan_cache_mode = force_custom_plan;
EXPLAIN (ANALYZE, BUFFERS) EXECUTE q(1);

SET plan_cache_mode = force_generic_plan;
EXPLAIN (ANALYZE, BUFFERS) EXECUTE q(1);

RESET plan_cache_mode;
```
观察点：
- force custom 时，planner 可以看当前参数。
- force generic 时，planner 不按当前参数值定制。
- 如果两者计划相同，不代表 GUC 没生效，可能是该 SQL 没有参数敏感性。
回到源码：
```text
choose_custom_plan()
  -> plan_cache_mode == PLAN_CACHE_MODE_FORCE_GENERIC_PLAN
  -> plan_cache_mode == PLAN_CACHE_MODE_FORCE_CUSTOM_PLAN
```
### 实验 3：DDL 后 revalidation
目标：观察 invalidation 延迟到下一次执行。 步骤：
```sql
DROP TABLE IF EXISTS pc_ddl;
CREATE TABLE pc_ddl(a int, b int);
INSERT INTO pc_ddl VALUES (1, 10);
PREPARE qddl AS SELECT * FROM pc_ddl;

EXECUTE qddl;
ALTER TABLE pc_ddl ADD COLUMN c int;
EXECUTE qddl;
```
观察点：
- 如果 result shape 改变且 statement 要求 fixed result，可能报错。
- 错误发生在下一次 `EXECUTE`。
- 这对应 lazy revalidation，而不是 DDL 立即重建所有 prepared statement。
回到源码：
```text
PlanCacheRelCallback()
  -> mark plansource/gplan invalid
GetCachedPlan()
  -> RevalidateCachedQuery()
  -> resultDesc compatibility check
```
### 实验 4：gdb 跟踪 refcount 交接
目标：确认 `GetCachedPlan()` 到 `PortalDefineQuery()` 的 ownership 边界。 断点：
```text
break GetCachedPlan
break PortalDefineQuery
break ReleaseCachedPlan
```
动作：
```sql
PREPARE q(int) AS SELECT * FROM pc_demo WHERE id = $1;
EXECUTE q(1);
```
观察：
```text
GetCachedPlan 返回后 plan->refcount 增加。
PortalDefineQuery 后 portal 持有该引用。
PortalDrop 或 cleanup 后 ReleaseCachedPlan 释放引用。
```
源码练习： 在测试分支临时加入断点或日志，不要把调试日志提交到课程仓库。
### 实验 5：extended protocol unnamed statement
目标：理解 unnamed statement 生命周期。 可以用支持 server-side prepare 的客户端驱动，或写一个最小 libpq 程序。 观察点：
- unnamed `Parse` 会替换前一个 unnamed statement。
- unnamed portal 也有类似短生命周期。
- SQL `PREPARE` 的 named statement 不等同于 protocol unnamed statement。
回到源码：
```text
exec_parse_message()
  -> named: StorePreparedStatement(stmt_name, psrc, false)
  -> unnamed: unnamed_stmt_psrc = psrc
exec_bind_message()
  -> CreatePortal(portal_name, ...)
```
## 12. 常见误区
### 误区 1：prepared statement 一定缓存最终 plan
不一定。 它一定保存 `CachedPlanSource`。 generic `CachedPlan` 是 optional。 custom plan 可以每次构造。
### 误区 2：有 prepared statement 就没有 planning cost
不对。 custom plan 仍然需要 planning。 invalidation 后也可能 re-analyze/rewrite/replan。 prepared
statement 节省的是可复用阶段的成本，不是取消所有规划。
### 误区 3：generic plan 总是更快
generic plan 省 planning cost。 但可能执行更慢。 尤其当参数值强烈影响选择性、分区剪枝或 join order 时。
### 误区 4：custom plan 总是更快
custom plan 可以更贴合参数。 但 planning 成本可能超过执行收益。 高 QPS 短查询尤其容易被 planning CPU 吃掉。
### 误区 5：`param_types` 保存的是本次参数值
不对。 `param_types` 是类型 OID 数组。 本次参数值在 `ParamListInfo` 里。
### 误区 6：invalidation 会阻塞 DDL
不对。 invalidation 是缓存过期通知。 并发语义依赖 lock。
### 误区 7：`pg_stat_statements` 能告诉我每次用了 generic 还是 custom
不直接。 `pg_stat_statements` 是 queryid 级聚合。 prepared statement 的 generic/custom 计数看
`pg_prepared_statements`。 单次计划形态看 `EXPLAIN EXECUTE`、gdb 或日志。
### 误区 8：不同连接会共享 prepared statement plan
不共享。 prepared statement hash table 是 per-backend。 连接池策略会显著改变 prepared statement
的有效性和内存成本。
## 13. 讨论题
1. 为什么 `CachedPlanSource` 要保存 raw parse tree，而不是只保存 rewritten query tree？
2. 为什么 `CreateCachedPlan()` 要在 parse analysis 之前调用？
3. `param_types` 和 `ParamListInfo` 分别回答什么问题？
4. 为什么 `GetCachedPlan()` 返回后必须尽快把 plan 引用交给 portal 或 ResourceOwner？
5. 一个 `CachedPlan` refcount 大于 0，但 `is_valid=false`，这是否矛盾？
6. 为什么 SQL `PREPARE` 禁止空 statement name，而 extended protocol 可以用 unnamed statement？
7. DDL 后 prepared statement 报错，为什么错误可能出现在下一次 `EXECUTE` 而不是 `ALTER TABLE` 时？
8. 如果某条 prepared statement 在生产上偶发变慢，你会如何区分 generic plan 问题、custom planning CPU、lock wait 和 executor I/O？
## 14. 本节小结
prepared statement 不是一个简单的 plan 指针缓存。 它是一组 backend-local、分层生命周期的状态。 核心链路是：
```text
PREPARE / Parse
  -> CreateCachedPlan()
  -> analyze/rewrite
  -> CompleteCachedPlan()
  -> StorePreparedStatement()

EXECUTE / Bind
  -> FetchPreparedStatement()
  -> ParamListInfo
  -> GetCachedPlan()
  -> PortalDefineQuery()
  -> PortalStart()
  -> PortalRun()
```
`CachedPlanSource` 保存 raw/analyzed/rewrite 语义、参数类型契约、结果描述、依赖对象、search_path、RLS/role 和
optional generic plan。 `CachedPlan` 保存 planned statement list，并用 refcount 保护执行期内存安全。
generic/custom 的选择发生在 `GetCachedPlan()`。 前几次有参数执行通常倾向 custom。 之后用 generic cost 与平均
custom cost 比较。 `plan_cache_mode` 和 cursor option 可以强制选择。 ownership 的关键边界是
`GetCachedPlan()` 到 `PortalDefineQuery()`。 这段之间不能新增可能 ERROR 的逻辑，否则会泄漏 plan refcount。
ResourceOwner 和 portal cleanup 处理引用释放。 MemoryContext 处理内存生命周期。 invalidation 处理语义过期。
这些机制各管一层，不能互相替代。 观测上，`pg_prepared_statements` 能看到 statement、参数类型和 generic/custom 计数。
`EXPLAIN EXECUTE` 能看到当前计划形态。 gdb 或源码日志才能直接看到
`choose_custom_plan()`、`CachedPlanSource->gplan` 和 `CachedPlan->refcount`。
本节可迁移的系统规律是：
```text
高性能缓存不能只缓存最终结果；
它必须缓存足够的语义来源、有效性证据和 ownership 信息，
才能在过期、参数变化和错误路径中安全重建或释放。
```
哪些判断仍然需要谨慎：
- generic/custom 哪个更快依赖 workload。
- custom planning 成本依赖 relation 数、join 形态、partition 数和硬件。
- revalidation 延迟依赖 DDL、invalidation 接收时机和下一次执行。
- SQL 视图只能观察一部分状态。
- 内核调试仍需要源码断点、日志或 profiler。
