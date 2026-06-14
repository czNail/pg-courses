# PostgreSQL Simple Query 如何进入 transaction command loop

## 课程定位

前置知识：熟悉 PostgreSQL backend 进程模型，知道 frontend/backend protocol 有
simple query 和 extended query 两条入口，并已读过本目录前面的
`ExecutorStart`、`ExecutorRun`、`ExecutorEnd` 生命周期。

本节唯一主问题：

```text
simple query 如何进入 transaction command loop、parse/analyze/rewrite/plan/execute，
并处理隐式事务边界？
```

核心矛盾：

```text
客户端把一个 query string 当作一条协议消息发送，
但服务端必须把它拆成一个或多个 SQL command，
并在事务状态机、snapshot、Portal、executor、ERROR cleanup 和 ReadyForQuery
之间维持精确边界。
```

学完后应能判断：

- simple Query message 和 SQL statement 不是同一个边界。
- raw parse、parse analysis、rewrite、planning、execution 的 snapshot 假设不同。
- 多语句 simple query 的 implicit transaction block 不是用户显式 `BEGIN`。
- `CommandComplete`、`CommitTransactionCommand()`、`ReadyForQuery` 的顺序有协议正确性含义。
- simple query 不是“直接调用 executor”，而是先进入 Portal 和 transaction command loop。

本课基于本地 `/home/nail/postgres` 源码，分支 `master`，
提交 `0e1f1ed157e90741e12a3715909e1b2d71ff9344`。

## 1. 本节在总主线中的位置

前面几节从 executor 内部看过 `QueryDesc -> ExecutorStart -> ExecutorRun
-> ExecutorFinish -> ExecutorEnd`。

本节把视角上移到 tcop 和协议层。

```text
PostgresMain message loop
  -> PqMsg_Query
  -> exec_simple_query()
  -> transaction command loop
  -> parser / analyzer / rewriter / planner
  -> unnamed Portal
  -> executor 或 ProcessUtility
  -> CommandComplete
  -> ReadyForQuery
```

这一层解释 executor 外部的三个问题。

第一，`ExecutorRun()` 使用的 snapshot、DestReceiver、QueryDesc owner 从哪里来。

第二，一条 simple Query message 为什么可能发送多个 `CommandComplete`。

第三，ERROR 后为什么有时回到 idle，有时停在 failed transaction block。

下一节 extended query 会把 Parse/Bind/Execute/Sync 拆成多个协议阶段。
本节只讨论 simple query：一个 `PqMsg_Query` 携带一个 query string，
服务端在同一轮处理里完成 parse/analyze/rewrite/plan/execute。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
exec_simple_query() 先在 transaction command 中 raw parse 整个 query string，
再逐个 RawStmt 做 analyze/rewrite/plan/PortalRun；
如果一条 simple Query message 含多个 RawStmt，
就用 implicit transaction block 默认把这些普通 statement 绑定到同一事务边界，
直到最后一个 statement 或显式事务控制命令改变边界。
```

这里的 tension 是：

```text
simple protocol 的历史兼容性和低协议开销
  vs
事务状态、命令计数、snapshot、Portal ownership 和 ERROR cleanup 必须可恢复
```

PostgreSQL 不把 query string 直接交给 executor。
它先用 raw parser 把 query string 切成 `RawStmt` list。
它在每个 `RawStmt` 周围建立 command tag、destination、snapshot、Portal 和事务推进边界。
它把真实事务状态交给 `xact.c` 的 `TBLOCK_*` 状态机。
它用 `Portal` 作为计划、执行、utility command 和 cleanup 的运行容器。
它最终用 `ReadyForQuery` 把恢复后的 transaction block status 回传给客户端。

源码里保留了真实工程痕迹。
simple query 被定义得像使用 unnamed statement 和 unnamed portal。
多语句 simple query 默认有 historical implicit transaction block。
`FETCH` binary cursor 的输出格式在 `exec_simple_query()` 里特殊处理。
utility command 可能在 `PortalRunUtility()` 内部改变 transaction context。
`ReadyForQuery` 才是协议轮次完成信号，不是 executor 完成信号。

本节所有源码细节都服务这个模型。
如果一个函数不能解释协议边界、statement 边界、transaction command 边界、
Portal 边界或 ERROR cleanup，就不展开。

## 3. 核心文件分工与阅读顺序

| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/tcop/postgres.c` | `PostgresMain()`、`exec_simple_query()`、`start_xact_command()`、`finish_xact_command()`。 |
| 2 | `src/backend/access/transam/xact.c` | `StartTransactionCommand()`、`CommitTransactionCommand()`、`AbortCurrentTransaction()`、implicit block。 |
| 3 | `src/backend/parser/parser.c` | `raw_parser()` 从 query string 生成 `RawStmt` list。 |
| 4 | `src/backend/parser/analyze.c` | `parse_analyze_fixedparams()`、`analyze_requires_snapshot()`。 |
| 5 | `src/backend/rewrite/rewriteHandler.c` | `QueryRewrite()` 让一个 `Query` 变成零个、一个或多个 query。 |
| 6 | `src/backend/optimizer/plan/planner.c` | `planner()` / `standard_planner()` 生成 `PlannedStmt`。 |
| 7 | `src/backend/tcop/pquery.c` | `PortalStart()`、`PortalRun()`、`PortalRunMulti()`、`ProcessQuery()`。 |
| 8 | `src/backend/executor/execMain.c` | `ExecutorStart()`、`ExecutorRun()`、`ExecutePlan()`、`ExecutorFinish()`、`ExecutorEnd()`。 |
| 9 | `src/backend/tcop/dest.c` | `ReadyForQuery()` 发送事务状态码。 |
| 10 | `src/include/utils/portal.h` | `PortalData` 的 status、resowner、queryDesc、snapshot。 |
| 11 | `src/include/executor/execdesc.h` | `QueryDesc` 是 Portal 和 executor 之间的运行句柄。 |
| 12 | `src/include/nodes/parsenodes.h` | `RawStmt`、`Query` 的语义边界。 |
| 13 | `src/include/nodes/plannodes.h` | `PlannedStmt` 是 executor 可消费的计划描述。 |

