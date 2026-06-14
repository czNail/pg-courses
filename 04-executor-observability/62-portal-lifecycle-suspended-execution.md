# PostgreSQL Portal lifecycle / suspended execution

## 课程定位
前置知识：已经理解 PostgreSQL backend message loop、simple query、extended query、prepared statement、`ExecutorStart()` / `ExecutorRun()` / `ExecutorEnd()` 的基本边界。
本节唯一主问题：
```text
Portal 如何支持 cursor、FETCH、暂停执行、holdable cursor 和 ERROR cleanup？
```
核心矛盾：
```text
executor 希望一次 query 有明确 start/run/end，
但 cursor、extended Execute(max_rows)、FETCH/MOVE、WITH HOLD 和 ERROR recovery
要求同一个可执行对象跨调用、跨协议消息，甚至跨事务边界保存状态。
```
学完后应能判断：
- 一个现象属于 protocol-level portal、SQL cursor、executor、tuplestore 还是 transaction cleanup。
- 哪些 portal 能暂停 executor，哪些只是把结果 materialize 后分批返回。
- 为什么 `PORTAL_ACTIVE -> PORTAL_READY` 是合法的 suspended execution。
- 为什么 `WITH HOLD` 保留的是 materialized result，不是活 executor。
- ERROR 后哪些 cleanup 能正常执行，哪些必须交给 transaction abort 和 `ResourceOwner` 兜底。
本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e90741e12a3715909e1b2d71ff9344`。
本节不展开 plan cache 的 generic/custom plan 选择，也不展开 `DestReceiver` 如何编码 `DataRow`。这些只在解释 Portal 边界时出现。

## 1. 本节在总主线中的位置
04 目录前面的 executor 主线是：
```text
QueryDesc
  -> ExecutorStart
  -> ExecutorRun
  -> ExecutorFinish
  -> ExecutorEnd
```
57 和 58 节把视角上移到协议层：
```text
PostgresMain()
  -> simple Query message
  -> extended Parse / Bind / Execute / Sync
  -> PortalRun()
```
本节站在 protocol 和 executor 之间。
Portal 不是 executor node。
Portal 也不是 wire protocol message。
Portal 是 backend-local session object。
它把 planned statement、参数、result format、snapshot、executor state、cursor position、tuplestore 和 cleanup hook 组合成一个可运行句柄。
如果只读 executor，会误以为 `ExecutorRun()` 总是跑完整棵计划树。
如果只读 protocol，会误以为 `Execute(max_rows)` 只是客户端限流。
真实链路是：
```text
Bind / DECLARE CURSOR / simple query
  -> CreatePortal()
  -> PortalDefineQuery()
  -> PortalStart()
  -> PortalRun() 或 PortalRunFetch()
  -> PortalRunSelect() 或 RunFromStore()
  -> PortalDrop() / PreCommit_Portals() / AtAbort_Portals()
```
本节只围绕 Portal lifecycle / suspended execution。
后续如果继续向外看，应追 `DestReceiver` 输出协议。
如果继续向内看，应追 executor node 如何在多次 `ExecutorRun()` 之间保留 node-local 位置。

## 2. 核心矛盾与一句话运行模型
一句话运行模型：
```text
Portal 把一个可执行查询变成 backend-local 状态机；
只有单个 SELECT 型 portal 保留活 executor 并按 count/FETCH 分批推进；
其他会返回 tuple 的策略通常先 run to completion 并写入 holdStore；
COMMIT、ERROR 和 CLOSE 再按 Portal、ResourceOwner、MemoryContext 分层收尾。
```
这里有三个 tension。
第一，执行边界：
```text
QueryDesc/EState/PlanState 需要清晰 start/end
  vs
cursor 和 Execute(max_rows) 需要“跑一段、停下来、稍后继续”
```
第二，事务边界：
```text
普通 cursor 依赖当前事务里的 snapshot、lock、ResourceOwner
  vs
WITH HOLD cursor 要在 COMMIT 后继续读取结果
```
第三，错误边界：
```text
cleanup 需要释放 executor、snapshot、cached plan、temp file 和 memory
  vs
ERROR 后不能假设 executor shutdown 本身仍然安全
```
PostgreSQL 没有把 executor 设计成全局可恢复协程。
它把复杂性压到 Portal 状态机和 tuplestore 边界。
`PORTAL_ONE_SELECT` 可以增量调用 executor。
`PORTAL_ONE_RETURNING`、`PORTAL_ONE_MOD_WITH`、`PORTAL_UTIL_SELECT` 会先物化结果。
`PORTAL_MULTI_QUERY` 不支持 partial execution。
`WITH HOLD` 在 commit 前把结果导入 `holdStore`。
ERROR 路径把 portal 标记为 failed，防止不完整状态再次运行。
因此本节的核心不是背字段，而是跟状态随时间推进。

## 3. 核心文件分工与阅读顺序
| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/utils/portal.h` | `PortalData`、`PortalStatus`、`PortalStrategy` 的语义边界。 |
| 2 | `src/backend/utils/mmgr/portalmem.c` | `CreatePortal()`、`PortalDefineQuery()`、`PortalDrop()`、commit/abort cleanup。 |
| 3 | `src/backend/tcop/pquery.c` | `PortalStart()`、`PortalRun()`、`PortalRunSelect()`、`PortalRunFetch()`、strategy dispatch。 |
| 4 | `src/backend/commands/portalcmds.c` | `DECLARE CURSOR`、`FETCH/MOVE`、`CLOSE`、`PersistHoldablePortal()`。 |
| 5 | `src/backend/tcop/postgres.c` | simple unnamed portal、extended Bind/Execute、`PortalSuspended`、top-level error recovery。 |
| 6 | `src/backend/executor/execMain.c` | `ExecutorRun()` 的 direction/count 边界，由 Portal 多次调用。 |
| 7 | `src/backend/executor/spi.c` | SPI cursor、pinned portal、procedure 内 transaction control。 |
| 8 | `src/include/tcop/pquery.h` | Portal run/fetch 对外声明。 |
| 9 | `src/include/commands/portalcmds.h` | SQL cursor command 对 Portal 层的调用边界。 |
| 10 | `src/backend/executor/tstoreReceiver.c` | tuplestore receiver 如何接住 materialized result。 |
推荐读法：先读 `portal.h` 的状态和策略，再读 `portalmem.c` 的 ownership，最后读 `pquery.c` 的 run/fetch 主链路。
不要从 `ExecutorRun()` 开始背调用栈。
本节的问题是“谁保存可继续执行的状态、谁决定本次取多少行、失败后谁负责收尾”。
源码里有一些历史痕迹必须保留。
simple query 也创建 unnamed portal。
extended `Bind` 创建 portal，`Execute` 才运行。
SQL `DECLARE CURSOR` 创建 portal，但不立刻取结果。
`RETURNING` 的结果可以分批读，但 DML 本身不能暂停在一半。
`PreCommit_Portals()` 可能运行用户定义代码来持久化 holdable cursor。
FAILED portal 的 cleanup 会避免再次执行可能失败的 executor shutdown。

