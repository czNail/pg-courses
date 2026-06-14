# PostgreSQL Event Trigger / ProcessUtility 边界
## 课程定位
前置知识：已经理解 PostgreSQL 的 parse / analyze / rewrite / planner / executor 主线，也知道普通 SQL 查询会进入 planner 和 executor，而 DDL、事务控制、COPY、VACUUM、EXPLAIN 等 utility command 会走 `ProcessUtility()`。本节唯一主问题：
```text
event trigger、ProcessUtility hook 和 executor/planner hook 的边界在哪里？
```
核心矛盾：PostgreSQL 允许扩展在 utility command 的入口观察或接管控制流，也允许 event trigger 在 DDL 的语义边界上做审计、收集和拒绝；但它必须避免把 DDL 当成普通查询计划，也必须避免让 event trigger 看到半初始化、不可 deparse、不可回滚清理的内部状态。学完后应能独立判断：
- 一个扩展应该挂 `ProcessUtility_hook`，还是创建 event trigger。
- 为什么 `planner_hook` 和 executor hook 不是 DDL 审计的正确边界。
- `ddl_command_start`、`ddl_command_end`、`sql_drop`、`table_rewrite` 分别处在什么时间点。
- `pg_event_trigger_ddl_commands()`、`pg_event_trigger_dropped_objects()`、`pg_event_trigger_table_rewrite_*()` 为什么只能在特定事件中调用。
- 为什么 `ProcessUtility_hook` 比 event trigger 更早、更宽，也更危险。
本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e9`。本节只讨论 event trigger / ProcessUtility boundary。不展开 rule rewrite。不展开普通 row-level trigger。不展开 planner path hook 的具体成本模型。它们只作为边界对照出现。
## 1. 本节在总主线中的位置
05 目录前面的课程已经追过：
```text
SQL text
  -> raw parse tree
  -> analyzed Query
  -> rewritten Query list
  -> planner
  -> PlannedStmt
  -> executor
```
这条线主要服务 `SELECT`、`INSERT`、`UPDATE`、`DELETE`、`MERGE` 这类可规划语句。utility command 的位置不同。DDL 在 parse/analyze 后通常被包在 `PlannedStmt` 里，但它的 `commandType` 是 `CMD_UTILITY`，核心语义在 `utilityStmt` 里的 parse node 上。它不会像普通查询那样生成 scan path、join path 和 executor plan tree。它进入：
```text
PortalRunUtility()
  -> ProcessUtility()
  -> optional ProcessUtility_hook
  -> standard_ProcessUtility()
  -> ProcessUtilitySlow() for event-trigger-supported DDL
```
本节在 planner 课程里的位置是一个边界课。它回答：
```text
哪些命令还属于 planner / executor 的世界；
哪些命令进入 ProcessUtility 的 DDL 世界；
哪些扩展点只适合观察完整 utility command；
哪些扩展点只适合改变普通查询执行。
```
event trigger 是 SQL 级 DDL hook。`ProcessUtility_hook` 是 C 扩展级 utility hook。planner hook 是查询优化 hook。executor hook 是查询执行 hook。这四者经常都被称作 hook，但它们保护的系统边界完全不同。一个简单判断是：
```text
如果你需要看 DDL 的 ObjectAddress、command tag、drop list、table rewrite reason，
你在 event trigger / ProcessUtility 边界。

如果你需要改变 Path、Plan 或执行 tuple，
你在 planner / executor 边界。
```
本节的 runtime truth 是：
```text
同一条 CREATE TABLE 可以被 ProcessUtility_hook 看到一次或多次，
可以触发 ddl_command_start / ddl_command_end，
可以产生 collected command；
但它不会进入 planner path search，也不会触发 executor scan hook。
```
这条现象能用源码解释，也能通过 SQL event trigger 和一个简单 C hook 验证。
## 2. 核心矛盾与一句话运行模型
一句话运行模型：
```text
ProcessUtility() 先给 C 扩展的 ProcessUtility_hook 一个最外层机会；
标准实现再把支持 event trigger 的 DDL 送入 ProcessUtilitySlow()；
ProcessUtilitySlow() 在完整命令边界上建立 EventTriggerQueryState，
按 ddl_command_start -> 执行 DDL -> 收集 command / drop / rewrite state
-> sql_drop -> ddl_command_end -> cleanup 的顺序推进。
```
这里的核心 tension 是：
```text
DDL 扩展性和可观测性
  vs
DDL 内部执行顺序、catalog 可见性、事务回滚、递归子命令和 planner/executor 语义边界
```
PostgreSQL 的选择不是给所有命令一个统一万能 hook。它把边界拆成几层。第一层是 `ProcessUtility_hook`。它能看到所有 utility command。它在标准权限检查、event trigger、命令分发之前运行。它适合实现 C 扩展级审计、拦截、转发、包装和兼容层。它也可以完全不调用 `standard_ProcessUtility()`。因此它不是“事件通知”。它是控制流接管点。第二层是 event trigger。它只覆盖被 event trigger 设施支持的 DDL command tag。它在标准 utility 执行过程里被调用。
它通过 SQL 函数运行，拿到 `EventTriggerData`，再通过专用 SRF 读取 collected command、dropped object 或 rewrite reason。它适合 DDL 审计、策略校验、DDL deparse、扩展对象跟踪。它不适合替代 parser、planner 或 executor。第三层是 planner hook。它以 `Query` 为输入，以 `PlannedStmt` 为输出。它处理普通可规划语句的搜索空间和计划选择。
它不接收 `CreateStmt`、`AlterTableStmt` 这种 utility parse node 作为优化对象。第四层是 executor hook。它以 `QueryDesc` / plan state 为中心，观察或改变普通执行计划的启动、运行、结束。它不会成为 DDL catalog 操作的通用入口。把这四层混在一起，最常见的 bug 是：
```text
在 planner hook 里试图审计 CREATE TABLE；
在 event trigger 里假设所有内部 DDL 子命令都会像顶层命令一样触发；
在 ProcessUtility_hook 里丢掉 standard_ProcessUtility() 导致 event trigger 不再运行；
在 ddl_command_start 里读取 ddl_command_end 才会收集完成的 command list。
```
本节的 mental model 是：
```text
ProcessUtility_hook 保护控制流入口；
event trigger 保护 DDL 语义事件；
planner hook 保护计划搜索；
executor hook 保护计划运行。
```
## 3. 核心文件分工与阅读顺序
| 顺序 | 文件 | 作用 |
| --- | --- | --- |
| 1 | `src/include/tcop/utility.h` | 定义 `ProcessUtilityContext`、`ProcessUtility_hook_type`、`AlterTableUtilityContext` 和 utility 入口声明。 |
| 2 | `src/backend/tcop/utility.c` | `ProcessUtility()`、`standard_ProcessUtility()`、`ProcessUtilitySlow()`、`ProcessUtilityForAlterTable()` 和 command tag 分发。 |
| 3 | `src/include/commands/event_trigger.h` | 定义 `EventTriggerData`、table rewrite reason、event trigger 对外入口。 |
| 4 | `src/backend/commands/event_trigger.c` | event trigger cache lookup、事件 firing、query state、command/drop/rewrite collection。 |
| 5 | `src/include/tcop/deparse_utility.h` | 定义 `CollectedCommand`、`CollectedATSubcmd`，解释 `pg_ddl_command` 背后的 command collection。 |
| 6 | `src/backend/commands/tablecmds.c` | `ALTER TABLE` 子命令收集、`ATRewriteTables()` 中的 `EventTriggerTableRewrite()` 调用点。 |
| 7 | `src/backend/commands/extension.c` | extension script 中普通语句走 planner/executor，utility 语句递归走 `ProcessUtility()`。 |
| 8 | `src/backend/catalog/dependency.c` | drop dependency traversal 中调用 `EventTriggerSQLDropAddObject()` 收集 dropped objects。 |
| 9 | `src/include/optimizer/planner.h` | 对照 `planner_hook_type` 的输入输出边界。 |
| 10 | `src/include/executor/executor.h` | 对照 executor hook 的 `QueryDesc` 生命周期边界。 |
推荐阅读路径：
```text
ProcessUtility()
  -> standard_ProcessUtility()
  -> ProcessUtilitySlow()
  -> EventTriggerBeginCompleteQuery()
  -> EventTriggerDDLCommandStart()
  -> command-specific DDL implementation
  -> EventTriggerCollectSimpleCommand() / EventTriggerAlterTable*
  -> EventTriggerSQLDrop()
  -> EventTriggerDDLCommandEnd()
  -> EventTriggerEndCompleteQuery()
