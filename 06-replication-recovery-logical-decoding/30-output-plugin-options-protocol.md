# PostgreSQL output plugin options 与 logical replication protocol 边界


## 课程定位

前置知识：已经理解 replication slot 保存逻辑解码进度，知道 `ReorderBuffer` 会把 WAL record 重组成事务级 change，也知道 logical decoding 的输出不是 WAL 原文，而是 output plugin 生成的消息流。
本节唯一主问题：
```text
plugin 如何声明 textual / binary output、streaming、two-phase 和 origin 过滤能力，
客户端 options 如何影响输出内容和协议行为？
```
核心矛盾：logical decoding 要给扩展和内置逻辑复制足够自由，让插件决定输出格式、过滤规则和协议扩展；但 walsender、SQL decoding 函数、apply worker 和 downstream client 又必须在连接开始时就知道哪些行为可以启用，哪些消息格式能解析，哪些 transaction 可以提前 streaming 或在 PREPARE 时输出。
学完后应能判断：
```text
一个 option 是 plugin 私有输出开关，
还是 logical replication protocol 的兼容性开关；
一个 callback 表示能力声明，
还是 startup 时真正启用的运行状态；
一个过滤发生在 WAL record 入 ReorderBuffer 前，
还是发生在 pgoutput 生成协议消息时。
```
本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e90741e12a3715909e1b2d71ff9344`。

## 1. 本节在总主线中的位置

前几节已经把逻辑解码拆成三段：
```text
WAL record
  -> logical/decode.c 识别 heap/xact/message record
  -> ReorderBuffer 按事务重排、spill、stream
  -> output plugin 把 change 转成外部可见格式
```
这一节只看第三段和它向前、向后的边界。
向前看，plugin 不是直接扫描 WAL。
它接收的是 `ReorderBufferTXN`、`Relation`、`ReorderBufferChange` 和 commit / prepare / stream 事件。
向后看，plugin 也不是直接拥有网络连接。
它把 bytes 追加到 `ctx->out`，再通过 `OutputPluginPrepareWrite()` / `OutputPluginWrite()` 交给 writer。
在 walsender 场景中，writer 是 `WalSndPrepareWrite()` 和 `WalSndWriteData()`。
在 SQL 函数场景中，writer 是 `LogicalOutputPrepareWrite()` 和 `LogicalOutputWrite()`。
所以本节的主线不是“pgoutput 有哪些参数”。
主线是：
```text
客户端把 options 带进 START_REPLICATION
  -> walsender 创建 LogicalDecodingContext
  -> framework 根据 callbacks 先推断能力
  -> startup_cb 解析 options 并修正运行状态
  -> decode / reorder / plugin 根据状态决定跳过、stream、two-phase 或输出格式
  -> writer 把 plugin payload 放进 SQL tuplestore 或 COPY BOTH WALData 消息
