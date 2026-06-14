# PostgreSQL ProcessUtility / DDL dispatch

## 课程定位

前置知识：已经读过 05 目录中 raw parser、parse analysis、type coercion、query rewrite 和 RLS / security barrier 课程，理解 `RawStmt`、`Query`、`PlannedStmt`、`Portal`、snapshot 与 command tag 的基本角色。 本节唯一主问题：
```text
ProcessUtility() 如何区分 DDL、transaction command、VACUUM、COPY、EXPLAIN 和其它 utility statement？
```
本节核心矛盾：PostgreSQL 希望所有“不是普通 plannable query”的命令都走一个统一入口，方便权限、事务边界、hook、event trigger、command tag、日志和 portal 执行协议统一处理；但 utility statement 的语义差异极大，事务控制不能触发 event trigger，DDL 需要 catalog / lock / invalidation，VACUUM 可能自己管理事务，COPY 有数据通道，EXPLAIN 又可能重新进入 planner。 学完后应能独立判断：一个 SQL 走 planner 还是走 utility wrapper；一个 utility statement 在 `standard_ProcessUtility()` 的 fast switch 中直接执行，还是进入 `ProcessUtilitySlow()` 的 DDL / event trigger 路径；以及 read-only、standby、parallel mode、transaction block、snapshot 和 `readOnlyTree` 这些边界分别在哪一层生效。 本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e9`。

## 1. 本节在总主线中的位置

前几节已经把 SQL 从文本推进到 parser、parse analysis 和 rewrite。 优化器课程通常接着追 `planner()`，但并不是所有 SQL 都进入优化器。 `CREATE TABLE`、`ALTER TABLE`、`COMMIT`、`VACUUM`、`COPY`、`EXPLAIN`、`SET`、`LISTEN` 这类命令会被分析成 `Query.commandType = CMD_UTILITY`。 它们在 `pg_plan_queries()` 中不会进入 `planner()`。 它们只会被包进一个 `PlannedStmt`，把真正的 utility node 放在 `PlannedStmt.utilityStmt`。 这就是本节的入口。 本节把 parser / analyzer / rewrite 和后续 DDL / event trigger 课程接起来：
```text
raw parser
  -> parse analysis
  -> Query(commandType = CMD_UTILITY, utilityStmt = ...)
  -> pg_plan_queries() 只包 PlannedStmt
  -> PortalRunUtility()
  -> ProcessUtility()
  -> standard_ProcessUtility()
  -> fast-path utility 或 ProcessUtilitySlow()
```
这条链路的关键不是“utility 没有计划”这么简单。 关键是 PostgreSQL 把差异很大的命令先统一成一个 portal 可执行对象，再在 `ProcessUtility` 层用 node tag 精确分派。 如果把 DDL、VACUUM、COPY、EXPLAIN 都理解成“绕过 planner 的语句”，诊断时会漏掉它们各自重新进入 planner、事务系统、event trigger 或 executor 的位置。 本节只回答 dispatch 问题。 下一节可以继续问：DDL 分派到具体 command 模块后，catalog update、dependency、invalidation、WAL 和事务 abort 如何保证一致性。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：
```text
ProcessUtility() 接收一个 commandType = CMD_UTILITY 的 PlannedStmt，
先允许 ProcessUtility_hook 接管，
再由 standard_ProcessUtility() 按 utilityStmt 的 nodeTag 做 switch：
事务命令、COPY、VACUUM、EXPLAIN 等直接进入专用执行函数；
有 event trigger 支持的 DDL 进入 ProcessUtilitySlow()；
DDL 内部生成的子命令再用 wrapper PlannedStmt 递归回 ProcessUtility()。
```
这里有三个分层判断。 第一层是 planner 边界。 `Query.commandType == CMD_UTILITY` 表示这个 `Query` 不是普通 `SELECT/INSERT/UPDATE/DELETE/MERGE` 的优化对象。 `pg_plan_queries()` 对它只创建 wrapper，不调用 `pg_plan_query()`。 第二层是 utility node tag。 `PlannedStmt.utilityStmt` 指向具体 parsetree，例如 `TransactionStmt`、`CopyStmt`、`VacuumStmt`、`ExplainStmt`、`CreateStmt`、`AlterTableStmt`。 `standard_ProcessUtility()` 的核心分类就是 `switch (nodeTag(parsetree))`。 第三层是 event trigger 支持边界。 事务命令、全局对象命令、backend-local 命令、VACUUM、COPY、EXPLAIN 通常在 fast switch 里直接执行。 大多数 DDL 进入 `ProcessUtilitySlow()`，因为那里包着 `EventTriggerBeginCompleteQuery()`、`EventTriggerDDLCommandStart()`、`EventTriggerCollectSimpleCommand()`、`EventTriggerSQLDrop()` 和 `EventTriggerDDLCommandEnd()`。 本节的系统 tension 可以压缩成：
```text
一个统一入口带来的协议一致性和扩展 hook
  vs
不同 utility statement 对事务、snapshot、锁、event trigger、数据通道和副作用的完全不同要求
```
PostgreSQL 的选择不是把每类 SQL 都做成独立顶层入口。 它把顶层协议统一到 `Portal` / `PlannedStmt` / `ProcessUtility`，再把语义差异保留在 node tag switch 和 command 模块中。 这是一种“统一壳，语义分派”的设计。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/tcop/postgres.c` | `pg_plan_queries()` 如何把 utility `Query` 包成 wrapper `PlannedStmt`；simple / extended query protocol 如何把 portal 推到执行阶段。 |
| 2 | `src/backend/tcop/pquery.c` | `PortalRunUtility()` 如何设置必要 snapshot、传入 `readOnlyTree`、调用 `ProcessUtility()`，以及 `PlannedStmtRequiresSnapshot()` 的例外列表。 |
| 3 | `src/include/tcop/utility.h` | `ProcessUtility_hook_type`、`ProcessUtility()`、`standard_ProcessUtility()`、`UtilityReturnsTuples()`、`UtilityContainsQuery()`、`CreateCommandTag()` 的公共边界。 |
| 4 | `src/backend/tcop/utility.c` | 本节主文件：read-only 分类、hook 调用、utility switch、DDL slow path、event trigger cleanup、command tag / log level 辅助函数。 |
| 5 | `src/backend/commands/explain.c` | `ExplainQuery()` 和 `ExplainOneUtility()` 如何让 EXPLAIN 自己成为 utility，又可能解释内部 query / utility。 |
| 6 | `src/backend/commands/copy.c` | `DoCopy()` 如何处理 COPY FROM / TO、权限、RLS fallback 和数据传输通道。 |
| 7 | `src/backend/commands/vacuum.c` | `ExecVacuum()` 作为 manual VACUUM / ANALYZE 的 utility 入口，如何包装参数后进入 `vacuum()`。 |
| 8 | `src/backend/commands/dbcommands.c` | `createdb()`、`DropDatabase()` 等全局对象命令为什么在 fast switch 中执行且禁止事务块。 |
| 9 | `src/backend/commands/tablecmds.c` | `AlterTable()`、`DefineRelation()` 等 DDL heavy path 的目标模块，展示 `ProcessUtilitySlow()` 只是分派层。 |
推荐阅读顺序不是从 `utility.c` 顶部一路背 switch。 更好的顺序是：
```text
pg_plan_queries()
  -> PortalRunUtility()
  -> ProcessUtility()
  -> standard_ProcessUtility()
  -> 一个 fast-path 分支
  -> 一个 ProcessUtilitySlow() DDL 分支
  -> 一个递归 subcommand 分支
```
这样能先建立“谁调用谁、谁拥有 tree、谁负责 snapshot”的时间线，再看命令差异。 `utility.c` 后半段还有 `CreateCommandTag()`、`GetCommandLogLevel()`、`UtilityReturnsTuples()`、`UtilityTupleDescriptor()`、`UtilityContainsQuery()`。 它们不是主分派入口，但它们和 dispatch 使用同一套 node tag 语义。 诊断日志、completion tag、Describe、EXPLAIN 内部 query 时经常会碰到这些辅助函数。