```
`ProcessUtilitySlow()` 的名字不是“性能很慢”。它表示这条路径需要 event trigger 支持设施。`standard_ProcessUtility()` 的注释明确说，不支持 event trigger 的命令留在标准 switch 里直接处理；支持 event trigger 的命令被送入 slow path。这个分工不是单纯性能优化。事务控制命令不能随便触发 event trigger cache 刷新，因为刷新 event trigger cache 需要处在有效事务上下文。阅读时不要只看哪个 case 调哪个函数。要跟状态：
```text
currentEventTriggerState 是否存在；
commandList 何时增长；
SQLDropList 何时增长；
table_rewrite_oid 何时临时设值；
CommandCounterIncrement() 放在哪里；
递归子命令的 context 是 complete query 还是 subcommand。
```
## 4. 关键数据结构与状态边界
### `ProcessUtilityContext`
`ProcessUtilityContext` 是 utility command 的调用来源标签。本课关注四个值：
```text
PROCESS_UTILITY_TOPLEVEL
PROCESS_UTILITY_QUERY
PROCESS_UTILITY_QUERY_NONATOMIC
PROCESS_UTILITY_SUBCOMMAND
```
它不是权限模式。它也不是事务隔离级别。它告诉 utility 层：
```text
这是顶层客户端命令；
这是一个完整 query 里的 utility 命令；
这是非 atomic query context；
还是更大 utility 命令内部生成的子命令。
```
`ProcessUtilitySlow()` 用它计算：
```text
isCompleteQuery = context != PROCESS_UTILITY_SUBCOMMAND
```
只有 complete query 才触发：
```text
ddl_command_start
sql_drop
ddl_command_end
EventTriggerBeginCompleteQuery()
EventTriggerEndCompleteQuery()
```
这就是 event trigger 不等于“每一个内部子步骤都触发一次”的第一层边界。
### `ProcessUtility_hook`
`ProcessUtility_hook_type` 的签名接收：
```text
PlannedStmt *pstmt
const char *queryString
bool readOnlyTree
ProcessUtilityContext context
ParamListInfo params
QueryEnvironment *queryEnv
DestReceiver *dest
QueryCompletion *qc
```
它看到的是整个 utility 入口的调用约定。它不只看 DDL。事务控制、COPY、VACUUM、EXPLAIN、CALL、CREATE、ALTER、DROP 都可能经过这里。源码注释提醒：
```text
同一个 queryString 可能被多个 semicolon-separated statement 共用；
CREATE SCHEMA 这类命令还可能递归调用 ProcessUtility；
hook 应使用 pstmt->stmt_location 和 pstmt->stmt_len 定位当前语句片段。
```
因此 `queryString` 不是 command identity。`queryString + stmt_location + stmt_len + context + utilityStmt tag` 才接近可诊断语义。`ProcessUtility_hook` 的生命周期是 C 层全局函数指针。它由 loadable module 安装。它不是 per transaction 状态。它也不是 SQL event trigger catalog 状态。
如果 hook 不调用 `standard_ProcessUtility()`，后面的标准权限检查、event trigger firing、命令执行都可能不发生。
### `EventTriggerData`
`EventTriggerData` 是 event trigger 函数拿到的 fmgr context。关键字段：
```text
event
parsetree
tag
```
它不是完整 DDL 执行结果。`ddl_command_start` 时，`parsetree` 表示即将执行的 utility parse node，catalog 变化还没有发生。`ddl_command_end` 时，主命令已经执行，event trigger 可通过 `pg_event_trigger_ddl_commands()` 读取收集的 command list。`sql_drop` 时，可通过 `pg_event_trigger_dropped_objects()` 读取 drop dependency traversal 过程中收集的对象。
`table_rewrite` 时，可通过 `pg_event_trigger_table_rewrite_oid()` 和 `pg_event_trigger_table_rewrite_reason()` 读取当前 rewrite target。raw field 不是语义。`EventTriggerData.parsetree` 必须和 event name、command tag、当前 `currentEventTriggerState` 一起解释。
### `EventTriggerQueryState`
`EventTriggerQueryState` 是 backend-local 状态。它不在 shared memory。它不跨 backend 可见。它挂在 `currentEventTriggerState` 这个 backend-local static 指针上。本地源码里的核心字段是：
```text
cxt
SQLDropList
in_sql_drop
table_rewrite_oid
table_rewrite_reason
commandCollectionInhibited
currentCommand
commandList
previous
```
`cxt` 是这次 complete query 的 event trigger state memory context。`SQLDropList` 保存当前命令 drop dependency traversal 发现的 dropped objects。`in_sql_drop` 是保护 `pg_event_trigger_dropped_objects()` 的运行期 flag。`table_rewrite_oid` 和 `table_rewrite_reason` 只在 `table_rewrite` trigger 正在执行时有效。
`commandCollectionInhibited` 用来屏蔽某些内部 DDL 的 command collection。`currentCommand` 支持复杂 command 的嵌套收集，特别是 `ALTER TABLE`。`commandList` 是 `ddl_command_end` 暴露给 `pg_event_trigger_ddl_commands()` 的列表。`previous` 支持 reentrant event trigger。
例如 event trigger 函数内部又执行 DDL，新的 complete query state 会压栈，结束后恢复外层 state。这解释了为什么它不是简单的单个全局 list。
### `CollectedCommand`
`CollectedCommand` 定义在 `src/include/tcop/deparse_utility.h`。它是 DDL command deparse 的内部中间形态。类型包括：
```text
SCT_Simple
SCT_AlterTable
SCT_Grant
SCT_AlterOpFamily
SCT_AlterDefaultPrivileges
SCT_CreateOpClass
SCT_AlterTSConfig
```
普通 DDL 通过 `EventTriggerCollectSimpleCommand()` 收集。`ALTER TABLE` 通过：
```text
EventTriggerAlterTableStart()
EventTriggerAlterTableRelid()
EventTriggerCollectAlterTableSubcmd()
EventTriggerAlterTableEnd()
```
收集为一个带 subcmd list 的复杂 command。`CollectedCommand.in_extension` 会记录当时是否处在 `creating_extension`。这让 event trigger 能区分普通 DDL 和 extension script 内部创建的对象。
### `SQLDropObject`
`SQLDropObject` 是 `event_trigger.c` 的内部结构。它保存：
```text
ObjectAddress address
schemaname
objname
objidentity
objecttype
addrnames
addrargs
original
normal
istemp
```
它由 `EventTriggerSQLDropAddObject()` 分配到当前 query state 的 `cxt` 中。drop 对象来自 dependency traversal。这意味着 `sql_drop` 的对象列表不是简单等于用户 SQL 里写的对象。它可以包括依赖对象。`original` 和 `normal` 帮助区分用户直接指定对象和依赖删除对象。temp schema 的处理也在这里过滤：其他 session 的 temp objects 不报告。
### `ObjectAddress`
event trigger 的对象身份用 `ObjectAddress` 表示。核心三元组是：
```text
classId
objectId
objectSubId
```
它不是 SQL 名字。它是 catalog object identity。名字、schema、identity string 都可能需要重新查 catalog。这有两个诊断后果。第一，`ddl_command_end` 时如果对象已经在同一 command 里被删除，`pg_event_trigger_ddl_commands()` 可能跳过无法再查 identity 的 command。第二，`sql_drop` 的 dropped object identity 是删除前收集出来的，不能用事后 catalog lookup 简化替代。
### `AT_REWRITE_*`
`event_trigger.h` 定义 table rewrite reason：
```text
AT_REWRITE_ALTER_PERSISTENCE
AT_REWRITE_DEFAULT_VAL
AT_REWRITE_COLUMN_REWRITE
AT_REWRITE_ACCESS_METHOD
```
它们是 bit mask。`pg_event_trigger_table_rewrite_reason()` 返回的不是枚举单值。同一次 rewrite 可能有多个原因。`table_rewrite` 的核心状态由 `ATRewriteTables()` 设置。触发点在真正创建 transient heap 并 copy data 之前。因此 event trigger 可以拒绝 rewrite，但它看到的不是已经完成的 rewrite 结果。
### planner / executor hook 对照状态
`planner_hook_type` 接收：
```text
Query *parse
const char *query_string
int cursorOptions
ParamListInfo boundParams
```
返回 `PlannedStmt *`。它的语义单位是可规划 query。`ExecutorStart_hook` / `ExecutorRun_hook` / `ExecutorFinish_hook` / `ExecutorEnd_hook`围绕 `QueryDesc`。它们的语义单位是 plan execution。event trigger 的语义单位是 DDL command event。`ProcessUtility_hook` 的语义单位是 utility command invocation。这四个单位不能互相替代。
## 5. 主流程源码 walkthrough
下面按一条 `CREATE TABLE` / `ALTER TABLE` / `DROP TABLE` 的 utility 主线走。每一步只问一个问题：
```text
当前边界能看到什么状态；
当前边界不能看到什么状态；
谁负责把状态交给下一阶段。
```
### 5.1 `ProcessUtility()`：最外层 C hook 边界
入口在 `src/backend/tcop/utility.c`。`ProcessUtility()` 做的事情非常少。它断言：
```text
pstmt is PlannedStmt
pstmt->commandType == CMD_UTILITY
queryString != NULL
qc 未提前设置 command tag
```
然后检查 `ProcessUtility_hook`。伪代码是：
```text
if (ProcessUtility_hook)
    ProcessUtility_hook(...);