## 4. 关键数据结构与状态

### 4.1. Portal 是 backend-local 状态
`PortalData` 分配在当前 backend 的内存中。
其他 backend 不能直接访问这个指针。
Portal 名称存在当前 backend 的 `PortalHashTable`。
`pg_cursors` 也只展示当前 session 可见的 SQL cursor。
所以 cursor 不是跨连接对象。
named portal 也不是 shared memory handle。
protocol-level unnamed portal 的名字是空串。
SQL cursor 不允许空名字，因为会和 unnamed portal 冲突。
内存关系可以压缩成：
```text
TopMemoryContext
  -> TopPortalContext
     -> PortalData
     -> PortalContext
     -> PortalHoldContext   (仅 holdStore 存在时)
```
`CreatePortal()` 在 `TopPortalContext` 中分配 `PortalData`。
它同时为 portal 创建 `portalContext`。
`portalContext` 可能跨多个 protocol message 存活。
它不是 `MessageContext` 那种每轮消息 reset 的短命对象。

### 4.2. `PortalStatus`
| 状态 | 语义 |
| --- | --- |
| `PORTAL_NEW` | 刚创建，还没有 query。 |
| `PORTAL_DEFINED` | `PortalDefineQuery()` 已保存 source text、command tag、计划列表。 |
| `PORTAL_READY` | `PortalStart()` 完成，可以运行或继续运行。 |
| `PORTAL_ACTIVE` | 正在运行，不允许普通 drop。 |
| `PORTAL_DONE` | 已完成，不能再运行。 |
| `PORTAL_FAILED` | 运行或 cleanup 出错，不能再运行。 |
状态机的特殊点是：
```text
READY -> ACTIVE -> READY
```
这条回边就是 suspended execution。
单 SELECT portal 一次 `PortalRun()` 只取 `count` 行时，如果还没到末尾，`PortalRun()` 返回 `false`，portal 从 `ACTIVE` 回到 `READY`。
`DONE` 和 `FAILED` 是终结状态。
不要直接写 `portal->status = PORTAL_ACTIVE`。
源码要求使用 `MarkPortalActive()`、`MarkPortalDone()`、`MarkPortalFailed()`，因为它们同时处理状态检查、subtransaction activity 和 cleanup hook。

### 4.3. `PortalStrategy`
| 策略 | 是否真暂停 executor | 典型来源 | 关键理由 |
| --- | --- | --- | --- |
| `PORTAL_ONE_SELECT` | 是 | `SELECT`、普通 cursor | 活 executor 可保留位置并继续拉取。 |
| `PORTAL_ONE_RETURNING` | 否 | DML `RETURNING` | DML 副作用和 trigger 必须完整执行。 |
| `PORTAL_ONE_MOD_WITH` | 否 | data-modifying CTE | 可能有副作用和 trigger。 |
| `PORTAL_UTIL_SELECT` | 否 | `EXPLAIN`、`SHOW` 等 | utility 先执行，再从 tuplestore 读结果。 |
| `PORTAL_MULTI_QUERY` | 否 | 多语句或 rewrite 后多计划 | 第一次运行必须到完成。 |
`ChoosePortalStrategy()` 在 `pquery.c` 中根据 `stmts`、`canSetTag`、`commandType`、`hasModifyingCTE`、`hasReturning` 和 utility 是否返回 tuple 选择策略。
这解释了一个常见现象：
```text
FETCH 可以分批返回 UPDATE ... RETURNING 的结果，
但 UPDATE 本身没有被暂停。
```
实际路径是先 `FillPortalStore()` 跑完 DML，再从 `holdStore` 分批返回。

### 4.4. query、plan 和 params
Portal 保存：
| 字段 | 语义 |
| --- | --- |
| `sourceText` | 原始 query 文本，用于执行、错误上下文、监控展示。 |
| `commandTag` | 原始 query 对外完成标识。 |
| `qc` | command completion 数据。 |
| `stmts` | `PlannedStmt` list。 |
| `cplan` | 来自 plan cache 时保存 `CachedPlan` 引用。 |
| `prepStmtName` | 来源 prepared statement 名字。 |
| `portalParams` | Bind 或 cursor 创建时绑定的参数。 |
| `queryEnv` | query environment。 |
`PortalDefineQuery()` 只保存这些信息。
源码注释强调它不能做复杂且可能 ERROR 的事。
原因是 caller 可能刚通过 `GetCachedPlan()` 增加 refcount。
如果 refcount 交给 portal 前抛错，就会泄漏 cached plan 引用。
extended Bind 因此遵守这个顺序：
```text
copy query string / statement name / params into portalContext
GetCachedPlan()
PortalDefineQuery()
```
raw field 不是语义。
`stmts` 必须结合 `strategy`、`cplan` 和 context lifetime 才能解释。

### 4.5. executor、tuplestore 和 position
`queryDesc` 是 Portal 和 executor 的运行句柄。
只有 `PORTAL_ONE_SELECT` 在 `PortalStart()` 中立即：
```text
CreateQueryDesc()
ExecutorStart()
portal->queryDesc = queryDesc
portal->tupDesc = queryDesc->tupDesc
```
`PORTAL_ONE_RETURNING`、`PORTAL_ONE_MOD_WITH`、`PORTAL_UTIL_SELECT` 在 `PortalStart()` 中只确定 `tupDesc`。
它们第一次运行时由 `FillPortalStore()` 创建 `holdStore` 并把结果写进去。
`queryDesc != NULL` 表示还有活 executor 需要 shutdown。
`holdStore != NULL` 表示结果可以从 tuplestore 读取。
二者不是等价状态。
cursor position 用三个字段表达：
| 字段 | 语义 |
| --- | --- |
| `atStart` | 位于第一行之前。 |
| `atEnd` | 已越过结果末尾。 |
| `portalPos` | 名义位置，0 是第一行之前，N 是取到第 N 行之后。 |
`portalPos` 不能单独解释。
`atStart` 推出 `portalPos == 0`，但 `portalPos == 0` 不必然表示 `atStart`。
backward fetch、`FETCH 0` 和 endpoint 调整都会让单字段判断出错。