## 4. 关键数据结构与状态

### 4.1. `Query.commandType` 与 `Query.utilityStmt`
parse analysis 后，普通 DML / SELECT 会形成可规划的 `Query`。 utility statement 也会被放进 `Query`，但 `commandType` 是 `CMD_UTILITY`。 关键边界是：
```text
Query.commandType == CMD_UTILITY
  -> Query.utilityStmt 指向具体 utility node
  -> planner 不解释这个 node 的内部语义
```
这并不表示这个 utility 完全不包含 query。 `EXPLAIN SELECT ...`、`CREATE TABLE AS SELECT ...`、`DECLARE CURSOR FOR SELECT ...`、`COPY (SELECT ...) TO ...` 都可能在 utility node 内部保存一个 query 或 raw query。 区别是外层 dispatch 由 `ProcessUtility()` 管，内部 query 是否重新 rewrite / plan 由具体 command 模块决定。
### 4.2. `PlannedStmt.utilityStmt`
`PlannedStmt` 对普通 query 保存 plan tree。 对 utility statement，它是一个 wrapper。 `pg_plan_queries()` 中的关键行为是：
```c
stmt = makeNode(PlannedStmt);
stmt->commandType = CMD_UTILITY;
stmt->canSetTag = query->canSetTag;
stmt->utilityStmt = query->utilityStmt;
stmt->stmt_location = query->stmt_location;
stmt->stmt_len = query->stmt_len;
```
这个 wrapper 让 portal 执行层不用关心“这是不是 planner 产物”。 `PortalRunMulti()` 只需要看：
```text
pstmt->utilityStmt == NULL
  -> ProcessQuery()
pstmt->utilityStmt != NULL
  -> PortalRunUtility()
```
不要把 `PlannedStmt` 这个名字误解成“一定已经规划过”。 对 utility 来说，它更多是执行协议的统一容器。
### 4.3. `nodeTag(parsetree)`
`standard_ProcessUtility()` 的分类不是按 SQL 字符串，也不是按 command tag。 它拿到：
```c
parsetree = pstmt->utilityStmt;
switch (nodeTag(parsetree))
```
所以真正的运行时分类单位是节点类型。 例如：
| node tag | 典型 SQL | 分派含义 |
| --- | --- | --- |
| `T_TransactionStmt` | `BEGIN`、`COMMIT`、`SAVEPOINT` | 进入事务命令子 switch，不走 event trigger。 |
| `T_CopyStmt` | `COPY ...` | 调用 `DoCopy()`，处理数据通道和 processed rows。 |
| `T_VacuumStmt` | `VACUUM`、`ANALYZE` | 调用 `ExecVacuum()`，再进入 vacuum 子系统。 |
| `T_ExplainStmt` | `EXPLAIN ...` | 调用 `ExplainQuery()`，可能重写和规划内部 query。 |
| `T_CreateStmt` | `CREATE TABLE` | 默认进入 `ProcessUtilitySlow()`，带 event trigger。 |
| `T_AlterTableStmt` | `ALTER TABLE` | 进入 slow path，先算锁级别，再调用 `AlterTable()`。 |
command tag 主要服务日志、completion、客户端协议和 `PreventCommandIfReadOnly()` 的错误信息。 dispatch 本身依赖 node tag。
### 4.4. `ProcessUtilityContext`
`ProcessUtilityContext` 表示这个 utility 的来源：
| context | 语义 |
| --- | --- |
| `PROCESS_UTILITY_TOPLEVEL` | 顶层客户端命令。 |
| `PROCESS_UTILITY_QUERY` | 非顶层客户端命令或 portal 中的普通上下文。 |
| `PROCESS_UTILITY_QUERY_NONATOMIC` | 允许非 atomic 行为的查询上下文，例如过程调用相关场景。 |
| `PROCESS_UTILITY_SUBCOMMAND` | DDL 内部生成的子命令。 |
`standard_ProcessUtility()` 用它计算：
```c
bool isTopLevel = (context == PROCESS_UTILITY_TOPLEVEL);
bool isAtomicContext =
  (!(context == PROCESS_UTILITY_TOPLEVEL ||
     context == PROCESS_UTILITY_QUERY_NONATOMIC) ||
   IsTransactionBlock());
```
`isTopLevel` 决定 `PreventInTransactionBlock()`、`RequireTransactionBlock()`、`WarnNoTransactionBlock()` 等边界。 `isAtomicContext` 影响 `DO` / `CALL` 这类可以执行内部事务控制的命令。
### 4.5. `readOnlyTree`
`readOnlyTree` 是 tree ownership 边界，不是事务 read-only。 `PortalRunUtility()` 调用：
```c
ProcessUtility(pstmt,
               portal->sourceText,
               (portal->cplan != NULL),
               ...);
```
如果 portal 绑定了 cached plan，`portal->cplan != NULL`，说明这个 tree 可能被 plan cache 持有。 `standard_ProcessUtility()` 在 `readOnlyTree` 为 true 时先 `copyObject(pstmt)`。 这样做的原因很具体：有些 utility 执行会做 parse transformation 或改写 node tree。 如果直接修改 plan cache 中的 tree，后续复用会看到被污染的状态。 所以 `readOnlyTree` 保护的是 node tree ownership。 它不表示“这个命令只读”。
### 4.6. `ParseState`
`standard_ProcessUtility()` 为每条 utility 创建短生命周期的 `ParseState`：
```c
pstate = make_parsestate(NULL);
pstate->p_sourcetext = queryString;
pstate->p_queryEnv = queryEnv;
```
很多 command 模块还需要 parse state 做名称解析、权限检查、表达式 transform、错误位置和 query environment。 例如 `DoCopy()` 处理 `COPY FROM WHERE` 时会 transform expression。 `ExplainQuery()` 会用 `pstate` 处理 EXPLAIN options 和内部 query hook。 执行结束后 `free_parsestate(pstate)`。 这说明 `ProcessUtility()` 不是单纯的函数指针跳转层。 它还给 command 模块建立了最小 parse / execution 上下文。
### 4.7. `QueryCompletion` 与 `canSetTag`
`QueryCompletion *qc` 是 command completion status 的输出位置。 `COPY` 分支会把 processed rows 写入：
```c
DoCopy(..., &processed);
SetQueryCompletion(qc, CMDTAG_COPY, processed);
```
事务命令中，`COMMIT` 或 `PREPARE TRANSACTION` 如果实际失败并回滚，也会把 completion tag 改成 `ROLLBACK`。 `PlannedStmt.canSetTag` 决定该 statement 是否能成为客户端看到的最终 tag。 rewrite 可能生成不能 set tag 的附加语句。 utility dispatch 不能只执行命令，还要维护客户端协议可见的 completion 语义。
### 4.8. event trigger 上下文
`ProcessUtilitySlow()` 的关键状态是 event trigger complete-query 上下文。 它只在 `context != PROCESS_UTILITY_SUBCOMMAND` 时执行完整事件：
```text
EventTriggerBeginCompleteQuery()
  -> EventTriggerDDLCommandStart()
  -> command-specific DDL
  -> EventTriggerCollectSimpleCommand()
  -> EventTriggerSQLDrop()
  -> EventTriggerDDLCommandEnd()
  -> EventTriggerEndCompleteQuery()
```
子命令仍可能收集到外层上下文，但不重新开始完整 query 事件。 这就是为什么 slow path 不是“慢一点的 switch”。 它维护 DDL 可观察事件的生命周期。

## 5. 主流程源码 walkthrough