else
    standard_ProcessUtility(...);
```
这说明 hook ordering 非常明确。`ProcessUtility_hook` 在标准实现之前。event trigger 在标准实现内部。所以：
```text
ProcessUtility_hook can wrap event triggers;
event triggers cannot wrap ProcessUtility_hook.
```
一个常见审计扩展会：
```text
log before
standard_ProcessUtility(...)
log after
```
如果它漏掉标准调用，DDL 不会被执行，event trigger 也不会被触发。如果它在标准调用前修改 parse tree，它必须尊重 `readOnlyTree`。`standard_ProcessUtility()` 在 `readOnlyTree` 为真时会 copy `pstmt`，防止 parse transformation 损坏原始树。hook 自己也要有相同边界感。
### 5.2 `standard_ProcessUtility()`：utility command 分类边界
`standard_ProcessUtility()` 先处理 utility 通用前置约束。重要状态包括：
```text
isTopLevel
isAtomicContext
readonly_flags
pstate
parsetree
```
它会用 `ClassifyUtilityCommandAsReadOnly()` 判断命令在：
```text
read-only transaction
parallel mode
recovery
```
下是否合法。不合法时通过：
```text
PreventCommandIfReadOnly()
PreventCommandIfParallelMode()
PreventCommandDuringRecovery()
```
报错。这一步发生在 event trigger slow path 之前。这意味着某些命令根本不会走到 event trigger。随后它创建 `ParseState`，设置：
```text
p_sourcetext = queryString
p_queryEnv = queryEnv
```
然后进入一个很大的 `switch (nodeTag(parsetree))`。这里分成两类。第一类是不能或不应该触发 event trigger 的 utility command。例如事务控制命令。第二类是 event-trigger-supported DDL。这些命令会进入 `ProcessUtilitySlow()`。源码注释强调：
```text
standard_ProcessUtility itself deals only with utility commands
for which we do not provide event trigger support.
Commands that do have such support are passed down to ProcessUtilitySlow.
```
这就是 event trigger 的 command tag 边界。它不是所有 utility command 的通用通知。
### 5.3 `ProcessUtilitySlow()`：complete query state 边界
`ProcessUtilitySlow()` 的第一组局部变量是：
```text
parsetree
isTopLevel
isCompleteQuery
needCleanup
commandCollected
address
secondaryObject
```
关键判断是：
```text
isCompleteQuery = context != PROCESS_UTILITY_SUBCOMMAND
```
然后：
```text
needCleanup = isCompleteQuery && EventTriggerBeginCompleteQuery();
```
只有 complete query 才会建立 event trigger query state。`PROCESS_UTILITY_SUBCOMMAND` 递归进来的子命令不会建立自己的 complete query state，也不会触发 `ddl_command_start` / `ddl_command_end`。这并不代表子命令完全不可见。它可能被外层 command 的 collection 逻辑收进 `commandList`。例如 `CREATE TABLE ... LIKE` 可能生成内部 index / constraint 相关语句。
`CREATE SCHEMA` 也可能递归处理内部语句。但是 firing point 是完整 command 边界，不是每个内部 helper 边界。
### 5.4 PG_TRY：cleanup 的硬边界
`ProcessUtilitySlow()` 用 `PG_TRY()` 包住整个执行主体。`PG_FINALLY()` 中：
```text
if (needCleanup)
    EventTriggerEndCompleteQuery();
```
这个结构是本节最重要的不变量之一。只要 `EventTriggerBeginCompleteQuery()` 返回 true，无论 DDL 成功还是 ERROR，都必须恢复 `currentEventTriggerState`，删除当前 state memory context。如果没有这个 cleanup，backend-local static 指针可能指向已失效状态，后续命令会读到错误的 drop list 或 command list。这也是为什么源码注释说：
```text
use of a PG_TRY block is mandatory
```
MemoryContext 管 event trigger state 的内存。PG_TRY / PG_FINALLY 管 ERROR 路径上的状态恢复。这两者不是同一个机制。
### 5.5 `EventTriggerBeginCompleteQuery()`：状态是否需要创建
`EventTriggerBeginCompleteQuery()` 并不是每个 DDL 都无条件分配 state。它先调用 `trackDroppedObjectsNeeded()`。本地源码中，该函数只在这些事件存在时返回 true：
```text
EVT_SQLDrop
EVT_TableRewrite
EVT_DDLCommandEnd
```
如果系统里只有 `ddl_command_start` trigger，那么不需要 command/drop/rewrite state。`ddl_command_start` 可以直接通过 `EventTriggerCommonSetup()` 找 trigger 并执行。这解释了一个细节：`ddl_command_end` 里如果发现 `currentEventTriggerState` 为空，会直接返回。这个判断不是可有可无。假设某个 `ddl_command_start` trigger 在执行时创建了一个新的 `ddl_command_end` trigger。
命令开始前并没有 end trigger，因此没有为 end collection 建 state。如果结束时直接查 cache 并运行新 trigger，它会看到一个没有完整 command list 的半边界。所以源码选择：
```text
没有 command start 时建立的 state，就不在当前命令末尾运行 end/drop/rewrite 读取型事件。
```
这是 ordering correctness。不是性能优化。
### 5.6 `EventTriggerDDLCommandStart()`：主命令执行前
`ProcessUtilitySlow()` 在执行 command-specific switch 前调用：
```text
if (isCompleteQuery)
    EventTriggerDDLCommandStart(parsetree);