```
这条链路解释一个常见现场问题：
```text
为什么同一个逻辑 slot，
用不同 START_REPLICATION options 连接，
会看到不同消息、不同协议版本要求、不同 streaming 行为，
甚至让 slot 的 two_phase 状态发生持久变化？
```

## 2. 核心矛盾与一句话运行模型

一句话运行模型：
```text
OutputPluginCallbacks 声明插件能处理哪些事件；
OutputPluginOptions 声明插件输出 payload 的基本属性；
LogicalDecodingContext 保存本次 decoding session 的启用状态；
START_REPLICATION options 决定 pgoutput/test_decoding 这次具体输出什么。
```
这里最容易混淆的是“声明能力”和“启用行为”。
`OutputPluginCallbacks` 是 capability surface。
如果 `_PG_output_plugin_init()` 填了 `stream_start_cb`、`stream_stop_cb` 等 callback，framework 才能认为插件具备 streaming 相关能力。
但具备能力不等于这次一定启用。
`pgoutput_startup()` 会根据客户端传入的 `streaming` option、请求的 `proto_version`、以及 `ctx->streaming` 当前值决定是否真的保留 streaming。
如果客户端没有请求 streaming，`pgoutput` 会把 `ctx->streaming` 关掉。
`two_phase` 也是类似的两阶段判断。
插件 callback 能力只是第一道门。
slot 是否允许 two-phase、客户端是否带 `two_phase` option、协议版本是否足够，都会参与最终状态。
`OutputPluginOptions` 则更窄。
在当前源码中，它只有两个字段：
```text
output_type
receive_rewrites
```
它不保存 `streaming`。
它也不保存 `two_phase`。
它更像 plugin 在 `startup_cb` 中告诉 framework：
```text
我输出的是文本 payload 还是二进制 payload；
我是否希望 ReorderBuffer 把 heap rewrite 期间的临时 heap change 也交给我。
```
客户端 options 再往后一层。
`START_REPLICATION SLOT ... LOGICAL ... (proto_version '4', publication_names 'pub', ...)` 的 options 进入 `ctx->output_plugin_options`。
framework 不理解大多数 plugin 私有参数。
它只保存这串 `DefElem` list。
真正解释这些参数的是 output plugin 的 `startup_cb`。
这就是本节 tension：
```text
plugin API 要足够通用，不能把所有插件参数固化进 core；
协议行为又必须足够早收敛，不能等发送半个事务后才发现 downstream 不兼容。
```

## 3. 核心文件分工与阅读顺序

阅读时不要从 `pgoutput.c` 顶部一路扫到底。
本节建议按状态推进顺序读。
| 顺序 | 文件 | 作用 |
| --- | --- | --- |
| 1 | `src/include/replication/output_plugin.h` | 定义 output plugin ABI：`OutputPluginOptions`、`OutputPluginCallbacks`、callback 类型。 |
| 2 | `src/include/replication/logical.h` | 定义 `LogicalDecodingContext`，这里保存 `callbacks`、`options`、`output_plugin_options`、`streaming`、`twophase`、`twophase_opt_given`。 |
| 3 | `src/backend/replication/repl_gram.y` | 解析 `START_REPLICATION SLOT slot LOGICAL lsn (options)`，把 options 变成 `DefElem` list。 |
| 4 | `src/backend/replication/walsender.c` | `StartLogicalReplication()` 创建 context，`XLogSendLogical()` 推进 WAL，`WalSndPrepareWrite()` / `WalSndWriteData()` 写 COPY BOTH。 |
| 5 | `src/backend/replication/logical/logical.c` | 加载插件、推断 capability、调用 `startup_cb`、包装 callback、处理写出状态和 cleanup。 |
| 6 | `src/backend/replication/pgoutput/pgoutput.c` | 内置 logical replication output plugin，解析 `proto_version`、`publication_names`、`binary`、`streaming`、`two_phase`、`origin`。 |
| 7 | `src/backend/replication/logical/proto.c` | 具体 logical replication wire message 的编码，包括 tuple binary/text、origin、streaming、two-phase 消息。 |
| 8 | `contrib/test_decoding/test_decoding.c` | 测试插件，用文本输出展示 option 解析、origin 过滤、rewrite、streaming 和 force-binary。 |
| 9 | `src/backend/replication/logical/worker.c` | 订阅端 apply worker 如何构造 `WalRcvStreamOptions`，成为远端 `pgoutput` 的客户端。 |
| 10 | `src/backend/replication/libpqwalreceiver/libpqwalreceiver.c` | 把 `WalRcvStreamOptions` 拼成真正的 `START_REPLICATION` 命令。 |
本节不会把 publication 的 row filter、column list 和 relation sync cache 展开成另一条主线。
它们只在解释 `pgoutput` 输出内容为什么变化时出现。

## 4. 关键状态：三层 options，不是一层开关

先把状态分层。
第一层是 plugin ABI。
`output_plugin.h` 里 `OutputPluginOptions` 当前只有：
```text
OutputPluginOutputType output_type;
bool receive_rewrites;
```
`output_type` 的枚举值是：
```text
OUTPUT_PLUGIN_BINARY_OUTPUT
OUTPUT_PLUGIN_TEXTUAL_OUTPUT
```
这个字段描述的是 plugin 写入 `ctx->out` 的 payload 性质。
它不是“网络协议是否是 COPY BOTH”。
walsender 逻辑复制总是在 replication protocol 的 COPY BOTH 中发送 `PqReplMsg_WALData`。
`output_type` 影响的是 SQL logical decoding 函数如何解释第三列 payload，以及文本函数能否接受这个 plugin。
`receive_rewrites` 则把选择传给 `ReorderBuffer`。
`CreateInitDecodingContext()` 和 `CreateDecodingContext()` 在调用 startup callback 后执行：
```text
ctx->reorder->output_rewrites = ctx->options.receive_rewrites;
```
后续 `reorderbuffer.c` 在看到 `relation->rd_rel->relrewrite` 时，如果 `output_rewrites` 为 false，就跳过 rewrite 临时 heap 的 change。
第二层是 `OutputPluginCallbacks`。
`LoadOutputPlugin()` 加载 shared library 的 `_PG_output_plugin_init` 符号。
插件在这个函数里填 `OutputPluginCallbacks`。
core 强制要求：
```text
begin_cb
change_cb
commit_cb
```
没有这些基本 callback，`LoadOutputPlugin()` 直接 ERROR。
其他 callback 表达扩展能力。
例如：
```text
filter_by_origin_cb
message_cb
truncate_cb
begin_prepare_cb / prepare_cb / commit_prepared_cb / rollback_prepared_cb
stream_start_cb / stream_stop_cb / stream_abort_cb / stream_commit_cb / stream_change_cb
stream_message_cb / stream_truncate_cb / stream_prepare_cb
```
这里的规则不是“填了一个 callback 就完整支持”。
`StartupDecodingContext()` 会用“至少填了一个相关 callback”来把 `ctx->streaming` 或 `ctx->twophase` 置为 true。
这样做是为了 later wrapper 能在真正进入该模式时检查缺失 callback，并报出明确错误。
第三层是本次 decoding session 的运行状态。
`LogicalDecodingContext` 中相关字段是：
```text
OutputPluginCallbacks callbacks;
OutputPluginOptions options;
List *output_plugin_options;
bool streaming;
bool twophase;
bool twophase_opt_given;
StringInfo out;
void *output_plugin_private;
```
`output_plugin_options` 是客户端传来的 `DefElem` list。
它不属于 core 固定语义。
`pgoutput` 会把它解释成 logical replication protocol 参数。
`test_decoding` 会把它解释成测试输出参数。
别的插件可以定义自己的参数。
所以一个参数的语义必须同时问三件事：
```text
这个参数是谁解析的？
它修改哪个状态字段？
它在哪条后续路径被读取？
```

## 5. 客户端 options 如何进入 server

逻辑复制启动命令在 `repl_gram.y` 里是：
```text
START_REPLICATION SLOT slot LOGICAL lsn plugin_options
```
`plugin_options` 是可空括号列表。
每个元素最终变成 `DefElem`。
`StartReplicationCmd` 在 `replnodes.h` 中保存：
```text
kind
slotname
startpoint
options
```
这说明 parsing 层只保留结构，不解释 plugin 参数。
`walsender.c` 的 `StartLogicalReplication()` 接到这个 command 后先做 core 级检查。
它调用：
```text
CheckLogicalDecodingRequirements(false)
ReplicationSlotAcquire(cmd->slotname, true, true)
CreateDecodingContext(cmd->startpoint, cmd->options, ...)
```
注意顺序。
`CreateDecodingContext()` 在发送 `CopyBothResponse` 之前执行。
所以 option 解析失败、协议版本不兼容、plugin 缺 callback 等错误，会以普通错误响应返回。
一旦 `CopyBothResponse` 发出，连接进入 COPY BOTH streaming。
之后的错误就会表现为 stream 中断或 walsender 退出，诊断成本更高。
订阅端 apply worker 也是同一条协议，只是命令由 server 内部拼出来。
`worker.c` 的 `run_apply_worker()` 先根据 subscription 创建 replication origin，再调用 `set_stream_options()`。
`set_stream_options()` 把本地 subscription 状态翻译成 `WalRcvStreamOptions`：
```text
proto_version
publication_names
binary
streaming_str
twophase
origin
```
然后 `libpqwalreceiver.c` 把它拼成命令：
```text
START_REPLICATION SLOT "slot" LOGICAL lsn
  (proto_version '4',
   streaming 'parallel',
   two_phase 'on',
   origin 'any',
   publication_names 'pub',
   binary 'true')
```
实际拼接有版本保护。
例如 publisher 版本低于 14 时不会发送 `binary`。
低于 15 时不会发送 `two_phase`。
低于 16 时不会发送 `origin`。
这不是安全装饰。
它保证老 publisher 不会因为看不懂新 option 而拒绝启动。

## 6. 创建 decoding context：先推断能力，再让 plugin 修正

`logical.c` 中的核心入口是 `StartupDecodingContext()`。
它做几件事：
```text
创建 "Logical decoding context" MemoryContext
加载 output plugin
创建 XLogReader
创建 ReorderBuffer
创建 SnapBuild
把 ReorderBuffer callback 指向 wrapper
根据 plugin callbacks 推断 ctx->streaming 和 ctx->twophase
保存 output_plugin_options
```
能力推断发生在 `startup_cb` 之前。
这很重要。
因为 startup callback 需要看到 framework 已经推断出的能力，再根据客户端 options 决定是否关闭或报错。
streaming 的初始判断大致是：
```text
ctx->streaming =
  stream_start_cb != NULL ||
  stream_stop_cb != NULL ||
  stream_abort_cb != NULL ||
  stream_commit_cb != NULL ||
  stream_change_cb != NULL ||
  stream_message_cb != NULL ||
  stream_truncate_cb != NULL;