推荐先读 `exec_simple_query()` 的时间线，再读 `xact.c` 的状态机，
最后回到 parser/analyzer/rewriter/planner/executor 的分层。
不要从 grammar 或 planner path 枚举开始。

## 4. 关键数据结构与状态

### `PqMsg_Query`

simple query 的协议入口在 `PostgresMain()`。

```text
case PqMsg_Query:
  SetCurrentStatementStartTimestamp()
  query_string = pq_getmsgstring(&input_message)
  exec_simple_query(query_string)
  send_ready_for_query = true
```

`PqMsg_Query` 是协议消息边界，不是 SQL statement 边界。
一个 query string 可以包含空串、一个 SQL statement、多个 SQL statement，
也可能在 raw parse 阶段失败。

### `MessageContext`

`MessageContext` 是每轮 message loop 的短生命周期内存。
每轮读新消息前，backend 会切到并 reset 它。
raw parsetree、simple query 最后一个 statement 的 query/plan tree、
以及部分传给 unnamed portal 的对象，依赖这个 message 级生命周期。
多 statement 时，中间 statement 会使用 `per-parsetree message context`，
让前面 statement 的 query/plan tree 在本轮消息结束前释放。

### `xact_started`

`xact_started` 是 `postgres.c` 的 backend-local 标志。
它不是事务状态本身，只表示 tcop 层是否已经打开一个 transaction command。

```text
start_xact_command()
  if !xact_started:
    StartTransactionCommand()
    xact_started = true
  enable_statement_timeout()

finish_xact_command()
  disable_statement_timeout()
  if xact_started:
    CommitTransactionCommand()
    xact_started = false
```

真实事务状态在 `CurrentTransactionState->blockState`。
`xact_started` 只是 tcop 层避免重复 start/finish 的协调位。

### `TransactionState.blockState`

本节关键状态不是 XID，而是 transaction block state。

| 状态 | 本节语义 |
| --- | --- |
| `TBLOCK_DEFAULT` | backend idle，不在 transaction command 中。 |
| `TBLOCK_STARTED` | 单个 transaction command 已开始，但不是事务块。 |
| `TBLOCK_BEGIN` | 刚执行 `BEGIN`，等待 command 收尾后进入显式事务块。 |
| `TBLOCK_INPROGRESS` | 显式事务块内正常执行 command。 |
| `TBLOCK_IMPLICIT_INPROGRESS` | simple multi-statement 或 pipeline 创建的隐式事务块。 |
| `TBLOCK_END` | `COMMIT` 已标记，等待 `CommitTransactionCommand()` 真正提交。 |
| `TBLOCK_ABORT` | 显式事务块失败，等待用户 `ROLLBACK`。 |
| `TBLOCK_ABORT_END` | 已收到退出失败事务块的命令，等待 cleanup。 |

`BeginImplicitTransactionBlock()` 只在 `TBLOCK_STARTED` 时把状态改成
`TBLOCK_IMPLICIT_INPROGRESS`。
`EndImplicitTransactionBlock()` 只把它改回 `TBLOCK_STARTED`。
真正的 commit/abort 工作发生在后续 `CommitTransactionCommand()` 或
`AbortCurrentTransaction()`。

### `RawStmt`

`RawStmt` 是 raw parser 输出的 statement 容器。
它包含未做语义分析的 `stmt`，以及 `stmt_location` / `stmt_len`。
raw parse 只读输入字符串和 grammar，因此可以在 failed transaction block 中做。
parse analysis 会访问 catalog、类型、函数、relation 和权限状态，因此不能在 failed
transaction block 中继续普通命令。
这个分离让 backend 在 aborted transaction 中仍能识别 `COMMIT` / `ROLLBACK`。

### `Query`

`Query` 是 parse analysis 后的语义树。
本节关注 `commandType`、`queryId`、`canSetTag`、`utilityStmt`、`rtable`、
`targetList`。
`Query.commandType == CMD_UTILITY` 只有和 `utilityStmt`、Portal strategy、
`ProcessUtility()` 调用上下文一起看才有完整语义。

### `PlannedStmt`

`PlannedStmt` 是 planning 后交给 Portal 的 statement。
utility command 不进入普通 planner，而是被包装成 `CMD_UTILITY PlannedStmt`。
optimizable query 会走 `pg_plan_query() -> planner() -> planner_hook 或
standard_planner()`。
`PlannedStmt` 仍不是运行时状态。
运行时状态从 `PortalStart()` / `ExecutorStart()` 才开始。

### `Portal`

simple query 使用 unnamed portal：

```text
CreatePortal("", true, true)
PortalDefineQuery(...)
PortalStart(...)
PortalRun(...)
PortalDrop(...)
```

`PortalData` 的关键字段是 `portalContext`、`resowner`、`status`、
`sourceText`、`stmts`、`queryDesc`、`portalSnapshot`、`qc`。
simple query 的 unnamed portal 不出现在 `pg_cursors`，
但它仍然是 execution ownership 和 ERROR cleanup 的关键边界。

### `QueryDesc`

`QueryDesc` 是 Portal 与 executor 之间的运行句柄。
`CreateQueryDesc()` 提供 `plannedstmt`、`sourceText`、`snapshot`、`dest`、
`params`、`queryEnv` 和 instrumentation flags。
`ExecutorStart()` 后补齐 `tupDesc`、`estate`、`planstate`。
`ExecutePlan()` 会设置 `already_executed`。
读 `QueryDesc` 时要区分 caller input、executor runtime state 和执行推进状态。