```
`ddl_command_start` 的语义是：
```text
DDL 即将执行；
catalog change 尚未完成；
trigger 可以检查和拒绝。
```
函数内先处理禁用边界。event trigger 在 standalone mode 下禁用：
```text
!IsUnderPostmaster
```
也可由 superuser-only GUC `event_triggers` 禁用。standalone mode 禁用有两个工程原因。第一，坏 event trigger 可能让数据库无法通过正常连接修复。第二，event trigger cache 依赖 system catalog index scan；如果 `pg_event_trigger` 索引损坏，standalone mode 需要成为逃生通道。通过 `EventTriggerCommonSetup()` 找到 runlist 后，`EventTriggerInvoke()` 逐个调用 trigger 函数。调用结束后：
```text
CommandCounterIncrement();
```
这保证 event trigger 自己做的 catalog change 对主命令可见。
### 5.7 `EventTriggerCommonSetup()`：cache snapshot 边界
`EventTriggerCommonSetup()` 做三件事。第一，根据 event 和 parse tree 计算 command tag。对于普通 event，用 `CreateCommandTag(parsetree)`。对于 login event，用 `CMDTAG_LOGIN`。第二，调用 `EventCacheLookup(event)` 找 event trigger cache。如果 cache list 是 NIL，快速返回。第三，按 command tag filter 选出要运行的 trigger function OID。源码特别强调：
```text
Once we start running command triggers,
or touch catalogs,
an invalidation might leave cachelist pointing at garbage.
So copy function OIDs into runlist first.
```
这是一条典型 PostgreSQL 规则。不要把 syscache / relcache 返回的内部指针跨 catalog-changing 边界长期保存。event trigger 在调用函数前复制 OID list。复制的是稳定身份，不是 cache item 指针。
### 5.8 `EventTriggerInvoke()`：函数调用边界
`EventTriggerInvoke()` 为一组 trigger 调用创建独立 memory context：
```text
"event trigger context"
```
每调用一个函数后：
```text
MemoryContextReset(context)
```
调用全部完成后删除 context。它还在多个 trigger 函数之间做：
```text
CommandCounterIncrement()
```
第一个 trigger 前不做，第二个及以后每个 trigger 前做。语义是：
```text
后一个 event trigger 能看到前一个 event trigger 的 catalog effects。
```
trigger 函数没有 SQL 参数。它通过 fmgr context 接收 `EventTriggerData`。函数内部用：
```text
CALLED_AS_EVENT_TRIGGER(fcinfo)
```
判断调用上下文。这也是为什么 event trigger 专用 SRF 需要检查当前 state flag。它们不能在普通 SQL 中随便调用。
### 5.9 command-specific DDL：对象地址和收集边界
`ProcessUtilitySlow()` 的 switch 执行具体 DDL。典型创建类命令返回：
```text
ObjectAddress address
ObjectAddress secondaryObject
```
如果这个 command 没有自己完成 collection，switch 结束后统一执行：
```text
EventTriggerCollectSimpleCommand(address, secondaryObject, parsetree);
```
`commandCollected` 表示该 command 已经由内部路径收集。例如：
```text
CREATE SCHEMA
CREATE TABLE
ALTER TABLE
CREATE INDEX
REINDEX
DROP
GRANT
ALTER TEXT SEARCH CONFIGURATION
```
等命令可能由内部函数直接收集，或根本不需要 simple command collection。这解释了为什么只在 `ProcessUtilitySlow()` 结尾看一次 `CollectSimpleCommand`无法理解全部 command list。一些命令在更深的模块里收集。
### 5.10 `EventTriggerCollectSimpleCommand()`：简单 command 收集
`EventTriggerCollectSimpleCommand()` 的边界是：
```text
DDL 已经执行；
ObjectAddress 已经知道；
ddl_command_end 还没触发。
```
它首先检查：
```text
currentEventTriggerState 是否存在；
commandCollectionInhibited 是否为 false。
```
如果没有 state 或 collection 被抑制，直接返回。它切到 `currentEventTriggerState->cxt`，分配 `CollectedCommand`，设置：
```text
type = SCT_Simple
in_extension = creating_extension
d.simple.address
d.simple.secondaryObject
parsetree = copyObject(parsetree)
```
然后 append 到 `commandList`。这里复制 parse tree 是 ownership 边界。原始 parse tree 的生命周期不一定适合 event trigger SRF 后续读取。复制到 query state context 后，直到 `EventTriggerEndCompleteQuery()` 才整体释放。
### 5.11 `ALTER TABLE`：复杂 command 收集
`ALTER TABLE` 是本节最值得细看的一条 DDL。`ProcessUtilitySlow()` 中：
```text
lockmode = AlterTableGetLockLevel(atstmt->cmds)
relid = AlterTableLookupRelation(atstmt, lockmode)
EventTriggerAlterTableStart(parsetree)
EventTriggerAlterTableRelid(relid)
AlterTable(atstmt, lockmode, &atcontext)
EventTriggerAlterTableEnd()
```
`EventTriggerAlterTableStart()` 不是马上 append command list。它创建一个 `SCT_AlterTable` command，放进 `currentCommand`。`AlterTable()` 进入 `tablecmds.c` 后，每个子命令执行完成时，会调用：
```text
EventTriggerCollectAlterTableSubcmd((Node *) cmd, address)
```
`EventTriggerAlterTableEnd()` 在最后把带 subcmds 的 `CollectedCommand`append 到 commandList。如果没有 subcmds，它会释放 current command，不收集空 ALTER TABLE。这说明 `pg_event_trigger_ddl_commands()` 看到的是结构化 command。不是简单地把 SQL 文本原样返回。
### 5.12 `ProcessUtilityForAlterTable()`：递归 utility 的 ordering 边界
`ALTER TABLE` 有时会生成内部 utility statement。例如 parse transformation 可能产生需要在某个 subcommand 前后执行的语句。这类语句通过 `ProcessUtilityForAlterTable()` 执行。它不是直接调用 `ProcessUtility()` 完事。它先做：
```text
EventTriggerAlterTableEnd();
```
然后构造 wrapper `PlannedStmt`，以：
```text
PROCESS_UTILITY_SUBCOMMAND
```
递归调用 `ProcessUtility()`。递归回来后再：
```text
EventTriggerAlterTableStart(context->pstmt->utilityStmt);
EventTriggerAlterTableRelid(context->relid);
```
源码注释说，这样做是为了保证 command events 的 ordering 与实际执行顺序一致。这就是复杂 DDL 的核心难点。不是“一个 SQL 文本对应一个 event trigger command”。而是内部执行顺序和可 deparse command list 要尽量一致。
### 5.13 `DROP`：drop list 收集边界
`DROP` 命令在 `ProcessUtilitySlow()` 中调用 `ExecDropStmt()`。真正的依赖删除在 `dependency.c`。dependency traversal 中会调用：
```text
EventTriggerSQLDropAddObject(thisobj, original, normal)
```
把对象加入当前 state 的 `SQLDropList`。收集发生在实际删除过程中。`sql_drop` event firing 发生在主命令执行之后、`ddl_command_end` 之前。`ProcessUtilitySlow()` 结尾的顺序是：
```text
EventTriggerSQLDrop(parsetree);
EventTriggerDDLCommandEnd(parsetree);
```
因此：
```text
sql_drop 可以读取 dropped objects；
ddl_command_end 可以读取 ddl commands；
二者都晚于主 DDL 执行。
```
`EventTriggerSQLDrop()` 会设置：
```text
currentEventTriggerState->in_sql_drop = true
```
只在 trigger 调用期间有效。`pg_event_trigger_dropped_objects()` 检查这个 flag。离开 `sql_drop` event 后再调用它会 ERROR。
### 5.14 `ddl_command_end`：command list 暴露边界
主 DDL 执行完成后，`EventTriggerDDLCommandEnd()` 被调用。它先检查 standalone mode 和 GUC。然后检查：
```text
if (!currentEventTriggerState)
    return;
```
之后找到 runlist。在调用 trigger 前：
```text
CommandCounterIncrement();
```
这保证主命令做的 catalog changes 对 event trigger 可见。event trigger 函数内部可以调用：
```text
pg_event_trigger_ddl_commands()
```
该 SRF 读取 `currentEventTriggerState->commandList`。它会跳过某些没有有效 object OID 的 simple command。例如 `IF NOT EXISTS` 尝试创建已存在对象时，返回 OID 可能是 invalid。源码没有强行查找已存在对象来 deparse，因为 parse tree 可能与创建原对象的语句不一致。这是一条 correctness 取舍。宁可不返回不可靠 command，也不构造一个看似确定但语义可能错误的 deparse 结果。
### 5.15 `table_rewrite`：ALTER TABLE rewrite 前边界
`table_rewrite` 不在 `ProcessUtilitySlow()` 结尾触发。它在 `tablecmds.c` 的 `ATRewriteTables()` 里触发。触发条件是：
```text
tab->rewrite > 0
tab->relkind != RELKIND_SEQUENCE
parsetree != NULL
```
调用点位于：
```text
检查系统表 / catalog table / other backend temp table
选择 tablespace / access method / persistence
真正 make_new_heap 和 copy data 之前
```
伪流程：
```text
ATRewriteTables()
  -> if table needs rewrite
       EventTriggerTableRewrite(parsetree, tab->relid, tab->rewrite)
       make_new_heap()
       ATRewriteTable()
       finish swap / rebuild work
