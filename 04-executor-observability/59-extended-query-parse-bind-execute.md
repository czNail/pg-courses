# PostgreSQL extended query Parse/Bind/Describe/Execute/Sync 边界
## 课程定位
前置知识：已经理解 PostgreSQL backend 是一个长生命周期进程，也已经读过 executor 生命周期中 `Portal`、`QueryDesc`、`EState`、`ExecutorStart()`、`ExecutorRun()` 与 `ExecutorEnd()` 的基本边界。
本节唯一主问题：
```text
Parse/Bind/Describe/Execute/Sync 如何把 prepared statement、portal 和 executor 分成多个协议阶段？
```
核心矛盾：客户端希望把一条 SQL 拆成可缓存、可绑定参数、可分批执行、可描述结果的多个网络消息；内核却必须保证事务状态、snapshot、plan cache 引用、Portal 资源和 executor 状态在 ERROR、pipeline、unnamed 对象替换和 Sync cleanup 下仍然一致。
学完后应能判断：一个 extended query 阶段到底创建了什么状态、还没有创建什么状态；prepared statement、portal、executor 三者谁持有什么资源；错误后为什么必须 skip till Sync；以及 `ReadyForQuery` 为什么是协议和事务 cleanup 的共同边界，而不是普通响应消息。
本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e9`。
## 1. 本节在总主线中的位置
04 目录前面的课程已经把 executor 内部生命周期拆成：
```text
ExecutorStart()
  -> ExecutorRun()
  -> ExecutorFinish()
  -> ExecutorEnd()
```
但真实客户端不是直接调用 executor。
客户端先进入 frontend/backend protocol message loop。
simple query 协议把 parse、analyze、rewrite、plan、execute 放进一个 `Query` 消息。
extended query 协议把它们拆成多个消息：
```text
Parse
  -> Bind
  -> Describe
  -> Execute
  -> Sync
```
这个拆分不是单纯网络优化。
它把三个对象的生命周期拆开：
| 对象 | 主要阶段 | 核心语义 |
| --- | --- | --- |
| prepared statement | `Parse` | 保存 raw/analyzed/rewrite 后的 `CachedPlanSource`，以及参数类型和结果描述。 |
| portal | `Bind` | 绑定具体参数、结果格式、`CachedPlan` 引用，并准备执行 envelope。 |
| executor | `Bind` / `Execute` | `PortalStart()` 可能启动 executor，`Execute` 才推进 tuple 流。 |
| transaction command | 多个消息之间 | `start_xact_command()` 可能跨消息保持打开，`Sync` 负责收口。 |
本节是 04 目录第 12 组 `Frontend/backend protocol、Portal 与 plan cache` 的第一节实作课之一。
它服务后续三类问题：
1. prepared statement 如何选择 generic plan 或 custom plan。
2. Portal 如何支持 suspended execution 和 cursor。
3. DestReceiver 如何把 executor tuple 写回客户端。
本节只追一条主线：
```text
一条 extended query 如何从 Parse 变成 prepared statement，
再从 Bind 变成 portal，
最后由 Execute 推进 executor，
并在 Sync 或 ERROR 下收尾。
```
## 2. 核心矛盾与一句话运行模型
一句话运行模型：
```text
Parse 在 backend-local plancache 中建立 CachedPlanSource；
Bind 用参数和结果格式创建 Portal 并取得 CachedPlan；
Describe 只暴露 prepared statement 或 portal 的参数和 tuple descriptor；
Execute 对 PortalRun() 做一次有限行数推进；
Sync 结束 extended protocol 期间延迟打开的 transaction command，并发送 ReadyForQuery。
```
这里的 tension 是：
```text
协议阶段要足够细，支持参数复用、结果描述、pipelining 和分批取数
  vs
