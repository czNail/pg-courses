# PostgreSQL replication origin 与 logical replication 回环防护
## 课程定位
前置知识：已经理解 logical decoding 如何从 WAL record 生成 `ReorderBufferChange`，`pgoutput` 如何生成 logical replication protocol，apply worker 如何在 subscriber 端执行远端事务。
本节唯一主问题：
```text
logical replication 如何用 replication origin 区分变更来源，
为什么双向复制、级联逻辑复制和手工写入仍然需要显式处理回环与冲突策略？
```
核心矛盾：
```text
系统必须在 WAL、decoding、output plugin 和 apply worker 之间传递“这条变更来自哪里”；
但 PostgreSQL core 不能替用户推断拓扑意图，也不能自动决定冲突应该保留哪一边。
```
学完后应能判断：
```text
pg_replication_origin catalog 保存的是 origin 身份，不是 per-row provenance。
session origin 决定当前 backend 写 WAL 时标记哪个来源。
origin LSN/time progress 用于 apply 重启和 crash recovery，不是冲突解决协议。
pgoutput 的 origin=none 只过滤“带任意 origin 的变更”，不是按名称精确过滤。
apply worker 会设置订阅自己的 origin，并在提交成功后推进 progress。
双向复制、级联复制和手工修复仍然需要明确拓扑和冲突策略。
```
本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e90741e12a3715909e1b2d71ff9344`。
本节主线：
```text
remote commit
  -> subscriber apply worker setup session origin
  -> 本地 WAL record 带 origin id
  -> logical decoding 读出 origin id
  -> output plugin filter_by_origin_cb 决定是否跳过
  -> pgoutput 可发送 ORIGIN message
  -> apply 成功提交后推进 remote_lsn/local_lsn
```
本节不把 PostgreSQL 内置 logical replication 扩展成完整多主复制方案。
origin 是来源标记和过滤机制，不是全局冲突裁决器。
## 1. 本节在总主线中的位置
前几节已经覆盖：
```text
WAL record
  -> logical/decode.c 解析
  -> ReorderBuffer 按事务重组
  -> pgoutput 输出 protocol message
  -> worker.c apply 到 subscriber
```
replication origin 是这条链路上的横切状态。
它回答的问题不是：
```text
这条 SQL 是否应该复制？
```
而是：
```text
这条 WAL change 是否由某个 replication origin 产生？
```
典型场景：
```text
node A 用户提交事务 T。
node B 的 apply worker 重放 T。
node B 写本地 WAL 时带上“这是订阅 A 的 origin”。
如果 node B 再作为 publisher，pgoutput 能看到这些 change 已经带 origin。
```
没有 origin，B 无法区分：
```text
B 本地用户写入
B 从 A apply 出来的写入
```
双向复制中，这个区分是防回环的最低条件。
```text
A -> B 的事务如果被 B 当成本地新事务再发回 A，
A 会收到自己原来的修改。
```
PostgreSQL 当前内置订阅只暴露两个 origin 输出选项：
```text
origin = any:
  发布端不因为 origin 过滤。
origin = none:
  发布端只发送没有 origin 的变更。