```
`EventTriggerTableRewrite()` 在调用 trigger 前临时设置：
```text
table_rewrite_oid = tableOid
table_rewrite_reason = reason
```
并用 `PG_TRY()` / `PG_FINALLY()` 确保失败时重置。所以 `pg_event_trigger_table_rewrite_oid()` 和`pg_event_trigger_table_rewrite_reason()` 只在 table_rewrite trigger 内有效。这不是一个 session 变量。它是当前 firing point 的临时状态。
### 5.16 extension script：utility 与 executor 的混合边界
`extension.c` 展示了一个容易混淆的场景。extension script 里可以有普通查询和 utility 语句。普通可规划语句会走：
```text
CreateQueryDesc()
ExecutorStart()
ExecutorRun()
ExecutorFinish()
ExecutorEnd()
```
utility 语句会走：
```text
ProcessUtility(stmt, sql, false, PROCESS_UTILITY_QUERY, ...)
```
这说明同一个 SQL script 不是整体属于 planner 或 ProcessUtility。它按每个 parsed statement 分类。普通 query 进入 planner/executor。utility command 进入 ProcessUtility。`EventTriggerCollectSimpleCommand()` 记录 `creating_extension`，使 `pg_event_trigger_ddl_commands()` 能暴露 command 是否在 extension 内。
这也是边界的一部分：event trigger 可以知道 DDL 发生在 extension creation context，但它不是 extension dependency machinery 本身。
### 5.17 planner hook 为什么不是 DDL 边界
`planner_hook` 的入口在 `planner.c`。它在 `planner()` 中检查：
```text
if (planner_hook)
    planner_hook(parse, query_string, cursorOptions, boundParams)
else
    standard_planner(...)
```
它的输入是 `Query *parse`。`CREATE TABLE` 的核心语义不是一个要生成 scan/join plan 的 `Query`。它是 `CreateStmt` 等 utility node，被包在 `PlannedStmt.utilityStmt` 中交给 `ProcessUtility()`。因此用 planner hook 做 DDL 审计，会天然漏掉 utility DDL。反过来，event trigger 也不适合修改 `RelOptInfo`、`Path` 或 `Plan`。它没有 `PlannerInfo root`。它也不会参与 path search。
### 5.18 executor hook 为什么不是 DDL 边界
executor hook 的入口在 `execMain.c`。它们围绕：
```text
ExecutorStart(QueryDesc *queryDesc, int eflags)
ExecutorRun(QueryDesc *queryDesc, ScanDirection direction, uint64 count)
ExecutorFinish(QueryDesc *queryDesc)
ExecutorEnd(QueryDesc *queryDesc)
```
普通 `SELECT`、DML、extension script 内的可规划语句会进入这里。但是 DDL 的 catalog 操作并不是一个 executor plan tree。`DefineRelation()`、`AlterTable()`、`performDeletion()` 这类函数直接操作 catalog、dependency、storage、lock、CommandCounter。它们不通过 `ExecScan`、`ExecModifyTable` 这种普通 plan node。
所以 executor hook 可以观察某些 DDL 内部主动执行的查询，但不能代表 DDL command 的语义边界。要观察 DDL，应看 `ProcessUtility_hook` 或 event trigger。要观察 DML 执行，再看 executor hook。
## 6. 生命周期 / ownership / cleanup
### 谁创建
`ProcessUtility_hook` 由 C 扩展在 `_PG_init()` 等加载路径中安装。函数指针是 backend 进程里的全局状态。event trigger 定义在 catalog `pg_event_trigger` 中。`CreateEventTrigger()` 写 catalog，并记录函数、事件名、tag filter、owner 和 dependency。每次执行 DDL 时，backend 通过 event trigger cache 找到当前可用 trigger。
`EventTriggerQueryState` 由 `EventTriggerBeginCompleteQuery()` 创建。只在需要跟踪 dropped objects、table rewrite 或 ddl command end commands 时创建。`CollectedCommand` 由 command 执行后的 collection 函数创建。`SQLDropObject` 由 dependency deletion 路径创建。table rewrite transient state 由 `EventTriggerTableRewrite()` 临时设置。
### 谁持有
`ProcessUtility_hook` 由扩展代码负责链式持有。如果多个扩展都安装 hook，每个扩展通常保存 previous hook，再在自己的 hook 中调用 previous 或 standard。PostgreSQL 核心不替扩展解决 hook 链顺序冲突。event trigger catalog tuple 由数据库持有。event trigger cache 按 backend 缓存 catalog 结果。cache invalidation 使下次 lookup 重建。
`EventTriggerQueryState` 由当前 backend 的 `currentEventTriggerState` 持有。嵌套执行时，`previous` 形成栈。`commandList`、`SQLDropList`、copied parsetree 都归 `state->cxt`。
### 谁释放
`EventTriggerEndCompleteQuery()` 删除当前 state 的 memory context。这会一次性释放：
```text
SQLDropObject
CollectedCommand
copied parsetree
command list cells
alter table subcmds
```
然后恢复 `previous`。`EventTriggerInvoke()` 的临时 memory context 更短。它只覆盖 event trigger 函数调用过程中的临时泄漏。每个 trigger 函数后 reset。整组调用结束后 delete。这两个 context 生命周期不同：
```text
event trigger state context: complete query 级；
event trigger invoke context: 单组 trigger 调用级。
```
不要把它们混为一谈。
### ERROR / abort 时谁兜底
`ProcessUtilitySlow()` 的 `PG_FINALLY()` 兜底 complete query state。`EventTriggerSQLDrop()` 用 `PG_TRY()` / `PG_FINALLY()` 兜底 `in_sql_drop`。`EventTriggerTableRewrite()` 用 `PG_TRY()` / `PG_FINALLY()` 兜底 rewrite oid / reason。`EventTriggerAlterTableEnd()` 的注释承认一个历史边界：
```text
FIXME this API isn't considering the possibility
that an xact/subxact is aborted partway through.
Probably it's best to add an AtEOSubXact_EventTriggers().
```
这不是普通用户需要操作的接口，但对内核研发很重要。它说明 command collection API 的子事务错误路径并不完美抽象。读到 event trigger 代码时，不能把所有 cleanup 都想象成已经有统一 AtEOXact 回调。当前实现主要靠 complete query 的 `PG_FINALLY()` 和 memory context reset 保证不泄漏跨命令状态。catalog changes 本身由事务 abort 回滚。backend-local event trigger collection state 由 memory context 删除。
外部资源仍由各自模块的 ResourceOwner / lock / relcache 机制处理。
### 长期对象如何失效
event trigger catalog 变化通过 cache invalidation 影响 `EventCacheLookup()`。`EventTriggerCommonSetup()` 在运行 trigger 前把 function OID 复制到 `runlist`。这是为了避免触发器函数执行期间 catalog change 使 cache list 指针失效。`ObjectAddress` 是长期 identity。`CollectedCommand` 里的对象地址不是对象仍存在的保证。
`pg_event_trigger_ddl_commands()` 查 identity 失败时可能跳过。`SQLDropObject` 在删除前收集名字和 identity，用于 drop 后仍能报告。这是一组不同的失效策略：
```text
event trigger cache: invalidation 后重建；
commandList object address: 事后 lookup，失败则跳过；
SQLDropObject identity: 删除期间预先 materialize；
rewrite oid/reason: firing 期间临时有效。
```
## 7. 正确性机制层次
event trigger 的正确性不是一个机制单独保证。它依赖多层边界叠加。
### command tag filtering
`CREATE EVENT TRIGGER ... WHEN TAG IN (...)` 依赖 command tag。`EventTriggerCommonSetup()` 用 `CreateCommandTag(parsetree)` 得到 tag。assert build 下还会交叉检查：
```text
command_tag_event_trigger_ok()
command_tag_table_rewrite_ok()
```
如果某个命令被错误地送进 event trigger firing point，debug cross-check 会暴露。这保护的是：
```text
event trigger 文档承诺的 command tag 集合
  与