内核状态必须有单一 owner、单一 cleanup 边界和可恢复的错误语义
```
如果 Parse 就创建 executor，参数还没有到，custom plan 无法用参数值。
如果 Bind 不创建 portal，Execute 就没有地方保存参数、结果格式、cursor 位置和 cached plan 引用。
如果 Execute 每次都从头运行，`max_rows` 和 `PortalSuspended` 就无法实现。
如果 Sync 不作为收口，Parse/Bind/Execute 之间打开的 transaction command 会在 pipeline 中没有稳定结束点。
如果 ERROR 后继续读并执行后续 extended 消息，客户端和服务端可能已经对消息边界、事务状态、portal 状态产生不同理解。
所以 PostgreSQL 选择：
```text
协议阶段拆细；
状态对象分层；
ERROR 后只跳到 Sync；
ReadyForQuery 只在确定事务状态可报告时发送。
```
这一课要建立的 mental model 是：
```text
prepared statement 是可被 Bind 消费的语义缓存；
portal 是一次参数化执行的可暂停 envelope；
executor 是 portal 内部可推进的 runtime state；
Sync 是 extended protocol batch 的事务和错误恢复边界。
```
## 3. 核心文件分工与阅读顺序
| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/tcop/postgres.c` | backend message loop、`exec_parse_message()`、`exec_bind_message()`、`exec_execute_message()`、Describe、Sync、ERROR skip-till-Sync。 |
| 2 | `src/include/utils/portal.h` | `PortalData`、`PortalStatus`、`PortalStrategy`、portal 与 `CachedPlan`、`QueryDesc`、snapshot、hold store 的状态边界。 |
| 3 | `src/backend/utils/mmgr/portalmem.c` | `CreatePortal()`、`PortalDefineQuery()`、`PortalDrop()`、`PortalErrorCleanup()`、事务和子事务结束时 portal cleanup。 |
| 4 | `src/backend/tcop/pquery.c` | `PortalStart()`、`PortalRun()`、`PortalRunSelect()`、`PortalRunMulti()`、`CreateQueryDesc()`，从 portal 进入 executor。 |
| 5 | `src/include/utils/plancache.h` | `CachedPlanSource` 与 `CachedPlan` 的结构边界、memory context 和 refcount 语义。 |
| 6 | `src/backend/commands/prepare.c` | SQL `PREPARE` / `EXECUTE` 与 protocol prepared statement 共享的 plan cache 语义。 |
| 7 | `src/backend/executor/execMain.c` | `ExecutorStart()` 与 `ExecutorRun()` 的外层 contract，帮助区分启动和推进。 |
| 8 | `src/include/tcop/tcopprot.h` | tcop 层对外入口声明，帮助确认哪些函数是模块边界。 |
推荐阅读顺序不是按文件名。
先读 `postgres.c`，因为协议消息决定时间轴。
再读 `portal.h`，因为 `PortalData` 是 Bind 与 Execute 之间跨消息保存状态的中心结构。
然后读 `portalmem.c`，看 portal 如何创建、替换、释放。
最后读 `pquery.c` 和 `execMain.c`，确认 Execute 只是通过 portal 推进 executor，不直接理解 plan tree 的每个节点。
本节要警惕一个常见误读：
```text
extended query 的 Parse 不等于 SQL parser 阶段；
它会做 parse/analyze/rewrite，并创建 CachedPlanSource。
```
另一个常见误读：
```text
Bind 不只是把参数放到数组里；
它创建 Portal、读取参数值、取得 CachedPlan，并调用 PortalStart()。
```
## 4. 关键数据结构与状态
### 4.1 `unnamed_stmt_psrc`
`postgres.c` 中有一个 backend-local 静态指针：
```c
static CachedPlanSource *unnamed_stmt_psrc = NULL;
```
它只保存 unnamed prepared statement。
named prepared statement 走 `commands/prepare.c` 的 hashtable。
unnamed statement 单独存放，是为了降低短生命周期 extended query 的开销。
这个指针不能跨 backend。
它不是 shared state。
它不受其他连接直接访问。
它的生命周期规则很硬：
```text
新的 unnamed Parse 会 drop 旧 unnamed statement；
Close Statement 且名字为空会 drop unnamed statement；
simple query 入口也会 drop unnamed statement；
backend 退出时随 backend-local 内存消失。
```
源码里 `drop_unnamed_stmt()` 先把全局指针置空，再 `DropCachedPlan(psrc)`。
这个顺序避免 `DropCachedPlan()` 中出错时留下悬挂指针。
### 4.2 `CachedPlanSource`
`CachedPlanSource` 是 Parse 阶段的核心产物。
本节不展开 plan cache 的 generic/custom 选择，那是后续课程。
这里只需要把它理解成：
```text
一条已经 parse/analyze/rewrite 的 prepared statement 语义容器。
```
它包含：
| 状态 | 本节语义 |
| --- | --- |
| `query_string` | 原始 SQL 文本，用于日志、pg_stat、错误上下文和后续执行。 |
| `raw_parse_tree` | 创建 plan source 时保存的 raw tree。 |
| `query_list` | analyze/rewrite 后的 query tree 列表。 |
| `param_types` / `num_params` | 参数类型边界，Bind 必须匹配。 |
| `resultDesc` | Describe statement 可用的结果 tuple descriptor。 |
| `fixed_result` | prepared statement 的结果形状被认为固定。 |
| `context` | plan source 自己的 memory context。 |
Parse 阶段调用 `CreateCachedPlan()`，再调用 `CompleteCachedPlan()`。
对 named statement，解析临时对象先放在 `MessageContext`，完成后复制到 plan source。
对 unnamed statement，源码创建 `unnamed_stmt_context`，直接在其中做 parsing 和 rewriting，再把它交给 `CompleteCachedPlan()`。
这不是语义差异。
它是生命周期和复制成本上的优化。
### 4.3 `PortalData`
`PortalData` 是 Bind 阶段的核心产物。
可以把它称为：
```text
一次可执行、可暂停、可关闭的参数化 query envelope。
```
关键字段不是孤立解释的。
它们要按 owner 和阶段分组。
| 字段 | 阶段 | 语义 |
| --- | --- | --- |
| `name` | CreatePortal | portal hash table key；unnamed portal 名字是空字符串。 |
| `prepStmtName` | PortalDefineQuery | 来源 prepared statement 名字；unnamed 时为 NULL。 |
| `portalContext` | CreatePortal | 参数、query string、格式数组、`QueryDesc` 等 portal-local 内存。 |
| `resowner` | CreatePortal | portal 持有的 snapshot、buffer pin、锁等外部资源 owner。 |
| `cleanup` | CreatePortal | 失败或 drop 时的 hook，通常指向 `PortalCleanup()`。 |
| `stmts` | PortalDefineQuery | `CachedPlan` 的 planned statement list。 |
| `cplan` | PortalDefineQuery | `GetCachedPlan()` 返回的 `CachedPlan` 引用。 |
| `portalParams` | PortalStart | Bind 解析出的具体参数值。 |
| `strategy` | PortalStart | `PORTAL_ONE_SELECT`、`PORTAL_MULTI_QUERY` 等执行策略。 |
| `status` | 全生命周期 | `NEW`、`DEFINED`、`READY`、`ACTIVE`、`DONE`、`FAILED`。 |
| `queryDesc` | PortalStart | 对 `PORTAL_ONE_SELECT`，这里保存已启动 executor 的 `QueryDesc`。 |
| `tupDesc` | PortalStart | Describe portal 和客户端 RowDescription 的依据。 |
| `formats` | PortalSetResultFormat | 每列 text/binary 输出格式。 |
| `atStart` / `atEnd` / `portalPos` | Execute | 分批 Execute 和 cursor fetch 的位置状态。 |
| `holdStore` / `holdContext` | 特殊策略 | RETURNING、utility select 或 holdable cursor 的 tuplestore。 |
`PortalStatus` 是读 Portal 代码的第一把尺子：
```text
PORTAL_NEW
  -> PORTAL_DEFINED
  -> PORTAL_READY
  -> PORTAL_ACTIVE
  -> PORTAL_READY 或 PORTAL_DONE
  -> PORTAL_FAILED
```
`PortalStart()` 成功后，portal 才能被 `PortalRun()` 推进。
`PortalRun()` 会临时标记 `PORTAL_ACTIVE`。
如果 `max_rows` 耗尽但没读完，portal 回到 `PORTAL_READY`，并返回 `false` 给 `exec_execute_message()`。
这就是 `PortalSuspended` 的内核状态来源。
### 4.4 `MessageContext`
`MessageContext` 是 backend main loop 中每个客户端消息的短生命周期 context。
`PostgresMain()` 每轮处理消息前会：
```text
MemoryContextSwitchTo(MessageContext)
MemoryContextReset(MessageContext)
initStringInfo(&input_message)
```
Parse、Bind、Describe、Execute 中很多临时解析结果都放在这里。
但跨消息状态不能只放在 `MessageContext`。
所以 Parse 要把结果放进 `CachedPlanSource`。
Bind 要把参数和格式放进 `portal->portalContext`。
Execute 在可能 `finish_xact_command()` 之前，会把 `sourceText` 和 `prepStmtName` 复制到 `MessageContext`。
这是因为 transaction finish 可能销毁 portal。
日志路径不能在 portal 已释放后继续读 portal 内存。
### 4.5 `xact_started`、`doing_extended_query_message`、`ignore_till_sync`
`postgres.c` 还有三类协议层状态。
`xact_started` 表示当前 backend 是否已经通过 `start_xact_command()` 打开了 transaction command。
extended query 下它可能跨多个消息保持 true。
Parse、Bind、Describe、Execute 都可以调用 `start_xact_command()`。
但 Parse 和 Bind 不会自己 `finish_xact_command()`。
通常要等 Sync。
`doing_extended_query_message` 在 `ReadCommand()` 读消息类型时尽早设置。
Parse、Bind、Close、Describe、Execute、Flush 都会被标记为 extended message。
`ignore_till_sync` 是 ERROR 后的协议恢复状态。
当 extended message 处理中发生 ERROR，顶层错误恢复会：
```text
AbortCurrentTransaction()
PortalErrorCleanup()
MemoryContextSwitchTo(MessageContext)
FlushErrorState()
ignore_till_sync = true
xact_started = false
```
之后 main loop 会跳过所有消息，直到读到 Sync。
Sync 会清掉 `ignore_till_sync`。
这不是性能优化。
这是协议同步和事务状态恢复机制。
## 5. 主流程源码 walkthrough
### 5.1 main loop: 先按消息类型建立协议状态
主流程从 `PostgresMain()` 的消息循环开始。
每轮循环在读下一条消息前重置 `MessageContext`。
如果 `send_ready_for_query` 为 true，先发 `ReadyForQuery()`。
然后 `ReadCommand()` 读取消息类型和消息体。
在读取消息体前，它会根据消息类型设置：
```text
PqMsg_Query:
  doing_extended_query_message = false
PqMsg_Parse / PqMsg_Bind:
  doing_extended_query_message = true
PqMsg_Close / PqMsg_Describe / PqMsg_Execute / PqMsg_Flush:
  doing_extended_query_message = true
PqMsg_Sync:
  ignore_till_sync = false
  doing_extended_query_message = false
```
这里的关键点是“尽早”。
如果读取消息体或后续处理出错，顶层错误恢复已经知道这是否属于 extended query protocol。
如果 `ignore_till_sync` 已经为 true，loop 会跳过后续消息，不进入 switch 执行。
所以 ERROR 后客户端发送的 Bind、Execute、Describe 不会继续改变 backend 状态。
只有 Sync 重新建立共同边界。
### 5.2 Parse: 创建 prepared statement 语义容器
Parse 消息在 switch 中先解包：
```text
stmt_name
query_string
numParams
paramTypes[]
```
然后进入 `exec_parse_message()`。
主链路：
```text
exec_parse_message()
  -> debug_query_string = query_string
  -> pgstat_report_activity(STATE_RUNNING, query_string)
  -> set_ps_display("PARSE")
  -> start_xact_command()
  -> 选择 MessageContext 或 unnamed_stmt_context
  -> pg_parse_query(query_string)
  -> 检查 prepared statement 只能有一条用户语句
  -> CreateCachedPlan(raw_parse_tree, query_string, commandTag)
  -> 必要时 PushActiveSnapshot(GetTransactionSnapshot())
  -> pg_analyze_and_rewrite_varparams()
  -> CompleteCachedPlan()
  -> StorePreparedStatement() 或 SaveCachedPlan()
  -> CommandCounterIncrement()
  -> ParseComplete
```
Parse 阶段做了 parse/analyze/rewrite。
它还没有绑定参数值。
它也通常没有创建 `CachedPlan`。
真正的 plan 获取发生在 Bind 的 `GetCachedPlan()`。
Parse 中有几个重要边界。
第一，prepared statement 不能包含多条用户语句。
源码检查 `list_length(parsetree_list) > 1` 后报错。
原因不是 parser 不能 parse 多条语句，而是 prepared statement 需要单一参数列表、单一结果描述和单一后续 Bind 语义。
第二，aborted transaction block 里只能允许 transaction exit statement。
Parse 必须在 parse analysis、rewrite 或 planning 可能访问 catalog 前拒绝普通语句。
否则错误状态下访问数据库会产生更多派生错误。
第三，snapshot 只在分析需要时设置。
`analyze_requires_snapshot(raw_parse_tree)` 为真时，Parse 推入 transaction snapshot。
分析完成后立即 `PopActiveSnapshot()`。
第四，unnamed statement 会先 drop 旧对象。
这意味着 unnamed statement 是“最后一次 Parse”的语义，不是历史列表。
客户端如果依赖 unnamed statement，必须清楚它会被下一次 unnamed Parse 替换。
### 5.3 Bind: 从 prepared statement 创建 portal
Bind 消息解包更复杂，所以 `postgres.c` 把字段读取放在 `exec_bind_message()` 内部。
主链路：
```text
exec_bind_message()
  -> 读取 portal_name 和 stmt_name
  -> FetchPreparedStatement() 或 unnamed_stmt_psrc
  -> pgstat_report_activity(STATE_RUNNING, psrc->query_string)
  -> start_xact_command()
  -> MemoryContextSwitchTo(MessageContext)
  -> 读取参数格式数组
  -> 读取参数个数并检查 psrc->num_params
  -> aborted transaction block 检查
  -> CreatePortal()
  -> 切到 portal->portalContext
  -> 复制 query_string 和 stmt_name
  -> 必要时 PushActiveSnapshot()
  -> 读取参数值并调用 text/binary input function
  -> 读取结果格式数组
  -> GetCachedPlan(psrc, params, NULL, NULL)
  -> PortalDefineQuery(portal, ..., cplan->stmt_list, cplan)
  -> PortalStart(portal, params, 0, InvalidSnapshot)
  -> PortalSetResultFormat(portal, numRFormats, rformats)
  -> BindComplete
```
Bind 的第一件事是找到 prepared statement。
named statement 走 `FetchPreparedStatement(stmt_name, true)`。
unnamed statement 直接读 `unnamed_stmt_psrc`。
如果 unnamed statement 不存在，报 `UNDEFINED_PSTATEMENT`。
Bind 的第二个边界是参数数量和格式。
`numPFormats` 可以是 0、1 或等于参数数量。
如果大于 1 且不等于 `numParams`，就是 protocol violation。
`numParams` 必须等于 `psrc->num_params`。
这是 Parse 阶段和 Bind 阶段之间的参数契约。
Bind 的第三个边界是 user-defined I/O function。
text 参数会调用类型输入函数。
binary 参数会调用类型 receive 函数。
这意味着 Bind 不是纯协议解码。
它可能执行用户定义代码。
所以 Bind 需要 transaction command，需要 snapshot，也需要错误上下文记录具体参数。
Bind 的第四个边界是 plan refcount。
源码有明确注释：
```text
不要在 GetCachedPlan() 和 PortalDefineQuery() 之间放可能 ERROR 的代码。
```
因为 `GetCachedPlan()` 增加 cached plan 引用。
这个引用要交给 portal。
如果中间 ERROR，引用可能泄漏。
所以 Bind 先把可能 OOM 的参数和字符串复制做完，再调用 `GetCachedPlan()`，随后立即 `PortalDefineQuery()`。
Bind 的第五个边界是 `PortalStart()`。
这一步经常被忽略。
对于 `PORTAL_ONE_SELECT`，`PortalStart()` 会创建 `QueryDesc` 并调用 `ExecutorStart()`。
对于 `PORTAL_ONE_RETURNING`、`PORTAL_ONE_MOD_WITH` 和 `PORTAL_UTIL_SELECT`，它主要确定 `tupDesc`，执行会推迟到 `PortalRun()`。
对于 `PORTAL_MULTI_QUERY`，它此时甚至不需要设置 `tupDesc`。
所以不能简单说“Bind 不接触 executor”。
更精确的说法是：
```text
Bind 可能启动 executor state，但不推进 tuple 流。
Execute 才调用 PortalRun() 推进执行。
```
### 5.4 Describe statement: 描述 prepared statement
Describe statement 使用 subtype `'S'`。
入口是 `exec_describe_statement_message()`。
主链路：
```text
exec_describe_statement_message(stmt_name)
  -> start_xact_command()
  -> MemoryContextSwitchTo(MessageContext)
  -> FetchPreparedStatement() 或 unnamed_stmt_psrc
  -> aborted transaction block 下，如果 resultDesc 存在则拒绝
  -> 发送 ParameterDescription
  -> 如果 psrc->resultDesc 存在，发送 RowDescription
  -> 否则发送 NoData
```
Describe statement 面向 prepared statement。
它回答两个问题：
```text
参数有哪些类型？
执行结果有哪些列？
```
它不创建 portal。
它也不绑定参数值。
但发送 RowDescription 可能需要 targetlist。
源码通过 `CachedPlanGetTargetList(psrc, NULL)` 获取计划的 primary targetlist。
因此 aborted transaction 状态下，如果 statement 会返回数据，Describe 会拒绝。
### 5.5 Describe portal: 描述已绑定 portal
Describe portal 使用 subtype `'P'`。
入口是 `exec_describe_portal_message()`。
主链路：
```text
exec_describe_portal_message(portal_name)
  -> start_xact_command()
  -> MemoryContextSwitchTo(MessageContext)
  -> GetPortalByName(portal_name)
  -> aborted transaction block 下，如果 portal->tupDesc 存在则拒绝
  -> 如果 portal->tupDesc 存在，发送 RowDescription
  -> 否则发送 NoData
```
Describe portal 面向已经 Bind 的 portal。
它可以使用 `portal->tupDesc` 和 `portal->formats`。
这解释了为什么结果格式属于 Bind/portal，而不是 Parse/prepared statement。
同一个 prepared statement 可以 Bind 出不同 result formats 的 portal。
### 5.6 Execute: 推进 portal
Execute 消息包含：
```text
portal_name
max_rows
```
入口是 `exec_execute_message()`。
主链路：
```text
exec_execute_message(portal_name, max_rows)
  -> GetPortalByName(portal_name)
  -> null query 则 EmptyQueryResponse
  -> 判断 portal 是否 transaction command
  -> 复制 sourceText 和 prepStmtName 到 MessageContext
  -> pgstat_report_activity(STATE_RUNNING, sourceText)
  -> BeginCommand(portal->commandTag, dest)
  -> CreateDestReceiver(dest)
  -> start_xact_command()
  -> execute_is_fetch = !portal->atStart
  -> aborted transaction block 检查
  -> max_rows <= 0 转为 FETCH_ALL
  -> PortalRun(portal, max_rows, true, receiver, receiver, &qc)
  -> receiver->rDestroy(receiver)
  -> completed ? EndCommand 或 PortalSuspended
```
Execute 不直接调用 `ExecutorRun()`。
它调用 `PortalRun()`。
`PortalRun()` 再根据 portal strategy 分流：
```text
PORTAL_ONE_SELECT:
  PortalRunSelect()
    -> PushActiveSnapshot(queryDesc->snapshot)
    -> ExecutorRun(queryDesc, direction, count)
    -> PopActiveSnapshot()
PORTAL_ONE_RETURNING / PORTAL_ONE_MOD_WITH / PORTAL_UTIL_SELECT:
  首次运行 FillPortalStore()
  再从 holdStore 取结果
PORTAL_MULTI_QUERY:
  PortalRunMulti()
    -> ProcessQuery() 或 PortalRunUtility()
    -> run to completion
```
`max_rows` 是 Execute 和 Portal 的分批边界。
如果 `PortalRun()` 返回 `false`，表示 count 耗尽但 portal 尚未到 end。
`exec_execute_message()` 会发送 `PortalSuspended`。
portal 保持 `PORTAL_READY`，下一次 Execute 同一个 portal 时继续推进。
源码用 `execute_is_fetch = !portal->atStart` 区分日志中的第一次 execute 和后续 fetch。
这不是 SQL `FETCH` 命令。
它是 extended protocol 对同一 portal 的继续读取。
Execute 完成后还有事务边界判断。
如果 portal 是 transaction command，或者设置了 `XACT_FLAGS_NEEDIMMEDIATECOMMIT`，就立即 `finish_xact_command()`。
否则：
```text
CommandCounterIncrement()
MyXactFlags |= XACT_FLAGS_PIPELINING
disable_statement_timeout()
```
这说明 extended query 的多个消息之间可以共享一个 transaction command。
### 5.7 Sync: 协议 batch 的收口
Sync 消息处理很短：
```text
PqMsg_Sync:
  pq_getmsgend(&input_message)
  EndImplicitTransactionBlock()
  finish_xact_command()
  valgrind_report_error_query("SYNC message")
  send_ready_for_query = true
```
短不等于简单。
Sync 是 extended query protocol 的恢复点。
它做三件事。
第一，结束 pipelining 期间可能打开的 implicit transaction block。
第二，调用 `finish_xact_command()`，也就是 `CommitTransactionCommand()`。
第三，让 loop 下一轮发送 `ReadyForQuery()`。
`ReadyForQuery()` 不只是告诉客户端“可以发下一条了”。
它同时携带事务状态。
客户端由此知道 backend 当前是 idle、in transaction，还是 failed transaction。
这就是为什么 ERROR 后不能立即发 ReadyForQuery。
如果 extended message 出错，backend 会跳过到 Sync。
只有 Sync 处理完，才能重新报告稳定事务状态。
## 6. 状态随时间推进的完整故事
用一个 unnamed sequence 串起来：
```text
Parse("", "select * from t where id = $1", [int4])
Bind("", "", [42], result_formats)
Describe("P", "")
Execute("", 10)
Execute("", 10)
Sync
```
状态推进可以压缩成一张表：
| 时刻 | 关键状态 | 还不存在或不应推进的状态 |
| --- | --- | --- |
| Parse 前 | `MessageContext` 已 reset，旧 unnamed statement 可能仍由 `unnamed_stmt_psrc` 指向。 | 新 prepared statement、portal、`QueryDesc` 都不存在。 |
| Parse 后 | 旧 unnamed statement 被 `drop_unnamed_stmt()` 替换；新的 `CachedPlanSource` 保存 query tree、参数类型和结果描述。 | 没有 portal，也没有 executor runtime state。 |
| Bind 后 | unnamed portal 被创建或替换；参数、结果格式、query string 进入 `portalContext`；`GetCachedPlan()` 的引用交给 portal；普通 SELECT 已经 `PortalStart()`。 | tuple 还没有从 executor 拉出。 |
| Describe portal 后 | server 用 `portal->tupDesc` 和 `portal->formats` 发送 RowDescription。 | cursor 位置不变，executor 不推进。 |
| 第一次 Execute 后 | `PortalRunSelect()` 调 `ExecutorRun(..., count=10)`；未读完时 `portalPos` 增加，返回 `PortalSuspended`。 | portal 没有结束，`QueryDesc` 和 executor state 仍由 portal 保留。 |
| 第二次 Execute 后 | 同一 portal 从当前位置继续读；如果到末尾，`portal->atEnd = true` 并发送 `CommandComplete`。 | prepared statement 没有重新 Parse，portal 也不一定立刻 drop。 |
| Sync 后 | `EndImplicitTransactionBlock()` 和 `finish_xact_command()` 收口，下一轮发送 `ReadyForQuery`。 | extended batch 的错误跳过状态不应继续保留。 |
这条故事的判断点是：
```text
Parse 之后只有 prepared statement；
Bind 之后才有 portal；
PortalStart 之后可能已有 executor state；
Execute 改变 portal 位置；
Sync 把协议 batch 和 transaction command 收口。
```
## 7. 生命周期 / ownership / cleanup
### 7.1 谁创建
`CachedPlanSource` 由 Parse 创建。
named statement 由 `StorePreparedStatement()` 放入 session-level prepared statement 表。
unnamed statement 由 `unnamed_stmt_psrc` 指向。
`Portal` 由 Bind 创建。
`CreatePortal()` 分配 `PortalData`，创建 `portal->portalContext`，创建 `portal->resowner`。
`CachedPlan` 由 Bind 中的 `GetCachedPlan()` 取得。
引用交给 `PortalDefineQuery()`。
`QueryDesc` 对普通 SELECT 由 `PortalStart()` 创建。
`ExecutorStart()` 也在 `PortalStart()` 中调用。
`DestReceiver` 由 Execute 创建。
它分配在 `MessageContext`，因为某些 utility command 可能在执行中改变 transaction context。
### 7.2 谁持有
prepared statement 持有 parse/analyze/rewrite 后的语义状态。
portal 持有一次 Bind 的参数值、结果格式、cached plan 引用和 executor 状态。
executor 不拥有 portal。
executor state 挂在 `QueryDesc` / `EState` / executor context 下，但其生命周期由 portal cleanup 驱动。
resource ownership 分两层。
portal 有自己的 `ResourceOwner`。
进入 `PortalStart()` 或 `PortalRun()` 时，`CurrentResourceOwner` 临时切到 `portal->resowner`。
deep code 获取的 snapshot、buffer pin、锁等资源会挂到当前 owner。
但事务 owner 仍然是上层边界。
普通 portal drop 时，非 FAILED 情况下锁可以转交给事务 owner，其他资源按 ResourceOwner phase 释放。
### 7.3 谁释放
unnamed prepared statement 由 `drop_unnamed_stmt()` 释放。
named prepared statement 由 `DropPreparedStatement()`、session cleanup 或相关命令释放。
portal 由以下路径释放：
```text
Close Portal
  -> PortalDrop(portal, false)
事务结束
  -> PreCommit_Portals()
  -> AtAbort_Portals()
  -> AtCleanup_Portals()
错误恢复
  -> PortalErrorCleanup()
普通 drop
  -> PortalDrop()
```
`PortalDrop()` 的顺序很重要：
```text
检查 pinned / active
调用 portal->cleanup
从 portal hash table 删除
PortalReleaseCachedPlan()
释放 holdSnapshot
ResourceOwnerRelease()
tuplestore_end()
MemoryContextDelete(holdContext)
MemoryContextDelete(portalContext)
pfree(portal)
```
它先从 hash table 删除，是为了避免后续 cleanup 出错时进入无限错误恢复循环。
这意味着错误场景下宁可泄漏少量内存，也要避免重复 drop 同一个 portal。
### 7.4 ERROR / abort 时谁兜底
顶层 `PostgresMain()` 错误恢复是 extended query 的兜底。
核心步骤：
```text
EmitErrorReport()
debug_query_string = NULL
AbortCurrentTransaction()
PortalErrorCleanup()
ReplicationSlotCleanup(false)
jit_reset_after_error()
MemoryContextSwitchTo(MessageContext)
FlushErrorState()
if doing_extended_query_message:
  ignore_till_sync = true
xact_started = false
```
`AbortCurrentTransaction()` 处理事务资源。
`PortalErrorCleanup()` 处理 active portal 状态。
`MessageContext` 清理消息级临时内存。
`ignore_till_sync` 处理协议级恢复。
这四层不能互相替代。
MemoryContext reset 不能释放所有外部资源。
ResourceOwner release 不能让客户端和服务端重新同步消息语义。
Sync cleanup 不能代替 executor end。
## 8. 正确性机制层次
extended query 的正确性不是由一个锁或一个 context 保证的。
它依赖多个边界叠加。
| 层次 | 机制 | 保证 | 不保证 |
| --- | --- | --- | --- |
| 协议同步 | `doing_extended_query_message` / `ignore_till_sync` / Sync | ERROR 后不继续执行半失效 pipeline。 | 不释放 executor 资源。 |
| 事务命令 | `start_xact_command()` / `finish_xact_command()` | Parse/Bind/Execute 能在事务语义下访问 catalog、运行 I/O 函数和提交命令。 | 不定义 prepared statement 生命周期。 |
| prepared statement | `CachedPlanSource` | 保存 query 语义、参数类型和结果描述。 | 不代表一次具体执行。 |
| portal | `PortalData` | 保存参数、格式、cursor 位置、cached plan ref 和 executor envelope。 | 不跨 backend 共享。 |
| plan cache refcount | `GetCachedPlan()` / `ReleaseCachedPlan()` | portal 使用期间 plan list 不被释放。 | 不保证 plan 永不过期。 |
| MemoryContext | `MessageContext` / `portalContext` / plan source context | 内存按消息、portal、prepared statement 生命周期释放。 | 不释放锁、snapshot、pin。 |
| ResourceOwner | `portal->resowner` / transaction owner | 外部资源在 ERROR、drop、commit、abort 中按 phase 释放或转移。 | 不保存 SQL 语义。 |
| snapshot | `PushActiveSnapshot()` / `RegisterSnapshot()` | parse analysis、parameter I/O、executor 读取期间有一致可见性边界。 | 不提供互斥。 |
| Portal status | `PORTAL_READY` / `ACTIVE` / `FAILED` | 防止 drop active portal，标记失败 portal 不可重跑。 | 不说明事务是否 committed。 |
| ReadyForQuery | `ReadyForQuery(whereToSendOutput)` | 把 backend 事务状态暴露给客户端。 | 不表示每个 portal 都被显式 Close。 |
正确性推理要按时间问：
```text
现在客户端消息读完整了吗？
当前 transaction command 是否打开？
当前对象是 prepared statement 还是 portal？
portal 是否已经 PortalStart？
executor 是否已经 ExecutorRun？
如果这里 ERROR，当前 owner 是谁？
客户端什么时候能再次收到 ReadyForQuery？
```
这组问题比背函数名更有用。
## 9. 错误路径 / 异常路径 / fallback
### 9.1 Parse 阶段多语句
Parse 允许空字符串。
空字符串会创建 `CMDTAG_UNKNOWN` 的 plan source。
Execute 这种 portal 会返回 `EmptyQueryResponse`。
但 Parse 不允许 prepared statement 中包含多条用户语句。
错误：
```text
cannot insert multiple commands into a prepared statement
```
这是协议形状限制，不是 SQL parser 能力限制。
如果允许多语句 prepared statement，Describe result、parameter binding、portal strategy 和 command tag 都会变复杂。
PostgreSQL 把这个复杂性留给 simple query protocol。
### 9.2 unnamed statement 被替换
unnamed statement 是单槽位。
新的 unnamed Parse 会释放旧 `unnamed_stmt_psrc`。
Close Statement 且名字为空也释放它。
simple query path 进入时也会 drop unnamed statement。
这个规则让短查询少走 hashtable，但要求客户端不要把 unnamed statement 当成稳定对象。
如果客户端需要稳定复用，就用 named prepared statement。
### 9.3 unnamed portal 被替换
Bind 创建 portal 时：
```text
portal_name == "":
  CreatePortal(portal_name, true, true)
portal_name != "":
  CreatePortal(portal_name, false, false)
```
unnamed portal 可以静默替换。
named portal 重名会报 duplicate cursor。
原因是 unnamed portal 被设计为“当前这次绑定”的快捷槽位。
named portal 则有显式生命周期。
### 9.4 Bind 参数输入函数 ERROR
Bind 解析参数时可能调用类型输入函数或 receive 函数。
这些函数可能 ERROR。
源码为单个参数设置 `bind_param_error_callback()`。
错误上下文可以显示 portal 名字和参数序号。
如果配置允许，还能记录参数文本。
ERROR 后，顶层恢复会 abort 当前事务，并进入 skip till Sync。
这说明参数错误不是普通客户端解析失败。
它可能已经运行了 backend 内部函数。
必须走事务错误恢复。
### 9.5 GetCachedPlan 到 PortalDefineQuery 之间不能 ERROR
Bind 中最关键的 leak window 是：
```text
cplan = GetCachedPlan(psrc, params, NULL, NULL)
PortalDefineQuery(portal, ..., cplan)
```
`GetCachedPlan()` 增加 refcount。
`PortalDefineQuery()` 把这个引用交给 portal。
如果两者之间插入可能 ERROR 的代码，portal 还没接手，引用可能泄漏。
源码注释明确禁止这样做。
这是内核 patch review 中很典型的局部正确性窗口。
### 9.6 aborted transaction block 中的限制
Parse、Bind、Describe、Execute 都有 aborted transaction block 检查。
它们允许 transaction exit statement。
它们拒绝普通语句。
Bind 还拒绝带参数的情形，因为不能冒险调用用户定义 I/O 函数。
Describe statement 和 portal 在结果描述需要 catalog access 时也会拒绝。
这不是重复防御。
每个阶段能做的事情不同，能安全允许的子集也不同。
### 9.7 ERROR 后 skip till Sync
extended message 处理中出错后，backend 不会继续处理后续 Bind/Execute。
它设置 `ignore_till_sync = true`。
main loop 会跳过所有消息，直到 Sync。
同时不会发送 ReadyForQuery。
客户端只有在发送 Sync 后，才能收到恢复后的 ReadyForQuery。
这保护了两个边界：
```text
消息流边界:
  客户端和服务端重新对齐。
事务状态边界:
  ReadyForQuery 报告的是恢复后的状态。
```
### 9.8 Execute suspended
Execute 的 `max_rows` 可以让 portal 未完成。
`PortalRun()` 返回 false。
server 发送 `PortalSuspended`。
这不是错误。
portal 保持可继续运行状态。
下一次 Execute 同一个 portal 会继续从 `portalPos` 推进。
这一路径解释了为什么 executor 不能只暴露“跑完整棵树”的接口。
## 10. 成本、资源与跨模块传播
### 10.1 网络往返和 pipeline
extended query 把一条 SQL 分成多条消息。
如果客户端逐条等待响应，网络 RTT 成本会放大。
如果客户端 pipeline 多条 Parse/Bind/Execute，再用 Sync 收口，RTT 可以下降。
但 pipeline 会让 backend 在多个消息之间保持 transaction command。
源码用 `XACT_FLAGS_PIPELINING` 和 implicit transaction block 处理这个状态。
所以 pipeline 不是“无状态批处理”。
它会改变事务边界和错误恢复方式。
### 10.2 plan cache 成本
Parse 创建 `CachedPlanSource`，但 Bind 通过 `GetCachedPlan()` 才取得具体 `CachedPlan`。
成本可能出现在：
```text
首次 Bind 需要 planning；
custom plan 会使用参数值重新规划；
generic plan 可复用但可能不如 custom plan；
catalog invalidation 会让 cached plan 失效并重建。
```
本节不展开 generic/custom heuristic。
但诊断 extended query 时要知道：慢的可能是 Bind，不一定是 Execute。
在日志里看到 `duration: ... bind ...` 变长时，应想到 `GetCachedPlan()`、参数 I/O 和 replanning。
### 10.3 参数 I/O 成本
text format 参数要做 encoding conversion 和 type input。
binary format 参数要调用 receive function，并检查 buffer 是否被完整消费。
这些成本随参数个数、参数大小、类型输入函数复杂度扩张。
大数组、json、numeric、geometry 或自定义类型可能让 Bind 本身成为热点。
不要把所有 CPU 都归给 executor。
### 10.4 portal 内存 retention
portal 的参数、格式、query string、`QueryDesc` 和 executor state 都挂在 `portal->portalContext` 或其子 context 下。
如果客户端创建 named portal 后长期不 Close，backend-local 内存会保留。
如果 Execute suspended，executor state 也可能保留。
这不是 shared memory 问题。
它是单 backend 长连接的 local memory retention。
观测上可以从 `pg_backend_memory_contexts` 看到 `PortalContext`。
但看不到所有外部资源的完整因果。
### 10.5 snapshot 与 xmin
普通 Parse/Bind 的 snapshot 很短。
它们用于 parse analysis、parameter I/O 或 planning。
`PORTAL_ONE_SELECT` 的 `QueryDesc` 注册 snapshot，在 executor 读取期间使用。
如果 portal suspended 并长期不关闭，这个 snapshot 相关状态可能延长可见性 horizon。
这会影响 VACUUM 能否移除旧 tuple。
具体影响依赖事务状态、cursor 类型、snapshot 注册方式和 portal cleanup。
不能只看 portal 名字就断言 bloat。
### 10.6 DestReceiver 与客户端背压
Execute 创建 `DestRemoteExecute` receiver。
executor 产出的 tuple 最终要走 libpq output buffer 和 socket。
如果客户端读得慢，backend 可能在 Client wait 或 socket write 上耗时。
从 SQL 视角看这是 Execute 慢。
从 executor 视角看可能不是节点计算慢。
这就是 04 目录 wait event 和 DestReceiver 后续课程要接上的边界。
### 10.7 跨模块连接
本节至少连接这些模块：
| 模块 | 连接点 |
| --- | --- |
| parser/analyzer/rewrite | Parse 中的 `pg_parse_query()` 与 `pg_analyze_and_rewrite_varparams()`。 |
| plan cache | Parse 创建 `CachedPlanSource`，Bind 调用 `GetCachedPlan()`。 |
| portal manager | Bind 创建 portal，ERROR/commit/drop 清理 portal。 |
| executor | `PortalStart()` 调 `ExecutorStart()`，Execute 经 `PortalRun()` 调 `ExecutorRun()`。 |
| transaction manager | `start_xact_command()`、`finish_xact_command()`、implicit transaction block。 |
| snapshot manager | Parse/Bind/PortalRun 中 active snapshot 和 registered snapshot。 |
| stats/activity | `pgstat_report_activity()`、query id、plan id、duration logging。 |
| protocol output | ParseComplete、BindComplete、RowDescription、CommandComplete、PortalSuspended、ReadyForQuery。 |
## 11. 观测与诊断入口
本节锚定的 runtime truth 是：
```text
同一条 SQL 在 extended query 下，耗时和状态可能分布在 Parse、Bind、Describe、Execute、Sync 多个消息中。
```
如果只盯单次 Execute，会漏掉参数 I/O、planning、portal 创建和 Sync cleanup。
### 11.1 日志
`postgres.c` 分别对 parse、bind、execute 做 duration logging。
典型日志语义：
```text
duration: ... parse <stmt>: <sql>
duration: ... bind <stmt>/<portal>: <sql>
duration: ... execute <stmt>/<portal>: <sql>
duration: ... execute fetch from <stmt>/<portal>: <sql>
```
如果 Bind 慢，优先怀疑：
```text
参数类型输入或 receive 函数；
GetCachedPlan() planning；
plan invalidation 后重建；
参数日志构造；
大量参数复制。
```
如果 Execute 慢，继续区分：
```text
executor 节点 CPU；
buffer / IO；
锁等待；
客户端输出背压；
PortalSuspended 后多次 Execute 的总耗时。
```
### 11.2 `pg_stat_activity`
Parse 会设置 ps display 为 `PARSE`。
Bind 会设置为 `BIND`。
Execute 会根据 command tag 设置显示。
activity query string 对 Parse 是当前 query string。
对 Bind 和 Execute 是 prepared statement 的 `psrc->query_string` 或 portal source text。
能看到的是 backend 当前正在处理哪个阶段。
看不到的是 portal 内部所有字段。
也看不到 plan cache refcount。
### 11.3 `pg_backend_memory_contexts`
可以观察 backend local memory context。
重点看：
```text
MessageContext
PortalContext
RowDescriptionContext
CacheMemoryContext
```
unnamed statement 的 context 可能以 `unnamed prepared statement` 出现。
portal context 的 identifier 对 named portal 是 portal 名字，对 unnamed portal 是 `<unnamed>`。
这能帮助判断长连接是否保留了 portal 或 prepared statement 相关内存。
但它不能说明锁、snapshot、buffer pin 是否还被 ResourceOwner 持有。
### 11.4 `pg_cursors`
`pg_cursors` 能看到 visible cursor。
它更适合 SQL cursor 和 named portal 诊断。
extended protocol 的 unnamed portal 通常不是面向用户长期观察的对象。
如果你看到 cursor 长期存在，应继续追：
```text
是否 holdable；
是否还有 active transaction；
是否正在 suspended；
是否客户端忘记 Close；
是否事务结束 cleanup 没到。
```
### 11.5 wait event
Parse/Bind/Execute 都可能等待。
常见方向：
```text
Bind:
  type input 函数内部访问 catalog 或 IO；
  planning 期间锁或 buffer IO；
Execute:
  Lock；
  BufferPin；
  IO；
  ClientWrite；
  IPC；
Sync:
  commit 期间 WAL flush；
  transaction cleanup 相关等待。
```
wait event 是当前等待点，不是完整因果。
如果看到 Execute 卡在 `ClientWrite`，executor 可能已经产出 tuple，只是客户端接收慢。
如果看到 Sync 慢，可能是 commit/WAL，而不是 executor。
### 11.6 gdb 断点
源码跟读最直接的断点：
```text
b exec_parse_message
b exec_bind_message
b exec_describe_statement_message
b exec_describe_portal_message
b exec_execute_message
b PortalStart
b PortalRun
b PortalRunSelect
b PortalDrop
b drop_unnamed_stmt
```
断在 Bind 时建议打印：
```text
p stmt_name
p portal_name
p psrc->num_params
p portal->status
p portal->portalContext
p portal->resowner
```
断在 Execute 时建议打印：
```text
p portal->status
p portal->strategy
p portal->atStart
p portal->atEnd
p portal->portalPos
p portal->queryDesc
```
断在错误恢复时建议打印：
```text
p doing_extended_query_message
p ignore_till_sync
p xact_started
```
### 11.7 能看到、只能推断、基本不可见
| 类型 | 状态 |
| --- | --- |
| 能直接看到 | 日志中的 parse/bind/execute duration，`pg_stat_activity` state/query/wait event，`pg_backend_memory_contexts` 的 context。 |
| 需要推断 | Bind 是否在 replanning，portal 是否因 suspended 保留 executor state，Sync 是否在 commit flush。 |
| 基本不可见 | `CachedPlan` refcount、`ignore_till_sync`、`PortalStatus`、`portalPos`、`CurrentResourceOwner` 切换。 |
高级诊断需要组合日志、wait event、gdb、perf 和源码断点。
不要把 `pg_stat_*` 当作完整 runtime reality。
## 12. 常见误区
### 12.1 把 protocol Parse 当成纯 parser
protocol `Parse` 不只是 `pg_parse_query()`。
它还会 analyze/rewrite，创建 `CachedPlanSource`，并建立参数类型和结果描述。
但它通常不取得最终 `CachedPlan`。
### 12.2 认为 Bind 不会接触 executor
Bind 会调用 `PortalStart()`。
对于普通 SELECT，`PortalStart()` 会创建 `QueryDesc` 并调用 `ExecutorStart()`。
更准确的边界是：
```text
Bind 可以启动 executor state；
Execute 才推进 executor tuple flow。
```
### 12.3 把 prepared statement 和 portal 混成一个对象
prepared statement 是可复用语义。
portal 是一次绑定后的执行 envelope。
一个 prepared statement 可以 Bind 多个 portal。
一个 portal 持有具体参数、格式、cursor 位置和 cached plan 引用。
### 12.4 认为 unnamed 对象只是名字为空
unnamed statement 和 unnamed portal 都有特殊替换规则。
unnamed statement 由单独静态指针保存。
unnamed portal 可以被 Bind 静默替换。
这些不是普通 hashtable key 为空字符串那么简单。
### 12.5 把 `PortalSuspended` 当错误
`PortalSuspended` 表示 Execute 的 `max_rows` 用完但 portal 未完成。
它是正常协议响应。
下一次 Execute 同一 portal 会继续推进。
### 12.6 认为 Sync 只是 flush
Flush 只是要求发送已产生的输出。
Sync 是 extended query batch 的事务和错误恢复边界。
Sync 会 `finish_xact_command()`，并使下一轮发送 `ReadyForQuery`。
### 12.7 只用 Execute 耗时解释慢 SQL
extended query 下慢点可能在 Parse、Bind、Execute 或 Sync。
Bind 慢可能是 parameter I/O 或 planning。
Sync 慢可能是 commit/WAL。
Execute 慢才更接近 executor tuple flow。
## 13. 课堂实验
### 13.1 源码断点实验：看对象何时出现
目标：证明 Parse、Bind、Execute 分别创建不同对象。
步骤：
```text
1. 启动本地 PostgreSQL，连接一个客户端。
2. 用支持 extended query 的客户端或测试程序发送参数化查询。
3. 在 gdb 中设置断点：
   b exec_parse_message
   b exec_bind_message
   b PortalStart
   b exec_execute_message
   b PortalRun
4. Parse 断点处打印 unnamed_stmt_psrc。
5. Bind 断点处打印 psrc、portal、portal->status。
6. PortalStart 后打印 portal->queryDesc。
7. Execute 前后打印 portal->atStart、portal->atEnd、portal->portalPos。
```
预期现象：
```text
Parse 后有 CachedPlanSource；
Bind 后有 Portal；
PortalStart 后普通 SELECT 有 QueryDesc；
Execute 后 portal position 改变。
```
回到源码解释：
```text
exec_parse_message()
  -> SaveCachedPlan()
exec_bind_message()
  -> CreatePortal()
  -> GetCachedPlan()
  -> PortalDefineQuery()
  -> PortalStart()
exec_execute_message()
  -> PortalRun()
```
### 13.2 SQL 现象实验：观察 max_rows 和 PortalSuspended
目标：证明 Execute 可以分批推进同一个 portal。
建议用一个小客户端直接使用 libpq extended API，或在驱动中设置 fetch size。
构造：
```sql
create table eq_demo(id int primary key, payload text);
insert into eq_demo
select g, repeat('x', 20)
from generate_series(1, 50) g;
```
客户端发送：
```text
Parse unnamed: select * from eq_demo order by id
Bind unnamed portal
Execute unnamed portal max_rows = 10
Execute unnamed portal max_rows = 10
Execute unnamed portal max_rows = 0
Sync
```
观察：
```text
前两次 Execute 应返回部分行并可能伴随 PortalSuspended；
最后一次 Execute 读完并返回 CommandComplete；
server 日志可能出现 execute fetch from。
```
源码解释：
```text
exec_execute_message()
  -> max_rows <= 0 ? FETCH_ALL
  -> PortalRun()
  -> PortalRunSelect()
  -> ExecutorRun(..., count)
```
### 13.3 错误路径实验：参数错误后必须 Sync
目标：观察 extended message ERROR 后 skip till Sync。
构造：
```sql
select $1::int;
```
客户端发送：
```text
Parse
Bind 参数值 = 'not-an-int'
Execute
Sync
```
预期：
```text
Bind 报 invalid input syntax；
后续 Execute 在 Sync 前不会真正执行；
Sync 后收到 ReadyForQuery。
```
源码断点：
```text
b bind_param_error_callback
b PostgresMain
```
在错误恢复处观察：
```text
doing_extended_query_message = true
ignore_till_sync = true
```
解释：
```text
Bind 调用类型输入函数产生 ERROR；
顶层恢复 AbortCurrentTransaction 和 PortalErrorCleanup；
协议层进入 skip-till-Sync；
Sync 清理状态并触发 ReadyForQuery。
```
## 14. 讨论题
1. 为什么 Parse 阶段要做 analyze/rewrite，而不是把所有工作推迟到 Bind？
2. 为什么 Bind 要先完成可能 OOM 的复制和参数解析，再调用 `GetCachedPlan()`？
3. 为什么 `GetCachedPlan()` 和 `PortalDefineQuery()` 之间不能插入可能 ERROR 的代码？
4. 如果 Execute 返回 `PortalSuspended`，portal 中哪些状态必须保留下来？
5. 为什么 unnamed statement 使用 `unnamed_stmt_psrc` 单独保存，而不是普通 prepared statement hashtable？
6. Describe statement 和 Describe portal 的结果格式信息为什么不完全来自同一个对象？
7. ERROR 后为什么不能立即发送 ReadyForQuery，而要跳过到 Sync？
8. 如果线上看到 Bind duration 很高，哪些模块应该先被排查？
9. MemoryContext 能释放 portal 内存，为什么还需要 `ResourceOwner` 和 `PortalDrop()`？
## 15. 本节小结
本节的核心链路是：
```text
PostgresMain message loop
  -> Parse 创建 CachedPlanSource
  -> Bind 创建 Portal 并取得 CachedPlan
  -> Describe 暴露 prepared statement 或 portal 的描述信息
  -> Execute 通过 PortalRun 推进 executor
  -> Sync 收口 transaction command 并触发 ReadyForQuery
```
核心状态边界是：
```text
CachedPlanSource:
  prepared statement 语义缓存
PortalData:
  一次参数化执行的 envelope
QueryDesc / EState:
  executor runtime state
MessageContext:
  单条协议消息临时内存
portalContext / resowner:
  portal 跨消息内存和外部资源 owner
```
ownership 规则是：
```text
Parse 拥有 prepared statement；
Bind 把 CachedPlan 引用交给 portal；
PortalStart 可能创建 executor state；
Execute 只推进 portal；
PortalDrop、事务结束和 ERROR cleanup 负责释放；
Sync 负责协议 batch 和 transaction command 收口。
```
错误路径的核心规律是：
```text
extended message 中 ERROR 后，backend abort 当前事务并清理 active portal，
然后 ignore till Sync，直到 Sync 后才重新 ReadyForQuery。
```
能观测的部分包括 parse/bind/execute duration、`pg_stat_activity`、wait event、`pg_backend_memory_contexts` 和 `pg_cursors`。
不能直接观测的部分包括 `PortalStatus`、plan refcount、`ignore_till_sync`、`portalPos` 和 `CurrentResourceOwner` 切换。
从本节可以迁移出的系统规律是：
```text
协议拆阶段会降低客户端重复工作和支持 pipelining，
但每个阶段都必须有明确的状态 owner、错误恢复点和最终收口消息；
否则网络协议的灵活性会变成 backend-local 状态泄漏和事务语义错位。
```
判断 extended query 问题时，先问：
```text
慢点在哪个消息阶段？
当前对象是 prepared statement、portal，还是 executor state？
ERROR 时谁会 cleanup？
客户端是否已经发送 Sync？
ReadyForQuery 报告的是哪个事务状态？
```
这些问题比“Parse/Bind/Execute 分别叫什么函数”更能帮助定位真实内核问题。