### 4.6. holdStore、snapshot、ResourceOwner
`PortalCreateHoldStore()` 创建 `holdContext` 和 `holdStore`：
```text
AllocSetContextCreate(TopPortalContext, "PortalHoldContext", ...)
tuplestore_begin_heap(randomAccess, true, work_mem)
```
第二个参数 `true` 表示可使用跨事务临时文件。
因此 `PortalDrop()` 即使在 error 条件下也要 `tuplestore_end()`。
`holdContext` 不是 `portalContext` 的子 context。
holdable cursor 持久化后会清理 `portalContext` 的子 context，但 `holdStore` 仍要存在。
Portal 有两个容易混淆的 snapshot 字段：
| 字段 | 语义 |
| --- | --- |
| `portalSnapshot` | portal 外层 active snapshot，主要用于 utility / 非原子执行边界。 |
| `holdSnapshot` | 保护 holdStore 中可能含 TOAST 引用的数据。 |
普通 `PORTAL_ONE_SELECT` 运行时主要使用 `queryDesc->snapshot`。
每次 `PortalRunSelect()` 调 executor 前 `PushActiveSnapshot(queryDesc->snapshot)`，运行后 `PopActiveSnapshot()`。
holdable cursor 的 `PersistHoldablePortal()` 通过 tuplestore receiver 强制 detoast，因此 held cursor 不需要长期保留原创建事务 snapshot 来保护 TOAST 数据。
每个 portal 还有自己的 `resowner`：
```text
ResourceOwnerCreate(CurTransactionResourceOwner, "Portal")
```
MemoryContext 管内存。
ResourceOwner 管 snapshot、pin、外部资源和 abort cleanup。
CachedPlan refcount 管 plan cache 引用。
PortalStatus 管可运行语义。
这四者不能互相替代。

### 4.7. cleanup hook
标准 cleanup hook 是 `PortalCleanup()`，定义在 `portalcmds.c`。
它看到 `portal->queryDesc` 时会先把 `portal->queryDesc = NULL`，再在非 FAILED 状态下执行：
```text
ExecutorFinish()
ExecutorEnd()
FreeQueryDesc()
```
先清空 `queryDesc` 是为了防止 cleanup 自己出错后再次尝试 shutdown 同一个 executor。
FAILED portal 不走正常 executor shutdown。
因为 ERROR 可能发生在 expression、trigger、FDW callback、receiver 或 shutdown 前的副作用 drain 中。
继续调用完整 shutdown 可能制造第二个 ERROR。
FAILED 路径改为依赖 transaction abort、ResourceOwner release 和 memory context cleanup。

## 5. 主流程源码 walkthrough

### 5.1. simple query 创建 unnamed portal
simple query 在 `postgres.c` 的 `exec_simple_query()` 中创建 unnamed portal：
```text
exec_simple_query()
  -> pg_analyze_and_rewrite_fixedparams()
  -> pg_plan_queries()
  -> CreatePortal("", true, true)
  -> PortalDefineQuery()
  -> PortalStart()
  -> PortalSetResultFormat()
  -> CreateDestReceiver()
  -> PortalRun(..., FETCH_ALL, ...)
  -> PortalDrop()
```
这个 portal `visible=false`，不会显示在 `pg_cursors`。
simple query 通常不利用暂停能力。
它创建 Portal 是为了统一 command tag、result format、executor/utility dispatch 和 cleanup 边界。

### 5.2. extended Bind 创建 protocol-level portal
extended query 的 `Parse` 创建 prepared statement，`Bind` 创建 portal：
```text
exec_bind_message()
  -> GetPreparedStatement()
  -> CreatePortal(portal_name, allowDup, dupSilent)
  -> copy params into portalContext
  -> GetCachedPlan()
  -> PortalDefineQuery()
  -> PortalStart()
  -> PortalSetResultFormat()
  -> BindComplete
```
unnamed portal 可以静默替换。
named portal 不允许重复创建。
`Bind` 不执行查询。
它只让 portal 进入 `PORTAL_READY`。
真正推进发生在 `Execute` message。

### 5.3. extended Execute 触发 suspended execution
`exec_execute_message(portal_name, max_rows)` 的主线是：
```text
GetPortalByName()
CreateDestReceiver(DestRemoteExecute)
start_xact_command()
completed = PortalRun(portal, max_rows, true, receiver, receiver, &qc)
if completed:
  EndCommand()
else:
  send PortalSuspended
```
`max_rows <= 0` 会转换为 `FETCH_ALL`。
`PortalRun()` 返回 `false` 时，backend 向客户端发送 `PortalSuspended`。
这不是 `CommandComplete`。
它表示这个 protocol-level portal 还可以继续 `Execute`。
再次执行同一个 portal 时，`exec_execute_message()` 用 `!portal->atStart` 判断日志上是否是 `execute fetch from`。
没有 Portal 保存 `queryDesc`、position 和 result format，`Execute(max_rows)` 就只能重新执行 query 并丢弃前面行，那会破坏语义和成本。

### 5.4. SQL DECLARE CURSOR 创建 named portal
SQL cursor 入口是 `PerformCursorOpen()`：
```text
PerformCursorOpen()
  -> check cursor name
  -> RequireTransactionBlock() for non-HOLD cursor
  -> QueryRewrite()
  -> pg_plan_query()
  -> CreatePortal(cursor_name, false, false)
  -> copy plan/query text/params into portalContext
  -> PortalDefineQuery()
  -> choose SCROLL or NO SCROLL
  -> PortalStart(..., GetActiveSnapshot())
```
`DECLARE CURSOR` 的 query 必须是 SELECT。
rewrite 后也必须只有一个 SELECT query。
非 holdable cursor 必须在 transaction block 中声明。
否则 statement 结束后事务资源立即释放，cursor 没有可见效果。
`DECLARE CURSOR WITH HOLD` 会在 commit 前转成 held cursor。
`PerformCursorOpen()` 不取结果。
真正取结果发生在 `FETCH` 或 `MOVE`。

