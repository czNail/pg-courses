# PostgreSQL DestReceiver / output protocol
## 课程定位
前置知识：已经理解 `PortalRun()`、`QueryDesc`、`ExecutorStart()`、`ExecutorRun()`、`ExecutorEnd()` 的基本生命周期。
前置知识：已经读过 frontend/backend protocol、simple query、extended query 和 cached plan 相关课程，知道客户端最终看到的是 protocol message。
本节唯一主问题：
```text
DestReceiver 如何把 executor tuple 输出到客户端、SPI、tuplestore、COPY 或 EXPLAIN？
```
核心矛盾：executor 需要一个统一、低侵入的逐 tuple 输出接口；但客户端、SPI、tuplestore、COPY 和 EXPLAIN 需要完全不同的协议、内存 ownership、完成消息、暂停语义和观测代价。
学完后应能判断：一行结果到底是在 executor 中产生，在 `DestReceiver` 中序列化，在 portal 中物化，在 SPI tuple table 中复制，在 COPY formatter 中输出，还是被 EXPLAIN 丢弃或计量。
本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e90741e12a3715909e1b2d71ff9344`。
本节不讲 scan/join/agg 如何产生 tuple，也不展开完整 wire protocol 格式。
本节只追一个边界：executor 已经拿到 `TupleTableSlot` 后，这个 slot 被谁接走、如何转换、何时释放、客户端或内部 caller 能看到什么。
## 1. 本节在总主线中的位置
04 目录前面的课程先从 executor 内部讲起。
第 1 到第 5 节讲 `ExecutorStart()` 到 `ExecutorEnd()`。
第 8 到第 10 节讲 `TupleTableSlot`、tuple descriptor、projection 和 per-tuple state。
第 25 到第 29 节讲 `EXPLAIN ANALYZE` 如何观测 executor。
第 57 到第 61 节补上 protocol loop、simple/extended query、prepared statement 和 cached plan。
现在要补上 executor 与外部消费者之间的一层：`DestReceiver`。
如果只读 `ExecProcNode()`，你会看到 plan node 一次返回一个 slot。
如果只读 libpq protocol，你会看到 `RowDescription`、`DataRow`、`CommandComplete`、`ReadyForQuery`。
两者之间不是直接相连的。
PostgreSQL 用一组小 callback 把它们连接起来：
```text
Portal / ProcessQuery
  -> CreateQueryDesc(..., DestReceiver *dest, ...)
  -> ExecutorRun()
     -> dest->rStartup()
     -> ExecProcNode()
     -> dest->receiveSlot()
     -> dest->rShutdown()
  -> EndCommand()
  -> receiver->rDestroy()
```
这组 callback 让同一个 executor plan 可以输出到多个目标。
普通客户端 `SELECT` 输出到 `printtup.c`。
PL/pgSQL 或扩展内部 SQL 输出到 SPI tuple table。
`DECLARE CURSOR WITH HOLD` 或某些 portal 路径输出到 tuplestore。
`COPY (SELECT ...) TO ...` 输出到 COPY subprotocol、文件或 program。
`EXPLAIN ANALYZE` 默认把被解释 query 的 tuple 丢弃。
`EXPLAIN (ANALYZE, SERIALIZE)` 则故意模拟客户端序列化成本，但仍不返回真实 query result。
这节课不是 receiver 枚举清单。
主线是一个 slot 从 executor hot path 离开时，统一 contract 如何把它交给目标专属 ownership。
这也是 observability 的盲区。
很多“SQL 执行慢”其实发生在类型输出函数、TOAST detoast、COPY 格式化、tuplestore spill、SPI tuple 复制或 socket backpressure 中。
`DestReceiver` 是把这些成本接到 executor 主循环的边界。
## 2. 核心矛盾与一句话运行模型
一句话运行模型：
```text
上层根据 CommandDest 创建 DestReceiver，
executor 在每次 ExecutorRun() 中启动 receiver，
然后把每个输出 TupleTableSlot 交给 receiveSlot()，
receiver 决定是发 DataRow、复制到 SPI、写 tuplestore、做 COPY 格式化，还是只计量序列化成本。
```
系统 tension 是：
```text
统一 executor hot path
  vs
多种输出目标的协议、内存、资源和错误语义完全不同
```
如果 executor 直接知道所有输出目标，hot path 会塞满客户端协议、SPI、COPY、tuplestore 和 EXPLAIN 分支。
utility command 和 plannable query 也难以共用输出通道。
扩展或内部模块想接走 tuple 时，必须侵入 executor。
PostgreSQL 选择把目标选择放在 executor 外，把逐行交付压缩成 `DestReceiver` contract。
`CommandDest` 是粗粒度目标枚举。
`DestReceiver` 是细粒度 callback object。
portal、SPI、COPY、EXPLAIN 在进入 executor 前完成目标选择和参数补充。
executor 只调用固定方法。
同样一条 `SELECT generate_series(1, 1000000)`，换目标后行为不同：
```text
普通客户端 SELECT
  -> printtup()
  -> 类型 output/send 函数
  -> DataRow message
  -> backend libpq/socket

SPI SELECT
  -> spi_printtup()
  -> ExecCopySlotHeapTuple()
  -> SPITupleTable

WITH HOLD cursor
  -> tstoreReceiveSlot_*()
  -> Tuplestorestate
  -> 后续 FETCH 再输出

COPY (SELECT ...) TO STDOUT
  -> copy_dest_receive()
  -> CopyOneRowTo()
  -> CopyData message

EXPLAIN ANALYZE
  -> None_Receiver 或 DestExplainSerialize
  -> 不返回被解释 query 的真实结果