### 5.1. 从 analyzed `Query` 到 wrapper `PlannedStmt`
主流程从 `postgres.c` 的 `pg_plan_queries()` 开始。 对普通 query：
```text
Query
  -> pg_plan_query()
  -> planner()
  -> PlannedStmt(planTree != NULL, utilityStmt = NULL)
```
对 utility query：
```text
Query(commandType = CMD_UTILITY)
  -> makeNode(PlannedStmt)
  -> commandType = CMD_UTILITY
  -> utilityStmt = query->utilityStmt
  -> 不调用 planner()
```
这一步回答了第一个边界问题：ProcessUtility 分派不是从 SQL 文本直接开始的。 parser / analyzer 已经给出了具体 node 类型。 `ProcessUtility()` 不需要重新识别 `CREATE`、`VACUUM` 或 `COPY` 关键字。 它只读 `utilityStmt` 的 node tag。
### 5.2. Portal 执行层选择 `ProcessQuery()` 还是 `ProcessUtility()`
`pquery.c` 的 `PortalRunMulti()` 负责执行 portal 中的 statement list。 判断逻辑很窄：
```text
if (pstmt->utilityStmt == NULL)
  ProcessQuery()
else
  PortalRunUtility()
```
普通 query 必须有 active snapshot。 utility command 默认由 `PortalRunUtility()` 自己决定需不需要 snapshot。 这避免了 `BEGIN`、`SET TRANSACTION` 这类命令在 transaction-snapshot 模式下过早冻结 snapshot。
### 5.3. `PlannedStmtRequiresSnapshot()`
`PortalRunUtility()` 先调用 `PlannedStmtRequiresSnapshot(pstmt)`。 源码默认认为大多数 utility 需要 snapshot。 例外列表包括：
```text
TransactionStmt
LockStmt
VariableSetStmt
VariableShowStmt
ConstraintsSetStmt
FetchStmt
ListenStmt
NotifyStmt
UnlistenStmt
CheckPointStmt
WaitStmt
```
注释强调：transaction control、LOCK、SET 必须不能设置 snapshot。 否则 `START TRANSACTION ISOLATION LEVEL REPEATABLE READ` 这类命令可能在还没设置好事务模式时就创建事务 snapshot。 这个判断发生在 dispatch 之前。 所以某个 utility 是否有 snapshot，不由具体 command 函数临时猜测，而是在 portal utility 执行入口统一处理。
### 5.4. `ProcessUtility()` 只做 hook 入口
`ProcessUtility()` 本体很短。 它检查：
```text
pstmt 是 PlannedStmt
pstmt->commandType == CMD_UTILITY
queryString != NULL
qc 未设置 commandTag
```
然后：
```text
if (ProcessUtility_hook)
  hook(...)
else
  standard_ProcessUtility(...)
```
hook 的约定是扩展通常会再调用 `standard_ProcessUtility()`。 如果扩展吞掉命令，就必须自己承担权限、事件、completion、snapshot 后果。 这也是 utility hook 风险比 planner hook 更高的原因。 planner hook 主要改变搜索空间或 plan。 utility hook 可以拦截 DDL、事务命令、COPY、VACUUM 和配置变更。
### 5.5. `standard_ProcessUtility()` 的入口边界
`standard_ProcessUtility()` 首先处理四件事。 第一，`check_stack_depth()`。 utility dispatch 可以递归，例如 `CREATE TABLE LIKE` 展开额外命令，`ALTER TABLE` 生成内部 `CREATE INDEX`。 第二，必要时复制 tree：
```text
if (readOnlyTree)
  pstmt = copyObject(pstmt)
```
第三，做 read-only / standby / parallel mode 分类：
```text
readonly_flags = ClassifyUtilityCommandAsReadOnly(parsetree)
if not strictly read-only and XactReadOnly or parallel:
  PreventCommandIfReadOnly()
  PreventCommandIfParallelMode()
  PreventCommandDuringRecovery()
```
第四，创建 `ParseState`。 然后才进入 `switch (nodeTag(parsetree))`。 这个顺序重要。 它保证大多数禁用场景在进入具体 command 模块前就被统一挡住。 但也保留了 command 模块的二次检查，例如 `COPY FROM` 目标是临时表还是普通表，需要打开 relation 后才能知道。
### 5.6. transaction command fast path
`T_TransactionStmt` 是第一类显式 fast path。 它再按 `TransactionStmt.kind` 分派：
```text
BEGIN / START
  -> BeginTransactionBlock()
  -> SetPGVariable(transaction options)

COMMIT
  -> EndTransactionBlock(chain)

ROLLBACK
  -> UserAbortTransactionBlock(chain)

SAVEPOINT
  -> RequireTransactionBlock()
  -> DefineSavepoint()

ROLLBACK TO
  -> RequireTransactionBlock()
  -> RollbackToSavepoint()

COMMIT PREPARED / ROLLBACK PREPARED
  -> PreventInTransactionBlock()
  -> FinishPreparedTransaction()
```
事务命令不进入 `ProcessUtilitySlow()`。 源码注释说得很直接：event trigger 代码可能需要刷新 event trigger cache，而 START TRANSACTION 这类命令还不一定处在有效事务上下文。 所以“是不是 DDL”不是唯一分界。 `ProcessUtilitySlow()` 的边界首先是 event trigger 是否适用，事务控制明确不适用。
### 5.7. backend-local 和 portal 命令 fast path
`DECLARE CURSOR`、`FETCH`、`CLOSE`、`PREPARE`、`EXECUTE`、`DEALLOCATE`、`SET`、`SHOW`、`DISCARD`、`LISTEN`、`NOTIFY`、`UNLISTEN` 都在 fast switch 中直接处理。 这些命令的共同点不是“都只读”。 它们的共同点是：大多不需要 DDL event trigger 的完整事件协议。 例如：
```text
DECLARE CURSOR
  -> PerformCursorOpen()

FETCH
  -> PerformPortalFetch()

PREPARE
  -> PrepareQuery()

EXECUTE
  -> ExecuteQuery()

SET
  -> ExecSetVariableStmt()

SHOW
  -> GetPGVariable()
```
有些会改 backend-local 状态。 有些会触发异步通知状态。 但它们不是 schema-level DDL。 因此它们避开 `ProcessUtilitySlow()`。
### 5.8. COPY fast path
`T_CopyStmt` 的分支很短：
```text
DoCopy(pstate, CopyStmt, stmt_location, stmt_len, &processed)
SetQueryCompletion(qc, CMDTAG_COPY, processed)
```
复杂性在 `commands/copy.c`。 `DoCopy()` 会区分：
```text
COPY table FROM ...
COPY table TO ...
COPY (query) TO ...
```
`COPY FROM` 写入目标表。 `COPY TO` 从 relation 或 query 输出。 如果 COPY 目标涉及 RLS，源码里有一个重要边界：
```text
COPY TO + RLS enabled
  -> 构造 SELECT
  -> 走 query-based COPY

COPY FROM + RLS enabled
  -> ERROR，提示使用 INSERT
```
所以 COPY 是 fast path，不代表它一定简单。 它只是没有 DDL event trigger。 它内部可能重新构造 query，让正常 query processing 处理 RLS policy。
### 5.9. VACUUM fast path
`T_VacuumStmt` 直接调用：
```text
ExecVacuum(pstate, VacuumStmt, isTopLevel)
```
`ExecVacuum()` 是 manual `VACUUM` 和 `ANALYZE` 的准备 wrapper。 它解析 options，填充 `VacuumParams`，建立 buffer access strategy，然后进入 `vacuum()`。 VACUUM 在 read-only 分类里很特别。 源码注释说它写 WAL，所以不是严格只读。 但它不会改变会影响 `pg_dump` 输出的数据库语义，因此允许在 read-only transaction 中运行。 同时它不支持 parallel worker 中执行。 这说明 `ClassifyUtilityCommandAsReadOnly()` 的“read-only”不是单一布尔值。 它区分：
```text
strictly read-only
OK in read-only transaction
OK in recovery
OK in parallel mode
not read-only
```
VACUUM 的行为只能在这些 flag 组合里解释。
### 5.10. EXPLAIN fast path
`T_ExplainStmt` 调用：
```text
ExplainQuery(pstate, ExplainStmt, params, dest)
```
`EXPLAIN` 自己是 utility。 但它内部保存的 `stmt->query` 是被解释的 query。 `ExplainQuery()` 会：
```text
ParseExplainOptionList()
query = castNode(Query, stmt->query)
post_parse_analyze_hook()
QueryRewrite()
foreach rewritten query:
  ExplainOneQuery()
```
如果内部 query 不是 utility，`ExplainOneQuery()` 会调用 planner，生成 plan，再输出。 如果内部 query 仍然是 utility，则进入 `ExplainOneUtility()`。 `ExplainOneUtility()` 对少数 utility 做特殊解释：
```text
CREATE TABLE AS
DECLARE CURSOR
EXECUTE
NOTIFY
```
其它 utility 输出：
```text
Utility statements have no plan structure
```
所以 `EXPLAIN CREATE TABLE AS SELECT ...` 和 `EXPLAIN VACUUM` 的行为不同。 前者可以解释内部 SELECT。 后者没有 plan structure。 这个差异来自 `ExplainOneUtility()`，不是 `ProcessUtility()` 主 switch。
### 5.11. 全局对象命令 fast path
`CREATE DATABASE`、`DROP DATABASE`、`CREATE TABLESPACE`、`DROP TABLESPACE`、role 相关命令等在 `standard_ProcessUtility()` 中直接执行。 典型例子：
```text
CREATE DATABASE
  -> PreventInTransactionBlock()
  -> createdb()

DROP DATABASE
  -> PreventInTransactionBlock()
  -> DropDatabase()

CREATE TABLESPACE
  -> PreventInTransactionBlock()
  -> CreateTableSpace()
```
注释反复写着：
```text
no event triggers for global objects
```
这说明 DDL 不等于一定进 slow path。 event trigger 支持的对象类型，才需要进入 slow path。 全局对象命令有 DDL 性质，但 event trigger 语义不同，因此留在 fast switch。
### 5.12. mixed fast / slow path
有些 node tag 需要先看 object type。 例如 `GrantStmt`：
```text
if EventTriggerSupportsObjectType(stmt->objtype)
  ProcessUtilitySlow()
else
  ExecuteGrantStmt()
```
`DropStmt`、`RenameStmt`、`AlterObjectDependsStmt`、`AlterObjectSchemaStmt`、`AlterOwnerStmt`、`CommentStmt`、`SecLabelStmt` 也类似。 这类命令说明 `nodeTag` 是第一层分类，但不是最后一层分类。 同一个 node tag 可能包含不同 object type。 如果 object type 支持 event trigger，就进 slow path。 如果不支持，就直接执行。
### 5.13. DDL default slow path
`standard_ProcessUtility()` 的 `default` 分支是：
```text
ProcessUtilitySlow(...)
```
也就是说，大多数没有在 fast switch 中列出的 DDL 走 slow path。 `ProcessUtilitySlow()` 先判断：
```text
isCompleteQuery = (context != PROCESS_UTILITY_SUBCOMMAND)
needCleanup = isCompleteQuery && EventTriggerBeginCompleteQuery()
```
然后用 `PG_TRY()` 包住 DDL 执行。 正常路径：
```text
EventTriggerDDLCommandStart(parsetree)
switch nodeTag(parsetree):
  CreateStmt -> transformCreateStmt() -> DefineRelation()
  AlterTableStmt -> AlterTableGetLockLevel() -> AlterTableLookupRelation() -> AlterTable()
  IndexStmt -> RangeVarGetRelidExtended() -> transformIndexStmt() -> DefineIndex()
  CreateExtensionStmt -> CreateExtension()
  ...
EventTriggerCollectSimpleCommand()
EventTriggerSQLDrop()
EventTriggerDDLCommandEnd()
```
`PG_FINALLY()` 中：
```text
if (needCleanup)
  EventTriggerEndCompleteQuery()
```
这保证 DDL 中途 ERROR 时 event trigger complete-query 上下文也会收尾。
### 5.14. `CREATE TABLE` 的内部子命令递归
`CreateStmt` 在 slow path 中先被 `transformCreateStmt()` 展开成一个 statement list。 列表可能包括：
```text
CreateStmt
CreateForeignTableStmt
TableLikeClause 展开出的更多语句
IndexStmt
AlterTableStmt
```
源码不能简单 `foreach()`，因为处理 `TableLikeClause` 时会向列表前面插入更多语句。 对真正的 `CreateStmt`，执行：
```text
DefineRelation()
EventTriggerCollectSimpleCommand()
CommandCounterIncrement()
NewRelationCreateToastTable()
```
对其它派生语句，构造 wrapper：
```c
wrapper = makeNode(PlannedStmt);
wrapper->commandType = CMD_UTILITY;
wrapper->canSetTag = false;
wrapper->utilityStmt = stmt;
wrapper->planOrigin = PLAN_STMT_INTERNAL;
ProcessUtility(wrapper, queryString, false,
               PROCESS_UTILITY_SUBCOMMAND, ...);
```
这解释了为什么 hook 可能看到一条用户 SQL 对应多次 `ProcessUtility()` 调用。 外层 queryString、stmt_location、stmt_len 可能仍然指向整条原始 SQL。 hook 必须结合 context 和 statement location 判断当前对象。
### 5.15. `ALTER TABLE` 的内部递归
`AlterTableStmt` slow path 先算锁。 `AlterTableGetLockLevel()` 根据子命令决定需要的 lock mode。 `AlterTableLookupRelation()` 获取 relation OID 和锁，并做基本权限检查。 然后建立 `AlterTableUtilityContext`：
```text
pstmt
queryString
relid
params
queryEnv
```
再调用：
```text
EventTriggerAlterTableStart()
EventTriggerAlterTableRelid()
AlterTable(atstmt, lockmode, &atcontext)
EventTriggerAlterTableEnd()
```
`AlterTable()` 内部有时会生成 utility 子命令。 它通过 `ProcessUtilityForAlterTable()` 递归执行。 该函数会先结束当前 alter-table event trigger context，执行子命令，再恢复外层 alter-table context。 这是一个典型的源码 awkwardness。 它不是架构图中理想的单层分派，而是为了保持 event trigger command ordering 与实际执行顺序一致。
### 5.16. 分派结束后的 `CommandCounterIncrement()`
`standard_ProcessUtility()` 在 switch 之后统一执行：
```text
free_parsestate(pstate);
CommandCounterIncrement();
```
注释提到这样能让命令效果可见，例如让 `PreCommit_on_commit_actions()` 看到。 DDL slow path 内部也可能在多个子命令之间显式 `CommandCounterIncrement()`。 所以 CCI 有两个层次：
```text
utility 顶层命令结束后统一 CCI
DDL 复合命令内部在需要看到前一步 catalog 结果时提前 CCI
```
不要把 CCI 理解成 executor 专属机制。 DDL / utility dispatch 也依赖 command id 推进让 catalog 变化在同一事务后续步骤中可见。