### 5.5. PortalStart 从 defined 变 ready
`PortalStart()` 要求 `portal->status == PORTAL_DEFINED`。
它保存 `ActivePortal`、`CurrentResourceOwner`、`PortalContext`，然后在 `PG_TRY()` 中切换到当前 portal：
```text
ActivePortal = portal
CurrentResourceOwner = portal->resowner
PortalContext = portal->portalContext
MemoryContextSwitchTo(PortalContext)
```
这一步决定后续 startup 内存和资源挂在哪个 owner 下。
随后 `ChoosePortalStrategy()` 决定 strategy。
`PORTAL_ONE_SELECT` 路径：
```text
PushActiveSnapshot(snapshot or GetTransactionSnapshot())
CreateQueryDesc(..., None_Receiver, ...)
ExecutorStart(queryDesc, eflags)
portal->queryDesc = queryDesc
portal->tupDesc = queryDesc->tupDesc
portal->atStart = true
portal->atEnd = false
portal->portalPos = 0
PopActiveSnapshot()
```
`CreateQueryDesc()` 初始 destination 是 `None_Receiver`。
真正输出目标会在每次 `PortalRunSelect()` 中重新设置。
这支持同一个 portal 被 `FETCH`、`MOVE` 或不同 protocol receiver 消费。
`PORTAL_ONE_RETURNING` 和 `PORTAL_ONE_MOD_WITH` 只确定 `tupDesc`。
`PORTAL_UTIL_SELECT` 通过 `UtilityTupleDescriptor()` 确定 `tupDesc`。
`PORTAL_MULTI_QUERY` 暂时不需要 `tupDesc`。
如果 `PortalStart()` 过程中 ERROR，catch block 会 `MarkPortalFailed()`，恢复全局指针并重新抛错。

### 5.6. PortalRun 按 strategy 推进
`PortalRun()` 的骨架：
```text
PortalRun()
  -> InitializeQueryCompletion()
  -> MarkPortalActive()
  -> save top-level ResourceOwner/MemoryContext/ActivePortal
  -> switch portal->strategy
  -> restore globals
  -> return completed
```
`MarkPortalActive()` 是运行前守门。
portal 必须是 `READY`。
`PORTAL_ONE_SELECT` 直接：
```text
PortalRunSelect(portal, true, count, dest)
portal->status = PORTAL_READY
result = portal->atEnd
```
如果没有到末尾，`result=false`，上层 extended protocol 发送 `PortalSuspended`。
`PORTAL_ONE_RETURNING`、`PORTAL_ONE_MOD_WITH`、`PORTAL_UTIL_SELECT` 第一次运行时如果没有 `holdStore`，先 `FillPortalStore()`，再 `PortalRunSelect()`。
它们对客户端看起来可以分批返回，但 query 副作用已经完成。
`PORTAL_MULTI_QUERY` 调 `PortalRunMulti()`，然后 `MarkPortalDone()`，总是返回 complete。

### 5.7. PortalRunSelect 执行 N rows forward/backward
`PortalRunSelect()` 处理两种数据源：
```text
queryDesc != NULL  -> 活 executor
holdStore != NULL  -> materialized result
```
它每次都把 `queryDesc->dest` 改成本次 `dest`。
源码注释明确说 executor 不能假设 destination 永远不变。
forward 简化逻辑：
```text
if atEnd or count <= 0:
  direction = NoMovement
else:
  direction = Forward
if count == FETCH_ALL:
  count = 0
if holdStore:
  nprocessed = RunFromStore()
else:
  PushActiveSnapshot(queryDesc->snapshot)
  ExecutorRun(queryDesc, direction, count)
  nprocessed = estate->es_processed
  PopActiveSnapshot()
update atStart / atEnd / portalPos
```
backward 路径先检查 `CURSOR_OPT_NO_SCROLL`。
如果不允许 backward scan，报错并提示用 `SCROLL`。
`NoMovementScanDirection` 不是简单跳过。
它仍让 destination startup/shutdown 发生。
这是输出协议边界。

### 5.8. SQL FETCH/MOVE 的方向语义
SQL `FETCH` 和 `MOVE` 进入：
```text
PerformPortalFetch()
  -> GetPortalByName()
  -> if MOVE then dest = None_Receiver
  -> PortalRunFetch(portal, direction, howMany, dest)
```
`PortalRunFetch()` 和 `PortalRun()` 一样会 `MarkPortalActive()`、切换 `ActivePortal` / `CurrentResourceOwner` / `PortalContext`，并在 catch 中 `MarkPortalFailed()`。
方向逻辑在 `DoPortalRunFetch()`。
`FETCH FORWARD/BACKWARD n` 最终变成 `PortalRunSelect()`。
`FETCH ABSOLUTE positive` 会比较目标位置和当前 `portalPos`，选择从当前位置移动或 rewind 后向前跳过。
`FETCH ABSOLUTE negative` 通常先跑到末尾再向后退。
`FETCH RELATIVE` 会移动到目标前再取一行。
`FETCH 0` 表示重新取当前行；如果正坐在一行上，源码会先 backward 一行到 `None_Receiver`，再 forward 一行到真实 receiver。
这解释了为什么 `portalPos`、`atStart`、`atEnd` 必须一起看。

### 5.9. RunFromStore 从 materialized result 读
`RunFromStore()` 不依赖 `QueryDesc` 或 `EState`：
```text
MakeSingleTupleTableSlot(portal->tupDesc, TTSOpsMinimalTuple)
dest->rStartup()
tuplestore_gettupleslot()
dest->receiveSlot()
ExecClearTuple()
dest->rShutdown()
ExecDropSingleTupleTableSlot()
```
它自己返回 processed row count。
因为没有 `EState`，receiver 运行在 caller 当前 memory context。
诊断时要区分：
- 从活 executor 读：profile 会看到 scan/join/agg node 继续运行。
- 从 `holdStore` 读：profile 主要看到 tuplestore 和 receiver。

### 5.10. FillPortalStore 让副作用完整执行
`FillPortalStore()` 服务 `PORTAL_ONE_RETURNING`、`PORTAL_ONE_MOD_WITH`、`PORTAL_UTIL_SELECT`。
它先创建 `holdStore` 和 `DestTuplestore` receiver：
```text
PortalCreateHoldStore()
CreateDestReceiver(DestTuplestore)
SetTuplestoreDestReceiverParams()
```
然后：
```text
RETURNING / MOD_WITH -> PortalRunMulti(..., treceiver, None_Receiver, &qc)
UTIL_SELECT          -> PortalRunUtility(..., treceiver, &qc)
```
这样 DML、data-modifying CTE、trigger 和 rewrite auxiliary query 不会停在半路。
“结果分批返回”和“副作用完整执行”被 tuplestore 解耦。