### `DestReceiver`

输出不由 executor 直接写 libpq。
executor 只调用 `dest->rStartup()`、`dest->receiveSlot()`、`dest->rShutdown()`。
`exec_simple_query()` 创建 receiver，并在远端输出时把 portal 结果格式传给 receiver。
这让 executor tuple 流和协议输出解耦。

### `ReadyForQuery`

`ReadyForQuery()` 发送 `PqMsg_ReadyForQuery`。
协议 3.0 起，它包含 transaction status byte。
`I` 表示 idle，`T` 表示 in transaction，`E` 表示 in failed transaction。
状态码来自 `TransactionBlockStatusCode()`，
本质是 `TransactionState.blockState` 的协议投影。

## 5. 主流程源码 walkthrough

1. `PostgresMain()` 每轮 loop 开始先 reset `MessageContext`，
必要时发送 `ReadyForQuery`，然后 `ReadCommand()` 阻塞等待客户端消息。

2. 读到 `PqMsg_Query` 后，backend 设置 statement timestamp，
从消息中取出 `query_string`，调用 `exec_simple_query(query_string)`，
返回后设置 `send_ready_for_query = true`。

3. `SetCurrentStatementStartTimestamp()` 在 message 分派时设置。
如果一个 simple Query message 含多个 SQL statement，它们共享这次
statement timestamp。

4. `exec_simple_query()` 入口设置 `debug_query_string`，
调用 `pgstat_report_activity(STATE_RUNNING, query_string)`，
并触发 `TRACE_POSTGRESQL_QUERY_START`。

5. 因此 `pg_stat_activity.query` 显示整段 query string，
不是当前 raw statement 的切片。

6. raw parse 前先调用 `start_xact_command()`。
如果 `xact_started == false`，它会调用 `StartTransactionCommand()` 并置 true。

7. `StartTransactionCommand()` 在 `TBLOCK_DEFAULT` 下启动事务并进入
`TBLOCK_STARTED`。
如果已经在显式事务块内，它基本不启动新事务，只确保 context 切到
`CurTransactionContext`。

8. simple query 随后调用 `drop_unnamed_stmt()`。
这不是为了 simple query plan cache，而是把 simple query 定义得像使用 unnamed
statement 和 unnamed portal，从而回收前一次 unnamed 操作留下的存储。

9. raw parse 在 `MessageContext` 中执行：

```text
oldcontext = MemoryContextSwitchTo(MessageContext)
parsetree_list = pg_parse_query(query_string)
MemoryContextSwitchTo(oldcontext)
```

10. `pg_parse_query()` 调用 `raw_parser(query_string, RAW_PARSE_DEFAULT)`，
返回 `RawStmt` list。
raw parse 只做语法解析，不访问 catalog。

11. raw parse 后，`check_log_statement(parsetree_list)` 决定是否记录 statement。
这个位置早于 analyze/rewrite/plan，所以日志中出现 statement 不代表它已经进入 executor。

12. 源码用 `use_implicit_block = (list_length(parsetree_list) > 1)` 判断是否使用
implicit block。
多语句 simple Query message 默认作为同一事务执行。

13. `exec_simple_query()` 随后按 `RawStmt` 循环。
每个 raw statement 先计算 `CommandTag`，设置 ps display，并调用 `BeginCommand()`。

14. 一个 `RawStmt` 正常完成时对应一个 `EndCommand()`。
rewrite 后可能有多个 `PlannedStmt`，但 command completion 仍按原 raw statement 粒度发送。

15. 在 analyze 前先检查：

```text
if IsAbortedTransactionBlockState()
   and !IsTransactionExitStmt(parsetree->stmt):
  ERROR
```

16. 这个 guard 必须早于 parse analysis，
因为 parse analysis、rewrite、planning 会访问数据库对象。
失败事务块内只允许能退出该状态的事务控制命令继续。

17. 循环内再次调用 `start_xact_command()`。
第一次通常不会重复启动事务，因为 `xact_started` 已经是 true。

18. 如果前一个 statement 是 `COMMIT`、`ROLLBACK` 或类似事务控制语句，
前一轮可能已经 `finish_xact_command()`，
下一轮就需要重新打开 transaction command。

19. 如果 `use_implicit_block` 为 true，就调用 `BeginImplicitTransactionBlock()`。
它在 `TBLOCK_STARTED` 时进入 `TBLOCK_IMPLICIT_INPROGRESS`。

20. 每轮都调用 `BeginImplicitTransactionBlock()` 是故意的。
如果中间遇到 `COMMIT` / `ROLLBACK`，
后续普通 statement 仍要按当前状态重新判断是否需要 implicit block。

21. 如果 `analyze_requires_snapshot(parsetree)` 返回 true，
源码执行 `PushActiveSnapshot(GetTransactionSnapshot())` 并设置 `snapshot_set = true`。

22. `analyze_requires_snapshot()` 当前等价于 `stmt_requires_parse_analysis()`。
这份 snapshot 是给 parse analysis、rewrite 和 planning 使用的，不是 executor snapshot。

23. 计划完成后会 `PopActiveSnapshot()`。
源码注释明确说不能复用它给 execution，
因为它是在锁住 query 中表之前取得的，复用会产生用户可见异常。

24. 多 statement 中不是最后一个的 `RawStmt` 会创建
`per-parsetree message context`。
最后一个 statement 直接使用 `MessageContext`。
这是峰值内存控制。

25. simple query 使用 fixed params 路径：

```text
pg_analyze_and_rewrite_fixedparams(parsetree, query_string, NULL, 0, NULL)
```

26. 内部调用 `parse_analyze_fixedparams()`，
它会创建 `ParseState`，调用 `transformTopLevelStmt()`，
可选生成 query jumble，调用 `post_parse_analyze_hook`，
并通过 `pgstat_report_query_id()` 上报 query id。