```
two-phase 的初始判断大致是：
```text
ctx->twophase =
  begin_prepare_cb != NULL ||
  prepare_cb != NULL ||
  commit_prepared_cb != NULL ||
  rollback_prepared_cb != NULL ||
  stream_prepare_cb != NULL ||
  filter_prepare_cb != NULL;
```
这看起来宽松。
它故意宽松。
如果插件只注册了一个 streaming callback，core 不会在 load 阶段就拒绝。
但一旦真的进入 streaming wrapper，缺失 mandatory callback 会报：
```text
logical streaming requires a stream_start_cb callback
logical streaming requires a stream_change_cb callback
```
two-phase 也是一样。
`begin_prepare_cb_wrapper()`、`prepare_cb_wrapper()`、`commit_prepared_cb_wrapper()`、`rollback_prepared_cb_wrapper()` 会在实际需要时检查 mandatory callback。
这让 plugin 作者能在错误路径看到具体缺哪一个 callback。
`CreateInitDecodingContext()` 用于创建 slot 时初始化。
它调用 startup callback 时 `is_init = true`。
`CreateDecodingContext()` 用于已有 slot 开始 decode。
它调用 startup callback 时 `is_init = false`。
`pgoutput` 利用这个差异：
```text
is_init = true:
  只声明 binary output
  禁用 streaming 和 twophase
is_init = false:
  解析客户端 options
  检查 protocol
  按 option 启用或关闭 streaming / two_phase
```
这解释了一个边界：
```text
CREATE_REPLICATION_SLOT ... LOGICAL pgoutput
不会因为缺 publication_names 而失败；
START_REPLICATION 使用 pgoutput 时必须带 publication_names 和 proto_version。
```

## 7. `output_type`：文本/二进制是 plugin payload，不是 WALData 外壳

`pgoutput_startup()` 直接设置：
```text
opt->output_type = OUTPUT_PLUGIN_BINARY_OUTPUT;
```
`test_decoding` 默认设置：
```text
opt->output_type = OUTPUT_PLUGIN_TEXTUAL_OUTPUT;
```
但 `test_decoding` 还有 `force-binary` option。
如果用户传 `force-binary = true`，它会把 `opt->output_type` 改为 binary。
这不是让 `test_decoding` 变成 `pgoutput` 协议。
它只是声明这次 SQL decoding 结果不能被文本函数当作数据库编码文本验证。
SQL logical decoding 函数在 `logicalfuncs.c` 里会检查：
```text
if (!binary &&
    ctx->options.output_type != OUTPUT_PLUGIN_TEXTUAL_OUTPUT)
    ERROR
```
也就是说：
```text
pg_logical_slot_get_changes()
  期望 textual plugin
pg_logical_slot_get_binary_changes()
  可以接 binary payload
```
walsender 路径不同。
`WalSndPrepareWrite()` 总是往 `ctx->out` 先写：
```text
PqReplMsg_WALData
dataStart
walEnd
sendtime
```
plugin 随后写的是 logical payload。
`WalSndWriteData()` 再把整个 buffer 放进 `CopyData`。
所以 walsender 侧看到的外壳总是 replication protocol 的 WALData。
`output_type` 不会把 COPY BOTH 改成文本协议。
这个边界对客户端实现很关键。
客户端不能看到 `OUTPUT_PLUGIN_TEXTUAL_OUTPUT` 就按普通 Query response 读。
它仍然要按 replication protocol 读 CopyData，再把 WALData 中的 payload 交给插件协议解析器。

## 8. `pgoutput` options：协议版本先收敛，输出内容再变化

`pgoutput` 是内置 logical replication 协议插件。
它在 `_PG_output_plugin_init()` 中注册了普通事务、message、truncate、origin filter、two-phase、streaming 等 callback。
因此 `StartupDecodingContext()` 初始会认为：
```text
ctx->streaming = true
ctx->twophase = true
```
但真正是否启用，要看 `pgoutput_startup()`。
`parse_output_parameters()` 先把客户端 options 解析到 `PGOutputData`：
```text
proto_version
publication_names
binary
messages
streaming
two_phase
origin
```
其中 `proto_version` 和 `publication_names` 是必需项。
缺任意一个都会 ERROR。
其他选项有默认值：
```text
binary = false
streaming = off
messages = false
two_phase = false
origin = any
```
协议版本检查紧接着发生。
`logicalproto.h` 当前定义：
```text
LOGICALREP_PROTO_MIN_VERSION_NUM = 1
LOGICALREP_PROTO_VERSION_NUM = 1
LOGICALREP_PROTO_STREAM_VERSION_NUM = 2
LOGICALREP_PROTO_TWOPHASE_VERSION_NUM = 3
LOGICALREP_PROTO_STREAM_PARALLEL_VERSION_NUM = 4
LOGICALREP_PROTO_MAX_VERSION_NUM = 4
```
因此 `pgoutput_startup()` 对请求建立硬边界：
```text
proto_version > 4:
  server only supports protocol 4 or lower
proto_version < 1:
  server only supports protocol 1 or higher
streaming = on 需要 proto_version >= 2
two_phase = true 需要 proto_version >= 3
streaming = parallel 需要 proto_version >= 4
```
这里的顺序体现一个设计原则：
```text
先确认双方能解析消息类型，
再允许后续路径生成这些消息。
```
如果没有这个检查，publisher 可能发出 `LOGICAL_REP_MSG_STREAM_START`、`LOGICAL_REP_MSG_STREAM_PREPARE` 等下游不认识的消息。
失败就会从“启动时明确报错”变成“复制流中途断开”。
`binary` option 的语义更细。
它不是 `OutputPluginOptions.output_type`。
`pgoutput` 的 `output_type` 永远是 binary protocol。
`binary = true` 影响 `proto.c` 中 tuple column value 的编码方式。
`logicalrep_write_tuple()` 会检查：
```text
if (binary && type has typsend)
  发送 LOGICALREP_COLUMN_BINARY
else
  发送 LOGICALREP_COLUMN_TEXT
```
所以即使客户端请求 `binary true`，某些类型没有合适的 send function 时仍会落回文本列值。
客户端要按每个 column 的标记解析，而不是全局假设所有列都是 binary。
`messages` option 决定 logical decoding message 是否输出。
`pgoutput_message()` 开头就是：
```text
if (!data->messages)
  return;