### 5.11. PortalRunMulti 总是 run to completion
`PortalRunMulti()` 对 `portal->stmts` 循环。
plannable query 走：
```text
ProcessQuery()
  -> CreateQueryDesc()
  -> ExecutorStart()
  -> ExecutorRun(..., Forward, 0)
  -> ExecutorFinish()
  -> ExecutorEnd()
  -> FreeQueryDesc()
```
utility 走 `PortalRunUtility()`。
多条 statement 之间执行 `CommandCounterIncrement()`。
每轮会 `MemoryContextDeleteChildren(portal->portalContext)` 回收临时内存。
所以 multi-query portal 没有 suspended 状态。
源码还处理 `CALL/DO` 内部 transaction control 可能让 `portal->stmts` 变成 `NIL` 的 awkward path，避免 dereference 已释放的 cached plan list。

### 5.12. holdable cursor 的 commit 前转化
holdable cursor 的关键不在 `FETCH`，而在 `PreCommit_Portals()`。
简化逻辑：
```text
for each portal:
  if active:
    detach resowner and portalSnapshot
  else if HOLD and created in current xact and status READY:
    if PREPARE TRANSACTION: ERROR
    HoldPortal(portal)
  else if created in previous transaction:
    keep it
  else:
    PortalDrop(portal, true)
```
`HoldPortal()`：
```text
PortalCreateHoldStore()
PersistHoldablePortal()
PortalReleaseCachedPlan()
portal->resowner = NULL
createSubid = activeSubid = InvalidSubTransactionId
```
`PersistHoldablePortal()` 把活 executor 变成 materialized result：
```text
copy tupDesc into holdContext
MarkPortalActive()
PushActiveSnapshot(queryDesc->snapshot)
if SCROLL:
  ExecutorRewind()
else if atEnd:
  direction = NoMovement
queryDesc->dest = DestTuplestore(detoast=true)
ExecutorRun(queryDesc, direction, 0)
portal->queryDesc = NULL
ExecutorFinish()
ExecutorEnd()
FreeQueryDesc()
reposition tuplestore
status = READY
PopActiveSnapshot()
MemoryContextDeleteChildren(portal->portalContext)
```
SCROLL WITH HOLD 需要保存整个结果集。
NO SCROLL WITH HOLD 通常只保存尚未取出的行。
commit 后继续 FETCH 时，读的是 `holdStore`，不是原 executor。

### 5.13. CLOSE、PortalDrop 与 top-level ERROR
`CLOSE cursor` 进入 `PerformPortalClose()`，最终 `PortalDrop(portal, false)`。
`CLOSE ALL` / `DISCARD ALL` 用 `PortalHashTableDeleteAll()`，会跳过当前 active portal。
`PortalDrop()` 顺序很关键：
```text
reject pinned portal
reject active portal
run cleanup hook if any
delete from PortalHashTable
release cached plan
unregister holdSnapshot
release/delete portal resowner if needed
tuplestore_end(holdStore)
delete holdContext
delete portalContext
pfree PortalData
```
先从 hash table 删除，是为了防止后续步骤 ERROR 时重复 drop 进入恢复循环。
top-level ERROR recovery 在 `postgres.c` 中报告错误、`AbortCurrentTransaction()`，然后调用 `PortalErrorCleanup()`。
普通 portal 的 abort cleanup 主要由 transaction callbacks 完成：
```text
AtAbort_Portals()
AtCleanup_Portals()
AtSubAbort_Portals()
AtSubCleanup_Portals()
```
`AtAbort_Portals()` 会把本事务 READY portal 标为 FAILED、运行 cleanup hook、释放 cached plan、把 `resowner` 置 NULL，并删除 `portalContext` 子 context。
`AtCleanup_Portals()` 再 drop 非 held、非 auto-held、非 active 的 portal。
如果 cleanup hook 还没跑，它会 warning 后跳过，避免 post-abort cleanup 阶段运行可能失败的用户代码。

## 6. 生命周期 / ownership / cleanup

### 6.1. 谁创建
simple query：
```text
exec_simple_query() -> CreatePortal("", true, true)
```
它是 unnamed、invisible、通常立即 run/drop。
extended Bind：
```text
exec_bind_message() -> CreatePortal(portal_name, ...)
```
它跨 Execute message 存活。
SQL cursor：
```text
PerformCursorOpen() -> CreatePortal(cursor_name, false, false)
```
它显示在 `pg_cursors`。
SPI：
```text
SPI_cursor_open / SPI_cursor_parse_open -> CreatePortal() -> PortalStart()
```
它服务 PL/pgSQL 和扩展内部 cursor。

### 6.2. 谁持有
| 对象 | owner |
| --- | --- |
| `PortalData` | `TopPortalContext` 和 backend-local portal hash。 |
| query/executor auxiliary memory | `portalContext`。 |
| held result | `holdContext` / `holdStore`。 |
| snapshots、pins、外部资源 | `portal->resowner` 或 transaction owner。 |
| cached plan | `portal->cplan` refcount。 |
| 可运行语义 | `PortalStatus`。 |

### 6.3. 谁释放
显式释放包括 `CLOSE`、`DISCARD ALL`、simple query 完成后的 `PortalDrop()`、extended unnamed portal 被新 Bind 替换、session end。
事务结束释放包括 `PreCommit_Portals()`、`AtAbort_Portals()`、`AtCleanup_Portals()`、subtransaction commit/abort/cleanup callbacks。
普通 non-holdable cursor 在 commit 时被 drop。
holdable cursor 在 commit 前 materialize，之后由 `CLOSE` 或 session end drop。

### 6.4. ERROR / abort 谁兜底
`PortalStart()`、`PortalRun()`、`PortalRunFetch()` 都用 `PG_TRY/PG_CATCH`。
catch 中：
```text
MarkPortalFailed(portal)
restore ActivePortal / CurrentResourceOwner / PortalContext
PG_RE_THROW()
```
`MarkPortalFailed()` 会运行 cleanup hook。
标准 cleanup hook 在 FAILED 状态下不会正常 shutdown executor。
随后 transaction abort 释放 ResourceOwner 中的资源，post-abort cleanup 删除 portal shell 和 memory contexts。