## 6. 生命周期 / ownership / cleanup

### 6.1. 谁创建 utility tree
raw parser 创建 raw parsetree。 parse analysis 把 utility node 挂到 `Query.utilityStmt`。 rewrite 后，`pg_plan_queries()` 把它挂到 wrapper `PlannedStmt.utilityStmt`。 `ProcessUtility()` 不拥有原始 tree 的长期生命周期。 它只在当前 portal execution 中读取或在必要时复制。
### 6.2. 谁持有 wrapper `PlannedStmt`
短查询路径中，wrapper list 属于当前 message / portal 相关 memory context。 prepared statement 或 cached plan 路径中，wrapper 可能被 plancache 持有。 这就是 `readOnlyTree` 的来源。 当 `portal->cplan != NULL`，`PortalRunUtility()` 告诉 `ProcessUtility()`：你看到的 tree 可能不是一次性对象。 执行层如果要 transform 或 scribble，必须先 copy。
### 6.3. `ParseState` 生命周期
`standard_ProcessUtility()` 创建 `ParseState`。 它把 `queryString` 和 `queryEnv` 放进去。 然后传给 command 模块。 switch 结束后 `free_parsestate(pstate)`。 如果 command 模块需要长期保存某些表达式或 query，它必须复制到自己的 owner context。 不能把 `ParseState` 中的临时指针当长期状态。
### 6.4. snapshot 生命周期
`PortalRunUtility()` 决定是否 push active snapshot。 如果需要：
```text
snapshot = GetTransactionSnapshot()
PushActiveSnapshotWithLevel(snapshot, portal->createLevel)
portal->portalSnapshot = GetActiveSnapshot()
```
执行后：
```text
if portalSnapshot != NULL && ActiveSnapshotSet()
  PopActiveSnapshot()
portal->portalSnapshot = NULL
```
注释中特别说，有些 utility 如 `VACUUM`、`WAIT FOR` 可能从下面 pop 掉 ActiveSnapshot stack。 因此收尾时不能盲目 assert 一定还有 active snapshot。 这个细节对错误诊断很重要。 如果你在 extension hook 中维护自己的 snapshot，必须理解 portal 层已经有一套条件化 snapshot 生命周期。
### 6.5. event trigger cleanup
`ProcessUtilitySlow()` 用 `PG_TRY()` / `PG_FINALLY()` 包 event trigger complete-query cleanup。 正常或 ERROR 都要保证：
```text
EventTriggerEndCompleteQuery()
```
只在 `needCleanup` 为 true 时执行。 `needCleanup` 又要求当前不是 `PROCESS_UTILITY_SUBCOMMAND`。 这避免子命令破坏外层 DDL event trigger 上下文。
### 6.6. command module 的资源 owner
`ProcessUtility()` 不统一持有 relation locks、buffer pins、tuplestore、copy state、vacuum state。 这些资源由具体命令模块创建和释放。 例如：
```text
COPY
  -> BeginCopyFrom()/BeginCopyTo()
  -> CopyFrom()/DoCopyTo()
  -> EndCopyFrom()/EndCopyTo()

VACUUM
  -> ExecVacuum()
  -> vacuum()
  -> relation-level vacuum_rel()

ALTER TABLE
  -> AlterTableLookupRelation()
  -> AlterTable()
```
如果中途 ERROR，PostgreSQL 依赖事务 abort、ResourceOwner、MemoryContext reset、smgr / relcache cleanup 和 command 模块的 `PG_TRY` cleanup 组合收尾。 `ProcessUtility()` 只保证它自己建立的 parse state 和 event trigger complete-query context 不泄漏。
### 6.7. completion ownership
`QueryCompletion` 由调用者提供。 `ProcessUtility()` 只填。 `qc` 可以为 NULL。 如果某个 utility 不能 set tag，调用方会传 `NULL` 或 alternative dest。 这解释了为什么内部子命令 wrapper 通常设置：
```text
canSetTag = false
dest = None_Receiver
qc = NULL
```
用户只应该看到外层命令的 completion，而不是 `CREATE TABLE` 内部展开出的每个 index / constraint 子命令。