```
这说明 `messages = false` 不是 core 不解码 logical message WAL。
它是 `pgoutput` 在 plugin 层不发送。

## 9. streaming：callback 能力、option 和 ReorderBuffer 压力共同作用

streaming 解决的是大事务不能等到 commit 才全部输出的问题。
没有 streaming 时，`ReorderBuffer` 必须把事务 change 留到 commit 之后再一次性交给 plugin。
事务很大时，会 spill 到磁盘，也会推迟下游可见进度。
streaming 打开后，`ReorderBuffer` 可以在事务未提交时把一个 chunk 交给 plugin。
但 streaming 不是单个布尔开关能解释的。
先有 callback 能力。
`pgoutput` 注册：
```text
stream_start_cb
stream_stop_cb
stream_abort_cb
stream_commit_cb
stream_change_cb
stream_message_cb
stream_truncate_cb
stream_prepare_cb
```
`test_decoding` 也注册了完整 streaming callbacks。
然后有客户端 option。
`pgoutput` 默认 `streaming = off`。
如果客户端传 `streaming 'on'`，需要 protocol >= 2。
如果传 `streaming 'parallel'`，需要 protocol >= 4。
`defGetStreamingMode()` 接受：
```text
0 / 1
false / true
off / on
parallel
```
如果值不是这些，会报：
```text
streaming requires a Boolean value or "parallel"
```
最后还有 runtime 压力。
即使 `ctx->streaming = true`，也不是每个事务都会拆成 stream chunks。
`ReorderBuffer` 会在内存压力、debug 选项或大事务状态下决定是否 stream。
这节课不展开 spill 策略，但要记住：
```text
streaming option 只是允许 streaming；
是否发生 streaming，还取决于事务大小、memory pressure 和 ReorderBuffer 判断。
```
`pgoutput_stream_start()` 生成 `STREAM START` 消息。
它还处理 origin：
```text
第一次 stream segment 可以发送 origin；
后续 segment 不重复发送 origin。
```
`pgoutput_stream_stop()` 生成 `STREAM STOP`。
`pgoutput_stream_commit()` 在事务最终 commit 时生成 `STREAM COMMIT`。
`pgoutput_stream_abort()` 在 streamed transaction abort 时通知下游丢弃已接收的块。
parallel streaming 还影响 abort 消息内容。
`pgoutput_stream_abort()` 中：
```text
write_abort_info = (data->streaming == LOGICALREP_STREAM_PARALLEL)
```
这说明 `streaming = parallel` 不是只改变 subscriber 执行模型。
它也改变 publisher 发送的某些协议字段。
订阅端 `worker.c` 会根据远端版本和 subscription 的 `streaming` 选项设置：
```text
streaming_str = "parallel"
streaming_str = "on"
streaming_str = NULL
```
这就是客户端 options 影响协议行为的完整闭环。

## 10. two-phase：slot 状态、客户端请求和 callback 必须同时成立

two-phase logical decoding 的目标是在 `PREPARE TRANSACTION` 时输出 prepared transaction，而不是等 `COMMIT PREPARED` 才当普通事务输出。
它需要更强的不变量。
prepared transaction 会跨越普通事务边界。
slot 必须知道从哪个 LSN 开始 two-phase decoding 是安全的。
`LogicalDecodingContext` 中有两个相关字段：
```text
twophase
twophase_opt_given
```
`twophase` 先由 callbacks 推断。
`twophase_opt_given` 表示本次 START_REPLICATION 的 plugin option 是否请求了 two-phase。
`pgoutput_startup()` 中：
```text
two_phase = true 且 proto_version < 3:
  ERROR
two_phase = true 且 ctx->twophase 为 false:
  ERROR
否则:
  ctx->twophase_opt_given = true
```
真正收敛在 `CreateDecodingContext()` 里：
```text
ctx->twophase &= (slot->data.two_phase || ctx->twophase_opt_given);
```
如果最终 `ctx->twophase` 为 true，而 slot 之前还没有启用 two-phase，代码会持久更新 slot：
```text
slot->data.two_phase = true
slot->data.two_phase_at = start_lsn
ReplicationSlotSave()
SnapBuildSetTwoPhaseAt(...)
```
这解释一个运维现象：
```text
带 two_phase 'on' 启动逻辑复制，
不只是本次连接的临时行为；
它可能把 slot 标记为 two_phase，并记录 two_phase_at。
```
创建 slot 时也可以带 `two_phase`。
`walsender.c` 的 `parseCreateReplSlotOptions()` 接受 `CREATE_REPLICATION_SLOT ... LOGICAL ... (two_phase true)`。
创建 slot 的路径会把该布尔值传给 `ReplicationSlotCreate()`。
但是创建 slot 时 `pgoutput_startup(is_init = true)` 会禁用 `ctx->twophase`。
这并不矛盾。
创建 slot 阶段只需要建立一致起点和 slot 元数据。
真正发送 prepared transaction 发生在后续 `START_REPLICATION`。
decode 路径在 `decode.c` 中通过 `FilterPrepare()` 决定。
如果 `ctx->twophase` 没启用，`FilterPrepare()` 直接返回 true，意思是：
```text
这个 PREPARE 在 prepare-time decoding 层面被过滤；
等 COMMIT PREPARED 时按普通事务处理。
```
如果插件提供 `filter_prepare_cb`，还可以按 gid 过滤。
`test_decoding` 中的 `pg_decode_filter_prepare()` 演示了这个能力：
```text
gid 包含 "_nodecode" 时过滤
```
一旦没有过滤，`DecodePrepare()`、`ReorderBufferPrepare()` 和 prepare callbacks 会把 transaction 在 PREPARE 时输出。
缺少 mandatory callback 时，wrapper 会报：
```text
logical replication at prepare time requires a prepare_cb callback
logical replication at prepare time requires a commit_prepared_cb callback
```
这类错误通常说明插件声明了部分 two-phase 能力，但 callback 集合不完整。

## 11. origin filtering：越早过滤，越少进入 ReorderBuffer

origin 过滤回答的是：
```text
带 replication origin 的 WAL change 是否要发给下游？
```
core 提供 `filter_by_origin_cb`。
`decode.c` 中 `FilterByOrigin()` 的规则很短：
```text
没有 callback:
  不过滤
有 callback:
  调用 filter_by_origin_cb_wrapper()
```
重要的是调用位置。
heap insert / update / delete / truncate / logical message 解码时，都会在 queue change 前检查 origin。
例如 heap insert 路径：
```text
确认 record 属于当前 database
  -> FilterByOrigin(ctx, XLogRecGetOrigin(r))
  -> 如果过滤，直接 return
  -> 否则分配 ReorderBufferChange 并 queue