27. simple query 这里没有 `$1` 参数类型推导。
参数化是 extended query 或 prepared statement 的主题。

28. parse analysis 返回 `Query` 后进入 `pg_rewrite_query()`。
utility command 直接成为单元素 query list。
普通 query 进入 `QueryRewrite()`。

29. rewrite 可能产生零个、一个或多个 `Query`。
rule、view、RLS、INSTEAD rule 都可能改变后续执行数量。
因此 `RawStmt` 粒度不等于 `Query` 粒度，也不等于 `PlannedStmt` 粒度。

30. 重写后的 query list 进入：

```text
pg_plan_queries(querytree_list, query_string, CURSOR_OPT_PARALLEL_OK, NULL)
```

31. utility command 被包装成 `CMD_UTILITY PlannedStmt`。
普通 query 调用 `pg_plan_query() -> planner()`，
再进入 `planner_hook` 或 `standard_planner()`。

32. `planner()` 返回后上报 plan id。
query id 在 parse analysis 后上报，plan id 在 planner 后上报。
每个 raw statement 开始前会把当前 query/plan id 清零。

33. 计划完成后创建 unnamed portal：

```text
portal = CreatePortal("", true, true)
portal->visible = false
PortalDefineQuery(portal, NULL, query_string, commandTag, plantree_list, NULL)
```

34. simple query 每个 raw statement 都用 unnamed portal 跑完即丢。
这里不深拷贝 query/plan tree，
因为它们位于 `MessageContext` 或 per-parsetree context，
会活到 portal 运行结束。

35. `PortalStart(portal, NULL, 0, InvalidSnapshot)` 选择 portal strategy。
常见策略包括 `PORTAL_ONE_SELECT`、`PORTAL_ONE_RETURNING`、
`PORTAL_ONE_MOD_WITH`、`PORTAL_UTIL_SELECT`、`PORTAL_MULTI_QUERY`。

36. 对 `PORTAL_ONE_SELECT`，`PortalStart()` 会 push execution snapshot，
创建 `QueryDesc`，调用 `ExecutorStart()`，把 `queryDesc` 和 `tupDesc` 保存到 portal，
随后 pop snapshot。

37. utility 和 multi-query 的真正执行通常在 `PortalRun()` 中发生。

38. simple query 默认 text result format。
如果当前 raw statement 是 `FETCH`，且目标 cursor 是 binary cursor，
`exec_simple_query()` 会把 format 改成 binary。

39. 然后创建 destination receiver：

```text
PortalSetResultFormat(portal, 1, &format)
receiver = CreateDestReceiver(dest)
SetRemoteDestReceiverParams(receiver, portal)
```

40. simple query 调用 `PortalRun(portal, FETCH_ALL, true, receiver, receiver, &qc)`。
`FETCH_ALL` 表示跑到完成，`isTopLevel = true` 表示来自客户端顶层命令。

41. `PortalRun()` 会 `MarkPortalActive(portal)`，
保存 `ActivePortal`、`CurrentResourceOwner`、`PortalContext`，
在 `PG_TRY` 中执行 strategy。
ERROR 时标记 portal failed 并恢复全局指针。

42. `PORTAL_ONE_SELECT` 进入 `PortalRunSelect()`。
如果有 `queryDesc`，它会设置 `queryDesc->dest`，
push `queryDesc->snapshot`，调用 `ExecutorRun()`，记录 processed 行数，再 pop snapshot。

43. `ExecutorRun()` 断言 active snapshot 等于 `estate->es_snapshot`。
executor 不自行决定事务 snapshot，
它要求 Portal 在正确边界激活。

44. `PORTAL_MULTI_QUERY` 进入 `PortalRunMulti()`。
对 plannable query，它 push copied transaction snapshot 并调用 `ProcessQuery()`；
同一 portal 的后续 query 使用 `UpdateActiveSnapshotCommandId()`。

45. `ProcessQuery()` 再走 `CreateQueryDesc()`、`ExecutorStart()`、`ExecutorRun()`、
`ExecutorFinish()`、`ExecutorEnd()`、`FreeQueryDesc()`。

46. 对 utility command，`PortalRunUtility()` 可能 push snapshot，
然后调用 `ProcessUtility()`，最后 pop snapshot。
因此 simple query 并不保证所有 statement 都进入 executor。

47. 进入 executor 后，链路是 `ExecutorStart()` 创建 `EState` 和 `PlanState`，
`ExecutorRun()` 调用 `ExecutePlan()`，
`ExecutorFinish()` drain 副作用，
`ExecutorEnd()` 注销 snapshot 并释放 executor context。

48. 本节只强调外层事实：
executor 的 snapshot、receiver、QueryDesc owner 和 cleanup 都来自 Portal/tcop。

49. `PortalRun()` 后，`exec_simple_query()` 调用 `receiver->rDestroy(receiver)`，
再调用 `PortalDrop(portal, false)`。

50. 如果当前 `RawStmt` 是最后一个，
源码先在需要时 `EndImplicitTransactionBlock()`，
再 `finish_xact_command()`。

51. 最后一个 statement 的 `finish_xact_command()` 发生在 `EndCommand()` 之前。
这样如果 end-of-transaction 报错，
客户端不会先收到成功 `CommandComplete` 再收到 commit error。

52. 如果当前 raw statement 是 `TransactionStmt` 且不是最后一个，
源码也立即 `finish_xact_command()`。
下一轮 statement 会重新 `start_xact_command()`。

53. 普通中间 statement 不提交。
它执行 `CommandCounterIncrement()` 和 `disable_statement_timeout()`。
`CommandCounterIncrement()` 让同一事务内后续 command 能看见前一 command 的效果。

54. 每个 raw parsetree 正常完成后调用 `EndCommand(&qc, dest, false)`。
`DataRow` 是 tuple 输出粒度，`CommandComplete` 是 raw statement 粒度，
`ReadyForQuery` 是 protocol cycle 粒度。