## 7. 正确性机制层次

### 7.1. node tag 保证语义入口
`nodeTag` 是 dispatch 的第一不变量。 同样的 SQL 字符串不会在 `ProcessUtility()` 中重新 parse。 只要进入 `ProcessUtility()`，它必须已经是 `CMD_UTILITY` wrapper。 这降低了运行期字符串判断成本，也避免 hook 或日志格式变化影响语义分派。
### 7.2. read-only 分类不是一个布尔值
`ClassifyUtilityCommandAsReadOnly()` 返回 flag 组合。 大致包括：
```text
COMMAND_IS_STRICTLY_READ_ONLY
COMMAND_OK_IN_READ_ONLY_TXN
COMMAND_OK_IN_RECOVERY
COMMAND_OK_IN_PARALLEL_MODE
COMMAND_IS_NOT_READ_ONLY
```
这些 flag 分别约束：
```text
read-only transaction
recovery / standby
parallel mode
strictly no-write semantics
```
例子：
| 命令 | 分类含义 |
| --- | --- |
| `EXPLAIN` / `SHOW` | 严格只读，parallel worker 中也安全。 |
| `SET` / `PREPARE` | 修改 backend-local 状态，read-only txn 和 recovery 可接受，但 parallel mode 不支持。 |
| `VACUUM` / `REINDEX` | 可能写 WAL，但不改变 pg_dump 语义，允许 read-only txn，不允许 parallel worker。 |
| `COPY FROM` | 先允许 read-only txn，具体目标是否非临时表由 `DoCopy()` 再挡。 |
| DDL / `TRUNCATE` | 非只读。 |
所以“为什么 read-only transaction 里能 VACUUM”不能用普通业务语义解释。 要回到源码中的 flag 定义和注释。
### 7.3. transaction block 边界
某些命令必须在事务块外：
```text
CREATE DATABASE
DROP DATABASE
CREATE TABLESPACE
DROP TABLESPACE
ALTER SYSTEM
CREATE INDEX CONCURRENTLY
DROP INDEX CONCURRENTLY
COMMIT PREPARED
ROLLBACK PREPARED
```
它们用 `PreventInTransactionBlock(isTopLevel, "...")`。 某些命令必须在事务块内：
```text
SAVEPOINT
RELEASE SAVEPOINT
ROLLBACK TO SAVEPOINT
LOCK TABLE
```
它们用 `RequireTransactionBlock(isTopLevel, "...")`。 这些检查属于 utility dispatch 层或 slow path 早期。 它们不是 parser 语法检查，也不是 executor 后期错误。
### 7.4. snapshot 边界
`PlannedStmtRequiresSnapshot()` 保护两个方向。 一方面，大多数 utility 需要 snapshot，因为 catalog lookup、expression evaluation、permissions 或 internal planning 需要一致视图。 另一方面，transaction control、SET、LOCK 等必须不能为了执行 utility 而冻结事务 snapshot。 这就是为什么 snapshot 判断在 `PortalRunUtility()`，不在每个 command 模块随意处理。
### 7.5. event trigger 边界
`ProcessUtilitySlow()` 的入口条件不是“命令执行很慢”。 它是 event trigger 支持边界。 `standard_ProcessUtility()` 的注释明确指出：这个划分不只是性能问题。 事务命令不应该触发 event trigger，因为 event trigger cache refresh 需要有效事务。 全局对象命令也标注没有 event trigger。 如果扩展在 `ProcessUtility_hook` 中绕过 slow path，就可能漏掉 DDL event trigger。
### 7.6. lock 与 catalog 可见性
DDL 的正确性不是由 `ProcessUtility()` 单独保证。 它只负责把 `AlterTableStmt` 分派给 `AlterTable()`、把 `CreateStmt` 分派给 `DefineRelation()`。 真正的对象锁、catalog tuple 写入、dependency 记录、relcache invalidation、WAL 由 command 模块和 catalog 层完成。 但 `ProcessUtilitySlow()` 会在某些 DDL 前先做关键顺序控制。 例如 `ALTER TABLE` 先算 lock mode 再 lookup relation。 `CREATE INDEX` 先用最终需要的 lock mode 获取 relation，避免后续重复 name lookup 锁到不同 relation，也避免 lock upgrade hazard。
### 7.7. command counter
`CommandCounterIncrement()` 让同一事务内后续命令能看到前一命令的 catalog 修改。 `CREATE TABLE` slow path 中，`DefineRelation()` 后要 CCI，再创建 toast table。 整个 utility command 结束后，`standard_ProcessUtility()` 再统一 CCI。 因此 DDL dispatch 的 correctness 依赖 command id 推进。 如果只看 transaction commit / abort，会漏掉同一事务内部可见性。
### 7.8. hook contract
`ProcessUtility_hook` 的参数包含：
```text
PlannedStmt *pstmt
queryString
readOnlyTree
ProcessUtilityContext context
ParamListInfo params
QueryEnvironment *queryEnv
DestReceiver *dest
QueryCompletion *qc
```
hook 必须尊重 `readOnlyTree`。 hook 必须理解同一个 queryString 可能对应多次 utility 调用。 hook 如果调用 `standard_ProcessUtility()`，通常要原样传递 context、dest、qc。 hook 如果改变命令执行顺序，可能破坏 event trigger ordering、completion tag 或 command counter 语义。