实际 ProcessUtilitySlow 调用点
```
之间的一致性。
### CommandCounterIncrement
CCI 在 event trigger 中非常关键。`ddl_command_start` 结束后 CCI，让 trigger 自己做的 catalog changes 对主命令可见。`ddl_command_end` 和 `sql_drop` 在 trigger 前 CCI，让主命令 catalog changes 对 trigger 可见。`EventTriggerInvoke()` 在多个 trigger 函数之间 CCI，让后一个 trigger 能看到前一个 trigger 的 effects。这不是 MVCC snapshot 的替代品。
它是同一事务内 command visibility 边界。没有这些 CCI，event trigger 会看到陈旧 catalog 状态，或者主命令看不到 trigger 创建的对象。
### lock and dependency
DDL 自己的正确性主要由 command-specific 模块保证。例如 `ALTER TABLE` 先用 `AlterTableGetLockLevel()` 计算 lock mode，再用 `AlterTableLookupRelation()` 获取 target relid。drop 路径由 dependency traversal 决定删除顺序和依赖对象。event trigger 只是插在这些模块的语义边界上。它不替代 heavyweight lock。它也不替代 dependency correctness。
### MemoryContext
event trigger state 使用 memory context 管生命周期。`EventTriggerEndCompleteQuery()` 不逐个释放 list node。它直接 delete state context。这要求所有需要跨 firing point 保留到 end 的对象都必须分配在 `state->cxt`。`EventTriggerCollectSimpleCommand()`、`EventTriggerCollectAlterTableSubcmd()`、`EventTriggerSQLDropAddObject()` 都切换到这个 context。
这保护的是 backend-local 内存生命周期。不是事务语义。
### PG_TRY / PG_FINALLY
`PG_TRY()` 保护的是 ERROR 路径的本地状态恢复。它不能阻止事务 abort。它也不能让已经 ERROR 的 command 继续执行。但它能保证：
```text
currentEventTriggerState 不悬挂；
in_sql_drop 不残留为 true；
table_rewrite_oid 不残留为旧值。
```
这类状态如果残留，会污染后续同 backend 命令。
### GUC and standalone escape hatch
`event_triggers` GUC 和 standalone mode 禁用是 operational correctness。坏 event trigger 可能阻止修复自身。standalone mode 禁用 event trigger 给管理员留下恢复路径。这也说明 event trigger 不是必须永远运行的 crash-safety 机制。它是数据库正常运行模式下的 DDL 扩展机制。
### planner/executor separation
planner hook 的正确性边界是：
```text
返回语义等价且 executor 可执行的 PlannedStmt。
```
executor hook 的正确性边界是：
```text
尊重 QueryDesc、snapshot、estate、instrumentation 和 tuple flow。
```
event trigger 的正确性边界是：
```text
在 DDL command event 的 catalog visibility 点运行，
不能假装自己拥有 planner root 或 executor estate。
```
把 event trigger 写成 planner transform，或者把 planner hook 写成 DDL trigger，都会破坏边界。
## 8. 错误路径 / 异常路径 / fallback
### event trigger 被禁用
`EventTriggerDDLCommandStart()`、`EventTriggerDDLCommandEnd()`、`EventTriggerSQLDrop()`、`EventTriggerTableRewrite()` 都检查：
```text
!IsUnderPostmaster || !event_triggers
```
满足则返回。用户现象是：DDL 仍执行，event trigger 不运行。这不是 silently broken。这是设计好的 escape hatch。
### 没有 current state
`ddl_command_end`、`sql_drop`、`table_rewrite` 都依赖 `currentEventTriggerState`。没有 state 时直接返回。这通常表示命令开始时没有需要 collection 的 trigger。即使命令执行过程中创建了相关 trigger，当前命令也不会 retroactively 变成可完整观察。这是 start-time boundary。
### trigger function 报错
event trigger function 报 ERROR 时，当前 DDL command 失败。事务按普通 ERROR 规则处理。本地 backend state 通过 `PG_FINALLY()` 清理。如果错误发生在 `ddl_command_start`，主 DDL 还没有执行。如果错误发生在 `ddl_command_end` 或 `sql_drop`，主 DDL 已经执行过，但整个事务仍可 abort 回滚。这就是 event trigger 能做策略拒绝的原因。它不是 post-commit callback。它仍在事务内。
### `pg_event_trigger_*` 在错误上下文调用
`pg_event_trigger_dropped_objects()` 要求：
```text
currentEventTriggerState != NULL
in_sql_drop == true
```
否则报 protocol violated。`pg_event_trigger_table_rewrite_oid()` 要求 rewrite oid 有效。`pg_event_trigger_table_rewrite_reason()` 要求 reason 非 0。`pg_event_trigger_ddl_commands()` 要求 current state 存在。这些函数不是普通 catalog view。它们读取的是 event firing 期间的 backend-local transient state。
### `IF NOT EXISTS` 的 invalid object OID
`pg_event_trigger_ddl_commands()` 遇到某些 simple command 的 invalid object OID 会跳过。典型场景是：
```text
CREATE TABLE IF NOT EXISTS existing_table ...
```
源码没有查已有对象来补一个地址。原因是 deparse 当前 parse tree 可能并不能代表已存在对象的真实创建语义。这是信息不完整时的 conservative fallback。
### table rewrite 不支持嵌套命令
`ATRewriteTables()` 注释说明：
```text
We don't support Event Trigger for nested commands anywhere, here included.
```
当来自 `AlterTableInternal()` 时，`parsetree` 可能为 NULL，因此不会触发 table rewrite event。这说明 table rewrite event 是顶层 DDL parse tree 边界上的事件。内部系统调用不一定可见。
### command collection 被抑制
`REFRESH MATERIALIZED VIEW CONCURRENTLY` 内部可能执行一些 DDL。源码在该路径中调用：
```text
EventTriggerInhibitCommandCollection()
EventTriggerUndoInhibitCommandCollection()
```
目的是避免内部 DDL 出现在 deparsed command queue 里。外层 refresh command 本身被收集即可。这是一种有意隐藏内部实现细节的边界。如果诊断时看到 event trigger 没列出某些内部 DDL，不要马上判断为 bug。先检查是否有 command collection inhibition。
## 9. 成本、资源与跨模块传播
event trigger 不在普通 tuple hot path。但它会放大 DDL 成本。成本主要来自五类。
### event trigger cache lookup
每次 firing point 都要查 event trigger cache。如果没有 trigger，快速返回。如果有 trigger，还要按 command tag filter 生成 runlist。成本随：
```text
event trigger 数量
tag filter 数量
firing point 数量
```
扩张。它通常不是 DDL 的主成本，但在大量自动化 DDL 场景中会进入火焰图。
### command collection memory
`ddl_command_end` 需要 command collection。`ALTER TABLE` 多子命令会复制多个 `AlterTableCmd`。`CREATE TABLE` 可能展开内部 command。每个 collected command 都分配在 query state context。成本随：
```text
DDL 子命令数
partition / inheritance fan-out
generated internal statements
parsetree size
```
扩张。这不是 shared memory 压力。它是当前 backend 的 local memory 压力。complete query cleanup 会整体释放。
### drop dependency traversal
`sql_drop` 的 dropped objects 由 dependency traversal 收集。一个 `DROP SCHEMA CASCADE` 可以产生大量 dependent objects。成本随 dependency graph 扩张。`SQLDropObject` 还会尝试 materialize schema/name/identity/type。这意味着 event trigger 打开后，大规模 drop 不只是删除 catalog tuple，还会有额外 identity 构造成本。
### CommandCounterIncrement
event trigger 引入额外 CCI。CCI 本身不是简单计数器递增。它会推进当前事务内 command visibility，让后续 catalog lookup 看到前一 command effects。在 DDL 密集、trigger 函数又执行 DDL 的场景里，CCI 数量会增加。诊断时不要把所有延迟都归因到 trigger 函数 SQL。CCI 和 cache invalidation 也可能贡献成本。
### trigger function 自身
event trigger 函数可以执行任意 SQL。它可能查 catalog，写 audit table，发 notice，甚至执行更多 DDL。系统只保证调用边界和 cleanup。它不保证 trigger 函数低成本。如果 trigger 函数做复杂 deparse 或递归 DDL，主要成本就在用户定义函数里。
### 跨模块连接
event trigger 连接至少六个模块。`tcop/utility.c` 提供 utility command 分发和 firing point。`commands/event_trigger.c` 提供 event trigger 管理和 state。`tcop/deparse_utility.h` 提供 command collection 类型。`commands/tablecmds.c` 提供 `ALTER TABLE` 子命令和 rewrite trigger。`catalog/dependency.
c` 提供 drop traversal 和 dropped object collection。`commands/extension.c` 提供 extension creation context。此外，event trigger 还依赖 syscache / relcache invalidation。它依赖 CommandCounter 和事务 visibility。它依赖 lock manager 但不替代 lock manager。它依赖 object address / dependency 但不替代 dependency。
后台进程方面，event trigger 本身没有 dedicated background worker。它主要在执行 DDL 的 backend 中同步运行。如果 event trigger 函数写 WAL、修改 catalog、创建对象，那会通过普通事务、WAL、checkpointer、walwriter、autovacuum 等已有机制向外传播。但 event trigger state 自身不需要后台进程推进。
## 10. 观测与诊断入口
本节的可观测 truth 是：
```text
DDL 会经过 ProcessUtility 和 event trigger firing point；
但不会生成普通 planner Path，也不会以 executor plan node 的形式执行 catalog DDL。
```
### SQL 侧直接观测
可以创建一个简单 event trigger 函数记录：
```sql
CREATE TABLE ddl_audit(
  ts timestamptz default clock_timestamp(),
  ev text,
  tag text
);