55. 空 query 没有 parsetree。
源码仍会 `finish_xact_command()` 并 `NullCommand(dest)`，
客户端收到 EmptyQueryResponse，再收到 `ReadyForQuery`。

56. `exec_simple_query()` 末尾处理 duration logging，
触发 `TRACE_POSTGRESQL_QUERY_DONE`，
并清空 `debug_query_string`。
ERROR path 中顶层 recovery 也会在报告错误后清空它。

## 6. 状态随时间推进的故事

以一条 simple Query message 为例：

```sql
CREATE TABLE sq_demo(a int);
INSERT INTO sq_demo VALUES (1);
SELECT * FROM sq_demo;
```

客户端只发送一个 `PqMsg_Query`。
backend 进入 `exec_simple_query()`，把整段文本放入 `debug_query_string`，
并把 `pg_stat_activity` 设置为 running。

`start_xact_command()` 让 `TBLOCK_DEFAULT` 进入 `TBLOCK_STARTED`。
`pg_parse_query()` 返回 3 个 `RawStmt`。
因为 list 长度大于 1，`use_implicit_block = true`。

第一个 `CREATE TABLE` 开始时调用 `BeginImplicitTransactionBlock()`，
`blockState` 进入 `TBLOCK_IMPLICIT_INPROGRESS`。
`CREATE TABLE` 是 utility。
它被 parse analysis 包装成 `CMD_UTILITY Query`，
planning 包装成 `CMD_UTILITY PlannedStmt`，
执行时进入 `ProcessUtility()`。

第一个 statement 不是最后一个，也不是事务控制 statement。
所以不会提交，只做 `CommandCounterIncrement()`。
第二个 `INSERT` 在同一 transaction 内执行。
由于 command id 已推进，它能看到前一条 DDL 建立的表。

第三个 `SELECT` 进入普通 planner/executor path。
`PortalStart()` 为它创建 `QueryDesc` 和 executor runtime state。
`PortalRunSelect()` 激活 execution snapshot 并调用 `ExecutorRun()`。

最后一个 statement 完成后执行：

```text
EndImplicitTransactionBlock()
finish_xact_command()
EndCommand()
```

`finish_xact_command()` 里的 `CommitTransactionCommand()` 真正提交事务。
`exec_simple_query()` 返回后，message loop 顶部发送 `ReadyForQuery('I')`。

如果第三个 statement ERROR，顶层 recovery 会 abort implicit transaction，
前两个 statement 的效果一起回滚。
如果这是显式事务块内的 ERROR，状态会停在 failed transaction block，
客户端随后看到 `ReadyForQuery('E')`。

这条故事说明：

```text
同一个 query string 可以有多个 command 边界，
多个 command 又可能共享一个 transaction boundary，
而 executor 只看见其中某些 PlannedStmt 的运行阶段。
```

## 7. 生命周期 / ownership / cleanup

query string 来自 frontend message buffer。
`debug_query_string` 指向它。
正常 path 在 `exec_simple_query()` 末尾清空。
ERROR path 在顶层 recovery 报告错误后清空。

raw parsetree list 分配在 `MessageContext`。
它活到本轮 message 处理结束。
下一轮 message loop reset `MessageContext` 时释放。

多 statement 中间项使用 `per-parsetree message context`。
正常顺序是 analyze/rewrite/plan、`PortalDefineQuery()`、`PortalStart()`、
`PortalRun()`、`PortalDrop()`、`MemoryContextDelete(per_parsetree_context)`。
最后一个 statement 直接放在 `MessageContext`。

tcop 层用 `xact_started` 管 start/finish 配对。
正常 path 是 `start_xact_command() -> StartTransactionCommand()`，
再由 `finish_xact_command() -> CommitTransactionCommand()` 收尾。
ERROR path 是 `AbortCurrentTransaction()` 后把 `xact_started = false`。
`AbortCurrentTransaction()` 根据 `TBLOCK_*` 状态决定直接回 idle，
还是停在 failed transaction block。

Portal 由 `CreatePortal()` 创建，由 `PortalDrop()` 正常释放。
`PortalRun()` 有局部 `PG_TRY`：active 时替换 global portal/resource owner/context，
ERROR 时 `MarkPortalFailed()` 并恢复全局指针。
顶层 ERROR recovery 还有 `PortalErrorCleanup()`。

`QueryDesc` 由 `CreateQueryDesc()` 创建，通常位于 portal context。
`ExecutorStart()` 创建 `EState` 和 `PlanState`，并注册 snapshot。
正常结束是 `ExecutorFinish()`、`ExecutorEnd()`、`FreeQueryDesc()`。
`ExecutorEnd()` 负责注销 snapshot 和释放 executor context。

本节至少有三类 snapshot。
analyze/rewrite/plan 的临时 snapshot 由 `exec_simple_query()` push/pop。
`PORTAL_ONE_SELECT` 的 execution snapshot 由 `PortalStart()` 获取并交给 executor 注册。
`PortalRunMulti()` 的 active snapshot 由 `PortalRunMulti()` push/update/pop。
不要把它们合并成“simple query 的 snapshot”。

`DestReceiver` 由 `exec_simple_query()` 创建。
executor 或 tuplestore path 调用它输出 tuple。
正常执行后 `receiver->rDestroy(receiver)` 释放 receiver 对象。

`MessageContext` 只释放内存。
Relation lock、buffer pin、snapshot ref、TupleDesc ref、portal resource owner
必须由 ResourceOwner、Portal、ExecutorEnd、transaction abort 等协议释放。

```text
MemoryContext cleanup 不能替代 ResourceOwner cleanup。
```

## 8. 正确性机制层次