## 8. 错误路径 / 异常路径 / fallback

### 8.1. read-only / recovery / parallel mode 拦截
`standard_ProcessUtility()` 在进入 command switch 前统一做限制。 典型错误来自：
```text
PreventCommandIfReadOnly()
PreventCommandIfParallelMode()
PreventCommandDuringRecovery()
```
但这不是全部。 某些命令需要打开对象后才能判断，例如 `COPY FROM` 到临时表可以在 read-only transaction 中允许，非临时表不允许。 因此 `ClassifyUtilityCommandAsReadOnly()` 有时只是给出外层可能性，command 模块还会二次检查。
### 8.2. transaction block 错误
`CREATE DATABASE` 或 `CREATE INDEX CONCURRENTLY` 在显式事务块中会 ERROR。 错误发生在命令实际修改 catalog 前。 这类错误路径的目标是避免不可原子化或需要跨事务阶段的操作被包进普通 transaction block。 对 `SAVEPOINT`、`LOCK TABLE` 则相反：不在 transaction block 中没有持久意义或语义错误，所以用 `RequireTransactionBlock()`。
### 8.3. event trigger ERROR cleanup
DDL slow path 的核心异常保障是：
```text
PG_TRY()
  DDL execution and event trigger collection
PG_FINALLY()
  EventTriggerEndCompleteQuery()
PG_END_TRY()
```
如果 `DefineRelation()`、`AlterTable()`、`DefineIndex()` 或 event trigger 本身 ERROR，complete-query context 仍然收尾。 catalog 修改由事务 abort 回滚。 event trigger 上下文由 `PG_FINALLY()` 回收。 这两个 cleanup 层次不能混为一谈。
### 8.4. COPY RLS fallback
`DoCopy()` 中，如果 `COPY TO table` 遇到 RLS enabled，它不会在 COPY 代码里重新实现 policy evaluation。 它构造一个 `SELECT`，让正常 query processing 处理 RLS。 这是一种正确性优先的 fallback：
```text
直接 relation COPY TO
  -> 发现 RLS
  -> 构造 SELECT targetlist / FROM
  -> query-based COPY
```
`COPY FROM` 遇到 RLS 则 ERROR，提示用 `INSERT`。 因为 COPY FROM 在行进入目标表前的 policy / generated column / where 处理顺序更敏感，源码选择不在这里复制 INSERT 的完整语义。
### 8.5. EXPLAIN utility fallback
`ExplainOneUtility()` 只对少数 utility 展示内部计划。 对大多数 utility：
```text
Utility statements have no plan structure
```
这不是 EXPLAIN 失败。 它表达的是：外层 utility 没有 optimizer plan tree。 如果 utility 内部没有可解释的 query，EXPLAIN 只能给 dummy group。
### 8.6. ActiveSnapshot 被下层弹出
`PortalRunUtility()` 的注释说，`VACUUM`、`WAIT FOR` 等 utility 可能从下层 pop active snapshot stack。 因此收尾逻辑是：
```text
if (portalSnapshot != NULL && ActiveSnapshotSet())
  PopActiveSnapshot()
```
而不是无条件 pop。 如果扩展假设执行 utility 后 active snapshot 一定存在，就可能在这些命令上出错。
### 8.7. unrecognized node type
`ProcessUtilitySlow()` 的 default 是：
```text
elog(ERROR, "unrecognized node type: %d", nodeTag(parsetree))
```
新增 utility node 时，如果只改 grammar / parsenodes，却没有补 `utility.c` 的 dispatch、command tag、log level、tuple descriptor 或 snapshot 判断，就会走到这里或其它辅助 switch 的 unknown case。 这类错误通常在开发期暴露。 课程里要记住：utility node 的生命周期横跨多个 switch，不是只加一个执行函数。

## 9. 成本、资源与跨模块传播

### 9.1. dispatch 本身不是瓶颈
`ProcessUtility()` 的顶层成本只是一次 hook 判断和一个 switch。 `ClassifyUtilityCommandAsReadOnly()` 也是 node tag switch。 对单条命令来说，这些 CPU 成本几乎不构成瓶颈。 真正成本在分派后的模块：
```text
DDL -> catalog access / locks / dependency / invalidation / WAL
COPY -> data parsing / tuple routing / constraint / WAL / file or socket IO
VACUUM -> heap scan / index cleanup / visibility map / FSM / WAL / cost delay
EXPLAIN -> rewrite / planning / optional execution / instrumentation
```
因此 profiling 时如果火焰图显示 `ProcessUtility()`，通常要继续展开到具体 command 函数。
### 9.2. event trigger 成本
进入 `ProcessUtilitySlow()` 的 DDL 可能触发 event trigger。 成本包括：
```text
event trigger cache lookup
command collection
object address collection
deparse support
user-defined event trigger function execution
```
这个成本随 DDL 子命令数量、对象数量、分区数量和 event trigger 函数复杂度扩张。 `CREATE TABLE` 里的 LIKE、constraints、indexes 可能生成多个内部命令。 `ALTER TABLE` 在分区或继承结构上可能传播到大量 child relations。
### 9.3. lock 成本与等待传播
`ProcessUtilitySlow()` 本身不维护 lock table。 但它把命令送入会拿 heavyweight lock 的模块。 例如：
```text
ALTER TABLE
  -> AlterTableGetLockLevel()
  -> AlterTableLookupRelation()

CREATE INDEX
  -> RangeVarGetRelidExtended(lockmode)
```
等待会出现在 `pg_locks` 和 `pg_stat_activity.wait_event_type = Lock`。 用户看到的是 DDL 卡住。 源码上要从 ProcessUtility dispatch 继续追到 command module 的 lock acquisition。
### 9.4. command counter 与 catalog cache 成本
DDL 子命令之间的 `CommandCounterIncrement()` 会让 catalog 修改对后续命令可见。 它也会推动 invalidation 处理和 cache 状态变化。 在大量 DDL 或复杂 `CREATE TABLE` 场景中，成本不只是 catalog insert。 还有：
```text
syscache / relcache invalidation
dependency lookup
object address resolution
event trigger command collection
CCI 后的 catalog 可见性推进
```
这些成本随对象数量和 dependency fan-out 增长。
### 9.5. COPY 资源传播
COPY 的成本沿数据通道传播。 `COPY FROM` 通常涉及：
```text
input parsing
encoding conversion
per-row expression / WHERE
constraints / triggers
tuple routing
WAL
indexes
```
`COPY TO` 涉及：
```text
scan or query execution
output conversion
file / program / client socket write
```
如果 `COPY TO` 因 RLS 变成 query-based COPY，成本模型会从直接 relation handling 转向正常 planner / executor 路径。 这个行为在性能诊断中很容易被误判。
### 9.6. VACUUM 资源传播
VACUUM 从 utility fast path 进入 `ExecVacuum()` 后，资源压力会传播到：
```text
heap pages
visibility map
free space map
index vacuum
WAL
autovacuum / manual vacuum cost delay
shared cost balance
```
manual VACUUM 的入口是 utility dispatch。 但它的慢点通常在 storage / access method 层。
### 9.7. EXPLAIN 成本传播
`EXPLAIN` 外层是 strictly read-only utility。 但 `EXPLAIN ANALYZE` 会执行内部 query。 `ExplainOnePlan()` 会创建 query descriptor、启动 executor、收集 buffer / WAL / memory / worker instrumentation。 所以不能只因为外层 `ExplainStmt` 被归为 read-only，就认为它没有执行成本或没有副作用。 `EXPLAIN ANALYZE INSERT ...` 会真的执行写入，除非被事务回滚。 分派层的 read-only 分类针对的是 EXPLAIN command 本身。 内部 query 的语义由 EXPLAIN 模块处理。
### 9.8. 跨模块边界
本节至少连接这些模块：
| 模块 | 边界 |
| --- | --- |
| parser / analyzer | 负责产生 utility node，不负责执行。 |
| planner | utility wrapper 不进入 planner，但 EXPLAIN / CTAS / COPY query 可能在内部重新进入 planner。 |
| portal / pquery | 负责 statement list、snapshot、dest、qc、canSetTag。 |
| transaction manager | 处理 transaction command、transaction block 限制、read-only 状态。 |
| event trigger | 只覆盖支持的 DDL，slow path 维护完整事件生命周期。 |
| command modules | 具体执行 DDL、COPY、VACUUM、EXPLAIN、SET、LOCK 等语义。 |
| relcache / syscache / invalidation | DDL command 模块修改 catalog 后触发缓存一致性机制。 |
ProcessUtility 是边界层，不是所有副作用的实现层。