```
当前没有内置：
```text
origin = include 'node_a'
origin = exclude 'node_b'
origin = keep original provenance through all cascades
```
所以本节的关键结论先放在前面：
```text
origin 能标记来源、过滤带来源的变更、恢复 apply 进度；
origin 不能表达业务拓扑的全部意图，也不能自动解决冲突。
```
## 2. 核心矛盾与一句话运行模型
一句话运行模型：
```text
apply worker 在 subscriber 上 setup 一个 session replication origin；
当前事务提交时，replorigin_xact_state 的 origin、origin_lsn、origin_timestamp 进入 WAL/commit metadata；
decoding 通过 XLogRecGetOrigin() 取得 origin id，并调用 output plugin 的 filter_by_origin_cb；
pgoutput 的 origin=none 会跳过所有带 origin 的变更；
提交成功后，apply worker 推进该 origin 的 remote_lsn/local_lsn progress。
```
这里的 tension 是：
```text
loop prevention 希望尽早丢弃会回流的变更；
crash safety 又要求只有真正成功提交的 apply 事务才能推进远端进度。
```
PostgreSQL 把状态拆成五层：
| 层次 | 状态 | 位置 | 作用 |
| --- | --- | --- | --- |
| identity | `roident` / `roname` | `pg_replication_origin` | 短 ID 与外部名字映射。 |
| session ownership | `session_replication_state` | backend pointer + shared slot | 当前 backend 使用哪个 origin。 |
| transaction mark | `replorigin_xact_state` | backend global | 当前事务写 WAL 时带哪个 origin 和远端 LSN/time。 |
| progress | `remote_lsn` / `local_lsn` | shared memory + checkpoint + redo | 已成功重放到远端哪个位置。 |
| output filter | `origin_id` | WAL record / decoding context | output plugin 是否跳过该来源。 |
这五层不能互相替代。
`pg_replication_origin` 中有 catalog 行，不代表它有 active progress。
`pg_replication_origin_status.remote_lsn` 推进，不代表没有业务冲突。
`origin=none` 能阻止带 origin 的 change 输出，不代表拓扑中所有路径无环。
`ORIGIN` protocol message 能告诉 downstream 事务带 origin 名字，不代表当前 apply worker 会把原始 origin 名称重新用于本地 WAL 标记。
## 3. 核心文件分工与阅读顺序
| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/catalog/pg_replication_origin.h` | `pg_replication_origin` catalog 的 `roident` / `roname`，唯一索引和 syscache。 |
| 2 | `src/include/replication/origin.h` | `InvalidReplOriginId`、`DoNotReplicateId`、`ReplOriginXactState`、origin API。 |
| 3 | `src/backend/replication/logical/origin.c` | origin 创建/删除、shared progress、checkpoint/redo、session setup/reset/advance、SQL wrapper。 |
| 4 | `src/backend/replication/logical/worker.c` | apply worker 如何创建并设置 origin、提交时设置 origin LSN/time、错误时清理。 |
| 5 | `src/backend/replication/logical/decode.c` | `XLogRecGetOrigin()`、`FilterByOrigin()`、commit/change 如何携带 origin。 |
| 6 | `src/backend/replication/pgoutput/pgoutput.c` | `filter_by_origin_cb`、`origin` option、ORIGIN message 发送。 |
| 7 | `src/backend/replication/logical/proto.c` | `logicalrep_write_origin()` / `logicalrep_read_origin()` 的 wire format。 |
| 8 | `src/backend/catalog/system_views.sql` | `pg_replication_origin_status` 只是 `pg_show_replication_origin_status()` 的 view。 |
推荐跟读：
```text
pg_replication_origin.h
  -> origin.h
  -> origin.c 的 ReplicationState 和 replorigin_xact_state
  -> worker.c 的 run_apply_worker()
  -> decode.c 的 FilterByOrigin()
  -> pgoutput.c 的 pgoutput_origin_filter()
  -> proto.c 的 ORIGIN message
```
如果要补齐 WAL 写入细节，再看：
```text
xact.c:
  commit record 如何写 origin_lsn/origin_timestamp。
xloginsert.c:
  XLOG_INCLUDE_ORIGIN 如何把 origin id 放进 WAL record。
```
本节会引用这两个相邻文件的事实，但主分析集中在用户要求的八个文件。
## 4. 关键数据结构与状态
### 4.1 `pg_replication_origin`: identity catalog
`src/include/catalog/pg_replication_origin.h` 定义 shared catalog：
| 字段 | 语义 |
| --- | --- |
| `roident` | 本地短 ID，会写入 WAL；注释强调它需要适合 `uint16`。 |
| `roname` | 外部 free-form origin name。 |
它有两个唯一索引：
```text
ReplicationOriginIdentIndex:
  roident -> row
ReplicationOriginNameIndex:
  roname -> row
```
并生成两个 syscache：
```text
REPLORIGIDENT
REPLORIGNAME
```
`origin.c` 的 `replorigin_by_name()` 和 `replorigin_by_oid()` 就走这两个 syscache。
关键边界：
```text
pg_replication_origin 保存身份映射。
它不保存 remote_lsn/local_lsn。
它不保存行级来源。
它不保存冲突策略。
```
`replorigin_create()` 有几个值得注意的实现点：
```text
限制 origin name 长度，避免 catalog 需要 TOAST。
用 DirtySnapshot 扫 catalog。
找第一个未使用的 16-bit roident。
插入 row 后 CommandCounterIncrement()。
```
这个创建过程是低频路径。
热路径不是 catalog update，而是 shared memory progress 更新。
### 4.2 `origin.h`: 特殊 ID 与事务状态
`src/include/replication/origin.h` 定义：
```text
InvalidReplOriginId = 0
DoNotReplicateId = PG_UINT16_MAX
```
`InvalidReplOriginId` 表示没有 origin。
普通本地用户写入通常处于这个状态。
`DoNotReplicateId` 是特殊内部值。
`replorigin_advance()` 对它直接返回，事务提交判断也排除它。
`ReplOriginXactState` 包含：
```text
origin
origin_lsn
origin_timestamp
```
`origin.c` 中的 backend global 初始为：
```text
origin = InvalidReplOriginId
origin_lsn = InvalidXLogRecPtr
origin_timestamp = 0
```
这不是 shared memory。
它描述的是当前 backend 接下来写 WAL 的事务应该带哪个来源。
apply worker setup origin 后，`replorigin_xact_state.origin` 会持续是该订阅的 origin。
每个远端事务提交前，worker 再设置本事务对应的 `origin_lsn` 和 `origin_timestamp`。
### 4.3 `ReplicationState`: active progress slot
`origin.c` 的 shared memory slot 是 `ReplicationState`：
| 字段 | 语义 |
| --- | --- |
| `roident` | slot 对应哪个 origin。 |
| `remote_lsn` | 已成功重放到远端哪个 LSN。 |
| `local_lsn` | 本地证明该重放已提交的 WAL LSN。 |
| `acquired_by` | 首次 acquisition 这个 origin 的 PID，0 表示没有记录。 |
| `refcount` | 当前多少进程正在使用这个 slot。 |
| `origin_cv` | drop/reset 等待使用者释放时的 condition variable。 |
| `lock` | 保护 `remote_lsn` 和 `local_lsn` 的 per-origin LWLock。 |
`max_active_replication_origins` 决定 shared memory slot 数量。
如果它为 0，很多操作会报错；status SRF 会返回 0 行。
锁分层：
```text
ReplicationOriginLock:
  保护 slot 数组、创建/删除、session acquisition。
state->lock:
  保护单个 origin 的 remote_lsn/local_lsn。
```
`session_replication_state` 是 backend-local 指针。
它缓存当前 backend 使用的 shared slot，避免每次 `replorigin_session_advance()` 都扫描数组。
### 4.4 progress 的 durable 语义
`remote_lsn` 和 `local_lsn` 要一起理解：
```text
remote_lsn:
  远端事务位置。
local_lsn:
  本地提交 WAL 位置。
```
checkpoint 时 `CheckPointReplicationOrigin()` 会：
```text
遍历 active origin slots。
读取 remote_lsn/local_lsn。
XLogFlush(local_lsn)。
把 roident 和 remote_lsn 写入 pg_logical/replorigin_checkpoint。
写 CRC 并 durable_rename。
```
这保证 checkpoint 文件不会承诺一个本地 WAL 尚未持久化的 remote progress。
启动时 `StartupReplicationOrigin()` 读取 checkpoint 文件。
redo 时 `replorigin_redo()` 处理 `XLOG_REPLORIGIN_SET` 和 `XLOG_REPLORIGIN_DROP`。
这套机制解决的是 crash/restart 后 apply progress 恢复。
它不证明所有表状态无冲突。
## 5. 主流程源码 walkthrough
### 5.1 apply worker 设置 origin
入口在 `worker.c` 的 `run_apply_worker()`：
```text
ReplicationOriginNameForLogicalRep(MySubscription->oid, InvalidOid, originname)
StartTransactionCommand()
originid = replorigin_by_name(originname, true)
if (!OidIsValid(originid))
  originid = replorigin_create(originname)
replorigin_session_setup(originid, 0)
replorigin_xact_state.origin = originid
origin_startpos = replorigin_session_get_progress(false)
CommitTransactionCommand()
set_stream_options(&options, slotname, &origin_startpos)
walrcv_startstreaming(...)
start_apply(origin_startpos)
```
普通 apply worker origin name：
```text
pg_<subscription oid>
```
tablesync worker name：
```text
pg_<subscription oid>_<relation oid>
```
这一段同时改变三类状态：
```text
catalog:
  如果 origin 不存在，创建 pg_replication_origin row。
shared memory:
  replorigin_session_setup() acquisition 一个 ReplicationState slot。
backend local:
  replorigin_xact_state.origin = originid。
```
`origin_startpos` 来自当前 origin progress。
它会成为 walreceiver logical streaming startpoint。
所以 apply worker 重启不是从零开始，而是从该 origin 已成功 apply 的远端位置继续。
`set_stream_options()` 还会设置：
```text
options->proto.logical.origin = pstrdup(MySubscription->origin)
```
这把订阅 catalog 中的 `suborigin` 传给上游 `pgoutput`。
### 5.2 `replorigin_session_setup()` 的 ownership
`origin.c` 的 `replorigin_session_setup(node, acquired_by)` 是 session origin 的核心。
普通路径传 `acquired_by = 0`，语义是：
```text
该 origin slot 必须当前没有被其他进程 acquisition。
成功后 acquired_by = MyProcPid。
refcount++。
session_replication_state 指向该 slot。
```
函数第一次执行时注册：
```text
on_shmem_exit(ReplicationOriginExitCleanup, 0)
```
这样进程退出时可以释放 session origin。
它还禁止同一个 backend setup 两个 session origin：
```text
session_replication_state != NULL
  -> ERROR "cannot setup replication origin when one is already setup"
```
parallel apply 是特殊场景。
`acquired_by != 0` 时，其他进程可以复用由 leader PID acquisition 的 origin slot。
源码注释明确要求这些进程维护 commit order。
这不是任意多进程共享。
### 5.3 订阅的 origin option 进入 pgoutput
`worker.c` 把 `MySubscription->origin` 放进 `WalRcvStreamOptions`。
上游 walsender 创建 `pgoutput` context 后，`pgoutput.c` 的 `parse_output_parameters()` 解析：
```text
origin = none:
  data->publish_no_origin = true
origin = any:
  data->publish_no_origin = false
```
默认是 `any`。
`pg_subscription.h` 中 `suborigin` 的默认值也是 `LOGICALREP_ORIGIN_ANY`。
`subscriptioncmds.c` 的解析代码只接受 `none` 和 `any`。
注释说明它现在用 string 类型，是为了未来可能扩展成按用户指定 origin name 过滤。
当前 `pgoutput` 实现没有这个能力。
### 5.4 decoding 中的 origin filter
`pgoutput.c` 在 `_PG_output_plugin_init()` 注册：
```text
cb->filter_by_origin_cb = pgoutput_origin_filter
```
`decode.c` 封装：
```text
FilterByOrigin(ctx, origin_id)
  -> 如果没有 callback，返回 false。
  -> 否则调用 filter_by_origin_cb_wrapper(ctx, origin_id)。
```
`pgoutput_origin_filter()` 的核心逻辑：
```text
if (data->publish_no_origin && origin_id != InvalidReplOriginId)
  return true;
return false;
```
返回 true 表示跳过。
`decode.c` 在多个早期位置调用它：
```text
logicalmsg_decode()
DecodeCommit() -> DecodeTXNNeedSkip()
DecodeInsert()
DecodeUpdate()
DecodeDelete()
DecodeTruncate()
```
heap change 入 reorder buffer 前会先检查 origin：
```text
if (FilterByOrigin(ctx, XLogRecGetOrigin(r)))
  return;
change->origin_id = XLogRecGetOrigin(r);
ReorderBufferQueueChange(...);
```
这意味着 `origin=none` 不只是最后少发网络消息。
它能让带 origin 的 change 不进入后续重组和 output 流程。
事务提交路径还会处理 origin LSN/time：
```text
origin_id = XLogRecGetOrigin(buf->record)
if commit record 有 XACT_XINFO_HAS_ORIGIN:
  origin_lsn = parsed->origin_lsn
  commit_time = parsed->origin_timestamp
ReorderBufferCommit(..., commit_time, origin_id, origin_lsn)
```
record-level origin id 用于过滤。
commit-level origin LSN/time 用于事务级来源信息和 apply progress。
### 5.5 apply commit 设置 origin LSN/time
`worker.c` 的 `apply_handle_commit_internal()` 在本地提交前设置：
```text
replorigin_xact_state.origin_lsn = commit_data->end_lsn;
replorigin_xact_state.origin_timestamp = commit_data->committime;
CommitTransactionCommand();
store_flush_position(commit_data->end_lsn, XactLastCommitEnd);
```
prepare、commit prepared、rollback prepared 路径也会在相应动作前设置 `origin_lsn` 和 `origin_timestamp`。
相邻的 `xact.c` 提交路径会在 `replorigin_xact_state.origin != InvalidReplOriginId` 时把 origin LSN/time 写入 commit record，并在提交成功后：
```text
replorigin_session_advance(replorigin_xact_state.origin_lsn, XactLastRecEnd)
```
这一步只能发生在本地提交成功之后。
如果提前推进 remote progress，apply 失败后发布端可能不再发送该事务。
### 5.6 WAL record 如何带 origin id
相邻源码中，`xact.c` 和 heap WAL 路径会设置：
```text
XLogSetRecordFlags(XLOG_INCLUDE_ORIGIN)
```
`xloginsert.c` 插入 WAL record 时检查：
```text
curinsert_flags 包含 XLOG_INCLUDE_ORIGIN
且 replorigin_xact_state.origin != InvalidReplOriginId
```
满足时，它把当前 origin id 写进 WAL record。
所以 decoding 可以在 `decode.c` 中用：
```text
XLogRecGetOrigin(r)
```
读出 origin id。
不要混淆两个概念：
```text
WAL record origin id:
  每条记录的来源标记，用于过滤。
commit record origin_lsn/origin_timestamp:
  事务在原始节点上的位置和时间，用于 progress 和冲突诊断。
```
### 5.7 pgoutput 发送 ORIGIN message
`pgoutput_send_begin()` 在发送第一个 change 前发送 BEGIN。
如果事务带 origin：
```text
send_replication_origin = txn->origin_id != InvalidReplOriginId
logicalrep_write_begin(ctx->out, txn)
send_repl_origin(ctx, txn->origin_id, txn->origin_lsn, send_replication_origin)
```
`send_repl_origin()` 会用 `replorigin_by_oid(origin_id, true, &origin)` 查 name。
如果查到：
```text
OutputPluginWrite(ctx, false)
OutputPluginPrepareWrite(ctx, true)
logicalrep_write_origin(ctx->out, origin, origin_lsn)
```
如果查不到，当前实现选择不发送 ORIGIN message。
源码注释列出过其他可能方案：报错、发送 unknown origin 等，但当前没有采用。
streaming transaction 中，`pgoutput_stream_start()` 只在第一个 stream 发送 origin，后续 stream 不重复发送。
`proto.c` 中 ORIGIN message 格式是：
```text
logicalrep_write_origin()
  -> LOGICAL_REP_MSG_ORIGIN
  -> origin_lsn
  -> origin string
logicalrep_read_origin()
  -> 读 origin_lsn
  -> 读 origin string
```
### 5.8 apply worker 对 ORIGIN message 的当前处理
`worker.c` 的 dispatcher 遇到 `LOGICAL_REP_MSG_ORIGIN` 时调用：
```text
apply_handle_origin(s)
```
当前实现只校验顺序：
```text
ORIGIN message 只能在 streaming transaction 内，
或者 remote transaction 内并且在实际写入前出现。
```
函数注释仍然写着：
```text
TODO, support tracking of multiple origins
```
它没有调用 `logicalrep_read_origin()` 并把 origin name 映射成当前事务的 `replorigin_xact_state.origin`。
这个事实决定了能力边界：
```text
pgoutput 能发送 ORIGIN message。
protocol 能表达 ORIGIN message。
当前内置 apply worker 主要用“本订阅自己的 session origin”标记重放产生的本地 WAL。
它不是完整保存每条变更原始 provenance 的级联系统。
```
### 5.9 progress 查询与推进
`replorigin_session_advance(remote_commit, local_commit)` 只更新当前 session origin：
```text
if local_lsn < local_commit:
  local_lsn = local_commit
if remote_lsn < remote_commit:
  remote_lsn = remote_commit
```
它只推进，不回退。
SQL `pg_replication_origin_advance(name, lsn)` 调用通用 `replorigin_advance()`，并允许 `go_backward = true`。
源码注释说它适合设置初始状态，不适合 replay 时使用，因为它没有一个已提交的 `local_commit` 可以传入。
查询进度：
```text
pg_replication_origin_session_progress(flush)
pg_replication_origin_progress(name, flush)
```
如果 `flush = true` 且 `local_lsn` 有效，会先 `XLogFlush(local_lsn)`。
`system_views.sql` 定义：
```text
CREATE VIEW pg_replication_origin_status AS
  SELECT *
  FROM pg_show_replication_origin_status();
```
`pg_show_replication_origin_status()` 遍历 active shared memory slots，返回：
```text
local_id
external_id
remote_lsn
local_lsn
```
因此 view 展示的是 active progress，不是 catalog 全量列表。
## 6. 生命周期 / ownership / cleanup
### 6.1 创建
创建 origin identity 的路径：
```text
SQL:
  pg_replication_origin_create(name)
apply worker:
  run_apply_worker()
    -> replorigin_by_name(originname, true)
    -> replorigin_create(originname)
```
`pg_replication_origin_create()` 会拒绝保留名字：
```text
any
none
pg_* 前缀
```
其中 `any` 和 `none` 是 logical replication option。
`pg_*` 是内部逻辑复制 origin 名空间。
### 6.2 持有
`replorigin_session_setup()` 持有 active slot。
普通 session：
```text
acquired_by = MyProcPid
refcount = 1
session_replication_state = slot
```
SQL wrapper `pg_replication_origin_session_setup(name, pid)` 还会：
```text
replorigin_xact_state.origin = origin
```
这意味着手工 session setup origin 后，后续事务会持续带这个 origin，直到 reset 或 session exit。
### 6.3 释放
显式释放：
```text
pg_replication_origin_session_reset()
  -> replorigin_session_reset()
  -> replorigin_xact_clear(true)
```
进程退出释放：
```text
ReplicationOriginExitCleanup()
  -> replorigin_session_reset_internal()
```
reset internal 会：
```text
如果 acquired_by == MyProcPid:
  acquired_by = 0
refcount--
session_replication_state = NULL
ConditionVariableBroadcast(origin_cv)
```
如果 leader origin 仍被其他 parallel apply worker 共享，显式 reset 会报错。
这是为了避免第一个 acquisition 进程提前释放共享 origin。
### 6.4 transaction-level cleanup
`replorigin_xact_clear(clear_origin)`：
```text
origin_lsn = InvalidXLogRecPtr
origin_timestamp = 0
if clear_origin:
  origin = InvalidReplOriginId
```
apply worker 有两个关键保护：
```text
start_apply() 的 PG_CATCH:
  replorigin_xact_clear(true)
InitializeLogRepWorker() 注册 before_shmem_exit:
  on_exit_clear_xact_state()
    -> replorigin_xact_clear(true)
```
`worker.c` 注释强调，即使设置 origin state 后只打印一条 LOG/DEBUG，也可能先处理 shutdown signal。
所以 exit callback 必须提前注册。
如果失败路径没有清理 transaction origin state，abort 或 shutdown 可能错误推进 origin progress。
后果是发布端不再重发未成功落地的事务。
### 6.5 drop
`replorigin_drop_by_name()` 会：
```text
打开 pg_replication_origin
LockSharedObject(... AccessExclusiveLock)
replorigin_state_clear(roident, nowait)
CatalogTupleDelete()
CommandCounterIncrement()
```
`replorigin_state_clear()` 如果看到 `refcount > 0`：
```text
nowait = true:
  ERROR，报告被 PID 或其他进程占用。
nowait = false:
  ConditionVariableSleep(WAIT_EVENT_REPLICATION_ORIGIN_DROP)
  被唤醒后重试。
```
清理 active slot 前会先写：
```text
XLOG_REPLORIGIN_DROP
```
redo 时 `replorigin_redo()` 会清空对应 shared slot。
## 7. 正确性机制层次
### 7.1 identity 与 progress 分离
PostgreSQL 没有在每次 apply commit 时更新 `pg_replication_origin` catalog。
它使用：
```text
catalog:
  保存 identity。
shared memory:
  保存 active progress。
checkpoint file:
  周期性持久化 progress。
WAL redo:
  恢复 SET/DROP 和 commit origin progress。
```
这样避免把高频 apply commit path 变成 catalog update path。
### 7.2 `local_lsn` 保护异步提交
origin progress 的 crash-safety 依赖：
```text
remote_lsn:
  远端已重放位置。
local_lsn:
  本地 commit WAL 位置。
```
checkpoint 必须先 `XLogFlush(local_lsn)`，再把 `remote_lsn` 写入 checkpoint file。
否则 checkpoint 可能承诺一个本地尚未持久化的远端进度。
### 7.3 filter 只是 filter
`FilterByOrigin()` 只问：
```text
output plugin 是否对这个 origin_id 感兴趣？
```
它不检查：
```text
publication relation membership
row filter
tuple 是否冲突
权限
目标端唯一键
业务 merge 策略
```
这些由 `pgoutput` 后续 relation/publication 逻辑、apply worker 和用户策略处理。
### 7.4 conflict detection 只使用 origin 作为证据
`worker.c` 在 UPDATE/DELETE 时可能调用：
```text
GetTupleTransactionInfo(..., &conflicttuple.origin, &conflicttuple.ts)
```
如果本地 tuple 的最后修改 origin 与当前 apply origin 不同：
```text
conflicttuple.origin != replorigin_xact_state.origin
```
会报告：
```text
CT_UPDATE_ORIGIN_DIFFERS
CT_DELETE_ORIGIN_DIFFERS
```
这说明 origin 能帮助诊断“不同来源修改过同一行”。
但报告冲突不等于自动解决冲突。
是否覆盖、忽略、报错、禁用订阅或人工处理，是 apply 逻辑和配置策略决定的。
## 8. 错误路径 / 异常路径 / fallback
### 8.1 `max_active_replication_origins = 0`
`replorigin_check_prerequisites()` 在需要 origin 功能时会报错：
```text
cannot query or manipulate replication origin when "max_active_replication_origins" is 0
```
`pg_show_replication_origin_status()` 用较宽松检查。
注释说明如果 slot 数为 0，就返回 0 行。
所以 status view 为空不一定表示没有 catalog origin。
### 8.2 recovery 中禁止 manipulation
如果 `RecoveryInProgress()` 且调用路径不允许 recovery：
```text
cannot manipulate replication origins during recovery
```
查询 progress 可以允许 recovery。
创建、删除、session setup 等 manipulation 不允许。
redo 自己通过 `replorigin_redo()` 更新状态。
### 8.3 slot 被占用
`replorigin_session_setup()` 和 `replorigin_advance()` 会拒绝被其他进程占用的 origin slot。
常见错误语义：
```text
replication origin with ID ... is already active for PID ...
replication origin with ID ... is already active in another process
```
这不是 catalog duplicate。
这是 active shared memory slot ownership 冲突。
### 8.4 ORIGIN message 找不到名字
`send_repl_origin()` 如果 `replorigin_by_oid(..., true, &origin)` 失败，当前实现不发送 ORIGIN message。
因此：
```text
没有 ORIGIN message
```
不必然等于：
```text
事务没有 origin id
```
也可能是 origin id 无法映射回 name。
### 8.5 apply ERROR 不能推进 progress
`start_apply()` 的 `PG_CATCH` 首先：
```text
replorigin_xact_clear(true)
```
注释说明这是为了防止 apply 失败时推进 origin progress。
如果失败事务的 progress 被推进，发布端会认为该远端 LSN 已经成功消费，之后不会重发。
这是 transaction loss 风险。
### 8.6 initial copy 的盲区
`subscriptioncmds.c` 对 `copy_data = true` 且 `origin = none` 有 warning 检查。
原因：
```text
initial table synchronization 复制的是已有 heap 数据。
已有 heap 数据没有可供 pgoutput origin filter 使用的 WAL record origin id。
```
所以 `origin=none` 对增量 WAL 流有效。
它不能证明 initial copy 没有复制来自其他 origin 的历史数据。
## 9. 成本、资源与跨模块传播
origin 成本主要在四处：
```text
WAL insertion:
  需要时把 origin id 写入 WAL record。
logical decoding:
  每条相关 record 读取 XLogRecGetOrigin()，并可能调用 filter callback。
apply commit:
  成功提交后更新 session origin remote_lsn/local_lsn。
checkpoint:
  遍历 max_active_replication_origins 个 slots，flush local_lsn，写 checkpoint file。
```
扩张变量：
| 变量 | 成本传播 |
| --- | --- |
| WAL record 数 | origin check 和 filter callback 次数。 |
| 事务数 | session progress advance 次数。 |
| active origin 数 | checkpoint 和 status view 遍历成本。 |
| `max_active_replication_origins` | shared memory 和 scan 上界。 |
| filtered change 比例 | `origin=none` 可减少 reorder/output/network 工作。 |
跨模块传播：
| 模块 | origin 角色 |
| --- | --- |
| catalog | name/id 映射。 |
| WAL insertion | 写 record-level origin id。 |
| transaction commit | 写 origin LSN/time，推进 commit timestamp。 |
| logical decoding | 按 origin 过滤，传递 origin 到 reorder buffer。 |
| pgoutput | 解析 `origin` option，发送 ORIGIN message。 |
| apply worker | setup session origin，提交后 advance，错误时清理。 |
| system view | 展示 active progress。 |
诊断“origin=none 没生效”时，不要只看一个点。
至少要确认：
```text
pg_subscription.suborigin 是否为 none。
walreceiver 是否把 origin=none 传给上游。
pgoutput 是否注册 filter_by_origin_cb。
上游 WAL record 是否真的带 origin id。
问题数据是否来自 initial copy_data。
是否存在手工写入没有 setup origin。
```
## 10. 观测与诊断入口
直接看 identity：
```sql
SELECT roident, roname
FROM pg_replication_origin
ORDER BY roident;
```
直接看 active progress：
```sql
SELECT local_id, external_id, remote_lsn, local_lsn
FROM pg_replication_origin_status
ORDER BY local_id;
```
看订阅请求的 origin option：
```sql
SELECT subname, suborigin
FROM pg_subscription
ORDER BY subname;
```
看当前 session 是否 setup origin：
```sql
SELECT pg_replication_origin_session_is_setup();
```
手工 session origin 实验入口：
```sql
SELECT pg_replication_origin_create('lab_origin');
SELECT pg_replication_origin_session_setup('lab_origin');
SELECT pg_replication_origin_xact_setup('0/1000000', clock_timestamp());
INSERT INTO t VALUES (1, 'marked');
COMMIT;
SELECT pg_replication_origin_session_progress(false);
SELECT pg_replication_origin_session_reset();
```
不要在生产 session 随手做这个实验。
session origin 会持续影响后续事务，直到 reset 或 session exit。
直接可见：
```text
origin catalog identity。
active progress remote_lsn/local_lsn。
subscription 的 suborigin。
apply worker error context 中的 origin name。
```
只能推断：
```text
某条已有 heap tuple 的完整来源历史。
initial copy_data 中每一行是否来自其他 origin。
origin=none 是否足以覆盖整张拓扑的回环路径。
```
常见 wait/log：
```text
WAIT_EVENT_REPLICATION_ORIGIN_DROP:
  drop 等待 active origin 释放。
processing remote data for replication origin "...":
  apply worker error context。
CT_UPDATE_ORIGIN_DIFFERS / CT_DELETE_ORIGIN_DIFFERS:
  不同 origin 修改冲突被报告。
```
## 11. loop prevention 的能力边界
### 11.1 能力
origin 能稳定提供：
```text
来源标记:
  本地 WAL record 可带 origin id。
输出过滤:
  pgoutput origin=none 可跳过所有带 origin 的变更。
进度恢复:
  apply worker 用 remote_lsn/local_lsn 重启后继续。
冲突证据:
  apply conflict detection 可比较 tuple 修改 origin 与当前 apply origin。
```
这足以支持简单防回环模式：
```text
A <-> B 双向复制
两边 subscription 都设置 origin=none
每边只输出本地用户写入
apply 产生的 WAL 不再发回对方
```
### 11.2 边界
origin 不能做：
```text
按 origin name 精确 include/exclude。
自动推断任意拓扑是否无环。
自动合并同时更新。
自动处理唯一键冲突。
自动区分 initial copy_data 中的历史来源。
```
`origin=none` 的实际含义是：
```text
不发送带任何 origin 的变更。
```
不是：
```text
只过滤从我自己回来的变更。
```
### 11.3 双向复制 caveat
双向复制常见误解：
```text
设置 origin=none 就等于完整多主复制。
```
实际只解决：
```text
apply 产生的变更不再被同一节点作为本地变更发出去。
```
它不解决：
```text
两边同时插入同一个 primary key。
两边同时 UPDATE 同一行。
一边 DELETE，另一边 UPDATE。
序列值在两边独立增长。
外键和唯一约束在两边观察顺序不同。
手工修复没有标记 origin。
```
所以双向复制还需要：
```text
单写或分片写入规则。
ID/sequence 分配策略。
冲突监控和处理策略。
必要时 retain_dead_tuples / track_commit_timestamp。
订阅错误后的 disable/skip/runbook。
```
origin 是必要来源信号，不是多主一致性协议。
### 11.4 级联复制 caveat
级联复制有两种不同目标。
不想转发上游数据：
```text
A -> B
B -> C 设置 origin=none
C 只收到 B 本地写入。
```
想继续转发上游数据：
```text
A -> B
B -> C 设置 origin=any
C 能收到 A 经 B apply 的数据。
```
第二种情况下，回环风险也被带进拓扑。
如果还有 C -> A 或 C -> B，就必须靠额外拓扑约束处理。
当前 apply worker 不保存多 origin provenance。
`apply_handle_origin()` 的 TODO 是这条边界的源码证据。
### 11.5 手工写入 caveat
普通 SQL 写入默认：
```text
replorigin_xact_state.origin = InvalidReplOriginId
```
因此它会被 `origin=none` 订阅发送。
维护修复时要特别小心：
```text
B 上手工修复一行。
B->A subscription 是 origin=none。
如果修复没有 setup origin，它会被当作 B 本地写入发回 A。
```
可选策略：
```text
维护窗口禁用相关 subscription。
显式 pg_replication_origin_session_setup() 标记修复来源。
让修复只发生在拓扑单写源。
应用层记录修复批次并过滤。
```
PostgreSQL 不会替你选择。
## 12. 常见误区
误区一：`pg_replication_origin` 保存进度。
实际：
```text
catalog 保存 identity；
progress 在 shared memory、checkpoint file 和 WAL redo 中维护。
```
误区二：`origin=none` 是“不要复制我自己的 origin”。
实际：
```text
它过滤所有带 origin 的变更。
```
误区三：ORIGIN message 会让 apply worker 保留原始 provenance。
实际：
```text
proto.c 能读写 ORIGIN message；
worker.c 当前只校验顺序，没有把它映射成事务 origin。
```
误区四：origin conflict 会自动解决。
实际：
```text
origin 可用于报告 UPDATE_ORIGIN_DIFFERS / DELETE_ORIGIN_DIFFERS；
冲突裁决仍在 apply/配置/应用层。
```
误区五：status view 为空等于没有 origin。
实际：
```text
pg_replication_origin_status 展示 active progress；
catalog identity 可能仍然存在。
```
误区六：手工 setup origin 只影响下一条语句。
实际：
```text
session origin 持续到 reset 或 session exit。
```
## 13. 课堂实验
### 实验一：源码跟读生命周期
断点：
```text
worker.c:
  run_apply_worker
  apply_handle_commit_internal
  start_apply
  on_exit_clear_xact_state
origin.c:
  replorigin_session_setup
  replorigin_session_advance
  replorigin_session_reset_internal
  pg_show_replication_origin_status
decode.c:
  FilterByOrigin
  DecodeCommit
pgoutput.c:
  parse_output_parameters
  pgoutput_origin_filter
  send_repl_origin
```
观察：
```text
originname 如何从 subscription oid 生成？
setup 后 acquired_by/refcount 如何变化？
commit 前 origin_lsn/origin_timestamp 何时设置？
advance 时 remote_lsn/local_lsn 分别是什么？
ERROR 路径是否先清理 replorigin_xact_state？
```
画两张图：
```text
backend-local:
  replorigin_xact_state 生命周期。
shared memory:
  ReplicationState roident/refcount/remote_lsn/local_lsn 生命周期。
```
### 实验二：`origin=none` 的过滤边界
拓扑：
```text
A -> B
B -> C
```
先让 `B -> C` 使用 `origin=none`。
在 A 写入：
```sql
INSERT INTO t VALUES (1, 'from A');
```
预期：
```text
B 收到。
C 不收到。
```
原因：
```text
B apply 产生的 WAL 带 origin；
B->C 的 pgoutput origin=none 过滤。
```
在 B 手工写入：
```sql
INSERT INTO t VALUES (2, 'manual on B');
```
预期：
```text
C 收到。
```
原因：
```text
普通手工写入没有 origin。
```
把 `B -> C` 改成 `origin=any` 后重复 A 写入。
观察 C 是否收到 A 经 B 级联的数据。
### 实验三：手工 session origin 风险
在测试库执行：
```sql
SELECT pg_replication_origin_create('lab_manual_origin');
SELECT pg_replication_origin_session_setup('lab_manual_origin');
SELECT pg_replication_origin_session_is_setup();
BEGIN;
SELECT pg_replication_origin_xact_setup('0/1000000', clock_timestamp());
INSERT INTO t VALUES (100, 'marked manual write');
COMMIT;
SELECT *
FROM pg_replication_origin_status
WHERE external_id = 'lab_manual_origin';
SELECT pg_replication_origin_session_reset();
```
问题：
```text
session setup 后状态是否持续？
commit 后 progress 是否出现？
如果忘记 reset，下一事务会发生什么？
```
### 实验四：initial copy 盲区
阅读 `subscriptioncmds.c` 中 `copy_data=true` 且 `origin=none` 的 warning 逻辑。
讨论：
```text
为什么 initial sync 无法像 WAL 增量那样过滤 origin？
为什么 warning 不是无条件 ERROR？
空表场景下 warning 的实际风险是什么？
```
## 14. 讨论题
1. 为什么 `pg_replication_origin` 只保存 `roident` 和 `roname`，不直接保存 `remote_lsn`？
2. `replorigin_session_setup()` 为什么要禁止普通 backend 同时 setup 两个 origin？
3. 为什么只看 `remote_lsn` 不足以判断 apply progress 已 durable？
4. `pgoutput_origin_filter()` 为什么当前只能表达 `any` / `none`？
5. 当前 `apply_handle_origin()` 不读取 origin string，这对级联复制意味着什么？
6. UPDATE/DELETE 冲突中，origin 能提供什么证据？哪些决策仍必须由策略处理？
7. `start_apply()` 的错误路径为什么必须先 `replorigin_xact_clear(true)`？
8. 为什么 `origin=none` 不能证明 initial `copy_data=true` 没有复制来自其他 origin 的历史数据？
## 15. 本节小结
本节核心链路：
```text
apply worker setup session origin
  -> 当前事务 WAL 携带 origin id
  -> commit record 保存 origin_lsn/origin_timestamp
  -> decode.c 用 XLogRecGetOrigin() 调 filter_by_origin_cb
  -> pgoutput origin=none 跳过带 origin 的变更
  -> apply commit 成功后推进 remote_lsn/local_lsn
```
核心状态：
```text
pg_replication_origin:
  持久 identity。
replorigin_xact_state:
  backend-local 当前事务来源标记。
ReplicationState:
  shared memory active progress。
ORIGIN message:
  output protocol 中的来源提示。
```
cleanup 不变量：
```text
session origin 必须 reset 或由 exit cleanup 释放。
ERROR/shutdown 必须清理 transaction origin state。
只有本地提交成功后才能推进 origin progress。
```
能力边界：
```text
origin 可以标记来源、过滤输出、恢复进度、辅助冲突诊断。
origin 不能自动证明拓扑无环，不能自动处理双写冲突，也不能追溯 initial copy_data 中每行历史来源。
```
可迁移规律：
```text
来源标记不是冲突协议。
一个稳定的 provenance bit 可以让 hot path 过滤掉明显不该转发的变更，
但只要业务意图不能从这个 bit 唯一推出，
拓扑设计、手工写入和冲突裁决就必须显式配置、观测和演练。
```