raw parse 和 analyze 分离，目标是 failed transaction block 中仍能识别
`COMMIT` / `ROLLBACK`。
机制是 raw parse 不访问 catalog，aborted transaction guard 在 analyze 前执行，
`IsTransactionExitStmt()` 放行事务退出命令。
不能误解为所有 parsing 都能在 aborted transaction 中安全完成。

implicit block 的目标是让一个 simple Query message 中多个普通 statement
默认同事务提交或回滚。
机制是 list 长度大于 1 时启用，每个 statement 开始都调用
`BeginImplicitTransactionBlock()`，最后一个 statement 前调用
`EndImplicitTransactionBlock()`，中间普通 statement 只做 `CommandCounterIncrement()`。
它不是用户显式 `BEGIN`。

事务控制命令的目标是让 `BEGIN` / `COMMIT` / `ROLLBACK`
在 multi-statement string 中仍生效。
机制是 `TransactionStmt` 后立即 `finish_xact_command()`，
下一轮 statement 再 `start_xact_command()`，
由 `xact.c` 的 `TBLOCK_*` 状态决定真正行为。

snapshot 获取顺序的目标是让 analysis/planning 能访问一致 catalog，
execution 能在正确锁顺序下读取用户数据。
机制是 analyze/rewrite/plan 前临时 push transaction snapshot，planning 后立即 pop，
Portal/executor 阶段重新设置 execution snapshot。
planning snapshot 不能复用给 execution。

command counter 的目标是让同一事务中后续 command 能看见前一 command 的效果。
机制包括中间普通 statement 后 `CommandCounterIncrement()`，
`PortalRunMulti()` 的 `UpdateActiveSnapshotCommandId()`，
以及 `CommitTransactionCommand()` 在事务块内 command 收尾时推进 command counter。

command completion 顺序的目标是避免客户端先收到成功 `CommandComplete`，
再收到同一 command 的 commit error。
机制是最后一个 statement 先 `finish_xact_command()`，再 `EndCommand()`。

ReadyForQuery 状态码的目标是告诉客户端 backend 是否 idle、in transaction、
failed transaction。
机制是 message loop 顶部调用 `ReadyForQuery()`，
而 `ReadyForQuery()` 调用 `TransactionBlockStatusCode()`。
`ReadyForQuery` 不是 executor 完成标记。

## 9. 错误路径 / 异常路径 / fallback

raw parse ERROR 会 longjmp 到 `PostgresMain()` 的顶层 recovery。
recovery 会清空 `error_context_stack`，禁用 timeout，`pq_comm_reset()`，
`EmitErrorReport()`，清空 `debug_query_string`，`AbortCurrentTransaction()`，
`PortalErrorCleanup()`，清理 replication slot，`jit_reset_after_error()`，
切回 `MessageContext`，`FlushErrorState()`，并把 `xact_started = false`。
如果错误发生在读消息过程中，协议边界可能丢失，backend 会 FATAL 关闭连接。

failed transaction block 中的普通命令会在 analyze 前被拒绝。
例如 `BEGIN; SELECT 1/0; SELECT 1;` 中第三条的 raw parse 可以完成，
但 aborted transaction guard 会报错。
这避免 failed transaction block 中继续访问 catalog 或执行普通 query。

analyze/rewrite/plan ERROR 可能已经创建 snapshot、parser state、部分锁或 planner 内存。
cleanup 依赖 `AbortCurrentTransaction()`、`AtEOXact_Snapshot(false, true)`、
`ResourceOwnerRelease()` 和 `MessageContext` reset。
显式事务块内会进入 `TBLOCK_ABORT`，后续 `ReadyForQuery` 为 `E`。
implicit block 内会 abort 并回到 idle，后续 `ReadyForQuery` 通常为 `I`。

execution ERROR 可能来自 executor node、AM、trigger、表达式函数、DestReceiver
或 `ProcessUtility()`。
`PortalRun()` 会标记 portal failed 并恢复全局指针。
顶层 recovery 随后 abort transaction。
implicit block 中前面普通 statement 一起回滚；
显式 block 中事务保留 failed 状态等待 `ROLLBACK`。

commit 阶段 ERROR 发生在 `EndCommand()` 之前。
如果 `CommitTransactionCommand()` 失败，客户端只看到 ERROR，不会先看到成功 command tag。
可能来源包括 deferred constraint、AFTER trigger、commit callback、WAL flush
或更底层事务收尾逻辑。

多语句 simple query 中遇到 `TransactionStmt`，
当前 transaction command 会立即 finish。
这让显式 `BEGIN` / `COMMIT` / `ROLLBACK` 的语义优先于默认 implicit block。

空 query 没有 parsetree。
源码仍会 `finish_xact_command()` 并 `NullCommand(dest)`。
客户端收到 EmptyQueryResponse，再收到 `ReadyForQuery`。

如果 ERROR 发生时 `pq_is_reading_msg()` 为 true，
服务端无法确定前后消息边界，只能终止连接。
这是协议正确性 fallback，不是事务 cleanup 能解决的问题。

## 10. 成本、资源与跨模块传播

每个 simple Query message 通常都要走 raw parse、analyze、rewrite、plan、
Portal create/start/run/drop、transaction command start/finish。
除非 SQL 本身使用 server-side prepared statement，
simple protocol 不跨消息保存 named prepared statement 或 named portal。
大量短 SQL 的瓶颈可能在 executor 外。

成本随 query string 长度、raw statement 数量、catalog lookup 次数、
RTE / relation / partition 数量、rewrite rule / view / RLS 扩展数量、
planner search space、executor tuple 数量、客户端读取速度和网络 backpressure 增长。

多 statement 共享 `MessageContext`。
中间 statement 用 per-parsetree context 降低峰值，
但最后一个 statement 的 query/plan tree 通常留到 message 结束。
把很多复杂 SQL 拼成一条 simple Query message，
可能比逐条发送产生更高的单轮内存峰值。