## 10. 观测与诊断入口

### 10.1. 运行时 truth
本节锚定的 runtime truth 是：
```text
同样都叫 utility statement，
有的只在 standard_ProcessUtility() fast switch 中执行，
有的会进入 ProcessUtilitySlow() 并触发 DDL event trigger，
有的还会在 command 模块内部重新进入 rewrite / planner / executor。
```
诊断要完成：
```text
看到 SQL 行为
  -> 判断 utility node tag
  -> 判断 fast path / slow path / internal query
  -> 回源码验证具体分支
```
### 10.2. `debug_print_parse` 和 `debug_print_plan`
用 `debug_print_parse` 可以看到 utility `Query` 中的 `utilityStmt`。 对普通 query，`debug_print_plan` 会打印 planner 产物。 对 utility wrapper，通常没有普通 plan tree。 但 `EXPLAIN` 内部 query 或 `COPY (SELECT)` 仍可能触发 planning。 观察时要区分外层 utility wrapper 和内部 query plan。
### 10.3. server log 与 command tag
`log_statement` 和 duration log 使用 command tag / log level。 `CreateCommandTag()` 和 `GetCommandLogLevel()` 也在 `utility.c`。 如果你新增 utility node 或调试日志分类，不能只看 `standard_ProcessUtility()`。 还要检查：
```text
CreateCommandTag()
GetCommandLogLevel()
UtilityReturnsTuples()
UtilityTupleDescriptor()
UtilityContainsQuery()
```
否则可能出现命令能执行，但日志、Describe、completion 或 EXPLAIN 行为不完整。
### 10.4. `pg_stat_activity`
`pg_stat_activity.query` 看到的是原始 query string。 它不能告诉你当前在 fast path 还是 slow path。 但 wait event 可以提示后续模块状态：
```text
Lock wait
  -> 多半在 command module 获取对象锁

IO wait
  -> COPY / VACUUM / DDL storage 操作

ClientRead / ClientWrite
  -> COPY 数据通道或结果输出
```
这类观测只能定位现象，不能直接证明 dispatch 分支。 要回源码或加断点。
### 10.5. `pg_locks`
DDL 卡住时，`pg_locks` 能看到 relation / database / transactionid 等 lock。 但 `ProcessUtility()` 本身不会显示成 lock owner。 你需要按命令类型继续追：
```text
ALTER TABLE -> tablecmds.c
CREATE INDEX -> indexcmds.c
DROP -> dependency / objectaddress / dropcmds.c
CREATE DATABASE -> dbcommands.c
```
dispatch 层只告诉你应该追哪个模块。
### 10.6. event trigger 实验观测
可以创建 event trigger 观察哪些命令进入 slow path。 例如对 `ddl_command_start` 和 `ddl_command_end` 写日志。 然后比较：
```text
CREATE TABLE t(a int);
ALTER TABLE t ADD COLUMN b int;
VACUUM t;
COPY t TO STDOUT;
EXPLAIN SELECT * FROM t;
CREATE DATABASE x;
```
你会看到 DDL event trigger 覆盖的是支持的 schema-level DDL。 VACUUM、COPY、EXPLAIN、事务命令和全局对象命令不会以同样方式触发。
### 10.7. gdb 断点
源码跟读最直接的断点：
```gdb
break pg_plan_queries
break PortalRunUtility
break ProcessUtility
break standard_ProcessUtility
break ProcessUtilitySlow
break DoCopy
break ExecVacuum
break ExplainQuery
```
在 `standard_ProcessUtility()` 中打印：
```gdb
p nodeTag(pstmt->utilityStmt)
p context
p readOnlyTree
p pstmt->stmt_location
p pstmt->stmt_len
```
对 `CREATE TABLE`，继续观察是否递归进入 `ProcessUtility()`。 对 prepared `EXPLAIN EXECUTE` 或 cached statement，观察 `readOnlyTree` 是否为 true。
### 10.8. 能看到、只能推断、看不到
| 类型 | 例子 |
| --- | --- |
| 能直接看到 | SQL 文本、command tag、wait event、locks、event trigger 是否触发、COPY processed rows、EXPLAIN 输出。 |
| 只能推断 | 当前走 fast path 还是 slow path、是否因为 object type 不支持 event trigger 而绕过 slow path、是否因 `readOnlyTree` copy 过。 |
| 基本不可见 | command 模块内部短生命周期 parse state、event trigger complete-query 内部队列、某些 CCI 边界，除非加断点或日志。 |
诊断时不要把可见 command tag 当作 dispatch 分支的完整证据。 `CREATE DATABASE` 和 `CREATE TABLE` 都是 DDL tag，但前者 fast path，后者 slow path。

## 11. 常见误区

### 11.1. 误区：utility statement 都不需要 planner
外层 utility wrapper 不进 planner。 但 utility 内部可能包含 query。 `EXPLAIN SELECT` 会规划内部 SELECT。 `CREATE TABLE AS SELECT` 会执行内部 query。 `DECLARE CURSOR FOR SELECT` 会保存内部 query。 `COPY (SELECT ...) TO` 会走 query execution。 正确说法是：`ProcessUtility()` 不把外层 utility node 当作普通 query plan；内部 query 由对应 command 模块决定是否 rewrite / plan / execute。
### 11.2. 误区：DDL 都走 `ProcessUtilitySlow()`
很多 schema-level DDL 走 slow path。 但全局对象命令、event trigger 不支持的对象类型、某些 grant / drop / rename / comment / security label 分支会 fast path。 判断时要看 `EventTriggerSupportsObjectType()` 和 `standard_ProcessUtility()` 的显式 case。
### 11.3. 误区：`readOnlyTree` 等于 read-only transaction
`readOnlyTree` 是内存 ownership 保护。 read-only transaction 由 `XactReadOnly` 和 `ClassifyUtilityCommandAsReadOnly()` 处理。 名字相似，但语义完全不同。
### 11.4. 误区：EXPLAIN 是普通 SELECT
`EXPLAIN` 外层是 `T_ExplainStmt` utility。 它通过 `ExplainQuery()` 解释内部 query。 所以 hook、snapshot、command tag 和 utility dispatch 都先看到 EXPLAIN，而不是内部 SELECT。
### 11.5. 误区：VACUUM 在 read-only transaction 中允许表示它不写
源码注释明确说 VACUUM 写 WAL，不是严格只读。 它允许 read-only transaction 的理由是不会改变 pg_dump 语义。 这和 standby、parallel mode、WAL 写入是不同维度。
### 11.6. 误区：event trigger cleanup 可以交给事务 abort
事务 abort 能回滚 catalog 修改。 但 event trigger complete-query context 是 backend-local 执行状态。 `ProcessUtilitySlow()` 必须用 `PG_FINALLY()` 清理。 这就是 `needCleanup` 的意义。
### 11.7. 误区：command tag 决定执行分支
command tag 服务客户端和日志。 执行分支由 node tag 和 object type 决定。 同样的 tag 或同类 tag 不一定走同一条路径。 新增 utility node 时要同时补多个辅助 switch。