### 6.5. 长期对象如何失效
Portal 不是 shared invalidation 对象。
它主要通过这些边界失效：
- transaction end drop non-holdable portals。
- plan cache refcount release。
- subtransaction abort 标记 failed。
- ERROR 后 failed 禁止继续运行。
- session end 清空 portal hash 和 contexts。
cached plan 的 catalog invalidation 是 plan cache 主题，不在本节展开。

## 7. 正确性机制层次
| 层次 | 机制 | 保证 | 不保证 |
| --- | --- | --- | --- |
| 状态机 | `PortalStatus` | 是否能 run/drop/continue | executor 内部资源健康 |
| 策略 | `PortalStrategy` | 哪类 query 可真暂停 | 所有返回 tuple 的语句都可暂停 |
| snapshot | `queryDesc->snapshot` / active snapshot | visibility 和 TOAST 可读性 | 并发互斥 |
| ResourceOwner | portal/xact owner | abort 时释放资源 | 内存释放 |
| MemoryContext | `portalContext` / `holdContext` | 批量释放内存 | snapshot、pin、lock 释放 |
| tuplestore | `holdStore` | result 跨调用读取 | 原 query 仍在运行 |
| cleanup hook | `PortalCleanup()` | 正常 drop 时 shutdown executor | FAILED 下完整 shutdown |
只允许单 SELECT 真暂停，是为了避免 DML 或 multi-query 的中间副作用对外暴露。
如果 `UPDATE RETURNING` 在返回 10 行后暂停，系统必须回答哪些 row 已更新、AFTER trigger 是否触发、rewrite auxiliary query 是否执行、client disconnect 后副作用是否继续。
PostgreSQL 选择保守规则：
```text
副作用必须 run to completion；
只有无副作用的单 SELECT executor state 可以挂起。
```
WITH HOLD 必须 materialize，因为普通 executor state 依赖当前事务的 snapshot、lock、resource owner、memory context 和可能的 buffer/tuple 引用。
COMMIT 后这些资源要释放。
因此 held cursor 保留 materialized result，而不是保留 executor。
FAILED portal 不强行 `ExecutorEnd()`，是为了避免 ERROR recovery 阶段再次进入可能失败的用户代码或损坏状态。

## 8. 错误路径 / 异常路径 / fallback

### 8.1. 运行非 READY portal
`MarkPortalActive()` 要求 portal 是 `PORTAL_READY`。
否则报错：
```text
portal "<name>" cannot be run
```
这覆盖已 done、failed、未 start、或 active 重入运行。

### 8.2. drop pinned 或 active portal
`PortalDrop()` 不允许 drop pinned portal，也不允许 drop active portal。
pinned portal 常见于 SPI/PL 内部 cursor loop。
如果 procedure 内执行 COMMIT/ROLLBACK，SPI 会先 `HoldPinnedPortals()`。
如果 abort 已经发生，`AtCleanup_Portals()` 可以强制 unpin，因为持有者也被错误路径中断。

### 8.3. backward fetch from NO SCROLL
`PortalRunSelect()` 和 `DoPortalRewind()` 会检查 `CURSOR_OPT_NO_SCROLL`。
尝试 backward scan 时抛错，并提示用 `SCROLL`。
未显式指定 SCROLL/NO SCROLL 时，`PerformCursorOpen()` 根据 plan 是否支持 backward scan 自动选择。
这避免为了兼容默认 backward fetch 给所有 cursor 引入 Materialize 成本。

### 8.4. WITH HOLD 与 PREPARE TRANSACTION
`PreCommit_Portals(isPrepare)` 遇到本事务创建的 holdable cursor 时，如果正在 `PREPARE TRANSACTION`，会报错：
```text
cannot PREPARE a transaction that has created a cursor WITH HOLD
```
这是因为 held cursor 的 result、snapshot、temp file 和 prepared transaction 状态组合起来语义不清。

### 8.5. utility command 内部事务控制
`PortalRun()` 特意保存 `TopTransactionResourceOwner` 和 `TopTransactionContext`。
原因是 `VACUUM`、`CLUSTER`、`CALL/DO` 等路径可能内部启动和提交事务。
恢复指针时要识别旧 pointer 是否等于旧 top owner，并改回新的 top owner。
这是 utility 历史语义对 Portal 层的侵入。

### 8.6. auto-held pinned portal
SPI 在 procedure 内部 transaction control 前调用：
```text
HoldPinnedPortals()
ForgetPortalSnapshots()
```
`HoldPinnedPortals()` 只允许 read-only `PORTAL_ONE_SELECT` 被 auto-held。
它把 pinned portal 转为 held cursor，并设置 `autoHeld=true`。
错误路径下 `PortalErrorCleanup()` drop 这些 auto-held portal。

## 9. 成本、资源与跨模块传播

### 9.1. 增量 SELECT 成本
`PORTAL_ONE_SELECT` 的成本随这些变量扩张：
| 变量 | 影响 |
| --- | --- |
| FETCH 次数 | 每批都有 PortalRun/ExecutorRun/receiver startup-shutdown 和协议往返。 |
| count 大小 | 小 count 降低单次延迟，但放大边界开销。 |
| backward scan | 可能要求 plan 支持 backward，或引入 Materialize，或直接禁止。 |
| cursor 存活时间 | snapshot/locks/resources 保留更久，可能拖住 vacuum horizon。 |
| 客户端速度 | backend 可能 idle in transaction 但资源仍被 cursor 持有。 |

### 9.2. holdable cursor 成本
WITH HOLD 的成本常发生在 COMMIT 前。
`PreCommit_Portals()` 可能突然执行：
```text
ExecutorRun(..., count=0)
  -> DestTuplestore
  -> detoast
  -> temp file spill
  -> ExecutorFinish/ExecutorEnd
```
成本随结果行数、tuple 宽度、TOAST/detoast、`work_mem`、是否 SCROLL、已取行数变化。
SCROLL WITH HOLD 可能保存整个结果集。
NO SCROLL WITH HOLD 通常只保存尚未取出的行。
`PortalCreateHoldStore()` 使用 `work_mem`。
超过内存预算后，tuplestore 可能写临时文件。
这会在 `log_temp_files`、`pg_stat_database.temp_files/temp_bytes` 和 commit latency 上表现出来。