analysis/planning 可能需要 transaction snapshot。
execution 又会设置自己的 snapshot。
在高并发系统中，snapshot 获取会连接到 ProcArray 成本。
这类成本不在单个 executor node 内。

rewrite 会放大工作量：

```text
rule / view / RLS
  -> extra Query
  -> extra planning
  -> PortalRunMulti
  -> extra executor 或 ProcessUtility work
```

simple query 每个 raw statement 创建并销毁 unnamed portal。
成本包括 portal context 分配、resource owner 切换、active portal 设置和恢复、
cleanup hook、receiver 创建销毁。
在复杂 query 中通常不显著；在极短 SQL workload 中可能可见。

返回大量行时，`DestReceiver`、libpq buffer、socket flush 和客户端读取速度会影响总耗时。
`ReadyForQuery()` 会 flush 输出。
慢客户端可能让 backend 看起来卡在 Client wait 或 socket 输出路径。

跨模块边界：

| 模块 | 本节边界 |
| --- | --- |
| protocol | `PqMsg_Query`、`CommandComplete`、`ReadyForQuery`。 |
| tcop | `exec_simple_query()` 组织主链路。 |
| xact | transaction command、implicit block、commit/abort。 |
| memory | `MessageContext`、per-parsetree context、portal context。 |
| parser/analyzer | raw parse 与语义分析分离。 |
| rewrite | 一个 statement 可能扩展成多个 query。 |
| planner | optimizable query 生成 plan，utility 只包装。 |
| portal | runtime owner、snapshot、cleanup。 |
| executor | `QueryDesc`、`EState`、tuple loop。 |
| dest | tuple 输出和 protocol completion。 |
| stats/log | activity、query id、plan id、duration、阶段 stats。 |

## 11. 观测与诊断入口

`pg_stat_activity` 能看到整段 query string、backend state 和等待事件：

```sql
SELECT pid, state, query, wait_event_type, wait_event
FROM pg_stat_activity
WHERE pid = pg_backend_pid();
```

它看不到当前是第几个 `RawStmt`，也看不到 implicit block 是否开启。

server log 可用入口：

```sql
SET log_statement = 'all';
SET log_duration = on;
SET log_parser_stats = on;
SET log_planner_stats = on;
SET log_executor_stats = on;
```

这些能粗略拆出 parser、planner、executor 和总 duration。
它们开销高，适合测试或短时诊断。

测试环境可用 debug print：

```sql
SET client_min_messages = log;
SET debug_print_parse = on;
SET debug_print_rewritten = on;
SET debug_print_plan = on;
```

这能看到 parse/rewrite/plan 层级，但日志量很大。

libpq 可以通过 transaction status 看到 `PQTRANS_IDLE`、`PQTRANS_INTRANS`、
`PQTRANS_INERROR`。
它们对应 `ReadyForQuery` 的 `I`、`T`、`E`。
psql prompt 可以用 `%x` 显示事务状态。

建议 gdb 断点：

```text
exec_simple_query
start_xact_command
finish_xact_command
BeginImplicitTransactionBlock
EndImplicitTransactionBlock
pg_parse_query
parse_analyze_fixedparams
QueryRewrite
planner
PortalStart
PortalRun
ProcessQuery
ExecutorStart
ExecutorRun
ReadyForQuery
AbortCurrentTransaction
```

关键变量：

```text
query_string
list_length(parsetree_list)
use_implicit_block
xact_started
CurrentTransactionState->blockState
commandTag
snapshot_set
portal->strategy
portal->status
queryDesc->estate
qc->commandTag
```

大量短 SQL 时，perf / flamegraph 可能显示热点在 scanner/parser、
parse analysis catalog lookup、planner、portal/context 创建销毁、
transaction start/commit、libpq output。
不要只从 executor node 解释所有慢 SQL。

观测边界：

| 状态 | 直接观测 | 入口 |
| --- | --- | --- |
| query string | 能 | `pg_stat_activity.query`、log。 |
| raw statement 数量 | 否 | gdb、debug print。 |
| implicit block | 否 | gdb 打印 `use_implicit_block` / `blockState`。 |
| transaction status | 能 | ReadyForQuery、libpq、psql prompt。 |
| analyze snapshot | 几乎不可见 | gdb。 |
| execution snapshot | 间接 | gdb、MVCC 现象。 |
| Portal strategy | 否 | gdb。 |
| executor rows/time | 能 | EXPLAIN ANALYZE、hooks、stats。 |

## 12. 常见误区

误区一：simple query 一次只能执行一条 SQL。
实际是一条 `PqMsg_Query` 可以包含多个 SQL statement。

误区二：多语句 simple query 每条自动提交。
实际默认使用 implicit transaction block。

误区三：implicit block 等同于用户显式 `BEGIN`。
实际它只是 `xact.c` 中的内部 block state。

误区四：raw parse、parse analysis、planning 可以统称一个 parse 阶段。
实际 raw parse 可以在 failed transaction 中做，analysis 和 planning 通常不可以。

误区五：planning snapshot 可以复用给 execution。
源码明确避免这样做，因为获取时机和锁顺序会造成用户可见异常。

误区六：Portal 只是 cursor。
simple query 的 unnamed portal 不可见，但仍是 execution ownership 和 cleanup 边界。

误区七：`CommandComplete` 表示整个协议轮次结束。
实际 `ReadyForQuery` 才表示 backend 可接收下一轮 query。

误区八：`ReadyForQuery('I')` 说明上一条 SQL 没出错。
它只说明当前事务块状态是 idle；是否出错要看前面的 ErrorResponse。

误区九：MemoryContext reset 可以替代 ResourceOwner cleanup。
内存、snapshot ref、relation lock、buffer pin 是不同资源。

误区十：所有慢 SQL 都能从 executor node 解释。
simple query 的固定成本可能主要在 executor 之外。