```
commit 路径也会在 `DecodeTXNNeedSkip()` 中检查 origin。
如果事务应该跳过，`DecodeCommit()` 会 `ReorderBufferForget()` 事务内容，而不是把它交给 plugin。
这说明 origin filtering 不是最后输出前的字符串过滤。
它能避免无关 origin 的 change 进入后续 reorder 和输出路径。
`pgoutput` 的 option 是 `origin`。
当前解析只接受：
```text
origin 'none'
origin 'any'
```
`origin 'none'` 会设置：
```text
data->publish_no_origin = true
```
`pgoutput_origin_filter()` 的规则是：
```text
如果 publish_no_origin 为 true，
并且 origin_id != InvalidReplOriginId，
返回 true 表示过滤。
```
`origin 'any'` 则不过滤。
`test_decoding` 用 `only-local` option 表达类似行为。
`pg_decode_filter()` 中：
```text
only_local && origin_id != InvalidReplOriginId
  -> return true
```
这里有一个对称但容易混淆的点。
过滤 origin 和发送 origin message 不是同一件事。
`pgoutput` 在事务有 origin 且未过滤时，可能调用 `send_repl_origin()`。
`send_repl_origin()` 通过 `replorigin_by_oid(origin_id, true, &origin)` 找 origin 名字。
找到才调用 `logicalrep_write_origin()`。
如果找不到，当前代码选择不发送 origin message。
所以：
```text
filter_by_origin_cb 决定这个 change / txn 是否进入输出；
logicalrep_write_origin() 决定输出中是否携带 origin 元数据。
```

## 12. `receive_rewrites`：插件请求看到 rewrite 临时 heap

`receive_rewrites` 是 `OutputPluginOptions` 中另一个字段。
它不是客户端通用协议 option。
当前两个内置示例里：
```text
pgoutput:
  不显式设置 receive_rewrites，palloc0 后默认 false
test_decoding:
  默认 false
  include-rewrites option 可以解析到 opt->receive_rewrites
```
这个字段在 startup callback 后传给 `ReorderBuffer`：
```text
ctx->reorder->output_rewrites = ctx->options.receive_rewrites;
```
后续 `reorderbuffer.c` 在输出 change 前检查 relation：
```text
if (relation->rd_rel->relrewrite && !rb->output_rewrites)
  goto change_done;
```
这条路径主要服务 DDL rewrite 期间的临时 heap。
普通逻辑复制不希望把 rewrite 内部过程暴露成用户 DML。
测试插件或诊断插件可能希望看到这些 change，帮助验证 rewrite 相关逻辑。
所以 `receive_rewrites` 的 mental model 是：
```text
plugin 告诉 ReorderBuffer：
我是否愿意接收 relrewrite 临时 relation 上的 change。
```
它不是 publication 过滤。
它也不是 row filter。
它发生在更靠近 ReorderBuffer 输出 change 的层次。

## 13. 输出写入：plugin 只能在 callback 窗口写

`OutputPluginPrepareWrite()` 和 `OutputPluginWrite()` 是 plugin 输出的边界。
`OutputPluginPrepareWrite()` 会检查：
```text
ctx->accept_writes
```
如果不在允许写的 callback 中调用，会报：
```text
writes are only accepted in commit, begin and change callbacks
```
`OutputPluginWrite()` 会检查是否先调用过 prepare：
```text
OutputPluginPrepareWrite needs to be called before OutputPluginWrite
```
这些检查不是 API 洁癖。
framework 需要知道当前输出对应哪个 LSN、哪个 xid，以及是否是事务结束位置。
wrapper 在调用 plugin callback 前设置：
```text
accept_writes = true
write_xid
write_location
end_xact
```
例如 `change_cb_wrapper()` 把 `write_location` 设为 change 的 LSN。
`commit_cb_wrapper()` 把它设为事务结束 LSN。
`filter_by_origin_cb_wrapper()` 和 `filter_prepare_cb_wrapper()` 则设置：
```text
accept_writes = false
```
所以过滤 callback 不能顺手输出诊断消息到 replication stream。
这保证了输出流仍然是事务事件驱动，而不是任意 callback 都能插入消息。
walsender writer 再把这些输出包装成 replication protocol：
```text
WalSndPrepareWrite()
  -> resetStringInfo(ctx->out)
  -> 写 PqReplMsg_WALData header
plugin callback
  -> logicalrep_write_*() 或 appendStringInfo()
WalSndWriteData()
  -> 填 sendtime
  -> pq_putmessage_noblock(PqMsg_CopyData, ctx->out->data, ctx->out->len)
  -> 必要时 ProcessPendingWrites()
```
这也解释为什么 `pgoutput` 的每个输出函数都围绕：
```text
OutputPluginPrepareWrite()
logicalrep_write_...()
OutputPluginWrite()
```

## 14. 主流程源码 walkthrough

这一节把上面状态串成一条真实路径。
假设客户端发送：
```sql
START_REPLICATION SLOT s LOGICAL 0/0
  (proto_version '4',
   publication_names 'pub',
   binary 'true',
   streaming 'parallel',
   two_phase 'on',
   origin 'none',
   messages 'true');