CREATE OR REPLACE FUNCTION log_ddl_start()
RETURNS event_trigger
LANGUAGE plpgsql
AS $$
BEGIN
  INSERT INTO ddl_audit(ev, tag)
  VALUES (TG_EVENT, TG_TAG);
END;
$$;

CREATE EVENT TRIGGER et_start
ON ddl_command_start
EXECUTE FUNCTION log_ddl_start();
```
执行：
```sql
CREATE TABLE et_demo(id int);
SELECT ev, tag FROM ddl_audit ORDER BY ts;
```
你会看到 `ddl_command_start` 和 tag。如果再创建 `ddl_command_end` trigger，可调用 `pg_event_trigger_ddl_commands()` 查看 object identity。
### `ddl_command_end` command list
示例函数：
```sql
CREATE OR REPLACE FUNCTION log_ddl_end()
RETURNS event_trigger
LANGUAGE plpgsql
AS $$
DECLARE
  r record;
BEGIN
  FOR r IN SELECT * FROM pg_event_trigger_ddl_commands()
  LOOP
    RAISE NOTICE 'end tag %, type %, identity %, in_extension %',
      r.command_tag, r.object_type, r.object_identity, r.in_extension;
  END LOOP;
END;
$$;
```
这个函数只能在 event trigger 上下文可靠运行。把 `pg_event_trigger_ddl_commands()` 当普通 SQL 查询调用，不是同一个语义。
### `sql_drop` dropped objects
示例：
```sql
CREATE OR REPLACE FUNCTION log_sql_drop()
RETURNS event_trigger
LANGUAGE plpgsql
AS $$
DECLARE
  r record;
BEGIN
  FOR r IN SELECT * FROM pg_event_trigger_dropped_objects()
  LOOP
    RAISE NOTICE 'drop type %, identity %, original %, normal %',
      r.object_type, r.object_identity, r.original, r.normal;
  END LOOP;
END;
$$;
```
`DROP TABLE ... CASCADE` 会显示依赖对象。这能把 runtime 现象回到 `dependency.c` 中的 collection 调用。
### `table_rewrite` rewrite reason
示例：
```sql
CREATE OR REPLACE FUNCTION block_rewrite()
RETURNS event_trigger
LANGUAGE plpgsql
AS $$
BEGIN
  RAISE EXCEPTION 'rewrite of %, reason % blocked',
    pg_event_trigger_table_rewrite_oid(),
    pg_event_trigger_table_rewrite_reason();
END;
$$;

CREATE EVENT TRIGGER et_rewrite
ON table_rewrite
EXECUTE FUNCTION block_rewrite();
```
再执行会触发 rewrite 的 `ALTER TABLE`。例如改变列类型或添加需要重写的默认值。如果命令被拒绝，说明 firing point 在实际 table rewrite 之前。
### gdb 断点
源码跟读断点建议：
```text
break ProcessUtility
break standard_ProcessUtility
break ProcessUtilitySlow
break EventTriggerDDLCommandStart
break EventTriggerCollectSimpleCommand
break EventTriggerDDLCommandEnd
break EventTriggerSQLDrop
break EventTriggerTableRewrite
break EventTriggerEndCompleteQuery
```
运行 `CREATE TABLE` 时，观察：
```text
context
nodeTag(pstmt->utilityStmt)
pstmt->stmt_location
pstmt->stmt_len
currentEventTriggerState
commandList length
```
运行 `ALTER TABLE` 时，再观察：
```text
currentCommand
d.alterTable.objectId
d.alterTable.subcmds
tab->rewrite
```
### 日志和 wait event
event trigger 自身没有专用 wait event。如果 trigger 函数执行 SQL，你会看到它造成的普通 lock wait、IO wait 或 catalog contention。`pg_stat_activity` 可看到当前 backend 正在执行 DDL 或 trigger 内部 SQL。但它不会告诉你 `commandList` 当前有几个元素。`pg_locks` 能看到 DDL 持有或等待的锁。但它不会告诉你 event trigger cache runlist。
`pg_stat_statements` 如果启用，可能看到 trigger 函数内部执行的 SQL。但它不能把这些 SQL 自动归因到某个 event trigger firing point。
### 能直接观测、只能推断、几乎不可见
能直接观测：
- event trigger 函数里的 `TG_EVENT`、`TG_TAG`。
- `pg_event_trigger_ddl_commands()` 输出的 command。
- `pg_event_trigger_dropped_objects()` 输出的 dropped object。
- table rewrite oid / reason。
- DDL lock wait。
- trigger 函数内部 SQL 的日志和错误。
只能推断：
- `ProcessUtility_hook` 是否调用了 standard path。
- command collection 是否被 inhibition 跳过。
- CCI 对 catalog visibility 的具体贡献。
- event trigger cache invalidation 是否刚发生。
几乎不可见：
- `currentEventTriggerState` 指针栈。
- `EventTriggerCommonSetup()` 复制出的 transient runlist。
- `CollectedCommand` 的完整内部 parsetree。
- `EventTriggerInvoke()` 每个函数后的 memory context reset。
这些需要 gdb、临时日志或源码插桩。
## 11. 常见误区
- `ProcessUtility_hook` 和 event trigger 不是同一层：hook 在标准 utility 实现之前，event trigger 在标准实现内部；hook 可以阻止 event trigger 发生，event trigger 不能拦截 hook。
- event trigger 不能观察所有 utility command：它只覆盖 event trigger 设施支持的 command tag，事务控制等命令不会走 event trigger slow path。
- `ddl_command_start` 不是读取 `pg_event_trigger_ddl_commands()` 的边界：command collection 是主 DDL 执行后给 `ddl_command_end` 用的状态。
- `sql_drop` 不只报告用户 SQL 里写的对象：它来自 dependency traversal，依赖删除对象也可能出现，`original` 和 `normal` 才帮助解释对象来源。
- `table_rewrite` 不是 rewrite 完成后通知：它在真正 rewrite 之前触发，trigger 可以阻止 rewrite，看到的是将要 rewrite 的 relation 和 reason bitmask。
- planner hook 不能审计 DDL：planner hook 面向 `Query -> PlannedStmt`，DDL utility node 的主语义在 `ProcessUtility()`。
- executor hook 不能以 DDL command 语义看到 `CREATE TABLE`：DDL 内部 SQL 可能进入 executor，但 catalog DDL 本身不是 executor plan node。
- `queryString` 不等于当前命令文本：同一 query string 可被多个 statement 共用，递归 utility 也可能传同一个 query string，诊断要结合 `stmt_location`、`stmt_len` 和 parse node。
- `EventTriggerQueryState` 不是事务级全局状态：它是 backend-local complete query state，用 memory context 管生命周期，用 `previous` 支持嵌套。
- event trigger 不适合实现所有 DDL policy：如果 policy 需要理解内部 storage rewrite、lock upgrade、dependency deletion 细节，它可能只能做粗粒度拒绝或审计。
## 12. 课堂实验
### 实验 1：画出 CREATE TABLE 的 firing order
目标：
```text
证明 ProcessUtility_hook 早于 event trigger；
证明 ddl_command_start 早于主 DDL；
证明 ddl_command_end 晚于 ObjectAddress collection。
```
步骤一：创建审计表：
```sql
CREATE TABLE et_log(
  id bigserial primary key,
  ts timestamptz default clock_timestamp(),
  ev text,
  tag text,
  detail text
);
```
步骤二：创建 start trigger：
```sql
CREATE OR REPLACE FUNCTION et_log_start()
RETURNS event_trigger
LANGUAGE plpgsql
AS $$
BEGIN
  INSERT INTO et_log(ev, tag, detail)
  VALUES (TG_EVENT, TG_TAG, 'before main DDL');
