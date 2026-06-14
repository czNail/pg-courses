# PostgreSQL frontend/backend protocol message loop
## 课程定位
前置知识：熟悉 PostgreSQL backend 进程模型，知道认证完成后每个客户端连接通常由一个 backend 进程服务。
前置知识：已经读过本目录前几节 Executor 生命周期，知道 `PortalRun()` 最终会进入 `ExecutorStart()` / `ExecutorRun()` / `ExecutorEnd()`。
本节唯一主问题：
```text
post-auth backend 如何在 frontend/backend protocol message loop 中分派 simple query 和 extended query？
```
核心矛盾：客户端希望把 SQL 发送、参数绑定、执行、取结果和同步点拆成灵活协议；backend 必须在一个长生命周期会话循环里保持事务边界、内存边界、错误恢复边界和协议同步。
学完后应能判断：一个现象到底发生在协议消息层、transaction command 层、Portal / plan cache 层，还是 executor 层。
学完后还应能解释：为什么 simple query 看起来像“一条字符串直接执行”，而源码里仍会创建 portal；为什么 extended query 的 `Parse` / `Bind` / `Execute` 不在每条消息后发送 `ReadyForQuery`。
本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e90741e12a3715909e1b2d71ff9344`。
本节不讲认证前 startup packet、SSL/GSS 协商和身份认证细节。
本节也不展开 plan cache 的 generic/custom plan 选择。
这些主题会在后续 `58` 到 `64` 课程中继续拆开。
## 1. 本节在总主线中的位置
04 目录前面的课程从 executor 内部往外讲。
第 1 节到第 5 节讲的是 `ExecutorStart()` 到 `ExecutorEnd()`。
第 25 节到第 29 节讲的是 `EXPLAIN ANALYZE` 如何观察执行器。
第 47 节到第 56 节讲的是如何从慢 SQL 和 profiler 回到执行器源码。
现在要补上更外层的一圈：客户端消息如何到达 executor。
这不是 executor 本身的一节课。
它是 executor 课程的入口层。
没有这一层，读者看到 `ExecutorRun()` 时会误以为所有 SQL 都以同一种方式进入执行器。
真实情况是：
```text
frontend message loop
  -> simple query path
     -> unnamed portal
     -> run to completion
     -> ReadyForQuery
  -> extended query path
     -> prepared statement
     -> portal
     -> Execute one step
     -> Sync as transaction/protocol boundary
```
本节只回答消息循环如何分派。
后续课程会继续追 simple query 的 transaction command 细节、extended query 的 parse/bind/execute 分阶段语义、prepared statement plan cache、Portal 暂停和 DestReceiver 输出协议。
阅读时要保持一个时间轴：backend 已经通过认证，`PostgresMain()` 进入普通会话循环，每轮读一个 frontend message，再决定是否推进事务、portal 和 executor。
## 2. 核心矛盾与一句话运行模型
一句话运行模型：
```text
PostgresMain() 在每轮循环开始用 ReadyForQuery 报告上一个同步点的事务状态，
随后 ReadCommand() 读一个 frontend message，
再按首字节分派到 exec_simple_query() 或 extended query 的 Parse/Bind/Execute/Sync 处理函数。
```
这里的系统 tension 是：
```text
协议灵活性和 pipeline/portal 能力
  vs