## 12. 课堂实验

### 实验 1：用断点画出 fast path / slow path
目标：验证 `ProcessUtility()` 如何按 node tag 分派。 步骤：
```gdb
break ProcessUtility
break standard_ProcessUtility
break ProcessUtilitySlow
break DoCopy
break ExecVacuum
break ExplainQuery
```
依次执行：
```sql
BEGIN;
COMMIT;
CREATE TABLE pu_t(a int);
ALTER TABLE pu_t ADD COLUMN b int;
VACUUM pu_t;
EXPLAIN SELECT * FROM pu_t;
COPY pu_t TO STDOUT;
DROP TABLE pu_t;
```
记录每条命令：
```text
nodeTag(pstmt->utilityStmt)
是否进入 ProcessUtilitySlow()
是否递归调用 ProcessUtility()
context 值
qc command tag
```
预期：
```text
TransactionStmt -> fast path
CreateStmt / AlterTableStmt / DropStmt(table) -> slow path
VacuumStmt -> fast path -> ExecVacuum
ExplainStmt -> fast path -> ExplainQuery
CopyStmt -> fast path -> DoCopy
```
### 实验 2：观察 event trigger 覆盖边界
目标：区分 DDL fast path 和 slow path。 创建 event trigger 函数，记录 `ddl_command_start` 和 `ddl_command_end`。 执行：
```sql
CREATE TABLE pu_e(a int);
ALTER TABLE pu_e ADD COLUMN b int;
CREATE DATABASE pu_db;
VACUUM pu_e;
EXPLAIN SELECT * FROM pu_e;
DROP TABLE pu_e;
```
比较日志。 解释：
```text
CREATE TABLE / ALTER TABLE / DROP TABLE
  -> 支持 event trigger
  -> ProcessUtilitySlow()

CREATE DATABASE
  -> global object，standard_ProcessUtility() 注释为 no event triggers

VACUUM / EXPLAIN
  -> utility fast path，不是 DDL event trigger 对象
```
### 实验 3：COPY TO RLS fallback
目标：观察 COPY fast path 内部重新走 query processing。 准备：
```sql
CREATE TABLE pu_rls(id int, tenant int);
ALTER TABLE pu_rls ENABLE ROW LEVEL SECURITY;
CREATE POLICY p ON pu_rls USING (tenant = current_setting('app.tenant')::int);
SET app.tenant = '1';
```
执行：
```sql
COPY pu_rls TO STDOUT;
COPY pu_rls FROM STDIN;
```
源码观察点：
```gdb
break DoCopy
break pg_analyze_and_rewrite_fixedparams
break planner
```
预期：
```text
COPY TO 发现 RLS 后构造 SELECT，转向 query-based COPY。
COPY FROM 遇到 RLS 报错，提示使用 INSERT。
```
### 实验 4：prepared utility 与 `readOnlyTree`
目标：理解 tree ownership。 执行：
```sql
PREPARE pu_ex AS SELECT 1;
EXPLAIN EXECUTE pu_ex;
```
再用 extended query protocol 或 prepared statement 场景观察 `PortalRunUtility()`。 断点：
```gdb
break PortalRunUtility
break standard_ProcessUtility
```
打印：
```gdb
p portal->cplan
p readOnlyTree
```
解释：如果 utility tree 来自 cached plan，`readOnlyTree` 为 true，`standard_ProcessUtility()` 会复制 wrapper，避免修改缓存对象。

## 13. 讨论题

1. 为什么 PostgreSQL 不让 `ProcessUtility()` 按 SQL 字符串判断命令类型，而依赖 parse analysis 后的 node tag？
2. `ProcessUtilitySlow()` 为什么不能包住所有 utility statement？事务命令触发 event trigger cache refresh 会有什么问题？
3. `readOnlyTree` 和 `XactReadOnly` 分别保护什么？为什么一个是 tree ownership，另一个是事务语义？
4. 为什么 `VACUUM` 可以被视为 OK in read-only transaction，却不是 strictly read-only？
5. `EXPLAIN SELECT` 的外层和内层分别是什么节点？planner 是在哪一层被调用的？
6. `CREATE DATABASE` 和 `CREATE TABLE` 都是 DDL，为什么 event trigger 覆盖不同？
7. 如果新增一个 utility statement，除了执行函数，还必须检查 `utility.c` 中哪些辅助 switch？
8. 为什么 `CREATE TABLE` 内部生成的子命令要用 wrapper `PlannedStmt` 递归回 `ProcessUtility()`，而不是直接调用对应 C 函数？

## 14. 本节小结

`ProcessUtility()` 的主链路是：
```text
Query(CMD_UTILITY)
  -> pg_plan_queries() wrapper PlannedStmt
  -> PortalRunUtility() snapshot / readOnlyTree
  -> ProcessUtility() hook
  -> standard_ProcessUtility() read-only classification + nodeTag switch
  -> fast path command 或 ProcessUtilitySlow()
```
核心状态是 `PlannedStmt.utilityStmt`。 它保存具体 utility node。 dispatch 的第一依据是 `nodeTag(parsetree)`，不是 SQL 字符串，也不是 command tag。 transaction command、COPY、VACUUM、EXPLAIN 和大量 backend-local 命令在 fast switch 中直接进入专用模块。 大多数支持 event trigger 的 DDL 进入 `ProcessUtilitySlow()`。 slow path 的重点是 event trigger lifecycle、DDL command collection、subcommand ordering 和 ERROR cleanup。 ownership 上，utility wrapper 可能来自一次性 portal，也可能来自 plancache。 `readOnlyTree` 为 true 时必须 copy。 `ParseState` 是当前 utility 执行的短生命周期上下文。 snapshot 由 `PortalRunUtility()` 条件化管理。 completion tag 由 `QueryCompletion` 和 `canSetTag` 控制。 正确性不是单个 switch 保证的。 它由 node tag、read-only flag、transaction block 检查、snapshot 边界、event trigger cleanup、command module locks、catalog CCI、invalidation 和事务 abort 共同组成。 异常路径中，read-only / standby / parallel mode 会在 dispatch 早期拦截；DDL ERROR 依赖 `PG_FINALLY()` 清理 event trigger context；COPY RLS 会 fallback 到 query-based COPY 或报错；EXPLAIN 对大多数 utility 只能输出 no plan structure。 观测上，SQL 文本、command tag、wait event、locks、event trigger 日志和 EXPLAIN 输出都只能看到一部分。 判断 fast path / slow path 常常需要断点或源码对照。 从本节抽象出的可迁移规律是：
```text
一个统一入口不等于统一语义；
内核常用 wrapper 把协议、hook、ownership、snapshot 和 completion 统一起来，
再用 typed node 与 context 把真正副作用分派给专门模块。
```
这条规律也适用于 planner hook、executor hook、table AM、FDW 和 event trigger：统一入口解决扩展和协议问题，正确性仍然落在具体状态生命周期和模块边界上。