### 9.3. 跨模块传播
| 模块 | Portal 边界 |
| --- | --- |
| frontend/backend protocol | Bind/Execute、named portal、`PortalSuspended`。 |
| plan cache | `CachedPlan` refcount 由 portal 持有和释放。 |
| executor | `QueryDesc` 和 `ExecutorRun(direction,count)` 多次推进。 |
| transaction manager | commit/abort/subxact callbacks 管生命周期。 |
| snapshot manager | active snapshot、registered snapshot、hold snapshot。 |
| tuplestore/temp file | held cursor 和 materialized result。 |
| SPI/PL | pinned portal、auto-held portal、procedure transaction control。 |
| observability | `pg_cursors`、`pg_stat_activity`、duration log、temp file log。 |
Portal 本身没有 shared memory 状态由后台进程推进。
但长事务 cursor 会间接影响 autovacuum cleanup horizon。
holdStore spill 会增加临时 I/O 压力。

## 10. 观测与诊断入口

### 10.1. 能直接看什么
`pg_cursors` 展示当前 session visible cursors：
```sql
SELECT name, statement, is_holdable, is_binary, is_scrollable, creation_time
FROM pg_cursors;
```
它来自 `portalmem.c` 的 `pg_cursor()`。
它看不到 unnamed protocol portal、invisible portal、`PortalStatus`、`portalPos`、`queryDesc`、`holdStore` 大小。
`pg_stat_activity` 能看到 backend state 和 query text，但看不到 Portal 内部状态。
`log_temp_files` 能提示 holdStore spill，但 sort/hash spill 也会产生日志，需要结合 SQL 行为和源码栈判断。
extended protocol 客户端能直接看到 `PortalSuspended`。

### 10.2. 普通 cursor 实验
```sql
BEGIN;
DECLARE c CURSOR FOR SELECT generate_series(1, 5);
SELECT name, is_holdable, is_scrollable FROM pg_cursors;
FETCH 2 FROM c;
FETCH 2 FROM c;
MOVE FORWARD 1 FROM c;
FETCH 1 FROM c;
CLOSE c;
COMMIT;
```
源码对应：
```text
PerformCursorOpen()
  -> PortalStart()
FETCH/MOVE
  -> PortalRunFetch()
  -> DoPortalRunFetch()
  -> PortalRunSelect()
CLOSE
  -> PortalDrop()
```
关键观察：`FETCH` 多次推进同一 portal，`MOVE` 改位置但不输出 tuple。

### 10.3. NO SCROLL backward error
```sql
BEGIN;
DECLARE c NO SCROLL CURSOR FOR SELECT generate_series(1, 3);
FETCH 1 FROM c;
FETCH BACKWARD 1 FROM c;
ROLLBACK;
```
预期第二个 `FETCH` 报错。
源码对应 `PortalRunSelect(forward=false)` 检查 `CURSOR_OPT_NO_SCROLL`。

### 10.4. WITH HOLD commit 后 fetch
```sql
BEGIN;
DECLARE c CURSOR WITH HOLD FOR SELECT generate_series(1, 5);
FETCH 2 FROM c;
COMMIT;
SELECT name, is_holdable FROM pg_cursors;
FETCH 2 FROM c;
CLOSE c;
```
源码对应：
```text
PreCommit_Portals()
  -> HoldPortal()
  -> PortalCreateHoldStore()
  -> PersistHoldablePortal()
  -> ExecutorRun(..., DestTuplestore)
  -> ExecutorEnd()
```
commit 后读取的是 `holdStore`。

### 10.5. holdStore spill 实验
```sql
SET work_mem = '64kB';
SET log_temp_files = 0;
BEGIN;
DECLARE c SCROLL CURSOR WITH HOLD FOR
  SELECT g, repeat('x', 1000)
  FROM generate_series(1, 200000) AS g;
COMMIT;
CLOSE c;
```
观察 `COMMIT` 延迟、server log temporary file、`pg_stat_database.temp_bytes`。
是否 spill 依赖 `work_mem`、tuple 宽度、行数、版本和临时文件环境。

### 10.6. extended protocol suspended execution
用 libpq 或测试客户端发送：
```text
Parse SELECT generate_series(1,5)
Bind portal p
Execute p max_rows=2
  -> DataRow 1, DataRow 2, PortalSuspended
Execute p max_rows=2
  -> DataRow 3, DataRow 4, PortalSuspended
Execute p max_rows=0
  -> DataRow 5, CommandComplete
Sync
  -> ReadyForQuery
```
断点建议：
```text
exec_execute_message
PortalRun
PortalRunSelect
ExecutorRun
```
打印：
```text
portal->status
portal->atStart
portal->atEnd
portal->portalPos
portal->queryDesc
portal->holdStore
```

### 10.7. gdb 状态跟踪
断点：
```text
b CreatePortal
b PortalStart
b PortalRun
b PortalRunSelect
b PortalRunFetch
b PersistHoldablePortal
b PortalDrop
b AtAbort_Portals
```
普通 cursor 的 `FETCH` 应看到：
```text
READY -> ACTIVE -> READY
portalPos 递增
queryDesc 非 NULL
holdStore 为 NULL
```
holdable cursor commit 时应看到：
```text
PersistHoldablePortal()
queryDesc 从非 NULL 变 NULL
holdStore 从 NULL 变非 NULL
```

### 10.8. 可见、可推断、不可见
能直接观测：`pg_cursors`、`pg_stat_activity`、server log、temp file log、protocol `PortalSuspended`。
只能推断：是否正在从 executor 读还是从 `holdStore` 读、commit 延迟中多少来自 `PersistHoldablePortal()`、cursor 是否拖住 vacuum horizon。
通常需要断点或插桩：`portal->status`、`portal->portalPos`、`portal->queryDesc`、`portal->holdSnapshot`、`portal->resowner` 何时置 NULL。
不要把 `pg_stat_activity.state='idle in transaction'` 直接解释成 cursor 问题。
它只说明 transaction open 且 backend 等客户端。
是否有 cursor 要结合 `pg_cursors`、SQL 行为和事务时间判断。