backend-local 状态必须在 ERROR、cancel、timeout、client disconnect 后保持可恢复和可同步
```
simple query 选择把一整个 query string 当作一个消息处理。
它可以包含多个 SQL command。
源码会逐个 raw parsetree 处理，但从协议层看仍是一个 `Q` message。
extended query 选择把生命周期拆成多个协议消息。
`Parse` 建立 `CachedPlanSource`。
`Bind` 建立 `Portal` 并取得 `CachedPlan`。
`Execute` 运行 portal，可能只取 `max_rows` 行。
`Sync` 才是协议层要求 backend 回到同步状态并发送 `ReadyForQuery` 的点。
这就形成一个核心差异：
simple query 的“消息边界”通常也是“本次执行完成边界”。
extended query 的“消息边界”不是执行完成边界。
extended query 的真正同步点由 `Sync` 决定。
PostgreSQL 不能把所有消息都处理成同一个函数调用。
原因不是 API 风格。
原因是不同协议消息持有的状态生命周期不同。
`Q` message 的临时解析、计划和 unnamed portal 大多可以随本轮 `MessageContext` reset 释放。
`Parse` message 创建的 prepared statement 可能跨多轮 message 存活。
`Bind` message 创建的 portal 可能跨一次或多次 `Execute` 存活。
`Execute` message 可能只推进 portal 一部分。
`Sync` message 才告诉 backend：这批 extended messages 到了一个客户端可理解的同步边界。
这也解释了为什么 observability 不能只盯 `ExecutorRun()`。
同一条 SQL 在 simple query 下，日志、pg_stat_activity、statement timeout 和 command complete 的边界，与 extended query 下可能不同。
## 3. 核心文件分工与阅读顺序
| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/tcop/postgres.c` | `PostgresMain()` 主循环、`ReadCommand()`、simple/extended message dispatch、事务命令边界和顶层 ERROR recovery。 |
| 2 | `src/backend/libpq/pqcomm.c` | socket 读写、`pq_startmsgread()` / `pq_getmessage()`、协议同步丢失检测、client read/write interrupt 边界。 |
| 3 | `src/backend/libpq/pqformat.c` | `StringInfo` message body 的 typed getter/setter，例如 `pq_getmsgint()`、`pq_getmsgstring()`、`pq_getmsgend()`。 |
| 4 | `src/backend/tcop/pquery.c` | `PortalStart()` / `PortalRun()` 如何把 portal 接到 executor 或 utility command。 |
| 5 | `src/backend/tcop/dest.c` | `ReadyForQuery()`、`BeginCommand()`、`EndCommand()` 如何形成客户端可见输出边界。 |
| 6 | `src/include/tcop/tcopprot.h` | `PostgresMain()`、`pg_parse_query()`、analyze/rewrite/plan 入口声明。 |
| 7 | `src/include/tcop/pquery.h` | `PortalStart()`、`PortalRun()`、`PortalRunFetch()` 的 portal 执行入口声明。 |
| 8 | `src/include/tcop/dest.h` | `CommandDest`、`DestReceiver`、`ReadyForQuery()` 等输出目标声明。 |
| 9 | `src/include/libpq/protocol.h` | `PqMsg_Query`、`PqMsg_Parse`、`PqMsg_Bind`、`PqMsg_Execute`、`PqMsg_Sync` 等消息字节。 |
| 10 | `src/include/libpq/libpq.h` | backend 侧 libpq 通信接口和 `ClientSocket` / `whereToSendOutput` 等边界。 |
| 11 | `src/include/libpq/pqformat.h` | protocol message 编解码 helper 的声明。 |
| 12 | `src/backend/utils/init/postinit.c` | `InitPostgres()` 如何在 `PostgresMain()` 进入消息循环前建立数据库、用户、GUC、cache 和 transaction 初始状态。 |
建议阅读顺序不是按文件名。
先读 `PostgresMain()`。
再读 `SocketBackend()` 和 `ReadCommand()`。
然后读 `exec_simple_query()`。
再读 `exec_parse_message()`、`exec_bind_message()`、`exec_execute_message()`。
最后才进入 `pquery.c` 看 portal 如何运行。
不要从 `libpq` 的所有函数开始背协议 API。
本节的主线是状态推进，不是 message helper 清单。
`pq_getmsgint()` 这种函数只在解释“为什么消息体必须先读完整、再逐字段解析”时有意义。
如果读者已经熟悉 PostgreSQL wire protocol，也仍然要读 `postgres.c`。
因为 backend 的实现边界和协议文档不是一回事。
协议文档告诉你 `P` 是 Parse、`B` 是 Bind、`E` 是 Execute。
源码告诉你这些消息何时启动事务、何时保留状态、何时跳到 error recovery、何时发送 `ReadyForQuery`。
## 4. 关键状态与结构
### 4.1. `PostgresMain()` 的 loop-local 状态
`PostgresMain()` 是 post-auth backend 的主入口。
认证前后有很多启动工作，但本节从 `InitPostgres()` 完成后开始理解。
关键 loop-local 状态包括：
| 状态 | 位置 | 语义 |
| --- | --- | --- |
| `send_ready_for_query` | `PostgresMain()` 局部变量 | 当前 loop 顶部是否要向客户端发送 `ReadyForQuery`。 |
| `ignore_till_sync` | `postgres.c` static 状态 | extended query 出错后，是否跳过后续消息直到 `Sync`。 |
| `doing_extended_query_message` | `postgres.c` static 状态 | 当前消息是否属于 extended query，用于 ERROR 后决定是否进入 skip-till-Sync。 |
| `DoingCommandRead` | `postgres.c` static 状态 | backend 是否正在等待客户端消息，用于 cancel、timeout、notify、sinval 等 interrupt 处理。 |
| `xact_started` | `postgres.c` static 状态 | tcop 层是否已经打开 transaction command。 |
| `MessageContext` | backend-local memory context | 每轮消息循环 reset 的短生命周期内存。 |
| `row_description_context` | backend-local memory context | 复用 RowDescription message buffer，减少频繁分配。 |
这些状态都是 backend-local。
其他 backend 不能直接读它们。
它们不是 shared memory。
它们也不是 SQL 可见状态。
`pg_stat_activity.state` 只能看到这些状态的一部分投影。
例如 `DoingCommandRead` 为 true 时，用户一般只能看到 backend 处于 idle 或 idle in transaction 等状态。
### 4.2. `StringInfoData input_message`
每轮 loop 会在 `MessageContext` 中创建一个 `StringInfoData input_message`。
`ReadCommand()` 把当前 frontend message body 读进这个 buffer。
后续 `pq_getmsgstring()`、`pq_getmsgint()`、`pq_getmsgbyte()` 从这个 buffer 顺序解析字段。
它的生命周期很短。
本轮 message 处理结束后，下轮 loop 开头 `MemoryContextReset(MessageContext)` 会释放它。
这解释了两个实现习惯。
第一，处理函数不会长期保存指向 `input_message.data` 的裸指针，除非把内容复制到更长生命周期的 context。
第二，extended query 中需要跨 message 存活的内容，必须被复制到 prepared statement context、portal context 或 plan cache 管理的 context。
`input_message` 的边界也是协议正确性的边界。
`pq_getmsgend()` 用来确认消息体已经被完整消费。
如果一个消息字段解析出错或消息体未按预期结束，backend 会报告 protocol violation。
### 4.3. `MessageContext`
`MessageContext` 是每轮消息处理的主临时 context。
它在 `PostgresMain()` 初始化一次。
每轮 loop 开头 reset。
simple query 的 raw parse tree、query tree、plan tree 和 unnamed portal 输入大多会落在这里或其子 context。
extended query 的 named prepared statement 不应依赖它。
unnamed prepared statement 会特殊处理。
`exec_parse_message()` 对 named statement 的策略是：在 `MessageContext` 里临时 parse，再把完成后的树拷贝进 plancache entry。
对 unnamed statement 的策略是：创建一个 `unnamed prepared statement` context，并在完成后把它挂到 `CachedPlanSource` 相关生命周期下。
这不是性能微调。
这是 ownership 边界。
如果把跨消息对象放错 context，下轮 message reset 会制造悬垂指针。
### 4.4. `CachedPlanSource` 与 prepared statement
`Parse` message 不执行 SQL。
它把 SQL 字符串、raw parse tree、analyze/rewrite 结果、参数类型和命令 tag 组装成 `CachedPlanSource`。
named statement 会通过 `StorePreparedStatement()` 存起来。
unnamed statement 会保存在 `unnamed_stmt_psrc`。
`drop_unnamed_stmt()` 用来释放先前的 unnamed statement。
simple query 开始时也会调用 `drop_unnamed_stmt()`。
这是一个兼容语义：simple Query 模式被定义得像使用 unnamed statement/portal。
所以 simple query 会清理 unnamed prepared state，避免 simple 和 extended 的 unnamed 状态互相污染。
### 4.5. `Portal`
`Portal` 是“可执行查询”的会话内对象。
它把 prepared statement、参数、计划、result format、execution strategy、snapshot/executor 状态连接起来。
simple query 也创建 portal。
simple query 创建的是 unnamed portal，通常不可见，并在跑完后立刻 `PortalDrop()`。
extended query 的 `Bind` 创建 portal。
unnamed portal 可以静默替换。
named portal 不允许随意替换。
`Execute` message 通过 portal name 找到 portal，然后调用 `PortalRun()`。
如果 `max_rows` 限制导致 portal 未完成，backend 发送 `PortalSuspended`，portal 保持可继续执行。
这就是 extended query 支持分批取结果的关键状态。
### 4.6. `whereToSendOutput` 与 `CommandDest`
`whereToSendOutput` 是当前 backend 的输出目标。
普通客户端连接通常是 `DestRemote`。
simple query 在 `exec_simple_query()` 中用这个目标创建 receiver。
extended query 执行时会把 `DestRemote` 调整为 `DestRemoteExecute`。
这是因为 extended query 的 `Execute` 输出路径需要使用 execute 协议语义。
`DestReceiver` 才是真正把 tuple 或 command completion 送到目标的抽象。
本节只需要记住：协议 message loop 不直接发送每一行数据。
它创建合适的 receiver，然后由 portal/executor 输出。
### 4.7. `ReadyForQuery` 的状态字节
`ReadyForQuery()` 不只是“我处理完了”。
它还向客户端报告事务状态。
状态字节大致对应：
| 状态 | 含义 |
| --- | --- |
| `I` | idle，不在事务块中。 |
| `T` | in transaction block。 |
| `E` | failed transaction block。 |
这个状态由 transaction machinery 判断。
`PostgresMain()` 只决定何时发送。
simple query 通常在 `Q` message 完成后让 `send_ready_for_query = true`。
extended query 通常在 `Sync` 完成后才这样做。
如果 ERROR 发生在 extended query message 中，则进入 `ignore_till_sync`，在 `Sync` 之前不发送 `ReadyForQuery`。
这保证客户端能用 `Sync` 恢复双方协议状态。
## 5. 主流程源码 walkthrough
### 5.1. post-auth 到主循环
`PostgresMain()` 开头先安装信号处理器。
普通 backend 会处理 `SIGHUP`、`SIGINT`、`SIGTERM`、`SIGQUIT`、`SIGPIPE`、`SIGUSR1`、`SIGFPE` 等。
随后调用 `BaseInit()`。
再允许 SIGINT 等信号。
如果是普通远程连接，会生成 cancel key。
然后调用：
```c
InitPostgres(dbname, InvalidOid, username, InvalidOid,
             (!am_walsender) ? INIT_PG_LOAD_SESSION_LIBS : 0,
             NULL);
```
`InitPostgres()` 之后，backend 才具备访问数据库、系统 cache、GUC 和事务基础设施的上下文。
然后 `SetProcessingMode(NormalProcessing)`。
然后 `BeginReportingGUCOptions()`。
普通客户端连接还会发送 `BackendKeyData`。
这时还没有进入用户 SQL 的 message loop。
`MessageContext` 和 `row_description_context` 在这里创建。
`EventTriggerOnLogin()` 在主循环前触发。
然后 `PostgresMain()` 建立最外层 `sigsetjmp` error recovery 点。
这就是本节的真实起点。
### 5.2. loop 顶部：先清理上一轮，再准备读下一条消息
每轮 loop 大致是：
```text
MemoryContextSwitchTo(MessageContext)
MemoryContextReset(MessageContext)
initStringInfo(&input_message)
InvalidateCatalogSnapshotConditionally()
if send_ready_for_query:
    report idle state
    flush stats
    ReportChangedGUCOptions()
    ReadyForQuery()
DoingCommandRead = true
firstchar = ReadCommand(&input_message)
disable idle timers
CHECK_FOR_INTERRUPTS()
DoingCommandRead = false
dispatch by firstchar
```
这段顺序很重要。
先 reset `MessageContext`，避免上一轮消息的临时内存泄漏进下一轮。
再考虑释放 catalog snapshot，避免 backend idle 时阻碍 global xmin 推进。
如果需要发送 `ReadyForQuery`，就先把 idle 状态、GUC 变化、notify、stats flush 等做完。
然后才进入 client read。
`ReadyForQuery()` 会 flush 上一轮尚未发出的输出。
这就是为什么很多客户端在看到 `ReadyForQuery` 前不能认为一个 simple query 周期结束。
### 5.3. `ReadCommand()` 与 `SocketBackend()`
`ReadCommand()` 只是选择来源。
远程连接调用 `SocketBackend()`。
standalone interactive backend 调用 `InteractiveBackend()`，并把输入当作 `Q` message。
普通路径在 `SocketBackend()` 中。
`SocketBackend()` 的核心步骤是：
```text
HOLD_CANCEL_INTERRUPTS()
pq_startmsgread()
qtype = pq_getbyte()
validate qtype and set maxmsglen
set doing_extended_query_message
maybe clear ignore_till_sync for Sync or Terminate
pq_getmessage(inBuf, maxmsglen)
RESUME_CANCEL_INTERRUPTS()
return qtype
```
这里有两个容易忽略的边界。
第一，message type byte 会先被读出来。
然后根据类型决定最大合理长度。
这样可以在协议失步时尽早 FATAL，而不是把垃圾长度当成巨大分配。
第二，`SocketBackend()` 会尽早设置 `doing_extended_query_message`。
如果后续处理这个 message 时 ERROR，顶层 error recovery 就知道是否要进入 skip-till-Sync。
`PqMsg_Sync` 是特殊的。
读到 `Sync` 时，`SocketBackend()` 会清除 `ignore_till_sync`，并把 `doing_extended_query_message` 标为 false。
因此即使 `PostgresMain()` 有 `if (ignore_till_sync && firstchar != EOF) continue;`，`Sync` 也不会被跳过。
这是协议恢复路径的关键细节。
### 5.4. simple query 分派
`PqMsg_Query` 对应 frontend protocol 的 `Q` message。
`PostgresMain()` 处理它时：
```text
SetCurrentStatementStartTimestamp()
query_string = pq_getmsgstring(&input_message)
pq_getmsgend(&input_message)
exec_simple_query(query_string)
send_ready_for_query = true
```
如果是 WAL sender，simple query 先尝试 replication command。
不是 replication command 时才走普通 `exec_simple_query()`。
本节聚焦普通 backend。
进入 `exec_simple_query()` 后，第一批状态变化是观测状态。
`debug_query_string = query_string`。
`pgstat_report_activity(STATE_RUNNING, query_string)`。
`TRACE_POSTGRESQL_QUERY_START(query_string)`。
如果开启 statement stats，会 `ResetUsage()`。
然后 `start_xact_command()`。
然后 `drop_unnamed_stmt()`。
这个顺序说明 simple query 不是“绕过 extended query 状态”。
它会主动清掉 unnamed prepared statement。
然后切到 `MessageContext`，调用 `pg_parse_query(query_string)`。
simple query 可以有多个 raw parsetree。
例如：
```sql
SELECT 1;
SELECT 2;
```
这仍是一个 `Q` message。
`exec_simple_query()` 会遍历 raw parsetree list。
### 5.5. simple query 中每个 raw parsetree 的路径
对每个 raw parsetree，`exec_simple_query()` 做：
```text
CreateCommandTag()
set_ps_display()
BeginCommand()
check aborted xact state
start_xact_command()
maybe BeginImplicitTransactionBlock()
maybe PushActiveSnapshot()
pg_analyze_and_rewrite_fixedparams()
pg_plan_queries()
PopActiveSnapshot()
CreatePortal("", ...)
PortalDefineQuery()
PortalStart()
PortalSetResultFormat()
CreateDestReceiver()
PortalRun(... FETCH_ALL ...)
receiver->rDestroy()
PortalDrop()
finish_xact_command() or CommandCounterIncrement()
EndCommand()
```
这条链解释了本目录前 1 到 5 节 executor 生命周期的入口。
`PortalRun()` 才会继续进入 `pquery.c`。
`PortalStart()` 会根据 portal strategy 决定何时创建 `QueryDesc` 和启动 executor。
simple query 对每个 command 发一个 `CommandComplete`。
源码中通过 `EndCommand(&qc, dest, false)` 完成。
如果 query string 没有 parsetree，会发送 `EmptyQueryResponse`。
最后 `exec_simple_query()` 再调用一次 `finish_xact_command()` 兜底，主要覆盖空输入等情况。
然后 duration logging、stats 输出、trace done、清空 `debug_query_string`。
回到 `PostgresMain()` 后，`send_ready_for_query = true`。
下一轮 loop 顶部发送 `ReadyForQuery`。
注意：`ReadyForQuery` 不是在 `exec_simple_query()` 内部直接发。
它由外层 loop 控制。
### 5.6. simple query 的隐式事务块
simple query 允许一个 `Q` message 中包含多个 SQL command。
源码注释明确说，这是历史原因。
如果 raw parsetree list 长度大于 1，`use_implicit_block = true`。
每个循环迭代会在需要时 `BeginImplicitTransactionBlock()`。
如果中间遇到 `BEGIN` / `COMMIT` / `ROLLBACK` 这类 transaction control statement，源码会在 command 后 `finish_xact_command()`，下一条 command 再开新的 transaction command。
这个行为不是 extended query 的通用语义。
它是 simple query 字符串多 command 的兼容语义。
诊断时不要把“一个 TCP message”误认为“一条事务”。
也不要把“一个 `Q` message”误认为“一次 executor 调用”。
一个 `Q` message 可以产生多个 `BeginCommand()` / `EndCommand()` 和多个 portal。
### 5.7. extended query: `Parse`
`PqMsg_Parse` 读取：
```text
statement name
query string
parameter type count
parameter type oid array
```
然后调用 `exec_parse_message(query_string, stmt_name, paramTypes, numParams)`。
`exec_parse_message()` 的主要状态变化是：
```text
debug_query_string = query_string
pgstat_report_activity(STATE_RUNNING, query_string)
set_ps_display("PARSE")
start_xact_command()
choose named or unnamed statement context
pg_parse_query()
reject multiple commands
CreateCachedPlan()
maybe PushActiveSnapshot()
pg_analyze_and_rewrite_varparams()
CompleteCachedPlan()
StorePreparedStatement() or SaveCachedPlan()
CommandCounterIncrement()
send ParseComplete
```
这里有几个分派层面的结论。
`Parse` 不调用 `PortalRun()`。
`Parse` 不发送 `ReadyForQuery`。
`Parse` 会启动 transaction command，但不会在函数末尾 `finish_xact_command()`。
源码注释明确说：open transaction command 只在客户端发送 `Sync` 时关闭。
这就是 extended query 与 simple query 的核心差异之一。
`Parse` 建立的是 prepared statement 生命周期。
它还不是 portal 生命周期。
如果 prepared statement 中包含多个 user statement，源码直接报错。
原因是 extended query protocol 要保持 result descriptor 和后续 Bind/Execute 语义简单。
### 5.8. extended query: `Bind`
`PqMsg_Bind` 复杂到让 `PostgresMain()` 把整个 field extraction 放到 `exec_bind_message()` 内部。
`Bind` message 读取 portal name、statement name、参数格式、参数值、结果格式。
它的主流程是：
```text
find CachedPlanSource by statement name
pgstat_report_activity(STATE_RUNNING, psrc->query_string)
set_ps_display("BIND")
start_xact_command()
read parameter format/value fields
check parameter count and formats
CreatePortal()
copy query string and names into portal context
maybe PushActiveSnapshot()
convert text/binary parameters
GetCachedPlan()
PortalDefineQuery()
PortalStart()
PortalSetResultFormat()
send BindComplete
```
`Bind` 是 prepared statement 到 portal 的边界。
`GetCachedPlan()` 取得 `CachedPlan`。
源码特别强调：`GetCachedPlan()` 到 `PortalDefineQuery()` 之间不要插入可能 ERROR 的代码。
原因是 plan refcount 将交给 portal 管理。
如果中间 ERROR，会泄漏 plancache refcount。
这是一条 ownership 不变量。
参数值存放到 portal 的 memory context。
参数 I/O 可能调用类型 input/receive 函数。
因此 `Bind` 可能需要 snapshot，也可能抛出用户函数或类型转换错误。
这类错误发生在 executor 之前。
不要看到 `ERROR: invalid input syntax for type integer` 就直接怀疑执行器。
### 5.9. extended query: `Execute`
`PqMsg_Execute` 读取 portal name 和 `max_rows`。
然后调用 `exec_execute_message(portal_name, max_rows)`。
主流程是：
```text
dest = whereToSendOutput
if dest == DestRemote:
    dest = DestRemoteExecute
portal = GetPortalByName(portal_name)
copy sourceText and prepStmtName into MessageContext
pgstat_report_activity(STATE_RUNNING, sourceText)
report queryId and planId from portal stmts
BeginCommand()
CreateDestReceiver(dest)
start_xact_command()
check aborted xact state
PortalRun(portal, max_rows or FETCH_ALL)
destroy receiver
if completed:
    finish_xact_command() for xact/immediate commit
    else CommandCounterIncrement()
    EndCommand()
else:
    send PortalSuspended
```
`Execute` 是 extended query 里真正可能进入 executor 的消息。
但它不一定执行完。
如果 `max_rows > 0` 且结果还有剩余，`PortalRun()` 返回 not completed。
backend 发送 `PortalSuspended`。
portal 保持位置。
后续 `Execute` 可以继续取。
这解释了为什么 executor 内部的一个 plan 可以跨多个 protocol message 被推进。
这也解释了为什么 `Portal` 必须有独立生命周期，而不能只是 `Execute` 函数栈上的局部对象。
### 5.10. extended query: `Flush`
`PqMsg_Flush` 只要求 backend flush 已经产生的输出。
源码处理非常短：
```text
pq_getmsgend(&input_message)
if whereToSendOutput == DestRemote:
    pq_flush()
```
`Flush` 不提交 transaction command。
`Flush` 不发送 `ReadyForQuery`。
它是 pipeline 场景下的输出推动点，不是语义同步点。
如果客户端想确认整个 extended query batch 已经回到同步状态，需要 `Sync`。
### 5.11. extended query: `Sync`
`PqMsg_Sync` 是 extended query protocol 的同步点。
`PostgresMain()` 中的处理是：
```text
pq_getmsgend(&input_message)
EndImplicitTransactionBlock()
finish_xact_command()
send_ready_for_query = true
```
下一轮 loop 顶部会发送 `ReadyForQuery`。
注意 `SocketBackend()` 在读到 `Sync` 时已经清除了 `ignore_till_sync`。
所以 ERROR 后的 skip-till-Sync 能在这里恢复协议。
`Sync` 的语义不是“执行一条 SQL”。
它是“结束这一批 extended query 消息，关闭隐式事务边界，向客户端报告事务状态”。
如果前面某个 extended message ERROR，backend 会忽略后续 non-Sync message。
到 `Sync` 时恢复同步。
### 5.12. `Close` 和 `Describe`
`Close` 可以关闭 prepared statement 或 portal。
`Close` subtype 为 `S` 时关闭 statement。
`Close` subtype 为 `P` 时关闭 portal。
unnamed statement 有特殊 `drop_unnamed_stmt()` 路径。
portal 关闭通过 `PortalDrop()`。
`Describe` 可以描述 statement 或 portal。
statement describe 调用 `exec_describe_statement_message()`。
portal describe 调用 `exec_describe_portal_message()`。
这两个消息服务 extended query 的元数据阶段。
它们可能触发 RowDescription 或 NoData。
它们不直接执行 executor。
本节不展开 descriptor 编码细节。
### 5.13. `Terminate`、EOF 和 COPY leftover
`PqMsg_Terminate` 表示客户端关闭连接。
EOF 表示连接意外断开。
这两者都会走正常 backend shutdown：
```text
if whereToSendOutput == DestRemote:
    whereToSendOutput = DestNone
proc_exit(0)
```
源码注释强调不要在这里塞更多 cleanup。
需要 cleanup 的状态应注册 `on_proc_exit` 或 `on_shmem_exit` callback。
`CopyData`、`CopyDone`、`CopyFail` 在普通主循环中会被接受但忽略。
这通常发生在 COPY 失败后，frontend 仍在发送 leftover COPY data。
这是协议容错的一部分。
## 6. 生命周期 / ownership / cleanup
### 6.1. 谁创建 post-auth message loop 的长期状态
`PostgresMain()` 创建 `MessageContext`。
`PostgresMain()` 创建 `row_description_context` 和复用 buffer。
`InitPostgres()` 建立数据库连接上下文、role 上下文、系统 cache、事务基础状态。
`BeginReportingGUCOptions()` 之后，GUC 变化可以报告给客户端。
cancel key 在普通远程连接中生成并通过 `BackendKeyData` 发送。
这些状态的 owner 是当前 backend。
它们不会跨 backend 共享。
### 6.2. 每轮 message 的内存生命周期
每轮 loop 开头 reset `MessageContext`。
`input_message` body、simple query 的许多临时树、duration logging 临时字符串、error callback 临时状态都可以挂在这里。
如果本轮消息 ERROR，顶层 error recovery 最后会切回 `MessageContext` 并 `FlushErrorState()`。
下一轮 loop reset 会清掉残留。
这不等同于释放所有资源。
buffer pins、catcache refs、locks、portal resource owner 等不是靠 `MessageContext` 单独保证。
### 6.3. Prepared statement 的生命周期
named prepared statement 由 `StorePreparedStatement()` 持有。
它可被 `Close` statement、SQL `DEALLOCATE`、session end 或错误 cleanup 路径释放。
unnamed prepared statement 由 `unnamed_stmt_psrc` 指向。
下一次 unnamed Parse 或 simple query 会通过 `drop_unnamed_stmt()` 清掉它。
因此 unnamed statement 是会话级但易失的状态。
它不是每个 message 的临时状态。
这也是连接池和驱动排查时常见误区。
### 6.4. Portal 的生命周期
simple query 创建 unnamed portal。
它通常在 `PortalRun()` 后立即 `PortalDrop()`。
extended query 的 portal 由 `Bind` 创建。
它可以被 `Execute` 多次推进。
它可以被 `Close` portal 释放。
unnamed portal 可以被下一次 unnamed Bind 静默替换。
portal 持有自己的 `portalContext`。
portal 也有 `resowner`，用来托管与执行相关的资源。
`PortalStart()` 会设置 `ActivePortal`、`PortalContext` 和 `CurrentResourceOwner`。
`PortalDrop()` 和 `PortalErrorCleanup()` 是理解 ERROR cleanup 的关键入口。
### 6.5. Cached plan refcount ownership
`Bind` 阶段调用 `GetCachedPlan()`。
返回的 `CachedPlan` 需要引用计数保护。
源码注释要求 `GetCachedPlan()` 和 `PortalDefineQuery()` 之间不能插入可能 ERROR 的代码。
因为 `PortalDefineQuery()` 之后，plan refcount 由 portal 释放路径接管。
这条规则体现了 PostgreSQL 内核常见模式：
```text
obtain refcounted object
  -> no-error handoff zone
  -> attach to owner
  -> owner cleanup releases it
```
课程后面讲 plan cache 时会继续展开。
本节只要求能在 `Bind` 路径里看出这个 ownership handoff。
### 6.6. ERROR 时谁兜底
`PostgresMain()` 的外层 `sigsetjmp` 是 session loop 的顶层 ERROR recovery。
进入 recovery 后，源码会：
```text
error_context_stack = NULL
HOLD_INTERRUPTS()
disable_all_timeouts(false)
QueryCancelPending = false
DoingCommandRead = false
pq_comm_reset()
EmitErrorReport()
debug_query_string = NULL
AbortCurrentTransaction()
PortalErrorCleanup()
ReplicationSlotRelease() when needed
ReplicationSlotCleanup(false)
jit_reset_after_error()
MemoryContextSwitchTo(MessageContext)
FlushErrorState()
maybe ignore_till_sync = true
xact_started = false
if pq_is_reading_msg(): FATAL
RESUME_INTERRUPTS()
```
这说明 ERROR cleanup 不是一个地方做完的。
transaction abort、portal cleanup、replication slot cleanup、JIT cleanup、libpq communication reset、message loop protocol state 都各有边界。
顶层 recovery 只做必须属于 session loop 的事。
源码注释也提醒：如果想在 recovery block 里加代码，先考虑是否应该放进 `AbortTransaction()`。
## 7. 正确性机制层次
### 7.1. 协议同步正确性
frontend/backend protocol 是有 message boundary 的。
backend 读错一个长度或漏读一个字段，就可能把下一条 message 的首字节当成当前 message body。
因此 `SocketBackend()` 先验证 message type。
然后 `pq_getmessage()` 按 length word 读完整 body。
处理函数最后调用 `pq_getmsgend()`。
如果 message type 不合法，源码选择 FATAL。
因为此时很可能已经失去边界同步。
如果 ERROR 发生在已经开始读取 message 期间，顶层 recovery 会检查 `pq_is_reading_msg()`。
如果仍处于 reading message 状态，会 FATAL。
原因是 backend 不再能保证下一次读从正确边界开始。
### 7.2. extended query ERROR 的 skip-till-Sync
extended query 支持 pipeline。
客户端可能已经发送 Parse、Bind、Execute、Describe、Flush、Sync 等多个 message。
如果中间一个 extended message ERROR，backend 不能假装后续 message 仍在正常上下文里执行。
所以顶层 recovery 检查 `doing_extended_query_message`。
如果为 true，则设置 `ignore_till_sync = true`。
后续 loop 会跳过消息。
读到 `Sync` 时，`SocketBackend()` 清除 ignore 状态。
然后 `PostgresMain()` 处理 `Sync`，调用 `finish_xact_command()` 并安排 `ReadyForQuery`。
这形成协议级恢复边界。
simple query 不走这个 skip-till-Sync 机制。
simple query ERROR 后直接回到下一轮 loop，并在合适时发送 `ReadyForQuery`。
### 7.3. transaction command 正确性
tcop 层用 `start_xact_command()` 和 `finish_xact_command()` 管理 transaction command。
`start_xact_command()` 在 `xact_started` 为 false 时调用 `StartTransactionCommand()`。
如果已经处于 pipeline 隐式状态，它可能调用 `BeginImplicitTransactionBlock()`。
它还负责启动 statement timeout 和 client connection check timeout。
`finish_xact_command()` 会禁用 statement timeout。
如果 `xact_started`，它调用 `CommitTransactionCommand()`。
然后把 `xact_started = false`。
simple query 通常在每个 command 或 query string 末尾收掉 transaction command。
extended query 则通常让 transaction command 跨 Parse/Bind/Execute 消息持续到 Sync。
这不是事务语义的全部。
显式 `BEGIN` 仍然会让 session 进入 transaction block。
`ReadyForQuery` 的状态字节才告诉客户端当前处于 idle、in transaction 还是 failed transaction。
### 7.4. snapshot 正确性
parse analysis 和 planning 有时需要 snapshot。
simple query 在 analyze/plan 前按需 `PushActiveSnapshot(GetTransactionSnapshot())`。
planning 后会 `PopActiveSnapshot()`。
源码特别说明，不把这个 snapshot 复用到执行阶段。
原因是执行时必须在锁定相关表之后使用合适 snapshot。
否则可能产生用户可见异常。
extended `Bind` 处理参数 I/O 或重新规划时也可能需要 snapshot。
这说明 snapshot 不是“进入 message 后取一次，全程复用”。
snapshot 生命周期跟 parse/analyze/plan/execute 子阶段有关。
### 7.5. cancel 和 timeout 正确性
`DoingCommandRead` 是 interrupt 处理的重要边界。
backend 等客户端输入时，如果用户 cancel 到达，通常应该被视为 no-op。
源码在读命令前设置 `DoingCommandRead = true`。
读完后先 `CHECK_FOR_INTERRUPTS()`，再设置 false。
`ProcessInterrupts()` 看到正在 command read 时，会按读等待语义处理 cancel、notify、sinval 和 stats update。
这避免了在没有正在执行的 SQL 时向客户端发送多余 ERROR。
idle-in-transaction timeout 和 idle-session timeout 只在进入 idle 状态时启用。
读到新 message 后在真正处理 message 前禁用。
这保证临界边界清晰。
### 7.6. MemoryContext 与 ResourceOwner 分层
`MessageContext` 解决短生命周期内存。
portal context 解决 portal 生命周期内存。
prepared statement / plancache context 解决跨 message 计划状态。
ResourceOwner 解决 pin、lock、refcount 等外部资源。
不要把 MemoryContext 当作 ResourceOwner。
ERROR 后 reset memory 可以清掉很多 C 指针对象。
但 buffer pin、catcache ref、lock、snapshot、portal executor 状态仍需要各自 cleanup 路径。
## 8. 错误路径 / 异常路径 / fallback
### 8.1. invalid message type
`SocketBackend()` 在读 message body 前验证 qtype。
未知 qtype 会 FATAL。
这比 ERROR 更重。
原因是 backend 很可能已经不知道后续 byte stream 的边界。
一旦边界不可信，继续服务连接比断开更危险。
观测上，客户端会看到连接关闭。
server log 会有 protocol violation。
### 8.2. message body 没读完整或解析不匹配
处理函数用 `pq_getmsgend()` 确认 body 被消费完。
如果 message 多出字段或字段不足，会 ERROR 或 FATAL，取决于具体位置。
`Bind` 中参数格式数量、参数数量、binary data 消费长度都被检查。
例如 binary receive 函数没有吃完整个参数 buffer，会报 invalid binary representation。
这类错误发生在协议/类型输入层，不是 executor 层。
### 8.3. ERROR while reading message
如果 ERROR 发生时 `pq_is_reading_msg()` 仍为 true，顶层 recovery 会 FATAL。
原因是 backend 可能不知道当前 message body 读到哪里了。
即使 transaction abort 成功，也不能恢复协议同步。
这是 libpq communication state 和 transaction state 分离的典型例子。
事务能 abort，不代表协议还能继续。
### 8.4. extended query ERROR before Sync
`Parse`、`Bind`、`Execute`、`Describe`、`Close` 等 extended message 中 ERROR 后，backend 设置 `ignore_till_sync`。
后续 non-Sync messages 仍会被读掉。
但不会执行。
读到 `Sync` 后才恢复并发送 `ReadyForQuery`。
这对 pipeline 客户端非常关键。
否则一个早期 ERROR 可能让后续 Execute 在错误上下文里继续运行。
### 8.5. aborted transaction block
如果当前 transaction block 已经 aborted，PostgreSQL 只允许少数 transaction exit command 继续。
simple query 在 parse analysis 前检查。
extended `Parse` 和 `Bind` 也有各自检查。
`Execute` 会检查 portal 里的 planned statements。
这解释了常见错误：
```text
current transaction is aborted, commands ignored until end of transaction block
```
它不是 parser 随机拒绝 SQL。
它是 tcop 层在进入需要 database access 的阶段前保护 aborted transaction state。
### 8.6. client disconnect
`SocketBackend()` 读 qtype 时返回 EOF。
如果当前有 open transaction，会报告 unexpected EOF with open transaction。
否则把输出目标改为 `DestNone`，记录 DEBUG。
回到 switch 后，EOF 走 session end。
`PqMsg_Terminate` 也是类似 shutdown。
需要 cleanup 的对象不应该依赖 switch case 里的手写代码。
它们应该走 proc exit callback、transaction abort、portal cleanup 或资源 owner。
### 8.7. COPY failure leftover messages
普通主循环看到 `CopyData`、`CopyDone`、`CopyFail` 会接受但忽略。
这常见于 COPY FROM STDIN 失败后，frontend 仍在发送数据。
backend 不把这类 leftover 直接当成 invalid protocol。
它选择吞掉，帮助连接恢复。
这体现了协议实现中的 fallback：能安全恢复边界时尽量恢复，不能恢复边界时 FATAL。
### 8.8. WAL sender 限制
`forbidden_in_wal_sender()` 会拒绝 extended query protocol 和 fastpath function call。
WAL sender 只支持它需要的 simple query replication command 语义。
这说明 `PostgresMain()` 是通用入口，但不同 backend role 的消息集合不完全相同。
诊断 replication connection 时，不要假设普通 SQL session 的 extended query 行为可用。
## 9. 成本、资源与跨模块传播
### 9.1. hot path 成本来自每个 message 周期
每轮 message loop 都会 reset `MessageContext`、初始化 input buffer、检查 catalog snapshot、处理 ReadyForQuery 条件、读 socket、处理 interrupts。
这些成本通常很小。
但在大量 tiny query 或 extended query pipeline 中会变成可见 CPU 成本。
例如每条 `SELECT 1` 都走完整 tcop loop、parser、planner、portal 和 executor。
这就是为什么 SQL 层的微小查询延迟不能只归因于 executor。
parser、planner、message dispatch、network flush 和 client round trip 都参与。
### 9.2. simple query 的成本随 raw command 数扩张
一个 `Q` message 可以包含多个 command。
成本不是按 message 数线性，而是按 raw parsetree 数、rewrite 结果数、planned statement 数和 portal run 次数扩张。
每个 command 会有 `BeginCommand()` / `EndCommand()`。
每个 command 通常创建 portal。
每个 command 可能需要自己的 snapshot、planning 和 receiver。
如果 query string 很长且包含很多 command，`MessageContext` 峰值和 per-parsetree context 会成为诊断点。
### 9.3. extended query 的成本拆到多个阶段
`Parse` 花 parser/analyze/rewrite 成本。
`Bind` 花参数解码、类型 input/receive、plan cache lookup、可能重规划、portal 创建成本。
`Execute` 花 executor 和输出成本。
`Sync` 花 transaction command 收尾、stats/GUC/ReadyForQuery 相关成本。
这对观测很重要。
客户端看到一次 prepared statement execute 慢，慢点可能在 `Bind` 的参数输入，也可能在 `Execute` 的执行器，也可能在 `Sync` 的提交收尾。
`log_min_duration_statement` 对 extended query 的日志表现也会按这些 message 阶段体现。
### 9.4. flush 与 round trip 成本
simple query 通常到 `ReadyForQuery` 才完成一个客户端可见周期。
extended query 可以用 `Flush` 让 backend 提前发送已有结果。
但 `Flush` 不关闭事务命令。
频繁 Flush 会增加 socket flush 和系统调用成本。
不 Flush 又可能让 pipeline 客户端等待输出。
这是 client protocol usage 的成本模型，不是 executor 算法问题。
### 9.5. transaction command 和 stats 成本传播
`ReadyForQuery` 前后是 stats flush 和 idle 状态上报的重要时机。
源码特意避免每条 batched message 都 flush stats。
原因是 extended pipeline 下频繁 flush 会拖慢处理，并且未提交统计会干扰 autovacuum 判断。
所以 pg_stat 观测可能滞后。
不要把 pg_stat 的刷新时机误读为 executor 刚刚才完成。
### 9.6. 跨模块边界
本节至少连接这些模块：
| 模块 | 边界 |
| --- | --- |
| libpq backend communication | 负责读写 protocol message、维护读写状态和 socket interrupt。 |
| transaction manager | `StartTransactionCommand()` / `CommitTransactionCommand()` 和 aborted state 判断。 |
| parser/rewrite/planner | simple query 与 Parse 阶段进入。 |
| plan cache | `CachedPlanSource`、`GetCachedPlan()`、refcount handoff。 |
| portal manager | query execution state、suspend/resume、portal cleanup。 |
| executor | `PortalRun()` 下游的 `ExecutorStart()` / `ExecutorRun()` / `ExecutorEnd()`。 |
| pgstat/activity | `pgstat_report_activity()`、queryId、planId、idle state、stats flush。 |
| timeout/interrupt | statement timeout、idle timeout、client connection check、cancel。 |
这些模块没有一个能单独解释 frontend/backend message loop。
本节的 mental model 是边界组合。
## 10. 观测与诊断入口
### 10.1. 能直接看到的状态
`pg_stat_activity.state` 能看到 backend 是 active、idle、idle in transaction，还是 idle in transaction aborted。
`pg_stat_activity.query` 能看到当前 reported query string。
simple query 在 `exec_simple_query()` 中报告原始 query string。
extended `Parse` 报告 query string。
extended `Bind` 和 `Execute` 报告 prepared statement 的 source text。
`pg_stat_activity.query_id` 可能在 Bind/Execute 中从 query tree 或 planned statement 报告。
server log 能看到 `log_statement`、`log_min_duration_statement`、`log_duration` 输出。
`ps` display 可以看到 `PARSE`、`BIND`、command tag 等阶段。
客户端协议层能看到 `ParseComplete`、`BindComplete`、`CommandComplete`、`PortalSuspended`、`ReadyForQuery`。
### 10.2. 只能间接推断的状态
`ignore_till_sync` 不能直接从 SQL 查询。
只能从 extended query ERROR 后客户端收到的错误、后续消息被跳过、直到 Sync 才 ReadyForQuery 来推断。
`doing_extended_query_message` 也不能直接看。
它只影响 ERROR recovery 策略。
`xact_started` 不能直接看。
能看到的是 transaction block 状态、ReadyForQuery 状态字节和 pg_stat_activity state。
portal 是否 suspended 可以通过客户端收到 `PortalSuspended`、游标行为或调试断点确认。
### 10.3. 几乎不可见的状态
`MessageContext` 每轮 reset 的细节通常不可见。
除非使用 gdb、`MemoryContextStats()` 或临时日志。
libpq backend 侧的 `PqCommReadingMsg` 和 `PqCommBusy` 也通常不可见。
它们只在异常路径影响是否 FATAL 或通信 reset。
plan cache refcount handoff 不可直接从 SQL 观察。
只能通过源码、断点、assert 或故意制造 ERROR path 实验确认。
### 10.4. gdb 断点建议
源码跟读可以设置断点：
```text
break PostgresMain
break SocketBackend
break exec_simple_query
break exec_parse_message
break exec_bind_message
break exec_execute_message
break PortalRun
break ReadyForQuery
```
simple query 实验中，观察 `firstchar == PqMsg_Query`。
extended query 实验中，观察 `PqMsg_Parse`、`PqMsg_Bind`、`PqMsg_Execute`、`PqMsg_Sync` 的顺序。
在 `exec_execute_message()` 里观察 `max_rows` 和 `completed`。
在 ERROR 实验中观察 `doing_extended_query_message` 和 `ignore_till_sync`。
### 10.5. 日志入口
可以临时开启：
```sql
SET log_statement = 'all';
SET log_duration = on;
SET log_min_duration_statement = 0;
```
这些日志会增加开销。
不要在生产上长期全量开启。
对 extended query，驱动是否使用 server-side prepare 会影响日志格式和阶段。
一些驱动默认先 simple query，执行多次后切换 prepared statement。
这会让同一 SQL 的日志表现随执行次数变化。
### 10.6. wait event 入口
backend 等客户端输入时，常见 wait event 是 Client 类读等待。
backend 向客户端输出被阻塞时，会出现在 Client 写等待相关路径。
这类等待不表示 executor 慢。
它表示 backend 在协议边界等待 frontend 或网络。
如果 `pg_stat_activity` 长时间 idle in transaction，问题可能是客户端已经收到 ReadyForQuery 状态 `T` 后没有继续提交或回滚。
如果 active 但 wait event 是 ClientWrite，可能是结果集发送被客户端消费速度限制。
### 10.7. runtime truth: simple 与 extended 的 ReadyForQuery 差异
本节锚定的可验证 runtime truth 是：
```text
simple query 的 Q message 跑完后，外层 loop 安排 ReadyForQuery；
extended query 的 Parse/Bind/Execute 不单独 ReadyForQuery，Sync 才安排 ReadyForQuery。
```
用 libpq pipeline 或 `psql` 的普通语句很难直接看到所有 extended messages。
更稳定的做法是写一个小 libpq 程序，使用 `PQsendPrepare()`、`PQsendQueryPrepared()` 或 pipeline mode。
也可以用 gdb 断点在 backend 侧直接确认。
看到现象后回到源码：
`PostgresMain()` 的 switch 决定哪些 case 设置 `send_ready_for_query`。
`SocketBackend()` 决定 Sync 如何解除 skip。
`exec_*_message()` 决定对应阶段是否关闭 transaction command。
## 11. 常见误区
误区一：把 simple query 当作“没有 portal”。
simple query 也创建 unnamed portal。
区别是它通常创建、运行、删除都在同一个 `exec_simple_query()` 循环里完成。
误区二：把 extended query 的 `Execute` 当作事务边界。
`Execute` 可以完成一个 portal，也可以只取部分行。
extended query 的协议同步边界是 `Sync`。
事务命令收尾通常也在 `Sync` 或特殊 immediate commit 路径。
误区三：认为每条 SQL 都对应一次 `ReadyForQuery`。
simple query 的一个 `Q` message 后通常会有。
extended query 的 `Parse`、`Bind`、`Execute` 后通常没有。
`ReadyForQuery` 是同步点，不是 SQL command complete。
误区四：把 `pg_stat_activity.state = idle` 理解成 backend 没有资源。
idle in transaction 可能持有 transaction snapshot、locks 或其他事务状态。
普通 idle 也可能刚刚 flush 完 stats 和 notify。
需要结合 ReadyForQuery 状态、事务状态和 workload。
误区五：把 ERROR cleanup 理解成只要 abort transaction。
顶层 recovery 还要恢复 libpq communication state、portal state、JIT state、replication slot、timeout、error context 和 extended protocol skip 状态。
误区六：把 message helper 当 public extension contract。
`pq_getmsg*` 是 backend 内部协议解析 helper。
扩展通常不应该依赖 tcop 内部 static 状态。
## 12. 课堂实验
### 实验一：用 gdb 比较 simple query 与 extended query
准备一个 debug build。
启动 PostgreSQL。
找到目标 backend PID。
附加 gdb。
设置断点：
```text
break SocketBackend
break exec_simple_query
break exec_parse_message
break exec_bind_message
break exec_execute_message
break ReadyForQuery
```
在 `psql` 中执行：
```sql
SELECT 1;
```
观察 backend 命中 `SocketBackend`、`exec_simple_query`、`ReadyForQuery`。
再用一个 libpq 程序调用 prepare/execute。
观察 `Parse`、`Bind`、`Execute`、`Sync` 的顺序。
记录哪些断点之间出现 `ReadyForQuery`。
预期结论：simple query 在 message 完成后回到 loop 发送 `ReadyForQuery`；extended query 的阶段性 complete message 不等于 ReadyForQuery。
### 实验二：观察 `max_rows` 导致的 `PortalSuspended`
使用 extended query protocol 创建一个返回多行的 portal。
第一次 `Execute` 设置 `max_rows = 1`。
观察 backend 在 `exec_execute_message()` 中：
```text
max_rows = 1
completed = false
```
客户端应收到一行结果和 `PortalSuspended`。
再次 Execute 同一 portal。
观察 portal 继续推进。
预期结论：portal 是跨 Execute message 的执行状态，不是一次函数调用就必须结束。
### 实验三：制造 extended query ERROR 并观察 skip-till-Sync
构造 extended query pipeline：
```text
Parse invalid statement
Bind another statement
Execute another portal
Sync
```
或者在 Bind 中传入错误类型参数。
在 backend 设置断点观察：
```text
doing_extended_query_message
ignore_till_sync
SocketBackend() reading PqMsg_Sync
```
预期结论：ERROR 后 backend 跳过后续 non-Sync message，直到 Sync 清除 skip 状态并发送 ReadyForQuery。
### 实验四：比较日志阶段
临时开启：
```sql
SET log_min_duration_statement = 0;
SET log_duration = on;
```
分别执行 simple query 和 prepared statement execute。
观察日志中 parse、bind、execute 或 statement 的差异。
预期结论：日志阶段反映 tcop message 处理边界，不应简单等同于 executor 时间。
## 13. 讨论题
1. 为什么 `PostgresMain()` 把 `ReadyForQuery` 放在 loop 顶部，而不是每个处理函数末尾直接发送？
2. extended query 出错后为什么要 skip until Sync，而不是直接继续处理后续 messages？
3. simple query 为什么也要创建 unnamed portal，而不是直接调用 executor？
4. `MessageContext` reset 能解决哪些资源，不能解决哪些资源？
5. `Bind` 中 `GetCachedPlan()` 到 `PortalDefineQuery()` 之间为什么不能插入可能 ERROR 的代码？
6. `Flush` 和 `Sync` 的协议语义差异是什么？
7. 为什么 parse/planning 用过的 snapshot 不能简单复用到 execution？
8. 当 `pg_stat_activity` 显示 ClientRead 或 ClientWrite 时，如何判断问题在客户端协议层还是 executor 层？
## 14. 本节小结
本节唯一主问题是：post-auth backend 如何在 frontend/backend protocol message loop 中分派 simple query 和 extended query。
核心链路是 `PostgresMain()`。
它在 loop 顶部处理 `ReadyForQuery`、idle 状态、stats、GUC 和 notify。
然后 `ReadCommand()` / `SocketBackend()` 读取一个 frontend message。
再根据 message type 分派。
simple query 的 `Q` message 进入 `exec_simple_query()`。
它解析 query string，遍历 raw parsetree，为每个 command 创建 unnamed portal，`PortalRun()` 到完成，然后 `EndCommand()`。
extended query 拆成 `Parse`、`Bind`、`Execute`、`Sync`。
`Parse` 建立 `CachedPlanSource`。
`Bind` 建立 portal 并完成 plan cache 到 portal 的 ownership handoff。
`Execute` 运行 portal，可能完成，也可能 `PortalSuspended`。
`Sync` 关闭 extended query batch 的隐式事务边界，并安排 `ReadyForQuery`。
核心状态是 backend-local。
`MessageContext` 管每轮 message 临时内存。
prepared statement 和 portal 有更长生命周期。
`xact_started`、`DoingCommandRead`、`doing_extended_query_message`、`ignore_till_sync` 和 `send_ready_for_query` 共同把事务、interrupt、ERROR recovery 和协议同步连在一起。
正确性来自多层机制。
protocol boundary 由 message type、length、`pq_getmsgend()` 和 `pq_is_reading_msg()` 保护。
transaction command 由 `start_xact_command()` / `finish_xact_command()` 管理。
snapshot 生命周期按 parse/analyze/plan/execute 子阶段切分。
portal 和 plan cache 通过 context、refcount 和 ResourceOwner 托管。
错误路径不是附录。
invalid message type 或 reading-message ERROR 会 FATAL。
extended query ERROR 会 skip until Sync。
aborted transaction block 会在进入需要 database access 的阶段前拒绝普通 command。
COPY leftover message 会被安全吞掉。
观测上，`ReadyForQuery`、`CommandComplete`、`ParseComplete`、`BindComplete`、`PortalSuspended`、`pg_stat_activity`、server log 和 wait event 都只是真实状态的投影。
它们能帮助定位边界，但不能替代源码状态。
可迁移规律是：
```text
长生命周期 session loop 不能只按函数调用理解；
必须同时跟踪 protocol sync point、transaction command、memory context、resource owner 和 ERROR recovery。
```
下一节可以继续追 simple query 内部如何把 parse/analyze/rewrite/plan/execute 和隐式事务边界串起来。