```
核心抽象是：
```text
executor 不拥有最终输出协议；
executor 只按 DestReceiver contract 交出 slot；
receiver 决定 tuple 的最终物理去向和可见性。
```
## 3. 核心文件分工与阅读顺序
| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/tcop/dest.h` | `CommandDest`、`DestReceiver` callback contract、`BeginCommand` / `EndCommand` / `ReadyForQuery` 声明。 |
| 2 | `src/backend/tcop/dest.c` | `CreateDestReceiver()` 分派；静态 receiver；`CommandComplete`、`EmptyQueryResponse`、`ReadyForQuery`。 |
| 3 | `src/backend/tcop/postgres.c` | simple query 和 extended `Execute` 何时创建 receiver、何时 `EndCommand()`、何时销毁。 |
| 4 | `src/backend/tcop/pquery.c` | `PortalRun()`、`PortalRunMulti()`、`FillPortalStore()` 如何把 portal 输出接到 receiver。 |
| 5 | `src/backend/executor/execMain.c` | `standard_ExecutorRun()` / `ExecutePlan()` 调用 `rStartup`、`receiveSlot`、`rShutdown`。 |
| 6 | `src/backend/access/common/printtup.c` | `DestRemote` / `DestRemoteExecute` 的 `RowDescription`、`DataRow`、text/binary output。 |
| 7 | `src/backend/executor/spi.c` | `DestSPI` 如何构造 `SPITupleTable` 并复制 slot tuple。 |
| 8 | `src/backend/executor/tstoreReceiver.c` | `DestTuplestore` 如何写 tuplestore、可选 detoast、可选 tuple conversion。 |
| 9 | `src/backend/commands/copyto.c` | `DestCopyOut` 如何把 query 输出接到 COPY TO。 |
| 10 | `src/backend/commands/explain.c` | `ExplainOnePlan()` 如何选择 `None_Receiver`、`IntoRel` 或 serialize receiver。 |
| 11 | `src/backend/commands/explain_dr.c` | `DestExplainSerialize` 如何复刻 `printtup()` 的序列化成本但不发送。 |
| 12 | `src/backend/executor/execTuples.c` | utility helper 如何用 receiver 输出虚拟 tuple。 |
| 13 | `src/include/executor/executor.h` | `QueryDesc.dest` 与 `TupOutputState.dest`。 |
建议先读 `dest.h`。
它不是宏观介绍，而是 contract。
再读 `execMain.c`，确认 executor 只认识 callback。
然后读 `printtup.c`，把普通客户端输出走通。
最后对照 SPI、tuplestore、COPY 和 EXPLAIN。
不要从 `pq_sendint32()`、CSV 转义或 tuplestore 内部 spill 开始。
这些是具体 receiver 的内部细节，不是本节主边界。
## 4. 关键数据结构与状态
### 4.1. `CommandDest`
`CommandDest` 回答“这条命令的输出目标大类是什么”。
本节关注这些值：
| 值 | 语义 |
| --- | --- |
| `DestNone` | 丢弃结果；内部执行、EXPLAIN 默认执行、rewrite 辅助 query 常用。 |
| `DestDebug` | standalone/debug 输出。 |
| `DestRemote` | 发给普通 frontend；startup 会自动发送 `RowDescription`。 |
| `DestRemoteExecute` | extended query `Execute` 输出；通常不自动发送 `RowDescription`。 |
| `DestRemoteSimple` | 简化远端输出，常见于 walsender/basebackup 等不需要普通 `printtup` catalog 信息的路径。 |
| `DestSPI` | 交给 SPI manager。 |
| `DestTuplestore` | 写入 `Tuplestorestate`。 |
| `DestIntoRel` | `CREATE TABLE AS` / `SELECT INTO` 类写入 relation。 |
| `DestCopyOut` | COPY TO query path 接收 tuple。 |
| `DestSQLFunction` | SQL-language function manager 输出。 |
| `DestTransientRel` | 写入 transient relation。 |
| `DestTupleQueue` | parallel tuple queue 输出。 |
| `DestExplainSerialize` | EXPLAIN serialize 计量输出序列化成本并丢弃数据。 |
`CommandDest` 不是 frontend protocol。
它是 backend 内部 tag。
`dest.h` 说明全局 `whereToSendOutput` 只允许 `DestNone`、`DestDebug`、`DestRemote`。
其他值只应作为单条命令或内部路径的目标。
这避免 backend 会话级输出状态被内部 receiver 污染。
### 4.2. `DestReceiver`
`DestReceiver` 是 callback object。
源码结构可以压缩成：
```c
struct _DestReceiver
{
    bool (*receiveSlot)(TupleTableSlot *slot, DestReceiver *self);
    void (*rStartup)(DestReceiver *self, int operation, TupleDesc typeinfo);
    void (*rShutdown)(DestReceiver *self);
    void (*rDestroy)(DestReceiver *self);
    CommandDest mydest;
};
```
`rStartup()` 是一次 executor run 的初始化。
它拿到 operation 和输出 tuple descriptor。
`receiveSlot()` 是 hot path。
它被每个输出 tuple 调用。
返回 `true` 表示继续。
返回 `false` 表示目标不再接收，executor 像到达本次输出终点一样停止 loop。
`rShutdown()` 是一次 executor run 的收尾。
它释放 run-local workspace。
`rDestroy()` 销毁 receiver object。
同一个 receiver object 可以被多次 `ExecutorRun()` 复用。
每次 run 应有 `rStartup()` / `receiveSlot()` / `rShutdown()`。
最终由创建者 `rDestroy()`。
因此 receiver 状态分两类。
object-lifetime 状态包括 `DR_printtup.portal`、`DR_copy.cstate`、tuplestore 参数。
run-lifetime 状态包括输出函数缓存、per-row context、DataRow buffer、tuple conversion map。
这两类状态不能混淆。
### 4.3. `QueryDesc.dest`
`CreateQueryDesc()` 接收 `DestReceiver *dest`。
`standard_ExecutorRun()` 从 `queryDesc` 取出 `operation` 和 `dest`。
executor 不关心这个 dest 来自客户端、SPI、COPY 还是 EXPLAIN。
plan node 只产生 slot。
`ExecutePlan()` 只把 slot 交给 `dest->receiveSlot()`。
这是 `DestReceiver` 抽象的最小边界。
### 4.4. `DR_printtup`
`printtup.c` 中普通客户端 receiver 的私有结构是 `DR_printtup`。
关键字段：
| 字段 | 语义 |
| --- | --- |
| `pub` | `DestReceiver` base，必须是第一个字段。 |
| `portal` | 输出来自哪个 portal，用于 target list 和 format。 |
| `sendDescrip` | startup 时是否发送 `RowDescription`。 |
| `attrinfo` / `nattrs` | 当前缓存的 tuple descriptor 身份和列数。 |
| `myinfo` | 每列 text output 或 binary send function 的 `FmgrInfo`。 |
| `buf` | 可复用 protocol message buffer，不放在 per-row context。 |
| `tmpcontext` | 每行 output/send 函数临时内存。 |
`DestRemote` 和 `DestRemoteExecute` 都走 `printtup_create_DR()`。
差别是 `sendDescrip`。
`DestRemote` 自动发送 `RowDescription`。
`DestRemoteExecute` 不自动发送。
extended query 中，客户端应通过 `Describe` 获取结果形状。
`Execute` 本身只负责产生 `DataRow` 与 completion。
### 4.5. `SPITupleTable`
SPI receiver 的目标不是网络。
它把每个 slot 复制成 heap tuple，放进当前 SPI 连接的 `SPITupleTable`。
关键状态：
| 状态 | 语义 |
| --- | --- |
| `_SPI_current` | 当前 SPI stack frame。 |
| `_SPI_current->tuptable` | 当前正在构造的结果表。 |
| `SPI_tuptable` | 对外暴露给 SPI caller 的结果表。 |
| `SPITupleTable.tuptabcxt` | 存放 tuple table 和 copied tuples 的 memory context。 |
| `SPITupleTable.vals` | `HeapTuple` 指针数组，满了会扩大。 |
| `SPITupleTable.tupdesc` | 结果 tuple descriptor copy。 |
| `SPITupleTable.subid` | 创建所在 subtransaction，用于 cleanup。 |
SPI 不能保存 executor slot 指针。
slot 会被 executor 节点复用。
SPI caller 需要在 executor 结束后仍能读取结果。
所以 `spi_printtup()` 使用 `ExecCopySlotHeapTuple()`。
### 4.6. `TStoreState`
tuplestore receiver 用于 cursor hold store、portal materialization 和一些 utility select 输出。
关键字段：
| 字段 | 语义 |
| --- | --- |
| `tstore` | 目标 `Tuplestorestate`。 |
| `cxt` | 包含 tuplestore 的 memory context。 |
| `detoast` | 是否强制取回 out-of-line TOAST datum。 |
| `target_tupdesc` | 可选目标 rowtype。 |
| `tupmap` | 输入和目标 descriptor 不一致时的转换 map。 |
| `mapslot` | tuple conversion 后的临时 slot。 |
| `outvalues` / `tofree` | detoast 路径 workspace。 |
`tstoreStartupReceiver()` 会根据参数改写 `receiveSlot` 指针。
不需要 detoast/map 时，用 `tstoreReceiveSlot_notoast()`。
需要 detoast 时，用 `tstoreReceiveSlot_detoast()`。
需要 tuple map 时，用 `tstoreReceiveSlot_tupmap()`。
startup 根据 tuple descriptor 和参数选择更窄 hot path，这是 receiver 内部优化。
### 4.7. `DR_copy` 与 `CopyToState`
`DestCopyOut` 的 receiver 私有状态很小。
`DR_copy` 主要保存 `cstate` 和 `processed`。
真正复杂的是 `CopyToStateData`。
它保存 format、目标、列列表、输出函数、per-row buffer、file/frontend/program sink 等。
`copy_dest_receive()` 做两件事：
```text
CopyOneRowTo(cstate, slot)
pgstat_progress_update_param(PROGRESS_COPY_TUPLES_PROCESSED, ++processed)
```
因此 COPY query 路径仍然用 executor。
但 row protocol 不是 `DataRow`。
它是 COPY 自己的 `CopyData`、file write 或 program pipe。
### 4.8. `SerializeDestReceiver`
`DestExplainSerialize` 定义在 `explain_dr.c`。
它用于 `EXPLAIN (ANALYZE, SERIALIZE TEXT|BINARY)`。
关键字段：
| 字段 | 语义 |
| --- | --- |
| `es` | 当前 `ExplainState`。 |
| `format` | text 或 binary，模拟 wire protocol format。 |
| `attrinfo` / `nattrs` | tuple descriptor 缓存。 |
| `finfos` | 每列 output/send function。 |
| `tmpcontext` | per-row temporary context。 |
| `buf` | 复用 DataRow buffer。 |
| `metrics` | 序列化字节、时间、buffer usage。 |
它刻意接近 `printtup()`。
但它不调用 `pq_endmessage_reuse()`。
它构造 DataRow-like buffer、统计长度和耗时，然后丢弃。
这让 EXPLAIN 可以测平时被默认丢弃的输出序列化成本。
### 4.9. `None_Receiver`
`None_Receiver` 是永久可用的 `DestNone` receiver。
它的 receive/startup/shutdown/destroy 都是 no-op 或返回 true。
源码把它做成静态对象，避免 portal、cursor、EXPLAIN 默认路径反复 palloc/free。
它表达一个真实路径：有些 query 必须执行副作用或 instrumentation，但不需要输出 tuple。
## 5. 主流程源码 walkthrough
### 5.1. simple query 中的 receiver
simple query 的入口在 `postgres.c`。
每个 raw parsetree 会创建 portal、确定 command tag、选择 dest。
核心顺序是：
```text
BeginCommand(commandTag, dest)
receiver = CreateDestReceiver(dest)
if dest == DestRemote:
    SetRemoteDestReceiverParams(receiver, portal)
PortalRun(..., receiver, receiver, &qc)
receiver->rDestroy(receiver)
PortalDrop(portal, false)
EndCommand(&qc, dest, false)
```
`BeginCommand()` 当前没有实际工作。
但它仍是 command 输出生命周期的边界。
`EndCommand()` 是客户端看到 `CommandComplete` 的地方。
simple query 源码注释强调：每个 raw parsetree 发送一次 `EndCommand`。
rewrite 产生的辅助 query 不会各自发送 command complete。
所以一条 SQL 经 rewrite 执行多个 `PlannedStmt`，客户端仍只看到一个最终 command tag。
### 5.2. extended `Execute` 中的 receiver
extended query 的 `Execute` 路径也在 `postgres.c`。
目标通常是 `DestRemoteExecute`。
顺序近似：
```text
BeginCommand(portal->commandTag, dest)
receiver = CreateDestReceiver(dest)
if dest == DestRemoteExecute:
    SetRemoteDestReceiverParams(receiver, portal)
PortalRun(..., receiver, receiver, &qc)
receiver->rDestroy(receiver)
EndCommand(&qc, dest, false)
```
关键差别是 row description。
`DestRemoteExecute` 的 `sendDescrip` 为 false。
`Execute` 不自动发送 `RowDescription`。
客户端应通过 `Describe` message 获取结果形状。
因此：
```text
DestRemote
  -> startup 发送 RowDescription

DestRemoteExecute
  -> startup 不发送 RowDescription
  -> receiveSlot 仍发送 DataRow
```
### 5.3. `CreateDestReceiver()` 分派
`dest.c` 的分派保留了真实源码的直接性：
```text
DestRemote / DestRemoteExecute -> printtup_create_DR(dest)
DestRemoteSimple              -> printsimpleDR
DestNone                      -> donothingDR
DestDebug                     -> debugtupDR
DestSPI                       -> spi_printtupDR
DestTuplestore                -> CreateTuplestoreDestReceiver()
DestIntoRel                   -> CreateIntoRelDestReceiver(NULL)
DestCopyOut                   -> CreateCopyDestReceiver()
DestSQLFunction               -> CreateSQLFunctionDestReceiver()
DestTransientRel              -> CreateTransientRelDestReceiver(InvalidOid)
DestTupleQueue                -> CreateTupleQueueDestReceiver(NULL)
DestExplainSerialize          -> CreateExplainSerializeDestReceiver(NULL)
```
这里有两类 receiver。
一类是静态 receiver，例如 `DestNone`、`DestDebug`、`DestRemoteSimple`、`DestSPI`。
它们的 private state 不在 receiver object 里，或不需要 private state。
另一类是 palloc receiver，例如 `DR_printtup`、`TStoreState`、`DR_copy`、`SerializeDestReceiver`。
调用者必须在足够长的 memory context 创建它们。
`DestExplainSerialize` 在 `dest.c` 里有分派，但 EXPLAIN 主路径直接调用 `CreateExplainSerializeDestReceiver(es)`，因为它必须带 `ExplainState`。
### 5.4. receiver 参数补充
`CreateDestReceiver()` 只选择类型。
有些 receiver 还要调用类型专属 setter。
remote receiver 需要 portal：
```text
SetRemoteDestReceiverParams(receiver, portal)
```
tuplestore receiver 需要 tuplestore、context、detoast 和可选目标 descriptor：
```text
SetTuplestoreDestReceiverParams(receiver,
                                portal->holdStore,
                                portal->holdContext,
                                true_or_false,
                                target_tupdesc_or_NULL,
                                map_failure_msg_or_NULL)
```
COPY receiver 需要 `CopyToState`。
源码直接设置：
```text
dest = CreateDestReceiver(DestCopyOut)
((DR_copy *) dest)->cstate = cstate
```
这看起来不够抽象。
但它避免给所有 receiver 设计一个膨胀的通用配置对象。
PostgreSQL 在这里选择小 base contract 加目标专属参数。
### 5.5. executor 固定调用顺序
`standard_ExecutorRun()` 是本节主链路核心。
它先判断是否会输出 tuple：
```text
sendTuples =
  operation == CMD_SELECT
  || plannedstmt->hasReturning
```
如果会输出：
```text
dest->rStartup(dest, operation, queryDesc->tupDesc)
```
然后进入 `ExecutePlan()`。
`ExecutePlan()` 的骨架是：
```text
for (;;)
{
    ResetPerTupleExprContext(estate);
    slot = ExecProcNode(planstate);
    if (TupIsNull(slot))
        break;
    if (estate->es_junkFilter)
        slot = ExecFilterJunk(estate->es_junkFilter, slot);
    if (sendTuples)
        if (!dest->receiveSlot(slot, dest))
            break;
    if (operation == CMD_SELECT)
        estate->es_processed++;
    if (count limit reached)
        break;
}
```
receiver 看到的是经过 junk filter 清理后的 slot。
`receiveSlot()` 在每个输出 tuple 上调用。
如果它返回 false，executor 结束本次 loop。
`ExecutorRun()` 退出前，如果启动过 receiver，会调用：
```text
dest->rShutdown(dest)
```
### 5.6. 普通客户端输出：`printtup`
普通客户端 receiver 是 `DR_printtup`。
startup 创建复用的 `StringInfoData buf`。
startup 创建 per-row `tmpcontext`。
如果 `sendDescrip` 为 true，startup 调用 `SendRowDescriptionMessage()`。
`SendRowDescriptionMessage()` 发送列名、源表 OID、源列号、类型 OID、长度、typmod、format code。
它会跳过 target list 中 resjunk 项。
第一行到来时，`printtup()` 可能调用 `printtup_prepare_info()`。
它按列缓存 text output 或 binary send function：
```text
format 0 -> getTypeOutputInfo()       -> OutputFunctionCall()
format 1 -> getTypeBinaryOutputInfo() -> SendFunctionCall()
other    -> ERROR "unsupported format code"
```
每一行的核心顺序是：
```text
slot_getallattrs(slot)
MemoryContextSwitchTo(tmpcontext)
pq_beginmessage_reuse(buf, PqMsg_DataRow)
pq_sendint16(buf, natts)
for each attr:
  NULL   -> length -1
  text   -> OutputFunctionCall() + pq_sendcountedtext()
  binary -> SendFunctionCall() + length + bytes
pq_endmessage_reuse(buf)
MemoryContextReset(tmpcontext)
```
`pq_endmessage_reuse()` 把 message 交给 backend libpq 输出通道。
它不保证客户端已经收到。
最终 flush 通常发生在 `ReadyForQuery()` 或其他协议边界。
这解释了为什么大结果集可能卡在 client write，而 `printtup()` 代码看起来只是构造 message。
### 5.7. `EndCommand()` 与 `ReadyForQuery()`
tuple 输出、command completion 和 protocol sync 是三层边界。
`receiveSlot()` 负责行。
`EndCommand()` 负责 `CommandComplete`。
`ReadyForQuery()` 负责同步点和事务状态字节。
`EndCommandExtended()` 对远端目标执行：
```text
BuildQueryCompletionString(completionTag, qc, force_undecorated_output)
pq_putmessage(PqMsg_CommandComplete, completionTag, len + 1)
```
对 SPI、tuplestore、COPY、EXPLAIN serialize 等目标则不发送。
`ReadyForQuery()` 对远端目标发送：
```text
PqMsg_ReadyForQuery
TransactionBlockStatusCode()
pq_flush()
```
所以不要把 `DataRow`、`CommandComplete`、`ReadyForQuery` 都归到 receiver。
receiver 只覆盖 row-level output。
### 5.8. SPI 输出：`DestSPI`
SPI 执行 query 时，`_SPI_execute_plan()` 选择 receiver。
如果 statement 不能 set tag，输出进 `DestNone`。
如果 caller 提供 `options->dest`，使用 caller receiver。
否则使用：
```text
CreateDestReceiver(DestSPI)
```
`_SPI_pquery()` 调用：
```text
ExecutorStart(queryDesc, eflags)
ExecutorRun(queryDesc, ForwardScanDirection, tcount)
_SPI_current->processed = queryDesc->estate->es_processed
ExecutorFinish(queryDesc)
ExecutorEnd(queryDesc)
```
`spi_dest_startup()` 创建 `SPITupleTable`。
它在 SPI procedure memory context 下创建 `SPI TupTable` 子 context。
它初始化 `alloced = 128`、`numvals = 0`、`vals` 数组和 `tupdesc` copy。
每行 `spi_printtup()`：
```text
if vals full:
    repalloc_huge(vals, alloced * 2)
vals[numvals] = ExecCopySlotHeapTuple(slot)
numvals++
```
执行结束后，`SPI_processed` 和 `SPI_tuptable` 暴露给 caller。
`_SPI_current->tuptable` 被置空，表示结果表 ownership 转给 SPI caller。
### 5.9. tuplestore 输出：portal hold store
`pquery.c` 的 `FillPortalStore()` 是一个典型 `DestTuplestore` 入口。
它用于 `PORTAL_ONE_RETURNING`、`PORTAL_ONE_MOD_WITH` 和 `PORTAL_UTIL_SELECT`。
主流程：
```text
PortalCreateHoldStore(portal)
treceiver = CreateDestReceiver(DestTuplestore)
SetTuplestoreDestReceiverParams(treceiver,
                                portal->holdStore,
                                portal->holdContext,
                                false,
                                NULL,
                                NULL)
PortalRunMulti(..., treceiver, None_Receiver, &qc)
treceiver->rDestroy(treceiver)
```
`portalcmds.c` 还有 hold cursor 路径。
它把已有 `queryDesc->dest` 改成 `DestTuplestore`，并传 `detoast = true`。
源码注释解释：这让 hold store 中的数据不再需要关联原 snapshot。
最简单 hot path 是：
```text
tuplestore_puttupleslot(myState->tstore, slot)
```
detoast 路径会取回 external datum，再用 `tuplestore_putvalues()`。
tuple map 路径先 `execute_attr_map_slot()` 到 `mapslot`，再写入 tuplestore。
### 5.10. COPY TO 输出：`DestCopyOut`
`COPY (SELECT ...) TO ...` 需要 executor 产生行，但输出不是 `DataRow`。
`BeginCopyTo()` 在 query COPY 路径里：
```text
dest = CreateDestReceiver(DestCopyOut)
((DR_copy *) dest)->cstate = cstate
queryDesc = CreateQueryDesc(plan, ..., dest, ...)
ExecutorStart(queryDesc, 0)
```
`DoCopyTo()` 对 query path 执行：
```text
ExecutorRun(cstate->queryDesc, ForwardScanDirection, 0)
processed = ((DR_copy *) cstate->queryDesc->dest)->processed
```
每行进入：
```text
copy_dest_receive()
  -> CopyOneRowTo(cstate, slot)
  -> pgstat_progress_update_param(PROGRESS_COPY_TUPLES_PROCESSED, ++processed)
```
如果目标是 frontend STDOUT，COPY 外层发送：
```text
CopyOutResponse
CopyData*
CopyDone
```
然后 command 层再发送 `CommandComplete` 和 `ReadyForQuery`。
relation COPY TO 不一定通过 `DestCopyOut`，它可以由 `CopyRelationTo()` 扫 relation 并直接 `CopyOneRowTo()`。
本节关注的是 query COPY 如何把 executor output 接到 COPY receiver。
### 5.11. EXPLAIN ANALYZE 输出边界
`ExplainOnePlan()` 先决定 instrumentation 选项，再选择 receiver：
```text
if (into)
    dest = CreateIntoRelDestReceiver(into)
else if (es->serialize != EXPLAIN_SERIALIZE_NONE)
    dest = CreateExplainSerializeDestReceiver(es)
else
    dest = None_Receiver
```
普通 `EXPLAIN ANALYZE SELECT ...` 会执行计划，但被解释 query 的结果 tuple 进入 `None_Receiver`。
这就是为什么你看到 node actual rows，却看不到 query result。
`EXPLAIN (ANALYZE, SERIALIZE TEXT|BINARY)` 使用 `DestExplainSerialize`。
它模拟 `DataRow` 序列化，调用 text output 或 binary send function，累计字节和耗时。
但它不会把这些行发给客户端。
`ExplainOnePlan()` 在销毁 receiver 前读取 metrics：
```text
serializeMetrics = GetSerializationMetrics(dest)
dest->rDestroy(dest)
```
即使在 explain 中，也显式调用 receiver destroy。
这保证动态 receiver 的 buffer、tmpcontext 和 metrics state 收尾。
### 5.12. utility tuple 输出
不是所有 tuple 输出都来自 plan node。
`execTuples.c` 提供 helper：
```text
begin_tup_output_tupdesc(dest, tupdesc, ...)
  -> dest->rStartup(dest, CMD_SELECT, tupdesc)
do_tup_output()
  -> slot 填 values/isnull
  -> dest->receiveSlot(slot, dest)
end_tup_output()
  -> dest->rShutdown(dest)
```
这说明 `DestReceiver` 是 PostgreSQL 内部通用 tuple result sink。
只要调用者能构造符合 descriptor 的 slot，就可以交给 receiver。
## 6. 生命周期 / ownership / cleanup
### 6.1. 谁创建
receiver 通常由进入 executor 前的上层模块创建。
simple query 和 extended query 由 `postgres.c` 创建。
SPI 由 `_SPI_execute_plan()` 创建或使用 caller 提供的 receiver。
COPY TO query path 由 `BeginCopyTo()` 创建。
EXPLAIN 由 `ExplainOnePlan()` 创建。
portal hold store 由 `FillPortalStore()` 或 hold cursor 相关路径创建。
executor 本身不创建 receiver。
### 6.2. 谁持有
`QueryDesc.dest` 持有指针，但通常不拥有 receiver。
owner 是创建它的上层调用者。
simple query 的 owner 是 `postgres.c`。
COPY query 的 receiver 存在于 `cstate->queryDesc->dest`，COPY command 负责结束 query desc 和 copy state。
SPI 的 `DestSPI` 是静态 receiver，真实结果状态在 `_SPI_current`。
tuplestore receiver object 是临时的，tuplestore 本体由 portal hold context 或 caller context 拥有。
receiver object 和目标资源不是同一个 owner。
`TStoreState` 可以销毁，`portal->holdStore` 仍然存在。
`DR_copy` 可以销毁，`CopyToState` 仍由 COPY lifecycle 管。
### 6.3. run-local workspace
`rStartup()` 分配 run-local workspace。
`rShutdown()` 释放 run-local workspace。
`printtup` 的 `buf` 和 `tmpcontext` 在 startup 创建、shutdown 删除。
`tstoreReceiver` 的 detoast arrays、conversion map、mapslot 在 startup/shutdown 之间有效。
`DestExplainSerialize` 的 `finfos`、`buf`、`tmpcontext` 也在 startup/shutdown 之间有效。
这允许 receiver object 在多次 `ExecutorRun()` 间复用。
portal 分批 fetch 需要这种分层生命周期。
### 6.4. tuple ownership
executor slot 的 ownership 不转移给 receiver。
receiver 必须在 `receiveSlot()` 返回前完成复制、序列化或写入。
| receiver | 策略 |
| --- | --- |
| `printtup` | 立即把 attributes 序列化进 protocol buffer。 |
| `DestSPI` | `ExecCopySlotHeapTuple()` 复制为 `HeapTuple`。 |
| `DestTuplestore` | `tuplestore_puttupleslot()` 或 detoast 后 `tuplestore_putvalues()`。 |
| `DestCopyOut` | 立即格式化到 COPY sink。 |
| `DestExplainSerialize` | 立即模拟序列化并计量。 |
| `DestNone` | 不读取 slot 内容。 |
扩展不能保存 `TupleTableSlot *` 等以后再读。
slot 会被 executor 复用、清空或释放。
### 6.5. ERROR cleanup
如果 `receiveSlot()`、类型输出函数或 COPY formatter `ereport(ERROR)`，控制流跳出 executor。
`PortalRun()` 的 `PG_TRY/PG_CATCH` 会把 portal 标成 failed，恢复 `ActivePortal`、`CurrentResourceOwner`、`PortalContext`，再抛出。
事务 abort 会 reset 相关 memory context，释放 executor state、portal-local children 和 receiver 分配的 context。
但 `rShutdown()` 不一定执行。
如果 ERROR 发生在 `receiveSlot()` 中，普通 C stack 不会自然走到 `standard_ExecutorRun()` 尾部。
所以 receiver 不能把关键外部资源只寄托在 `rShutdown()`。
`printtup` 的临时资源在 memory context 里。
COPY file/program 资源由 COPY command cleanup 负责。
SPI tuple table 挂在 SPI live list 上，subtransaction cleanup 由 `AtEOSubXact_SPI` 兜底。
这体现 PostgreSQL ERROR 模型：memory context、ResourceOwner、portal cleanup、subtransaction cleanup 共同兜底。
### 6.6. command completion ownership
receiver 不拥有 command completion。
`QueryCompletion qc` 由 portal/executor/utility 路径填充。
`EndCommand()` 根据 `qc` 和 `dest` 决定是否给客户端发送 `CommandComplete`。
SPI 的 `SPI_processed` 与 `SPI_result` 是 SPI API 结果。
COPY 的 completion tag 是 COPY command 结果。
EXPLAIN 的输出是 EXPLAIN command 的结果，不是被解释 query 的 command complete。
## 7. 正确性机制层次
### 7.1. tuple descriptor contract
`dest.h` 要求 `receiveSlot()` 接收的 slot descriptor 与 `rStartup()` 的 `typeinfo` 一致。
`printtup()` 仍会检查 descriptor identity 和列数。
如果 descriptor 变化，会重新准备输出函数缓存。
这只是防御，不是鼓励 mid-run 改 descriptor。
正确路径应保持输出 descriptor 稳定。
### 7.2. junk filter boundary
executor 在调用 receiver 前可能执行 `ExecFilterJunk()`。
这保证 resjunk 列不会进入最终输出。
DML `RETURNING` 或 row identity 内部列不应该泄漏到 receiver。
receiver 不理解 executor 内部 junk 列。
它只看最终输出 descriptor。
### 7.3. type output correctness
`printtup()` 和 `DestExplainSerialize` 依赖类型 text output 或 binary send function。
format code 0 走 text output。
format code 1 走 binary send。
其他 format code 报错。
receiver 不理解每种类型内部布局。
wire output 的类型正确性来自 type system。
### 7.4. memory context boundary
output/send function 可能 palloc。
`printtup()` 使用 per-row `tmpcontext`。
COPY TO 使用 `rowcontext`。
EXPLAIN serialize 也使用 per-row context。
SPI tuple table 使用 `tuptabcxt` 保存结果。
tuplestore receiver 把持久数据写入 caller 指定 context 管理的 tuplestore。
正确性来自：
```text
短生命周期临时内存每行 reset；
跨 executor 生命周期的结果必须复制或写入长期 owner；
ERROR 由 context cleanup 兜底。
```
### 7.5. protocol ordering
普通远端输出顺序是：
```text
RowDescription
DataRow*
CommandComplete
ReadyForQuery
```
extended `Execute` 可能是：
```text
DataRow*
CommandComplete 或 PortalSuspended
Sync 后 ReadyForQuery
```
COPY TO STDOUT 是：
```text
CopyOutResponse
CopyData*
CopyDone
CommandComplete
ReadyForQuery
```
receiver 只保证 row-level 输出。
`dest.c` 和 protocol loop 负责 command/protocol sync。
COPY command 自己负责 COPY subprotocol。
### 7.6. snapshot 不属于 receiver
`DestReceiver` 不选择 snapshot。
`PortalRunMulti()`、`BeginCopyTo()`、`ExplainOnePlan()` 会在进入 executor 前设置 snapshot 或 command id。
receiver 不决定“看到哪些 tuple”。
它只决定“已经看到的 tuple 去哪里”。
## 8. 错误路径 / 异常路径 / fallback
### 8.1. `receiveSlot()` 返回 false
`receiveSlot()` 返回 false 表示目标不再接收 tuple。
`ExecutePlan()` break。
这不是 ERROR。
它类似消费者提前停止。
内置 receiver 大多返回 true，但 contract 保留 early stop 能力。
返回 false 不表示 plan 已经 EOF。
它只结束本次 `ExecutePlan()` loop。
### 8.2. extended protocol 丢弃不可描述输出
`PortalRunMulti()` 中有特殊逻辑：
```text
if dest is DestRemoteExecute:
    dest = None_Receiver
```
原因是某些 portal strategy 没有机制在 `Execute` 时发送额外 `RowDescription`。
rewrite 给非 SELECT command 增加的 SELECT 辅助 query，客户端既不期待也不知道形状。
所以这些 tuple 被丢弃。
simple Query protocol 下情况不同，因为它可以按 command 发送 row description。
### 8.3. output function ERROR
类型 output/send function 报错时，ERROR 从 `receiveSlot()` 抛出。
客户端可能已经收到部分 `DataRow`。
但不会收到成功的 `CommandComplete`。
simple query 路径会在最后一个 raw parsetree 的事务收尾之后才 `EndCommand()`。
这样避免先发 command complete，后又报事务结束错误。
这也是 command completion 不放进 executor 的原因。
### 8.4. SPI consistency
`_SPI_pquery()` 结束后会检查 tuple count。
如果结果是 `SPI_OK_SELECT` 或 DML `RETURNING`，并且目标是 `DestSPI`，会调用 `_SPI_checktuples()`。
失败会 `elog(ERROR, "consistency check on SPI tuple count failed")`。
这防止 SPI API 暴露的 `SPI_processed` 与 tuple table 内容不一致。
### 8.5. COPY relation changed
`BeginCopyTo()` 在 query COPY 中会检查 plan relation OID 是否仍包含原先锁住的 relation。
不满足时会报：
```text
relation referenced by COPY statement has changed
```
这发生在 receiver 之前。
COPY 需要先保证 source relation 语义没有漂移，再把 executor output 接给 `DestCopyOut`。
### 8.6. EXPLAIN serialize 与 `CREATE TABLE AS`
`ExplainOnePlan()` 中 `into` 优先于 `serialize`。
解释 `CREATE TABLE AS` 时，执行 query 必须使用 `IntoRel` receiver 产生建表副作用。
如果同时指定 `SERIALIZE`，serialize metrics 可能是 0。
源码注释说明这是合理的，因为这些数据本来不会发给客户端。
### 8.7. `NullCommand()`
空 query string 不进入 executor。
`NullCommand(dest)` 对远端目标发送 `EmptyQueryResponse`。
extended query protocol 需要这个可识别响应，避免客户端无法判断空执行的结束。
这再次说明输出协议不等于 executor tuple 输出。
没有 tuple 时仍可能需要协议消息。
### 8.8. `rShutdown()` 不一定是 finally
普通成功路径会调用 `rShutdown()`。
ERROR longjmp 可能跳过它。
所以动态 receiver 要把内存放进合适 context，把外部资源交给上层 lifecycle。
这是 PostgreSQL C 错误模型下的基本约束。
## 9. 成本、资源与跨模块传播
### 9.1. 每行 callback 与 slot deforming
每输出一行都会调用 `dest->receiveSlot()`。
这是一次间接调用。
窄表、大量行、CPU cache 友好的 query 上，receiver 成本可能变得可见。
`printtup()` 会 `slot_getallattrs()`。
如果上游还没有 materialize 所有列，输出阶段承担 deforming 成本。
成本随 `输出行数 * 输出列数` 扩张。
### 9.2. 类型输出函数
text protocol 调用每列 type output function。
binary protocol 调用每列 send function。
`numeric_out()`、`jsonb_out()`、数组、复杂 varlena 都可能成为热点。
普通 `EXPLAIN ANALYZE` 默认丢弃输出，可能低估真实 SELECT。
`EXPLAIN (ANALYZE, SERIALIZE TEXT|BINARY)` 用于补这部分。
### 9.3. TOAST 与 detoast
普通 `printtup()` 可能在类型输出函数中 detoast。
tuplestore receiver 在 `detoast=true` 时显式取回 out-of-line datum。
COPY 输出也可能通过 output functions 或 formatter 触发 detoast。
这类成本随大 varlena 列、TOAST 外部存储比例和输出格式扩张。
### 9.4. 网络 backpressure
`pq_endmessage_reuse()` 只把 message 放入后端输出通道。
当输出缓冲或 socket 无法吸收数据，backend 会在写客户端时等待。
`pg_stat_activity.wait_event` 可能显示 client write 相关等待。
这时 executor 看起来慢，实际瓶颈在客户端读取、网络、TLS 或驱动 fetch 策略。
### 9.5. SPI 复制和内存放大
`DestSPI` 为每行复制 `HeapTuple`。
`vals` 指针数组按倍数扩张。
结果越大，SPI procedure context 压力越大。
这和普通客户端 streaming 输出不同。
普通 `printtup()` 每行序列化后可以 reset 临时内存。
SPI 结果要留给 caller。
### 9.6. tuplestore spill
`DestTuplestore` 调用 `tuplestore_puttupleslot()`。
真正内存/临时文件策略在 tuplestore 模块。
WITH HOLD cursor 或 materialized result 大时，成本传播到 `work_mem`、temp file IO、temp tablespace 和后续 FETCH。
看到 cursor fetch 慢时，要检查第一次填充 hold store 的成本。
### 9.7. COPY 格式化与 progress
COPY text/csv 需要转义 delimiter、quote、NULL 字符串和行结束。
binary 需要二进制头尾和 per-field length。
json 格式有对象构造成本。
`copy_dest_receive()` 每行更新 `PROGRESS_COPY_TUPLES_PROCESSED`。
COPY 有自己的 progress 入口，不依赖 executor rows 指标。
### 9.8. 跨模块传播
| 模块 | 边界 |
| --- | --- |
| tcop/protocol | 选择 `CommandDest`，发送 command completion 和 sync。 |
| portal | 管理 strategy、snapshot、hold store、分批 fetch。 |
| executor | 产生 slot，调用 receiver。 |
| type system | 提供 output/send function。 |
| backend libpq | 管理 message buffer、socket write、flush。 |
| SPI | 管理 nested execution、tuple table、subtransaction cleanup。 |
| tuplestore | 管理 materialized result、memory/spill、read pointer。 |
| COPY | 管理 COPY subprotocol、sink、progress。 |
| EXPLAIN | 管理 instrumentation、serialize metrics、plan rendering。 |
本节核心状态不是 shared memory，没有后台进程直接推进。
但输出压力会影响全局资源：client write wait、temp IO、COPY file IO、backend memory。
## 10. 观测与诊断入口
### 10.1. runtime truth
本节锚定的 runtime truth 是：
```text
同一条 executor plan，换一个 DestReceiver，输出成本、内存 ownership、客户端可见协议和可观测指标都会改变。
```
诊断时要完成：
```text
看到慢或内存增长
  -> 判断是否发生在 plan node、receiver、协议 flush 或 receiver 目标资源中
  -> 回到具体 receiver 源码验证
```
### 10.2. 可直接观测
| 入口 | 能看到 |
| --- | --- |
| driver/libpq trace | `RowDescription`、`DataRow`、`CommandComplete`、`ReadyForQuery` 顺序。 |
| `EXPLAIN (ANALYZE)` | executor node rows/timing；默认不含真实结果序列化。 |
| `EXPLAIN (ANALYZE, SERIALIZE TEXT)` | text output 和 DataRow-like 构造成本。 |
| `EXPLAIN (ANALYZE, SERIALIZE BINARY)` | binary send 和 DataRow-like 构造成本。 |
| `pg_stat_activity.wait_event` | 采样时可能看到 client write。 |
| `pg_stat_progress_copy` | COPY processed rows。 |
| server log duration | 粗粒度包含后端执行和输出时间。 |
SPI tuple table 通常不能从系统视图直接看到。
tuplestore spill 也不是 receiver 层直接暴露。
需要结合 temp file log、`pg_stat_database.temp_bytes`、`EXPLAIN`、gdb 或 memory context 推断。
### 10.3. profiling 入口
如果问题是 CPU 输出成本，常见热点包括：
```text
printtup
OutputFunctionCall
SendFunctionCall
numeric_out
jsonb_out
CopyOneRowTo
CopyAttributeOutCSV
tuplestore_puttupleslot
ExecCopySlotHeapTuple
serializeAnalyzeReceive
```
如果 flamegraph 顶部是这些函数，不要把问题直接归因到 scan 或 join。
### 10.4. gdb 断点
普通客户端输出：
```gdb
break standard_ExecutorRun
break ExecutePlan
break printtup_startup
break printtup
break EndCommandExtended
break ReadyForQuery
```
观察：
```text
queryDesc->dest->mydest
queryDesc->tupDesc->natts
((DR_printtup *) queryDesc->dest)->sendDescrip
estate->es_processed
```
SPI：
```gdb
break spi_dest_startup
break spi_printtup
break _SPI_pquery
```
COPY：
```gdb
break CreateCopyDestReceiver
break copy_dest_receive
break CopyOneRowTo
break SendCopyBegin
break SendCopyEnd
```
EXPLAIN serialize：
```gdb
break CreateExplainSerializeDestReceiver
break serializeAnalyzeReceive
break GetSerializationMetrics
```
### 10.5. SQL 对照实验入口
对照默认 EXPLAIN 和 serialize：
```sql
EXPLAIN (ANALYZE, TIMING OFF)
SELECT repeat(md5(i::text), 20)
FROM generate_series(1, 200000) AS g(i);

EXPLAIN (ANALYZE, TIMING OFF, SERIALIZE TEXT)
SELECT repeat(md5(i::text), 20)
FROM generate_series(1, 200000) AS g(i);
```
第一条默认不序列化被解释 query 的结果。
第二条模拟 text output。
差异主要来自 receiver/output function，而不是 scan 本身。
COPY 对照：
```sql
COPY (
  SELECT repeat(md5(i::text), 20)
  FROM generate_series(1, 200000) AS g(i)
) TO STDOUT WITH (FORMAT csv);
```
这里 executor 仍产生 slot，但 receiver 进入 COPY formatter。
客户端看到 COPY subprotocol，不是 `DataRow`。
### 10.6. 观测盲区
普通 `EXPLAIN ANALYZE` 不等于用户收到最后一字节的时间。
`SERIALIZE` 测 output/send function 和 DataRow-like buffer 构造。
它不测真实网络、客户端读取速度、TLS、驱动 decode。
`pg_stat_statements` total time 不告诉你具体在哪个 receiver。
wait event 只能显示采样时等待。
CPU 时间花在 `numeric_out()` 时，wait event 可能没有线索。
## 11. 常见误区
1. executor 直接发送结果给客户端。
实际是 executor 调用 `DestReceiver`，客户端协议在 `printtup.c`、`dest.c` 和 backend libpq 中完成。
2. `EXPLAIN ANALYZE` 等价于真实 SELECT 成本。
默认 `EXPLAIN ANALYZE` 使用 `None_Receiver`，不测真实 result serialization 和 network。
3. `CommandComplete` 是 receiver 发的。
receiver 发 tuple；`EndCommand()` 发 command completion；`ReadyForQuery()` 发 sync。
4. SPI 保存 executor slot。
SPI 保存复制出来的 heap tuple，不能保存 slot 指针。
5. tuplestore receiver 一定只是内存 append。
tuplestore 可能 spill，hold cursor 还可能 detoast。
6. COPY TO 只是普通 SELECT 的显示格式。
COPY query 使用 executor 产生 slot，但输出协议、formatter、progress 和 sink 属于 COPY。
7. `receiveSlot()` 返回 false 表示错误。
false 是 early stop contract；错误通过 `ereport(ERROR)`。
8. `rDestroy()` 会释放所有目标资源。
`rDestroy()` 只释放 receiver object；目标资源由 portal、SPI、COPY、tuplestore 或 EXPLAIN state 拥有。
## 12. 课堂实验
### 实验 1：普通 SELECT 输出边界
目标：确认 `RowDescription`、`DataRow`、`CommandComplete`、`ReadyForQuery` 属于不同源码层。
步骤：
```text
1. 启动 debug build。
2. gdb attach backend。
3. 设置断点：
   - printtup_startup
   - printtup
   - EndCommandExtended
   - ReadyForQuery
4. 执行：
   SELECT i, i::text FROM generate_series(1, 3) AS g(i);
```
预期：
```text
printtup_startup 先发送 RowDescription。
printtup 命中 3 次。
EndCommandExtended 发送 SELECT 3。
ReadyForQuery 发送事务状态字节并 flush。
```
### 实验 2：默认 EXPLAIN 与 SERIALIZE
SQL：
```sql
EXPLAIN (ANALYZE, TIMING OFF)
SELECT repeat(md5(i::text), 40)
FROM generate_series(1, 100000) AS g(i);

EXPLAIN (ANALYZE, TIMING OFF, SERIALIZE TEXT)
SELECT repeat(md5(i::text), 40)
FROM generate_series(1, 100000) AS g(i);
```
断点：
```gdb
break ExplainOnePlan
break serializeAnalyzeReceive
break printtup
```
预期：
```text
第一条使用 None_Receiver，不命中 serializeAnalyzeReceive。
第二条命中 serializeAnalyzeReceive，但不命中 printtup。
客户端只收到 EXPLAIN 结果，不收到被解释 query 的真实结果。
```
### 实验 3：COPY query receiver
SQL：
```sql
COPY (
  SELECT i, repeat('x', 10)
  FROM generate_series(1, 5) AS g(i)
) TO STDOUT WITH (FORMAT csv);
```
断点：
```gdb
break CreateCopyDestReceiver
break copy_dest_receive
break CopyOneRowTo
break printtup
```
预期：
```text
CreateCopyDestReceiver 命中一次。
copy_dest_receive 命中 5 次。
CopyOneRowTo 命中 5 次。
printtup 不负责 COPY result rows。
```
长 COPY 中可观察：
```sql
SELECT * FROM pg_stat_progress_copy;
```
### 实验 4：SPI tuple table ownership
目标：理解 `DestSPI` 为什么复制 tuple。
步骤：
```text
1. 用 PL/pgSQL 或 C extension 触发 SPI_execute。
2. 断点 spi_dest_startup、spi_printtup、_SPI_pquery。
3. 观察 SPITupleTable.tuptabcxt、numvals、alloced。
4. 执行结束后观察 SPI_tuptable 与 _SPI_current->tuptable 的 ownership 转移。
```
源码链路：
```text
ExecutorRun
  -> spi_printtup
  -> ExecCopySlotHeapTuple
  -> _SPI_current->tuptable
  -> SPI_tuptable
```
### 实验 5：WITH HOLD cursor 与 detoast
SQL：
```sql
CREATE TABLE dr_toast AS
SELECT i, repeat(md5(i::text), 2000) AS payload
FROM generate_series(1, 1000) AS g(i);

BEGIN;
DECLARE c CURSOR WITH HOLD FOR SELECT payload FROM dr_toast;
COMMIT;
FETCH 5 FROM c;
```
源码阅读：
```text
portalcmds.c
  -> SetTuplestoreDestReceiverParams(... detoast = true ...)
tstoreReceiver.c
  -> tstoreReceiveSlot_detoast
```
讨论：为什么 hold store 不能只保存外部 TOAST pointer？
## 13. 讨论题
1. 为什么 PostgreSQL 没有让 executor 直接调用 `pq_send*()`？
2. `DestRemote` 和 `DestRemoteExecute` 为什么都用 `printtup_create_DR()`，但 row description 行为不同？
3. `receiveSlot()` 返回 false 应该被理解为错误、EOF，还是消费者提前停止？
4. 默认 `EXPLAIN ANALYZE` 会低估哪些 workload 的真实用户感知成本？
5. 扩展实现 receiver 时，为什么不能保存 `TupleTableSlot *` 到全局 list？
6. SPI receiver 为什么复制 heap tuple，而 `printtup` 可以只序列化后丢掉？
7. COPY TO STDOUT 的 `CopyData` 与普通 SELECT 的 `DataRow` 分别从哪里发出？
8. tuplestore receiver 的 detoast 选项服务哪个 correctness 问题？它和 MVCC visibility 是同一层机制吗？
9. 为什么 `EndCommand()` 不适合放进 `DestReceiver.rShutdown()`？
10. flamegraph 显示 `numeric_out()` 或 `jsonb_out()` 时，你如何区分 plan node 成本和 output serialization 成本？
## 14. 本节小结
本节唯一主问题是：
```text
DestReceiver 如何把 executor tuple 输出到客户端、SPI、tuplestore、COPY 或 EXPLAIN？
```
核心链路是：
```text
CommandDest
  -> CreateDestReceiver()
  -> QueryDesc.dest
  -> ExecutorRun()
  -> rStartup()
  -> receiveSlot() per tuple
  -> rShutdown()
  -> EndCommand() / ReadyForQuery()
  -> rDestroy()
```
`CommandDest` 只是目标枚举。
真正的 output behavior 在 receiver callback 和 receiver 私有状态里。
ownership 边界是：
```text
executor owns slot production；
receiver owns immediate consumption or copying；
portal/SPI/COPY/tuplestore/EXPLAIN owns destination resource；
tcop owns command completion and protocol sync。
```
正确性来自 descriptor contract、junk filter、type output/send function、memory context、上游 snapshot 边界和 protocol ordering 的组合。
ERROR 时不能依赖 `rShutdown()` 一定执行。
receiver 必须让 memory context、ResourceOwner、portal cleanup、SPI subtransaction cleanup 或 COPY cleanup 能兜底。
观测上，默认 `EXPLAIN ANALYZE` 看到 executor 执行，不看到真实 result serialization。
`EXPLAIN (SERIALIZE)` 能测一部分 output function 和 DataRow-like 构造成本，但不测真实网络和客户端 decode。
COPY 有自己的 progress 和 subprotocol。
SPI 和 tuplestore 多数要通过源码、断点、memory context 或间接指标推断。
可迁移系统规律是：
```text
高频 producer 不应该理解所有 consumer；
用极小 callback contract 交出对象；
让 consumer 在自己的生命周期和资源边界内复制、序列化或丢弃；
由上层协议负责 command completion 和 sync。
```
判断性能和 correctness 问题时，不要只问“plan 为什么慢”。
还要问：
```text
tuple 在哪里 materialize？
在哪里序列化？
在哪里复制？
在哪里缓存或 spill？
在哪里真正 flush 给客户端？
```
这些答案通常就在具体 `DestReceiver` 实现里。