```
第一步，`repl_gram.y` 把命令解析成 `StartReplicationCmd`。
`cmd->kind = REPLICATION_KIND_LOGICAL`。
`cmd->slotname = "s"`。
`cmd->startpoint` 是 LSN。
`cmd->options` 是一串 `DefElem`。
第二步，`walsender.c` 的 `StartLogicalReplication()` 获取 slot。
它在发送 COPY BOTH 之前调用 `CreateDecodingContext(cmd->startpoint, cmd->options, ...)`。
第三步，`StartupDecodingContext()` 创建 decoding context。
它加载 slot 中记录的 plugin。
如果 slot 是 `pgoutput`，`LoadOutputPlugin()` 找到 `_PG_output_plugin_init()`。
`pgoutput` 填满 callbacks。
framework 因此初步得到：
```text
ctx->streaming = true
ctx->twophase = true
```
第四步，`CreateDecodingContext()` 调用 `startup_cb_wrapper()`。
wrapper 设置错误上下文：
```text
slot "s", output plugin "pgoutput", in the startup callback
```
然后调用 `pgoutput_startup(ctx, &ctx->options, false)`。
第五步，`pgoutput_startup()` 设置：
```text
opt->output_type = OUTPUT_PLUGIN_BINARY_OUTPUT
```
随后解析 `ctx->output_plugin_options`。
它确认 `proto_version` 在 1 到 4。
它确认 `publication_names` 存在。
它看到 `streaming = parallel`，要求 protocol >= 4。
它看到 `two_phase = true`，要求 protocol >= 3。
它看到 `origin = none`，设置 `publish_no_origin = true`。
第六步，回到 `CreateDecodingContext()`。
它执行：
```text
ctx->twophase &= (slot->data.two_phase || ctx->twophase_opt_given)
```
如果 slot 尚未标记 two-phase，但本次请求通过检查，就把 slot 的 `two_phase` 和 `two_phase_at` 持久化。
然后：
```text
ctx->reorder->output_rewrites = ctx->options.receive_rewrites
```
对 `pgoutput` 来说通常是 false。
第七步，`StartLogicalReplication()` 发送 `CopyBothResponse`。
它设置 walsender 状态到 catchup。
然后调用 `WalSndLoop(XLogSendLogical)`。
第八步，`XLogSendLogical()` 读 WAL record。
读到一条 heap change 时，`LogicalDecodingProcessRecord()` 进入 `decode.c`。
`DecodeInsert()` 先检查 database，再调用 `FilterByOrigin()`。
因为 `origin = none`，如果 record 有非本地 origin，就直接 return。
这个 change 不进入 `ReorderBuffer`。
第九步，对未过滤的 change，`ReorderBufferQueueChange()` 保存。
如果事务很大，且 streaming 被允许，`ReorderBuffer` 后续可能调用 stream wrappers。
`stream_start_cb_wrapper()` 设置 `accept_writes = true` 和当前 LSN，然后调用 `pgoutput_stream_start()`。
`pgoutput_stream_start()` 写 `STREAM START`，必要时写 origin message，并把 `data->in_streaming = true`。
第十步，普通 change 输出进入 `pgoutput_change()`。
如果 `data->in_streaming`，它会在 change 消息中携带 xid。
如果 `binary = true`，`logicalrep_write_tuple()` 对有 send function 的类型发送 binary column。
没有 send function 的类型仍发送 text column。
第十一步，commit 到来。
非 streaming 事务走 `pgoutput_commit_txn()`。
streamed 事务走 `pgoutput_stream_commit()`。
two-phase prepared transaction 如果在 PREPARE 时输出，则走 prepare 相关 callbacks。
第十二步，`WalSndWriteData()` 把 plugin 生成的 buffer 作为 `CopyData` 发给客户端。
客户端回报 flush LSN 后，slot 的 confirmed_flush 才能推进。

## 15. 生命周期 / ownership / cleanup

`LogicalDecodingContext` 的内存归属很清楚。
`StartupDecodingContext()` 创建：
```text
"Logical decoding context"
```
`ctx`、`ctx->out`、plugin private data 的子 context、ReorderBuffer、SnapBuild、XLogReader 都挂在这个生命周期附近。
`FreeDecodingContext()` 负责收尾：
```text
shutdown_cb_wrapper(ctx)
ReorderBufferFree(ctx->reorder)
FreeSnapshotBuilder(ctx->snapshot_builder)
XLogReaderFree(ctx->reader)
MemoryContextDelete(ctx->context)
```
plugin 可以创建自己的 child context。
`pgoutput` 创建：
```text
logical replication output context
logical replication cache context
logical replication publication list context
```
它还在 `ctx->context` 上注册 reset callback，用于清理 `RelationSyncCache`。
shutdown 时不需要逐个释放这些 child context。
`MemoryContextDelete(ctx->context)` 会统一回收。
`test_decoding` 创建 `"text conversion context"`。
`pg_decode_shutdown()` 中显式 `MemoryContextDelete(data->context)`。
这两种风格都有效。
关键是 plugin private state 不能假设跨 decoding session 存活。
每次 `START_REPLICATION` 都会创建新的 `LogicalDecodingContext`。
但 process-local static 状态可能跨 session 存活。
`pgoutput` 的 publication syscache callback 就用 static bool 防止重复注册。
错误路径下，`startup_cb_wrapper()` 和其他 wrappers 会把错误上下文压入 `error_context_stack`。
如果 plugin 在 callback 内 ERROR，用户能看到：
```text
slot "...", output plugin "...", in the ... callback, associated LSN ...
```
MemoryContext 和 ResourceOwner 的外层 abort 机制会清掉本次 command 资源。
如果已经进入 walsender streaming，进程退出路径还会释放 slot。
本节重点是：
```text
plugin 自己的私有内存可以交给 ctx->context；
外部语义状态如 slot->data.two_phase 是持久 slot 元数据，不会因 session cleanup 自动回滚。
```

## 16. 正确性机制层次

这一节的 correctness 不是一个机制保证的。
它由几层边界组合出来。
第一层是 ABI 边界。
`OutputPluginCallbacks` 规定 core 何时能调用 plugin。
wrapper 负责设置 `accept_writes`、LSN、xid、error context。
plugin 不能绕过这些状态随便写。
第二层是协议版本边界。
`pgoutput_startup()` 在 stream 开始前拒绝不兼容组合。
这样下游不会在半个事务中看到自己不能解析的消息类型。
第三层是 slot 持久状态。
two-phase 不是普通 session flag。
一旦本次启动决定启用并持久化 `slot->data.two_phase`，slot 后续 decoding 要遵守 `two_phase_at`。
第四层是 origin 和 database 过滤。
`decode.c` 在 queue change 前检查 database 和 origin。
这减少了无关 change 进入 reorder buffer 的机会。
但 commit 路径仍要处理 invalidation 语义，不能简单把所有 skip 都当 abort。
第五层是 output write ordering。
`OutputPluginPrepareWrite()` 到 `OutputPluginWrite()` 之间的 buffer 对应一个确定 LSN。
`last_write` 决定 walsender 是否把该 LSN 暴露为可确认的位置。
streaming 中间 chunk 的 LSN 可以帮助反馈进度，但不能单独证明整个事务可确认。
第六层是 subscriber 兼容性。
`worker.c` 和 `libpqwalreceiver.c` 根据远端 server version 决定是否发送新 option。
这层不是 publisher 的 correctness，但它让跨版本逻辑复制拓扑可运行。

## 17. 错误路径 / 异常路径 / fallback

第一类错误发生在 plugin 加载。
如果 shared library 没有 `_PG_output_plugin_init`：
```text
output plugins have to declare the _PG_output_plugin_init symbol
```
如果缺 `begin_cb`、`change_cb` 或 `commit_cb`：
```text
output plugins have to register a begin callback
```
这类错误在 context 创建期间发生，不会进入 COPY BOTH。
第二类错误发生在 option 解析。
`pgoutput` 遇到重复 option 会报：
```text
conflicting or redundant options
```
缺必需项会报：
```text
option "proto_version" missing
option "publication_names" missing
```
未知 option 走：
```text
unrecognized pgoutput option: ...
```
`test_decoding` 的未知 option 则是：
```text
option "..." = "..." is unknown
```
这说明未知参数由 plugin 报错，错误文本也由 plugin 决定。
第三类错误是协议能力不匹配。
例如：
```text
streaming 'on' + proto_version '1'
  -> requested proto_version=1 does not support streaming, need 2 or higher