## 13. 课堂实验

### 实验 1：证明 multi-statement simple query 的 implicit transaction

先清理：

```bash
psql -X -c "DROP TABLE IF EXISTS sq_implicit"
```

发送一条 simple Query message：

```bash
psql -X -v ON_ERROR_STOP=0 -c "CREATE TABLE sq_implicit(a int); INSERT INTO sq_implicit VALUES (1); SELECT 1/0;"
```

检查：

```bash
psql -X -c "SELECT to_regclass('public.sq_implicit')"
```

预期：表不存在。
解释：三个 statement 在同一 `PqMsg_Query` 中，`use_implicit_block = true`，
第三个 statement ERROR，`AbortCurrentTransaction()` 回滚整个 implicit block。

对照逐条发送：

```bash
psql -X -c "CREATE TABLE sq_implicit(a int)"
psql -X -c "INSERT INTO sq_implicit VALUES (1)"
psql -X -v ON_ERROR_STOP=0 -c "SELECT 1/0"
psql -X -c "TABLE sq_implicit"
```

预期：表存在且有一行。

### 实验 2：观察 failed transaction block

在 psql 中：

```sql
\set PROMPT1 '%/%R%x%# '
BEGIN;
SELECT 1/0;
SELECT 1;
ROLLBACK;
```

观察：错误后 prompt 显示 failed transaction 状态；
`SELECT 1` 在 analyze 前被拒绝；
`ROLLBACK` 被 raw parse 识别为 transaction exit statement；
`ROLLBACK` 后回到 idle。

源码对应：

```text
IsAbortedTransactionBlockState()
TransactionBlockStatusCode()
```

### 实验 3：拆 parse/rewrite/plan 阶段

测试环境执行：

```sql
SET client_min_messages = log;
SET debug_print_parse = on;
SET debug_print_rewritten = on;
SET debug_print_plan = on;
SELECT * FROM pg_class WHERE oid = 'pg_class'::regclass;
```

再执行：

```sql
CREATE TEMP TABLE sq_dbg(a int);
```

比较普通 SELECT 和 utility command 的路径：
SELECT 有 plan tree；utility 被包装成 `CMD_UTILITY PlannedStmt`；
utility 执行进入 `ProcessUtility()`。

## 14. 讨论题

1. 为什么 raw parse 必须和 parse analysis 分开？
2. 多语句 simple query 为什么默认需要 implicit transaction block？
3. `BeginImplicitTransactionBlock()` 为什么只改 block state，而不直接 commit 或 start 完整事务？
4. planning snapshot 为什么不能复用给 execution？
5. `CommandCounterIncrement()` 在多语句 simple query 中解决什么可见性问题？
6. 为什么最后一个 statement 要先 `finish_xact_command()` 再 `EndCommand()`？
7. unnamed portal 不可见，为什么 simple query 仍需要 Portal？
8. rewrite 产生多个 query 后，`CommandComplete` 应该按哪个粒度发送？
9. executor 中 ERROR 后，哪些 cleanup 属于 Portal，哪些属于 transaction abort？
10. `ReadyForQuery('E')` 能定位 executor 中哪一步失败吗？
11. 大量短 SQL 为什么可能瓶颈在 parser/planner/transaction/protocol？
12. simple query 和 extended query 最容易混淆的事务边界是什么？

## 15. 本节小结

本节主链路：

```text
PostgresMain()
  -> PqMsg_Query
  -> exec_simple_query()
  -> start_xact_command()
  -> pg_parse_query()
  -> for each RawStmt:
       BeginCommand()
       start_xact_command()
       BeginImplicitTransactionBlock()
       PushActiveSnapshot() for analyze/plan if needed
       pg_analyze_and_rewrite_fixedparams()
       pg_plan_queries()
       PopActiveSnapshot()
       CreatePortal("")
       PortalDefineQuery()
       PortalStart()
       PortalRun()
       PortalDrop()
       finish_xact_command() or CommandCounterIncrement()
       EndCommand()
  -> ReadyForQuery()
```

核心边界：

- `PqMsg_Query` 是协议消息边界。
- `RawStmt` 是 SQL command 循环边界。
- `xact_started` 是 tcop 层 transaction command 协调标志。
- `TransactionState.blockState` 是真实事务块状态。
- `MessageContext` 是 message 生命周期内存边界。
- `Portal` 是 plan/executor/utility 的运行和 cleanup 容器。
- `QueryDesc` 是 Portal 和 executor 之间的运行句柄。
- `ReadyForQuery` 是协议轮次完成和事务状态上报边界。

ERROR path 的收尾：raw parse 失败也回到顶层 recovery；
failed transaction block 中只允许 transaction exit statement 继续；
Portal ERROR 会标记 failed 并恢复全局指针；
transaction abort 释放 ResourceOwner、snapshot、lock、buffer pin 等资源；
MessageContext reset 只负责短生命周期内存；
ReadyForQuery 报告恢复后的事务块状态。

能观测的是 `pg_stat_activity`、server log、debug print、ReadyForQuery transaction status、
gdb 断点和 profile 栈。
不能直接观测的是当前 raw statement 序号、implicit block 开关、
analysis snapshot 与 execution snapshot 的具体差异、rewrite 后完整 causality。

可迁移规律：

```text
当外部协议允许一个粗粒度请求携带多个内部操作时，
内核不能把协议边界直接当成事务、资源或执行边界；
它必须建立嵌套状态机，
让每个内部操作有自己的 command、snapshot、owner 和 cleanup，
同时在外层协议上仍表现为一次完整请求。
```

本节判断依赖协议模式、SQL 形态、显式事务控制、rewrite 规则、扩展 hook、
短 SQL 比例和客户端读取速度。

稳定 mental model：

```text
protocol message
  != SQL statement
  != transaction command
  != Portal execution
  != executor node loop
```