## 11. 常见误区
1. 把 Portal 等同于 SQL cursor。SQL cursor 使用 Portal，但 simple query、extended protocol portal、SPI cursor 也使用 Portal。
2. 以为 `Execute(max_rows)` 重新执行 query。它是继续推进同一个 portal。
3. 以为所有返回 tuple 的语句都能暂停。`UPDATE RETURNING` 只能分批读已物化结果，DML 本身不能暂停。
4. 把 WITH HOLD 理解成保留 snapshot。主路径是 materialize，commit 后不保留活 executor。
5. 把 MemoryContext 当 ResourceOwner。前者管内存，后者管 snapshot、pin、外部资源。
6. 以为 ERROR 后一定完整 `ExecutorEnd()`。FAILED portal 会避免正常 shutdown，交给 abort cleanup。
7. 只看 `pg_cursors` 判断所有 portal。它看不到 unnamed/invisible portal，也看不到内部 status 和 position。

## 12. 课堂实验

### 实验 1：画出普通 cursor 状态变化
```sql
BEGIN;
DECLARE c CURSOR FOR SELECT generate_series(1, 4);
FETCH 1 FROM c;
FETCH 2 FROM c;
CLOSE c;
COMMIT;
```
断点：
```text
PerformCursorOpen
PortalStart
PortalRunFetch
PortalRunSelect
PortalDrop
```
记录 `strategy`、`status`、`portalPos`、`queryDesc`、`holdStore`。
结论应回到：暂停执行是 Portal 在多次 `ExecutorRun()` 之间保存 `QueryDesc` 和 position。

### 实验 2：比较普通 cursor 与 WITH HOLD
```sql
BEGIN;
DECLARE c CURSOR WITH HOLD FOR SELECT generate_series(1, 10);
FETCH 3 FROM c;
COMMIT;
FETCH 2 FROM c;
CLOSE c;
```
断点：
```text
PreCommit_Portals
HoldPortal
PersistHoldablePortal
PortalRunSelect
RunFromStore
PortalDrop
```
记录 commit 前后 `queryDesc` 和 `holdStore` 的变化。
结论应回到：WITH HOLD 跨事务保留 materialized result，不保留原 executor。

### 实验 3：制造 ERROR 并观察 cleanup
```sql
BEGIN;
DECLARE c CURSOR FOR SELECT 1 / (g - 2) FROM generate_series(1,3) AS g;
FETCH ALL FROM c;
ROLLBACK;
```
断点：
```text
PortalRunFetch
PortalRunSelect
MarkPortalFailed
AtAbort_Portals
AtCleanup_Portals
PortalDrop
```
预期 `g=2` division by zero，portal 进入 FAILED，abort cleanup 再删除 portal。

### 实验 4：观察 holdStore temp file
```sql
SET log_temp_files = 0;
SET work_mem = '64kB';
BEGIN;
DECLARE c SCROLL CURSOR WITH HOLD FOR
  SELECT g, repeat(md5(g::text), 50)
  FROM generate_series(1, 100000) AS g;
COMMIT;
CLOSE c;
```
观察 server log、`COMMIT` latency、`pg_stat_database.temp_bytes`。
如果没有 spill，增大行数或 tuple 宽度。

## 13. 讨论题
1. 为什么 `PORTAL_ACTIVE` 可以回到 `PORTAL_READY`，而 `PORTAL_DONE` / `PORTAL_FAILED` 不能回退？
2. 为什么 `UPDATE ... RETURNING` 不能像 SELECT cursor 一样在 executor 中途暂停？
3. `portalPos` 为什么不能单独解释 cursor 位置？
4. 普通 cursor 长时间不关闭，可能怎样影响 vacuum horizon？
5. 为什么 `PortalDefineQuery()` 要避免可能 ERROR 的复杂工作？
6. 为什么 `PortalDrop()` 先删 hash entry，再释放 cached plan、tuplestore 和 memory context？
7. `pg_cursors` 能诊断什么，又看不到哪些状态？
8. WITH HOLD commit 后能 fetch，是否意味着创建事务 snapshot 仍然活着？为什么？

## 14. 本节小结
本节核心链路：
```text
CreatePortal
  -> PortalDefineQuery
  -> PortalStart
  -> PortalRun / PortalRunFetch
  -> PortalRunSelect / RunFromStore
  -> PortalDrop 或 PreCommit_Portals/HoldPortal
  -> AtAbort/AtCleanup error cleanup
```
Portal 的核心状态是 backend-local。
`PortalStatus` 决定能否运行。
`PortalStrategy` 决定是否能真暂停 executor。
`queryDesc` 表示是否还有活 executor。
`holdStore` 表示是否已经 materialize。
`atStart/atEnd/portalPos` 共同表示 cursor position。
ownership 分层：
- `portalContext` 管 per-portal 内存。
- `holdContext` 管 held tuplestore。
- `ResourceOwner` 管 snapshot、pin、外部资源。
- `CachedPlan` refcount 由 portal 持有和释放。
- cleanup hook 负责正常 executor shutdown。
suspended execution 的本质：
```text
单 SELECT portal 在多次 ExecutorRun 之间保存 QueryDesc 和 cursor position；
一次调用没取到末尾时，状态从 ACTIVE 回到 READY。
```
holdable cursor 的本质：
```text
commit 前把剩余或全部结果 materialize 到 holdStore，
关闭 executor，
后续 FETCH 变成 tuplestore read。
```
ERROR cleanup 的本质：
```text
先把 portal 标为 FAILED 防止再运行，
再由 transaction abort 和 ResourceOwner 释放外部资源，
最后 post-abort cleanup 删除 portal shell 和内存。
```
可观测性边界：
`pg_cursors` 能看 visible SQL cursor。
extended protocol 能看 `PortalSuspended`。
temp file log 能提示 holdStore spill。
`PortalStatus`、`portalPos`、`queryDesc` 和 `holdStore` 边界通常需要断点或插桩。
可迁移系统规律：
```text
当执行对象必须跨调用或跨事务延长生命周期时，
不要把整个执行栈都做成长生命周期对象；
保留可暂停的最小状态，
把不能安全暂停的副作用先 drain 或 materialize，
并让 ERROR cleanup 按语义状态、外部资源、内存三层分别收尾。
```
仍依赖上下文的判断：
- FETCH 批大小取决于网络延迟、客户端速度和每行执行成本。
- holdStore 是否 spill 取决于 `work_mem`、行数、tuple 宽度和版本实现。
- cursor 是否拖住 vacuum 取决于事务持续时间、snapshot 和表更新压力。
- protocol-level portal 的暂停通常需要客户端协议、断点或插桩验证。