two_phase 'on' + proto_version '2'
  -> requested proto_version=2 does not support two-phase commit, need 3 or higher
streaming 'parallel' + proto_version '3'
  -> requested proto_version=3 does not support parallel streaming, need 4 or higher
```
第四类错误是 callback 集合不完整。
framework 初始只看到“至少有一个相关 callback”。
真正进入 wrapper 时才检查 mandatory callback。
这类错误通常会带 callback 名称。
它比 load 阶段一次性判断更利于插件开发定位。
第五类是 output API 使用错误。
在 `filter_by_origin_cb` 中调用 `OutputPluginPrepareWrite()` 会失败。
连续调用 `OutputPluginWrite()` 而没有 prepare 也会失败。
这些错误说明 plugin 破坏了 callback 生命周期。
第六类是 SQL decoding 的 textual/binary 不匹配。
使用 `pg_logical_slot_get_changes()` 读取 `pgoutput` 或 `force-binary` 的 `test_decoding` 时，会因为文本函数期望 textual data 而 ERROR。
使用 binary SQL 函数则绕过这个 textual 检查。
第七类是 fallback。
`binary = true` 时 column 没有 `typsend`，`proto.c` 会发送 text column。
`origin` id 找不到名字时，`send_repl_origin()` 当前选择不发送 origin message。
`two_phase` 未启用时，prepared transaction 在 prepare-time decoding 层面被过滤，后续在 `COMMIT PREPARED` 作为普通事务处理。
这些 fallback 都不是无声 bug。
它们是为了在信息不足或能力不足时保持协议可解析、事务语义可完成。

## 18. 成本、资源与跨模块传播

options 不是零成本。
`binary = true` 会改变每个 column 的输出路径。
对有 `typsend` 的类型，`OidSendFunctionCall()` 生成 binary datum。
对没有 `typsend` 的类型，仍走 `OidOutputFunctionCall()`。
所以成本随：
```text
行数
列数
类型输出函数成本
TOAST datum 大小
publication column list
```
一起扩张。
`messages = true` 会让 logical message record 进入输出流。
如果 workload 大量使用 `pg_logical_emit_message_*`，这会增加 network payload 和 downstream apply 解析成本。
`streaming = on` 可以减少大事务在 publisher 上的积压，但会增加协议消息数量。
一个大事务可能拆成多段：
```text
STREAM START
若干 INSERT/UPDATE/DELETE/MESSAGE/TRUNCATE
STREAM STOP
...
STREAM COMMIT 或 STREAM ABORT
```
这提高了下游提前接收和释放 publisher 内存压力的能力。
但也增加了下游 spool、parallel apply、abort 丢弃和 restart 处理的复杂性。
`streaming = parallel` 还影响 subscriber worker 调度。
`worker.c` 中 leader apply worker 可能把 streaming block 分发给 parallel apply worker。
这里的瓶颈会从 publisher ReorderBuffer 内存转移到 subscriber 的文件集、worker 可用性和 apply 冲突处理。
`two_phase = true` 让 prepared transaction 更早进入下游。
它提高 prepared transaction 语义保真度。
但 slot 需要保存 `two_phase_at`，subscriber 也需要处理 prepare、commit prepared、rollback prepared。
如果订阅端初始同步还没完成，`worker.c` 会让 subscription 的 two_phase 处于 PENDING，直到 tablesync ready 才启动时带 `two_phase`。
这体现了跨模块传播：
```text
plugin option
  -> slot persistent metadata
  -> SnapBuild two_phase_at
  -> publisher protocol messages
  -> subscriber pg_subscription 状态
  -> apply worker transaction handling
```
`origin = none` 可以在 decode 阶段减少无关 origin change 的 reorder 成本。
但它也影响业务语义。
在多主、级联复制或复制回环场景中，过滤 origin 可能避免回环，也可能让某些外部来源的数据不再传递。
这个判断不能只看 publisher。
必须结合拓扑。

## 19. 观测与诊断入口

最直接的 runtime truth 是：
```text
同一个 slot，用不同 START_REPLICATION options，
会在 startup 阶段改变 ctx->streaming / ctx->twophase / PGOutputData，
随后改变输出消息类型和过滤范围。
```
能直接看到的状态包括：
```text
pg_replication_slots.two_phase
pg_replication_slots.two_phase_at
pg_stat_replication.state
pg_stat_replication.sent_lsn / write_lsn / flush_lsn / replay_lsn
server log 中 startup callback 错误上下文
subscriber 日志中的 two_phase 状态 DEBUG1
```
能从输出推断的状态包括：
```text
是否出现 S/E/c/A/p 等 streaming 消息
是否出现 b/P/K/r 等 two-phase 消息
是否出现 O origin 消息
column value 是 LOGICALREP_COLUMN_BINARY 还是 LOGICALREP_COLUMN_TEXT
logical message 是否被发送
```
几乎不能直接从 SQL 视图看到的是：
```text
ctx->streaming 当前 session 布尔值
ctx->twophase_opt_given
PGOutputData->binary
PGOutputData->publish_no_origin
ReorderBuffer 当前是否因为 origin filter 跳过了某条 change
```
这些通常要靠 gdb、临时日志或协议抓包判断。
建议断点：
```text
StartLogicalReplication
CreateDecodingContext
startup_cb_wrapper
pgoutput_startup
parse_output_parameters
FilterByOrigin
pgoutput_stream_start
pgoutput_stream_commit
WalSndWriteData
```
诊断 `pgoutput` option 失败时，先看错误发生阶段。
如果还没进入 COPY BOTH，通常是 startup option 或 callback 能力问题。
如果已经进入 streaming 后中断，优先看 walsender log、subscriber apply worker error context 和最后一个已发送 LSN。
诊断 binary/text 问题时，不要只看 `output_type`。
要区分：
```text
plugin payload 是 textual 还是 binary；
pgoutput 每个 column 是 text representation 还是 binary representation；
replication protocol 外壳是否仍然是 WALData。
```
诊断 origin 问题时，也要区分：
```text
change 被 FilterByOrigin 提前过滤；
change 被发送但没有 O message；
subscriber apply 时更新自己的 replication origin；
subscription origin 选项阻止 publisher 发送某些来源。
```

## 20. 常见误区

误区一：`OutputPluginOptions` 包含 streaming 和 two_phase。
当前源码不是。
`OutputPluginOptions` 只有 `output_type` 和 `receive_rewrites`。
streaming 和 two-phase 在 `LogicalDecodingContext` 中，由 callbacks、slot 状态和 startup options 共同收敛。
误区二：`binary = true` 等价于 `OUTPUT_PLUGIN_BINARY_OUTPUT`。
不等价。
`pgoutput` 的 `output_type` 始终是 binary protocol。
`binary = true` 控制 tuple column value 是否优先用 type send function 编码。
误区三：注册了 streaming callback 就一定会 streaming。
不一定。
callback 只是能力。
`pgoutput` 默认关闭 streaming。
即使启用，也要等 ReorderBuffer 判断事务需要 streaming。
误区四：`origin = none` 只是下游 apply 过滤。
不是。
publisher decode 阶段就会通过 `filter_by_origin_cb` 提前跳过有 origin 的 WAL change。
误区五：two-phase 是一次连接的临时 option。
不完全是。
`START_REPLICATION` 带 `two_phase 'on'` 可能把 slot 持久标记为 two-phase，并设置 `two_phase_at`。
误区六：SQL decoding 和 walsender 使用同一种输出约束。
不是。
SQL textual 函数会拒绝 binary output plugin。
walsender 只把 plugin payload 包进 replication protocol，不因为 `output_type` 改变外层 COPY BOTH。
误区七：unknown option 是 core 统一报错。
不是。
START_REPLICATION parser 只构造 `DefElem`。
大多数 option 名称由 plugin 自己解析和报错。

## 21. 课堂实验

### 实验一：确认 startup 阶段收敛状态
准备一个 logical slot：
```sql
SELECT pg_create_logical_replication_slot('s_pgoutput', 'pgoutput');
```
用 gdb 在 walsender 上断：
```text
break CreateDecodingContext
break startup_cb_wrapper
break pgoutput_startup
break parse_output_parameters
```
从客户端执行：
```sql
START_REPLICATION SLOT s_pgoutput LOGICAL 0/0
  (proto_version '4', publication_names 'pub', streaming 'parallel',
   binary 'true', two_phase 'on', origin 'none');