END;
$$;

CREATE EVENT TRIGGER et_start
ON ddl_command_start
EXECUTE FUNCTION et_log_start();
```
步骤三：创建 end trigger：
```sql
CREATE OR REPLACE FUNCTION et_log_end()
RETURNS event_trigger
LANGUAGE plpgsql
AS $$
DECLARE
  r record;
BEGIN
  FOR r IN SELECT * FROM pg_event_trigger_ddl_commands()
  LOOP
    INSERT INTO et_log(ev, tag, detail)
    VALUES (TG_EVENT, TG_TAG, r.object_type || ':' || r.object_identity);
  END LOOP;
END;
$$;

CREATE EVENT TRIGGER et_end
ON ddl_command_end
EXECUTE FUNCTION et_log_end();
```
步骤四：执行：
```sql
CREATE TABLE et_t(id int);
SELECT ev, tag, detail FROM et_log ORDER BY id;
```
预期解释：start 行先出现。end 行能看到 object identity。这对应 `ProcessUtilitySlow()` 的：
```text
EventTriggerDDLCommandStart()
execute DDL
EventTriggerCollectSimpleCommand()
EventTriggerDDLCommandEnd()
```
源码练习：在 gdb 中断到 `EventTriggerCollectSimpleCommand()`，打印 `address` 和 `parsetree`。再断到 `pg_event_trigger_ddl_commands()`，观察它读取的是 `currentEventTriggerState->commandList`。
### 实验 2：观察 DROP CASCADE 的 dropped object 列表
目标：
```text
证明 sql_drop 的对象列表来自 dependency traversal，
不是用户 SQL 文本。
```
步骤：
```sql
CREATE TABLE et_parent(id int primary key);
CREATE TABLE et_child(id int references et_parent(id));

CREATE OR REPLACE FUNCTION et_log_drop()
RETURNS event_trigger
LANGUAGE plpgsql
AS $$
DECLARE
  r record;
BEGIN
  FOR r IN SELECT * FROM pg_event_trigger_dropped_objects()
  LOOP
    RAISE NOTICE 'drop: type=%, identity=%, original=%, normal=%',
      r.object_type, r.object_identity, r.original, r.normal;
  END LOOP;
END;
$$;

CREATE EVENT TRIGGER et_drop
ON sql_drop
EXECUTE FUNCTION et_log_drop();

DROP TABLE et_parent CASCADE;
```
观察：输出可能包含表、约束等多个对象。解释：drop 路径通过 dependency graph 找到实际删除对象。`EventTriggerSQLDropAddObject()` 在删除过程中 materialize identity。`sql_drop` firing 点再暴露这批对象。源码跟读：
```text
performDeletion()
  -> findDependentObjects()
  -> reportDependentObjects()
  -> EventTriggerSQLDropAddObject()
  -> deleteOneObject()
```
具体调用点以本地源码为准。重点不是背函数顺序，而是确认对象列表来自 dependency traversal。
### 实验 3：阻止 table rewrite
目标：
```text
证明 table_rewrite event 在实际 rewrite 前触发。
```
步骤：
```sql
CREATE TABLE et_rw(a int);
INSERT INTO et_rw SELECT generate_series(1, 10);

CREATE OR REPLACE FUNCTION et_block_rewrite()
RETURNS event_trigger
LANGUAGE plpgsql
AS $$
BEGIN
  RAISE EXCEPTION 'blocking rewrite oid %, reason %',
    pg_event_trigger_table_rewrite_oid(),
    pg_event_trigger_table_rewrite_reason();
END;
$$;

CREATE EVENT TRIGGER et_table_rewrite
ON table_rewrite
EXECUTE FUNCTION et_block_rewrite();

ALTER TABLE et_rw ALTER COLUMN a TYPE bigint;
```
预期：`ALTER TABLE` 报错。表结构不改变。解释：`ATRewriteTables()` 在 `make_new_heap()` 和数据 copy 之前调用 `EventTriggerTableRewrite()`。event trigger ERROR 使整个 DDL 失败。源码断点：
```text
break ATRewriteTables
break EventTriggerTableRewrite
break make_new_heap
```
确认 `EventTriggerTableRewrite()` 先于 `make_new_heap()`。
## 13. 讨论题
1. 为什么 `ProcessUtility_hook` 不能简单替代 event trigger？
2. 为什么 event trigger 不能简单替代 `ProcessUtility_hook`？
3. `ddl_command_start` 里创建的 `ddl_command_end` trigger 为什么不应该观察当前命令的 end event？
4. `EventTriggerCommonSetup()` 为什么要复制 function OID runlist，而不是保存 event trigger cache item 指针？
5. `pg_event_trigger_dropped_objects()` 为什么必须限制在 `sql_drop` event trigger 内调用？
6. `ALTER TABLE` 为什么需要 `currentCommand` 和 subcmd list，而不是直接把每个 subcommand 都作为 simple command append？
7. 如果一个 C 扩展的 `ProcessUtility_hook` 没有调用 previous hook 或 `standard_ProcessUtility()`，会影响哪些下游机制？
8. 为什么 executor hook 可能看到 event trigger 函数内部的 INSERT，却不能说明 CREATE TABLE 本身经过 executor？
9. table rewrite reason 为什么设计成 bit mask，而不是单个 enum？
10. 如果 `pg_event_trigger_ddl_commands()` 查不到 object identity，为什么跳过比猜测一个 identity 更安全？
## 14. 本节小结
本节唯一主问题是：
```text
event trigger、ProcessUtility hook 和 executor/planner hook 的边界在哪里？
```
核心链路是：
```text
ProcessUtility()
  -> optional ProcessUtility_hook
  -> standard_ProcessUtility()
  -> ProcessUtilitySlow()
  -> EventTriggerBeginCompleteQuery()
  -> ddl_command_start
  -> command-specific DDL
  -> command / drop / rewrite collection
  -> sql_drop
  -> ddl_command_end
  -> EventTriggerEndCompleteQuery()
```
`ProcessUtility_hook` 是 C 扩展控制流入口。它最早、最宽、也最危险。event trigger 是 SQL 级 DDL 事件边界。它不覆盖所有 utility command，也不代表 planner 或 executor。planner hook 的单位是 `Query -> PlannedStmt`。executor hook 的单位是 `QueryDesc` 的执行生命周期。DDL 的核心语义单位是 utility parse node 和 `ObjectAddress`。
`EventTriggerQueryState` 是 backend-local complete query state。`commandList` 服务 `ddl_command_end`。`SQLDropList` 服务 `sql_drop`。`table_rewrite_oid/reason` 服务 `table_rewrite`。这些状态都有严格 firing context。ownership 由 MemoryContext 管。ERROR cleanup 由 `PG_TRY()` / `PG_FINALLY()` 管。
catalog visibility 由 `CommandCounterIncrement()` 管。object deletion correctness 由 dependency 模块管。DDL 并发正确性由各 command 的 lock 策略管。event trigger 只是站在这些边界上观察或拒绝。可观测入口包括 event trigger SQL 函数、server log、gdb 断点、`pg_locks` 和触发器内部 SQL 的统计。
看不到或只能推断的状态包括 `currentEventTriggerState` 栈、runlist、command collection inhibition 和每次 memory context reset。本节可迁移规律是：
```text
扩展点不是按“能不能被调用”分类；
而是按它拥有的状态、生命周期、cleanup 义务和下游契约分类。
```
在 PostgreSQL 内核里，正确选择 hook 的第一步不是找最早入口，而是确认你要改变的是：
```text
utility control flow；
DDL semantic event；
planner search space；
executor tuple flow。
```
只有边界选对，后面的权限、锁、visibility、cleanup、观测和性能判断才可能成立。