```
观察：
```text
ctx->output_plugin_options
ctx->streaming
ctx->twophase
ctx->twophase_opt_given
((PGOutputData *) ctx->output_plugin_private)->binary
((PGOutputData *) ctx->output_plugin_private)->publish_no_origin
```
目标是把“option 字符串”映射成“context 状态”。
### 实验二：比较 textual 和 binary SQL 函数边界
创建 `test_decoding` slot：
```sql
SELECT pg_create_logical_replication_slot('s_test', 'test_decoding');
```
先用文本函数读取：
```sql
SELECT * FROM pg_logical_slot_get_changes('s_test', NULL, NULL);
```
再传：
```sql
SELECT * FROM pg_logical_slot_get_changes(
  's_test', NULL, NULL,
  'force-binary', 'true'
);
```
预期文本函数会因为 plugin 声明 binary output 而失败。
然后改用 binary 函数验证边界。
目标是区分 `output_type` 与 `pgoutput binary option`。
### 实验三：观察 origin 过滤位置
在 `decode.c` 下断点：
```text
break FilterByOrigin
break DecodeInsert
break DecodeTXNNeedSkip
```
用 `origin 'none'` 和 `origin 'any'` 分别启动 `pgoutput`。
构造带 replication origin 的 change。
观察有 origin 的 record 是否在 queue change 前被跳过。
目标是确认 origin filtering 发生在 publisher decode/reorder 边界，而不是 subscriber apply 末端。
### 实验四：触发协议版本错误
启动时传：
```sql
START_REPLICATION SLOT s_pgoutput LOGICAL 0/0
  (proto_version '1', publication_names 'pub', streaming 'on');
```
预期在进入 COPY BOTH 前报错。
再传：
```sql
START_REPLICATION SLOT s_pgoutput LOGICAL 0/0
  (proto_version '3', publication_names 'pub', streaming 'parallel');
```
预期需要 protocol 4。
目标是理解 `pgoutput_startup()` 为什么必须早检查协议版本。

## 22. 讨论题

1. 为什么 `OutputPluginOptions` 不保存 `streaming` 和 `two_phase`，而是把它们放在 `LogicalDecodingContext`？
2. 如果插件只注册了 `stream_change_cb`，但没有 `stream_start_cb`，为什么 PostgreSQL 不在 load 阶段立刻拒绝？
3. `origin = none` 为什么应该尽量早过滤，而不是等 `pgoutput_change()` 里再判断？
4. `binary = true` 时仍可能发送 text column，这对客户端解码器有什么要求？
5. 为什么 `two_phase 'on'` 可能需要持久修改 slot 元数据，而 `streaming 'on'` 不需要？
6. 如果 SQL textual 函数允许读取 binary output plugin，会破坏哪些假设？
7. `receive_rewrites` 为什么属于 plugin startup option，而不是 publication option？
8. apply worker 为什么要根据 publisher server version 决定是否发送 `origin`、`two_phase`、`binary` 等 options？

## 23. 本节小结

本节的核心链路是：
```text
START_REPLICATION options
  -> StartReplicationCmd.options
  -> LogicalDecodingContext.output_plugin_options
  -> plugin startup_cb 解析
  -> ctx->options / ctx->streaming / ctx->twophase / plugin private data
  -> decode、reorder、pgoutput、writer 共同决定最终输出
```
核心状态分三层。
`OutputPluginCallbacks` 是能力声明。
`OutputPluginOptions` 是 plugin 对 output payload 和 rewrite change 的声明。
`LogicalDecodingContext` 是本次 session 的实际运行状态。
不要把这三层压成一个“options 开关”。
`output_type` 区分 textual/binary payload。
它影响 SQL decoding 函数的接受范围。
它不改变 walsender COPY BOTH 外壳。
`pgoutput` 的 `binary` option 则是 logical replication protocol 内部 tuple column 编码选择。
streaming 和 two-phase 都需要 callback 能力。
但 streaming 是否启用取决于客户端 option 和协议版本。
two-phase 还依赖 slot 持久状态，并可能通过 `START_REPLICATION` 改写 slot 的 `two_phase` / `two_phase_at`。
origin filtering 通过 `filter_by_origin_cb` 尽早发生。
它减少无关 WAL change 进入 ReorderBuffer。
发送 origin message 是另一件事，不要混在一起。
错误路径的可诊断性来自两个设计：
```text
startup 阶段尽早检查 protocol/options；
callback wrapper 给每个 plugin callback 加 error context。
```
可迁移规律：
```text
扩展型协议不能只靠一个 options map 解决兼容性；
它需要把“能力声明、会话启用、持久状态、消息版本、输出 ownership”
分成不同层次，并在跨层边界尽早失败。
```
下一节可以沿着这个输出边界继续往下看：当 `pgoutput` 已经决定发送某个 relation 的 change 后，publication、row filter、column list 和 relation schema cache 如何共同决定具体行内容。
