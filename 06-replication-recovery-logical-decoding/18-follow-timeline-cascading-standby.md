# PostgreSQL 级联 standby timeline 追随与 PITR 历史边界

## 课程定位
前置知识：已经理解 physical replication 中 walsender 发送 WAL、walreceiver 接收 WAL、startup process replay WAL。
也已经知道 promotion 会结束 recovery、选择新 timeline、写出 timeline history file。
本节唯一主问题：
```text
级联 standby 如何发现上游 timeline 变化并继续追随正确历史，
为什么 recovery_target_timeline 会影响 PITR 和故障转移后的可恢复范围？
```
核心矛盾：
```text
standby 必须沿一条确定历史顺序 replay WAL，不能把分叉后的两条历史混在一起；
但级联拓扑里的上游可能被 promote，新的 timeline 先表现为旧 timeline 到头和新 history file 出现，
并不是一个所有下游同时可见的全局状态切换。
```
PostgreSQL 把这个问题拆成三层：
```text
walreceiver:
  从上游获取 WAL bytes、primaryTLI、next_tli 和 timeline history file。
startup process:
  用 recovery_target_timeline、recoveryTargetTLI、expectedTLEs 和 replayLSN 决定本机允许进入哪条历史。
walsender:
  支持 START_REPLICATION ... TIMELINE，
  对 historic timeline 只发送到 switchpoint，然后告诉客户端 next_tli。
```
学完后应能判断：
- 为什么 `recovery_target_timeline = 'latest'` 能让级联 standby 跟随上游 promotion 后的新历史。
- 为什么 `recovery_target_timeline = 'current'` 不是“当前上游最新 timeline”。
- 为什么 numeric timeline 会把 PITR 固定在指定分支。
- 为什么 `expectedTLEs` 是 recovery 可接受历史的边界。
- 为什么上游 promotion 后，下游通常先看到 old timeline end，再由 startup process 重启到 new timeline。
- 为什么 time / LSN / restore point target 都必须和 timeline target 一起解释。
本课基于本地 `/home/nail/postgres` 源码，
分支 `master`，
提交 `0e1f1ed157e90741e12a3715909e1b2d71ff9344`。

## 1. 本节在总主线中的位置
前几节已经把物理复制拆成了：
```text
primary walsender
  -> standby walreceiver
  -> pg_wal segment
  -> startup process ReadRecord()
  -> WAL replay / Hot Standby visibility
```
本节在这条链上加入 timeline。
timeline 不是 WAL 文件名里的装饰字段。
它回答的是：
```text
某个 LSN 区间属于哪一条数据库历史。
```
promotion 不会改写过去的 WAL。
它从某个 switchpoint 之后开始写入新 timeline。
同时写出 `000000NN.history`，说明新 timeline 从哪个父 timeline 的哪个 LSN 分叉。
所以恢复时的问题不是简单找下一个 WAL 文件。
真正的问题是：
```text
当前 standby 允许把哪些 timeline 拼成一条连续历史？
```
这个允许集合就是 `expectedTLEs`。
它在 `src/backend/access/transam/xlogrecovery.c` 中是 startup process 的 static list。
本节主线如下：
```text
recovery_target_timeline
  -> recoveryTargetTLI
  -> expectedTLEs
  -> tliOfPointInHistory()
  -> RequestXLogStreaming()
  -> START_REPLICATION ... TIMELINE
  -> historic timeline streaming
  -> walreceiver 获取 history file
  -> rescanLatestTimeLine()
  -> 更新 expectedTLEs 并继续 replay
```
本节不展开 redo resource manager。
也不展开 logical decoding slot 的 snapshot / catalog / confirmed_flush 语义。
这些问题都依赖 timeline，但不是本节唯一主问题。

## 2. 核心矛盾与一句话运行模型
一句话运行模型：
```text
startup process 根据 recovery_target_timeline 选择 recoveryTargetTLI，
用 readTimeLineHistory() 得到 expectedTLEs；
当本地 archive / pg_wal 不能提供下一段 WAL 时，
它用 tliOfPointInHistory(record_begin_lsn, expectedTLEs) 算出应请求的 timeline，
让 walreceiver 发送 START_REPLICATION <lsn> TIMELINE <tli>；
上游 walsender 如果发现这是 historic timeline，就只发送到 switchpoint 并返回 next_tli；
walreceiver 拉取缺失的 history file 后等待 startup process；
只有 latest 目标会触发 rescanLatestTimeLine()，验证新 timeline 是当前历史的合法后代，
然后替换 expectedTLEs 并重启 streaming。
```
这个模型里有两个“最新”。
第一个是上游 `IDENTIFY_SYSTEM` 返回的 `primaryTLI`。
它表示连接目标当前最高 timeline。
它由 `walreceiver.c` 调用 `walrcv_identify_system()` 获得。
第二个是本机 `recoveryTargetTLI`。
它表示 startup process 当前愿意恢复到的目标 timeline。
它受 `recovery_target_timeline` 控制。
二者不能混同。
walreceiver 可以发现上游有更高 timeline。
但它不能自行决定本机恢复历史必须跳过去。
原因是 timeline target 会改变 PITR 语义。
同一个 wall-clock time 在分叉后可能对应两份不同数据状态。
同一个数值 LSN 在分叉后也不能脱离 timeline 单独解释。
本节核心不变量：
```text
一条 WAL record 只有在它的 page / file timeline 属于 expectedTLEs，
并且 record LSN 落在该 TimeLineHistoryEntry 的 begin/end 区间内时，
才属于本次 recovery 允许 replay 的历史。
```
`expectedTLEs` 不是性能 cache。
它是 recovery 状态机的 correctness boundary。

## 3. 核心文件分工与阅读顺序
| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/access/timeline.h` | `TimeLineHistoryEntry` 的 `tli`、`begin`、`end` 区间语义。 |
| 2 | `src/backend/access/transam/timeline.c` | `readTimeLineHistory()`、`existsTimeLineHistory()`、`findNewestTimeLine()`、`tliOfPointInHistory()`、`tliSwitchPoint()`。 |
| 3 | `src/include/access/xlogrecovery.h` | `RecoveryTargetTimeLineGoal` 的 `CONTROLFILE`、`LATEST`、`NUMERIC`。 |
| 4 | `src/backend/access/transam/xlogrecovery.c` | `recoveryTargetTLI`、`expectedTLEs`、`WaitForWALToBecomeAvailable()`、`rescanLatestTimeLine()`、`checkTimeLineSwitch()`。 |
| 5 | `src/backend/replication/walreceiver.c` | `WalReceiverMain()`、`WalRcvFetchTimeLineHistoryFiles()`、`WalRcvWaitForStartPosition()`。 |
| 6 | `src/backend/replication/libpqwalreceiver/libpqwalreceiver.c` | `libpqrcv_startstreaming()` 构造 `START_REPLICATION ... TIMELINE`，`libpqrcv_endstreaming()` 接收 next timeline。 |
| 7 | `src/backend/replication/walsender.c` | `StartReplication()`、`SendTimeLineHistory()`、`WalSndSegmentOpen()`、`XLogSendPhysical()`。 |
| 8 | `src/backend/access/transam/xlogarchive.c` | `RestoreArchivedFile()` 如何让 archive 中的 history file 参与 `latest` 探测。 |
| 9 | `src/backend/access/transam/xlog.c` | `StartupXLOG()` 在 recovery 结束后选新 timeline、写 history file、清理旧 WAL。 |
| 10 | `src/include/replication/walreceiver.h` | `receiveStartTLI`、`receivedTLI`、`WalRcvStreamOptions.physical.startpointTLI`。 |
推荐阅读顺序不是从 walsender 主循环开始。
更好的路径是：
```text
timeline.h 区间模型
  -> timeline.c 读取和查找 history
  -> xlogrecovery.c 把 history 变成 expectedTLEs
  -> walreceiver.c 拉取上游 history file
  -> libpqwalreceiver.c 发送 START_REPLICATION TIMELINE
  -> walsender.c 对 historic timeline 发送到 switchpoint
  -> xlogrecovery.c latest rescan 并替换 expectedTLEs
```
这能避免一个常见误读：
```text
上游 promotion 后，是 walsender 主动把下游切到新 timeline。
```
实际不是。
walsender 只服务客户端显式请求的 timeline。
下游是否切到新 timeline，最终由 startup process 决定。

## 4. `TimeLineHistoryEntry`：一条 history，不是一张 DAG
`TimeLineHistoryEntry` 在 `src/include/access/timeline.h`。
字段很少，但语义很强：
| 字段 | 语义 |
| --- | --- |
| `tli` | 这一段 WAL 属于哪个 timeline。 |
| `begin` | 该 timeline 在这条 history 上的起始 LSN，inclusive。 |
| `end` | 该 timeline 在这条 history 上的结束 LSN，exclusive；invalid 表示延伸到无穷。 |
`readTimeLineHistory(targetTLI)` 返回 newest-to-oldest 的 list。
每个 entry 的 `begin/end` 拼出一条从起点到无穷的连续线。
例子：
```text
TLI 1: begin invalid, end 0/70000000
TLI 2: begin 0/70000000, end 0/A0000000
TLI 3: begin 0/A0000000, end invalid
```
`readTimeLineHistory(3)` 表示的是：
```text
从 TLI 1 走到 0/70000000；
从 TLI 2 走到 0/A0000000；
从 TLI 3 继续到未来。
```
它不是完整分叉图。
如果还有一条 `TLI 4` 从 `TLI 2` 的另一个 LSN 分叉，
`readTimeLineHistory(3)` 不会包含 `TLI 4`。
这解释了为什么 `latest` 不能只看 timeline ID 最大。
要看新 timeline 是否是当前目标 history 的合法后代。
`timeline.c` 中几个函数构成基本 API：
```text
readTimeLineHistory(targetTLI):
  读取 targetTLI 的 history file；
  targetTLI == 1 时没有 history file；
  找不到 history file 时返回只有 targetTLI 的 dummy history。
existsTimeLineHistory(probeTLI):
  探测 history file 是否存在；
  archive recovery 下会尝试 RestoreArchivedFile(..., "RECOVERYHISTORY", ...)。
findNewestTimeLine(startTLI):
  从 startTLI + 1 连续探测 history file；
  遇到第一个不存在的 TLI 停止。
tliOfPointInHistory(ptr, history):
  在 history 中找 begin <= ptr < end 的 entry；
  返回该 LSN 在这条 history 上的 TLI。
tliSwitchPoint(tli, history, &nextTLI):
  返回某个 tli 在这条 history 上切到 nextTLI 的 LSN。
```
`readTimeLineHistory()` 找不到文件时返回 dummy history，
不是说 history file 可有可无。
它允许 timeline 1 或无父 timeline 的场景继续。
在 streaming recovery 中，`XLogFileReadAnyTLI()` 还会避免过早把 dummy list 保存到 `expectedTLEs`。
这样 walreceiver 仍有机会先从上游拿到真实 history file。

## 5. `recovery_target_timeline`：目标策略
`recovery_target_timeline` 的 check / assign hook 在 `xlogrecovery.c`。
解析结果是 `RecoveryTargetTimeLineGoal`：
```text
"current" -> RECOVERY_TARGET_TIMELINE_CONTROLFILE
"latest"  -> RECOVERY_TARGET_TIMELINE_LATEST
number    -> RECOVERY_TARGET_TIMELINE_NUMERIC
```
numeric 的值保存在 `recoveryTargetTLIRequested`。
运行时真正使用的是 `recoveryTargetTLI`。
初始化时，`InitWalRecovery()` 先从 control file 得到基线：
```text
if minRecoveryPointTLI > checkpoint ThisTimeLineID:
  recoveryTargetTLI = minRecoveryPointTLI
else:
  recoveryTargetTLI = checkpoint ThisTimeLineID
```
然后 `validateRecoveryParameters()` 按用户目标修正：
```text
numeric:
  rtli != 1 时必须存在 history file；
  recoveryTargetTLI = rtli。
latest:
  recoveryTargetTLI = findNewestTimeLine(recoveryTargetTLI)。
current:
  保持 control file 给出的 recoveryTargetTLI。
```
这三个设置的边界不同。
| 设置 | 真正含义 | 典型用途 | 风险 |
| --- | --- | --- | --- |
| `current` | 使用本地 control file / minRecoveryPoint / checkpoint 的当前历史 | 恢复到备份原始历史，不主动追新分支 | 级联下游不会因上游 promotion 自动跟随新 TLI。 |
| `latest` | 启动时和等待 WAL 时寻找最新合法后代 | standby 跟随 failover；PITR 恢复到归档里最新历史 | archive 中新 history file 会改变 PITR 所在分支。 |
| numeric | 固定到指定 timeline | 精确恢复旧分支或指定 failover 分支 | 指错分支会找不到 target 或恢复出另一份业务历史。 |
源码中的全局默认值是：
```text
recoveryTargetTimeLineGoal = RECOVERY_TARGET_TIMELINE_LATEST
```
但这不意味着 timeline follow 是 walreceiver 的自然行为。
它仍然是 startup process 按 `latest` 策略做出的恢复决策。
`current` 特别容易误解。
它不是当前连接上游的 `primaryTLI`。
它对应 `RECOVERY_TARGET_TIMELINE_CONTROLFILE`。
也就是本地 control file 已经记录的恢复基线。

## 6. `expectedTLEs`：恢复允许历史
`expectedTLEs` 是 `xlogrecovery.c` 中的 static `List *`。
源码注释给出它的职责：
```text
它是 recoveryTargetTLI 及其已知父 timeline 的 TimeLineHistoryEntry list；
newest first；
只有这些 TLI 允许出现在要读取的 WAL 中；
也只有这些 TLI 会作为候选 WAL 文件被打开。
```
它把用户目标变成 runtime boundary。
可以把三层状态压成一条链：
```text
recovery_target_timeline:
  用户策略或默认策略。
recoveryTargetTLI:
  当前理解的目标 timeline。
expectedTLEs:
  从目标 timeline 回溯父 timeline 得到的一条允许历史。
```
`expectedTLEs` 被四类路径使用。
第一类是启动校验。
checkpoint 所在位置必须在目标 history 上：
```text
tliOfPointInHistory(CheckPointLoc, expectedTLEs) == CheckPointTLI
```
如果不成立，恢复报：
```text
requested timeline ... is not a child of this server's history
```
第二类是 `minRecoveryPoint` 校验。
`ControlFile->minRecoveryPoint - 1` 在目标 history 上的 TLI，
必须等于 `ControlFile->minRecoveryPointTLI`。
否则说明本地数据目录需要的最小一致性点不在用户指定历史上。
第三类是 WAL page 校验。
`ReadRecord()` 检查 `xlogreader->latestPageTLI` 是否属于 `expectedTLEs`。
如果 WAL segment 中混入另一条历史的 page TLI，
会报 `unexpected timeline ID ... in WAL segment ...`。
第四类是 streaming timeline 选择。
`WaitForWALToBecomeAvailable()` 不直接用 `recoveryTargetTLI` 请求 streaming。
它用：
```text
tli = tliOfPointInHistory(tliRecPtr, expectedTLEs)
RequestXLogStreaming(tli, ptr, PrimaryConnInfo, ...)
```
这里用的是 `tliRecPtr`。
它代表目标 WAL record 的 begin position。
`RecPtr` 可能指向 page header 或 segment header。
真正决定 record 属于哪个 timeline 的，是 record begin LSN。
还有一个边界是 `curFileTLI`。
`curFileTLI` 是当前输入 WAL 文件名中的 TLI。
它不一定等于 replayTLI。
promotion 时，旧 timeline 最后一个 segment 的有效前缀可能被复制到新 timeline 文件。
所以恢复可能从新 TLI 文件里读到旧 timeline 的 WAL 前缀。
但顺序扫描不允许 `curFileTLI` 倒退。
这防止父 timeline 延伸到更高 segment number 时，
恢复错误地回头选择父 timeline 上的文件。

## 7. 启动阶段 walkthrough：从 target 到 history
timeline 决策的启动入口：
```text
StartupXLOG()
  -> InitWalRecovery()
     -> readRecoverySignalFile()
     -> validateRecoveryParameters()
     -> read checkpoint / backup_label
     -> 构造或读取 expectedTLEs
     -> 校验 checkpoint 和 minRecoveryPoint
  -> PerformWalRecovery()
```
第一步，`InitWalRecovery()` 从 `pg_control` 选择基线 `recoveryTargetTLI`。
如果 `minRecoveryPointTLI` 高于 checkpoint 的 `ThisTimeLineID`，
说明本地数据目录已经要求恢复到更高 timeline 的最小一致性点。
所以要用 `minRecoveryPointTLI`。
第二步，`validateRecoveryParameters()` 解释 `recovery_target_timeline`。
numeric 要求目标 history file 存在，timeline 1 例外。
latest 调用 `findNewestTimeLine()`。
current 保持 control file 结果。
第三步，`findNewestTimeLine()` 连续探测 history file。
它不搜索任意缺口后的更高编号。
遇到第一个不存在的 TLI 就停止。
这个实现看似朴素，但对 recovery 有重要影响。
如果 archive 或上游漏掉中间 history file，
更高编号 timeline 可能不会被视为 newest。
因此 walreceiver 会尽量拉取 startpoint 到 primaryTLI 之间的所有缺失 history file。
第四步，`readTimeLineHistory(recoveryTargetTLI)` 得到 `expectedTLEs`。
之后 checkpoint 必须位于这条 history 上。
否则 base backup 与目标 timeline 不相容。
第五步，startup process 还会调用：
```text
restoreTimeLineHistoryFiles(checkPoint.ThisTimeLineID, recoveryTargetTLI)
```
这不是因为本机一定需要每个中间 file。
目标 timeline 的 history file 已经包含父链。
但级联下游可能需要中间 history file。
如果本节点 failover 后使用不同 archive，
保存这些文件也能让旧 timeline PITR 更可操作。

## 8. streaming 阶段 walkthrough：请求正确 TLI
当本地 archive / pg_wal 找不到下一段 WAL，
`WaitForWALToBecomeAvailable()` 进入 `XLOG_FROM_STREAM`。
关键路径：
```text
if startWalReceiver:
  if fetching checkpoint:
    ptr = RedoStartLSN
    tli = RedoStartTLI
  else:
    ptr = RecPtr
    tli = tliOfPointInHistory(tliRecPtr, expectedTLEs)
  curFileTLI = tli
  RequestXLogStreaming(tli, ptr, PrimaryConnInfo, PrimarySlotName, ...)
```
`RequestXLogStreaming()` 把请求写进 walreceiver shared state。
`WalRcvData` 在 `walreceiver.h` 中保存：
```text
receiveStart:
  本次 streaming 起点 LSN。
receiveStartTLI:
  本次 streaming 起点 timeline。
flushedUpto:
  walreceiver 已 flush 到的 LSN。
receivedTLI:
  flushedUpto 所属 timeline。
```
startup process 之后用 `GetWalRcvFlushRecPtr(&latestChunkStart, &receiveTLI)` 查看 walreceiver 进度。
只有当：
```text
RecPtr < flushedUpto
receiveTLI == curFileTLI
```
它才会把 streaming 文件打开为当前输入。
如果 `readFile < 0` 且 `expectedTLEs` 尚未初始化，
源码会执行：
```text
expectedTLEs = readTimeLineHistory(recoveryTargetTLI)
readFile = XLogFileRead(readSegNo, receiveTLI, XLOG_FROM_STREAM, false)
```
注意这里用 `recoveryTargetTLI`，
不是 `receiveTLI`。
原因是 `latest` 和 archive 可能已经让目标 timeline 更新。
`receiveTLI` 只是当前收到的文件 timeline。
它不一定代表本次 recovery 的目标 history。
这一点是级联环境里的关键细节。
下游可能先从旧 timeline 收到一段 WAL，
同时已经从 archive 或上游拿到更新的 history file。
startup process 必须以恢复目标为准，而不是以当前收到文件为准。

## 9. walreceiver：发现上游 timeline，但不决定切换
walreceiver 主循环在 `WalReceiverMain()`。
连接上游后，它先执行：
```text
primary_sysid = walrcv_identify_system(wrconn, &primaryTLI)
```
上游 `walsender.c` 的 `IdentifySystem()` 返回 system identifier、timeline、WAL 位置和 database name。
如果上游仍在 recovery，它用 `GetStandbyFlushRecPtr(&currTLI)`。
如果上游是 primary，它用 `GetFlushRecPtr(&currTLI)`。
walreceiver 检查：
```text
primaryTLI < startpointTLI
```
如果上游最高 timeline 还低于本机请求 timeline，
说明连错节点或历史落后。
错误是：
```text
highest timeline ... of the primary is behind recovery timeline ...
```
然后它调用：
```text
WalRcvFetchTimeLineHistoryFiles(startpointTLI, primaryTLI)
```
这一步总是尽力执行。
即使当前不打算追随更高 timeline，也会拉取缺失 history file。
原因是未来本机可能被 promote。
如果不知道上游已经使用过哪些 timeline ID，
本机可能选出相同 ID，造成混乱。
`WalRcvFetchTimeLineHistoryFiles()` 做的事：
```text
for tli in first..last:
  if tli != 1 && !existsTimeLineHistory(tli):
    walrcv_readtimelinehistoryfile(wrconn, tli, &fname, &content, &len)
    校验 fname == TLHistoryFileName(tli)
    writeTimeLineHistoryFile(tli, content, len)
```
`libpqrcv_readtimelinehistoryfile()` 发送：
```text
TIMELINE_HISTORY <tli>
```
上游 `SendTimeLineHistory()` 返回一行两列：
```text
filename
content
```
这说明 walreceiver 的职责是搬运和持久化信息。
它可以知道 `primaryTLI`。
它可以把 `00000002.history` 写到 `pg_wal`。
但它不替换 `expectedTLEs`。
所以它不决定本机恢复历史。

## 10. `START_REPLICATION TIMELINE` 与 historic streaming
walreceiver 从 shared state 取到 `receiveStart` 和 `receiveStartTLI` 后，
`libpqrcv_startstreaming()` 构造物理复制命令：
```text
START_REPLICATION [SLOT "..."] <lsn> TIMELINE <startpointTLI>
```
这个 `TIMELINE` 让下游能明确请求一条历史：
```text
即使上游已经在 TLI 3，
下游仍可请求从 TLI 2 的某个 LSN 开始发送。
```
返回值语义：
```text
true:
  上游进入 COPY_BOTH，开始 streaming。
false:
  上游接受命令但没有进入 copy mode；
  通常表示请求 timeline 在该 startpoint 没有更多 WAL，
  也就是上游已在该点或之前切到另一条 timeline。
```
上游处理入口是 `walsender.c` 的 `StartReplication()`。
它先决定当前可发送位置：
```text
if RecoveryInProgress():
  FlushPtr = GetStandbyFlushRecPtr(&FlushTLI)
else:
  FlushPtr = GetFlushRecPtr(&FlushTLI)
```
如果客户端指定 timeline：
```text
cmd->timeline == FlushTLI:
  sendTimeLineIsHistoric = false
cmd->timeline != FlushTLI:
  sendTimeLineIsHistoric = true
  timeLineHistory = readTimeLineHistory(FlushTLI)
  switchpoint = tliSwitchPoint(cmd->timeline, timeLineHistory, &sendTimeLineNextTLI)
  sendTimeLineValidUpto = switchpoint
```
如果请求 startpoint 晚于该 timeline 在上游历史中的 switchpoint，
walsender 报：
```text
requested starting point ... on timeline ... is not in this server's history
```
`XLogSendPhysical()` 对 historic timeline 的边界是：
```text
SendRqstPtr = sendTimeLineValidUpto
```
当 `sentPtr` 到达 `sendTimeLineValidUpto`，
walsender 发送 CopyDone。
`StartReplication()` 随后返回一行：
```text
next_tli
next_tli_startpos
```
walreceiver 的 `libpqrcv_endstreaming()` 读取 `next_tli`。
它忽略 `next_tli_startpos`。
实际重启点由 startup process 根据本地 replay 位置和 history 决定。
还有一个文件层细节在 `WalSndSegmentOpen()`。
如果正在发送 historic timeline，
且要打开的 segment 正好包含 switchpoint，
walsender 可能改用 next timeline 的 segment 文件。
原因是 promotion 时，
旧 timeline 最后一个 segment 的有效前缀会复制到新 timeline 的起始 segment。
旧 TLI 文件可能不存在。
但 switchpoint 之前的 bytes 相同。
所以诊断时不要只看文件名 TLI。
真实语义由 `sendTimeLine`、`sendTimeLineValidUpto` 和 history 区间共同决定。

## 11. 级联 standby 上游 promotion：完整状态故事
考虑三节点拓扑：
```text
P -> A -> B
```
P 是原 primary。
A 是 P 的 standby，同时给 B 提供 cascading walsender。
B 是 A 的下游 standby。
初始状态：
```text
P 写 TLI 1 WAL
  -> A 接收并 replay TLI 1
  -> A walsender 给 B 发送 TLI 1
  -> B 接收并 replay TLI 1
```
A 被 promote。
A 结束 recovery，执行大致路径：
```text
StartupXLOG()
  -> FinishWalRecovery()
  -> newTLI = findNewestTimeLine(recoveryTargetTLI) + 1
  -> XLogInitNewTimeline(EndOfLogTLI, EndOfLog, newTLI)
  -> writeTimeLineHistory(newTLI, recoveryTargetTLI, EndOfLog, reason)
  -> 设置 InsertTimeLineID
  -> 写 end-of-recovery record
```
A 的新 timeline 假设为 TLI 2。
B 此时还在按旧请求工作：
```text
START_REPLICATION <lsn> TIMELINE 1
```
A 的 walsender 看到：
```text
FlushTLI = 2
cmd->timeline = 1
```
于是进入 historic streaming：
```text
sendTimeLine = 1
sendTimeLineIsHistoric = true
sendTimeLineValidUpto = switchpoint
sendTimeLineNextTLI = 2
```
A 把 TLI 1 发送到 switchpoint。
到达 switchpoint 后：
```text
A walsender:
  CopyDone
  next_tli = 2
B walreceiver:
  libpqrcv_endstreaming(&primaryTLI)
  primaryTLI = 2
  WalRcvFetchTimeLineHistoryFiles(1, 2)
  写入 00000002.history
  WalRcvWaitForStartPosition()
  WakeupRecovery()
```
B 的 startup process 被唤醒。
如果 B 是 `recovery_target_timeline = 'latest'`：
```text
WaitForWALToBecomeAvailable()
  -> stream source 暂时失败或 walreceiver waiting
  -> rescanLatestTimeLine(replayTLI, replayLSN)
     -> findNewestTimeLine(recoveryTargetTLI) 得到 2
     -> readTimeLineHistory(2)
     -> 确认旧 recoveryTargetTLI 在新 history 中
     -> 确认新 timeline forkpoint 没有早于当前 replayLSN
     -> recoveryTargetTLI = 2
     -> expectedTLEs = newExpectedTLEs
     -> LOG "new target timeline is 2"
  -> RequestXLogStreaming(2, ptr, ...)
```
B 随后发送：
```text
START_REPLICATION <lsn> TIMELINE 2
```
并继续沿 A 的新历史 replay。
如果 B 是 `current` 或 numeric `1`：
```text
B 可以 replay TLI 1 到 switchpoint；
但不会因为看到 00000002.history 就把恢复目标改成 2。
```
它可能继续等待 TLI 1 上的更多 WAL。
在非 standby PITR 场景中，它也可能按 recovery target 结束。
这就是 `recovery_target_timeline` 影响故障转移后可恢复范围的直接原因。

## 12. `rescanLatestTimeLine()`：latest 的安全阀
`rescanLatestTimeLine()` 只服务 `RECOVERY_TARGET_TIMELINE_LATEST`。
它嵌在 startup process 等 WAL 的 retry 逻辑中。
核心步骤：
```text
oldtarget = recoveryTargetTLI
newtarget = findNewestTimeLine(recoveryTargetTLI)
if newtarget == recoveryTargetTLI:
  return false
newExpectedTLEs = readTimeLineHistory(newtarget)
确认 oldtarget 在 newExpectedTLEs 中
确认 oldtarget 的 end >= replayLSN
recoveryTargetTLI = newtarget
list_free_deep(expectedTLEs)
expectedTLEs = newExpectedTLEs
restoreTimeLineHistoryFiles(oldtarget + 1, newtarget)
LOG "new target timeline is ..."
return true
```
第一道检查：
```text
新 timeline 必须是当前 target 的后代。
```
如果新 history 中找不到 oldtarget，
startup process 只 LOG：
```text
new timeline ... is not a child of database system timeline ...
```
它不会因为 timeline ID 更大就跳过去。
第二道检查：
```text
新 timeline 的 forkpoint 不能早于当前 replayLSN。
```
如果当前节点已经 replay 过旧 timeline 上 forkpoint 之后的 WAL，
就不能回头进入另一条分支。
源码边界是：
```text
currentTle->end < replayLSN:
  LOG "new timeline ... forked off ... before current recovery point"
  return false
```
真正改变后续行为的是替换 `expectedTLEs`。
替换之后：
```text
XLogFileReadAnyTLI()
  会按新 history 扫描候选 TLI。
tliOfPointInHistory()
  会按新 history 选择 streaming TLI。
checkTimeLineSwitch()
  会接受新 history 中合法的 timeline switch。
```
诊断 timeline follow 时，
不要只问：
```text
有没有 00000002.history？
```
还要问：
```text
startup process 有没有把 recoveryTargetTLI 和 expectedTLEs 更新到 2？
```

## 13. PITR 可恢复范围：target 必须落在选定 history 上
PITR target 包括：
```text
recovery_target_time
recovery_target_lsn
recovery_target_name
recovery_target_xid
```
这些 target 都沿当前 recovery history 判断。
它们不是全局坐标。
例如：
```text
recovery_target_time = '2026-06-11 10:00:00'
```
真实含义是：
```text
沿 recovery_target_timeline 选出的 history replay，
遇到满足 time target 的 record 后停止。
```
如果 failover 后存在：
```text
TLI 1:
  原 primary 的旧历史。
TLI 2:
  standby promotion 后的新历史。
```
同一个 wall-clock time 可能对应两份不同业务状态。
`latest` 倾向进入 TLI 2。
numeric `1` 固定在旧分支。
`current` 使用本地 control file 的当前历史。
启动校验会先挡住不相容历史。
如果 base backup 的 checkpoint 不在目标 history 上，
会报：
```text
requested timeline ... is not a child of this server's history
```
如果 `minRecoveryPoint` 不在目标 history 上，
会报目标 timeline 不包含 minimum recovery point。
`checkTimeLineSwitch()` 还会防止在到达 `minRecoveryPoint` 前切到错误高 timeline。
源码注释解释了典型场景：
archive 中有一条新 timeline，
但它在 min recovery point 所在 timeline 之前分叉。
如果这时跳过去，
本节点将永远不可能在正确 timeline 上访问 minimum recovery point。
因此可恢复范围至少同时要求：
```text
目标 history file 存在；
checkpoint 在目标 history 上；
minRecoveryPoint 在目标 history 上；
当前 replayLSN 没有越过目标分支点；
所需 WAL bytes 仍在 pg_wal、archive 或上游 streaming 中。
```
还有一个少见但重要的 `.partial` 边界。
`CleanupAfterArchiveRecovery()` 处理 recovery 结束后的旧 timeline partial segment。
如果 switchpoint 在 segment 中间，
新 timeline 文件包含旧 timeline 的有效前缀。
旧 timeline 最后一个文件可能被改名为 `.partial` 并归档。
archive recovery 正常不会自动读取 `.partial`。
所以要 PITR 到旧 timeline 的尾部时，
管理员可能需要手动恢复这个 partial 文件并去掉后缀。
这说明：
```text
timeline history、WAL 文件名、archive 策略和 PITR 范围是同一个问题的不同侧面。
```

## 14. 正确性机制层次
timeline 追随不是靠单个判断保证正确。
它由几层机制组合。
| 层次 | 机制 | 保证 |
| --- | --- | --- |
| 历史模型 | `TimeLineHistoryEntry.begin/end` | LSN 属于哪条 history path。 |
| 用户目标 | `recovery_target_timeline` | 选择当前恢复想进入的分支。 |
| 本地边界 | `expectedTLEs` | 限定可读 WAL 的 TLI 集合和区间。 |
| streaming 协议 | `START_REPLICATION ... TIMELINE` | 下游明确请求某条历史。 |
| 上游边界 | `sendTimeLineValidUpto` | historic timeline 只能发送到 switchpoint。 |
| retry / fallback | `rescanLatestTimeLine()` | latest 只追合法后代且不能回跳。 |
| crash safety | history file + end-of-recovery record | promotion 后新 timeline 被持久化和传播。 |
| 文件清理 | `RemoveNonParentXlogFiles()` | 切新 timeline 后清理不属于新历史的未来 WAL 文件。 |
这些机制分别保证不同事。
`history file` 只描述路径。
它不保证 WAL segment 仍然存在。
`primaryTLI` 只说明上游最高 timeline。
它不代表本机应该跟随。
`receivedTLI` 只说明 walreceiver flush 的 WAL 来自哪条 timeline。
它不代表 startup process 已经接受新 history。
`expectedTLEs` 才是本机恢复时的允许历史。

## 15. 错误路径与异常边界
第一类错误：上游 timeline 落后。
walreceiver 连接后发现：
```text
primaryTLI < startpointTLI
```
说明连接到了旧节点、错误节点或缺失 history 的节点。
错误边界在 walreceiver。
第二类错误：请求 timeline 不在上游历史中。
walsender 在 `StartReplication()` 中调用 `readTimeLineHistory(FlushTLI)` 和 `tliSwitchPoint()`。
如果请求 TLI 不在上游当前 history 中，
会报：
```text
requested timeline ... is not in this server's history
```
如果请求 startpoint 晚于该 timeline 的 switchpoint，
会报：
```text
requested starting point ... on timeline ... is not in this server's history
```
第三类边界：旧 timeline 正常到头。
这不是错误。
walreceiver 可能看到：
```text
primary server contains no more WAL on requested timeline ...
walreceiver ended streaming and awaits new instructions
```
如果目标是 `latest`，
后续应看到 startup process 记录：
```text
new target timeline is ...
restarted WAL streaming ... on timeline ...
```
如果没有，
应怀疑 `recovery_target_timeline` 不是 latest，
或新 timeline 没通过后代 / forkpoint 校验。
第四类错误：WAL page TLI 不在 expected history。
`ReadRecord()` 检查 `latestPageTLI`。
如果 page header 的 timeline 不在 `expectedTLEs` 中，
会报 `unexpected timeline ID ... in WAL segment ...`。
常见原因：
- archive 混入另一条分支的同号 segment。
- 手工复制 `pg_wal` 时覆盖了正确 segment。
- numeric timeline 指错。
- history file 缺失或被错误替换。
第五类边界：新 timeline 分叉点早于当前 replay 点。
`rescanLatestTimeLine()` 记录：
```text
new timeline ... forked off current database system timeline ... before current recovery point
```
这表示本节点已经 replay 了旧分支上 forkpoint 之后的 WAL。
为了 correctness，不能切入另一条分支。

## 16. 成本、资源与跨模块传播
timeline follow 通常不是 CPU hot path。
成本主要在慢路径和资源传播。
`findNewestTimeLine()` 会逐个探测 history file：
```text
startTLI + 1
startTLI + 2
...
直到第一个不存在的 TLI
```
archive recovery 下，每次探测可能执行 `RestoreArchivedFile()`。
如果 archive 很慢、timeline 很多、`wal_retrieve_retry_interval` 又较短，
`latest` rescan 会拖慢追随新 timeline 的速度。
WAL retention 是另一个边界。
history file 告诉系统应该走哪条路。
它不保证路上的 WAL bytes 还在。
相关模块：
| 模块 | 影响 |
| --- | --- |
| archive | history file 和 WAL segment 是否能恢复。 |
| physical replication slot | 上游是否保留下游未接收的 WAL。 |
| `pg_wal` cleanup | 旧 segment 是否被移除或 recycled。 |
| `RemoveNonParentXlogFiles()` | 切新 timeline 后清理不属于新历史的未来 WAL 文件。 |
级联拓扑里还有传播延迟。
上游 promotion 后，下游要经历：
```text
上游写 history file
  -> 旧 timeline streaming 到 switchpoint
  -> walreceiver endstreaming
  -> 拉取新 history file
  -> startup process rescan latest
  -> 请求新 timeline streaming
```
每一步都可能受网络、archive、WAL flush、walreceiver restart、replay 速度影响。
physical cascading 和 logical walsender 的唤醒边界也不同。
`XLogWalRcvFlush()` 在 standby 上会 `WalSndWakeup(true, false)`。
physical cascading 可以发送已接收并 flush 的 WAL。
`ApplyWalRecord()` replay 后会 `WalSndWakeup(switchedTLI, true)`。
logical walsender 在 standby 上只能解码已 replay 的 WAL。
如果发生 timeline switch，physical walsender 也要被唤醒，以便发现当前发送 timeline 已变成 historic。

## 17. 观测与诊断入口
第一层，文件。
查看 `pg_wal`：
```text
00000002.history
00000003.history
00000002000000000000000A
00000003000000000000000A
```
history file 里要看父 timeline 和 switchpoint。
不要只看最高编号。
要确认目标 timeline 的祖先链包含当前节点已经 replay 的历史。
第二层，SQL。
`pg_stat_wal_receiver` 有：
```text
status
receive_start_lsn
receive_start_tli
written_lsn
flushed_lsn
received_tli
latest_end_lsn
sender_host
sender_port
```
诊断查询：
```sql
select status,
       receive_start_lsn,
       receive_start_tli,
       flushed_lsn,
       received_tli,
       latest_end_lsn,
       sender_host,
       sender_port
from pg_stat_wal_receiver;
```
它能看见 walreceiver 当前请求和收到的 timeline。
它看不见 startup process 的 `expectedTLEs`。
还可以看：
```sql
select pg_is_in_recovery();
select pg_last_wal_receive_lsn();
select pg_last_wal_replay_lsn();
select * from pg_control_checkpoint();
select * from pg_control_recovery();
```
`pg_control_checkpoint()` 提供 checkpoint timeline。
`pg_control_recovery()` 提供 minimum recovery point 和 timeline。
它们帮助解释 `current` 的本地基线。
第三层，日志。
重点搜索：
```text
fetching timeline history file for timeline ...
started streaming WAL from primary ... on timeline ...
restarted WAL streaming ... on timeline ...
primary server contains no more WAL on requested timeline ...
walreceiver ended streaming and awaits new instructions
new target timeline is ...
selected new timeline ID: ...
walsender reached end of timeline ...
requested timeline ... is not in this server's history
unexpected timeline ID ... in WAL segment ...
```
这些日志按时间排序后，
可以重建：
```text
上游何时 promotion；
下游何时拉到 history file；
startup process 是否执行 latest rescan；
下游是否从新 TLI 重启 streaming；
失败发生在上游历史、本地 expectedTLEs，还是 WAL 文件缺失。
```
第四层，gdb。
推荐断点：
```text
xlogrecovery.c:
  validateRecoveryParameters()
  WaitForWALToBecomeAvailable()
  rescanLatestTimeLine()
  checkTimeLineSwitch()
  XLogFileReadAnyTLI()
walreceiver.c:
  WalRcvFetchTimeLineHistoryFiles()
  WalRcvWaitForStartPosition()
libpqwalreceiver.c:
  libpqrcv_startstreaming()
  libpqrcv_endstreaming()
walsender.c:
  StartReplication()
  XLogSendPhysical()
  WalSndSegmentOpen()
```
在 startup process 打印：
```text
recoveryTargetTimeLineGoal
recoveryTargetTLI
expectedTLEs
curFileTLI
replayTLI
replayLSN
```
在 walsender 打印：
```text
FlushTLI
sendTimeLine
sendTimeLineIsHistoric
sendTimeLineValidUpto
sendTimeLineNextTLI
sentPtr
```

## 18. 课堂实验
实验 1：三节点级联 promotion。
拓扑：
```text
P -> A -> B
```
要求：
```text
A 开启 hot_standby = on 和 max_wal_senders > 0。
B 的 primary_conninfo 指向 A。
B 使用 recovery_target_timeline = latest。
```
步骤：
```text
1. 在 P 上写入数据，确认 A 和 B 都 replay。
2. 停止或隔离 P。
3. 在 A 上 pg_promote()。
4. 观察 A 的 pg_wal 出现 00000002.history。
5. 观察 B 的日志和 pg_stat_wal_receiver。
```
预期：
```text
B 先请求旧 timeline；
A walsender 到达 old timeline switchpoint；
B walreceiver 拉取 00000002.history；
B startup process 记录 new target timeline is 2；
B walreceiver restarted ... on timeline 2。
```
实验 2：把 B 改成 `current`。
重复实验 1。
观察：
```text
B 可能看到上游 primaryTLI 更高，也可能拉到 history file；
但 startup process 不会把 recoveryTargetTLI 更新为新 timeline。
```
对比是否缺少：
```text
new target timeline is ...
restarted WAL streaming ... on timeline 2
```
这个实验验证：
```text
发现 history file != 接受 history。
```
实验 3：PITR 到旧分支。
准备包含 TLI 1 和 TLI 2 的 archive。
分别恢复：
```text
recovery_target_time = 同一个 failover 后时间
recovery_target_timeline = 'latest'
recovery_target_time = 同一个 failover 后时间
recovery_target_timeline = '1'
```
比较：
```text
latest 是否进入 TLI 2；
numeric 1 是否停在旧分支；
缺 WAL 时错误表现是文件缺失、timeline mismatch，还是 minimum recovery point 不满足。
```
实验 4：断点观察 `expectedTLEs`。
在 B 的 startup process 断：
```text
rescanLatestTimeLine()
```
打印：
```text
recoveryTargetTLI
replayTLI
replayLSN
newtarget
currentTle->tli
currentTle->end
```
函数返回后再打印 `recoveryTargetTLI` 和 `expectedTLEs`。
这个实验要确认：
```text
latest 切换的实际动作是替换 expectedTLEs。
```

## 19. 常见误区
1. 把 `latest` 理解成 walreceiver 自动追上游最新 timeline。
真正切换在 startup process 的 `rescanLatestTimeLine()`。
2. 把 `current` 理解成当前上游 timeline。
源码里它对应 `RECOVERY_TARGET_TIMELINE_CONTROLFILE`。
3. 认为 `.history` 文件存在就一定能切过去。
还要满足 checkpoint、minRecoveryPoint、当前 replayLSN 和 forkpoint 边界。
4. 只看 WAL 文件名里的 TLI。
switchpoint 所在 segment 可能从 next TLI 文件读取 old timeline 前缀。
5. 认为 PITR 的时间点是全局唯一。
分叉后同一时间可以落在不同业务历史上。
6. 把 walsender 返回 `next_tli` 当作下游已经切换。
`next_tli` 只是“旧 timeline 到头”的信息。
7. 认为 physical cascading 必须等上游 standby replay 后才能转发。
`GetStandbyFlushRecPtr()` 允许在同一 TLI 上发送已 flush 未 replay 的 WAL。

## 20. 讨论题
1. 为什么 `expectedTLEs` 保存一条 newest-to-oldest path，而不是完整 timeline DAG？
2. 为什么 `current` 在 failover 后可能让级联下游停在旧历史？
3. walreceiver 已经写入 `00000003.history`，startup process 仍可能拒绝切到 TLI 3，原因有哪些？
4. 为什么 `WaitForWALToBecomeAvailable()` 用 record begin LSN 的 `tliRecPtr` 选择 streaming timeline？
5. 为什么 historic timeline streaming 到 switchpoint 就结束，而不是上游直接继续发送 next timeline？
6. `pg_stat_wal_receiver.received_tli` 能证明本机已经接受新 history 吗？
7. 如果 archive 中存在更高编号但非后代 timeline，`findNewestTimeLine()` 和 `rescanLatestTimeLine()` 分别如何处理？
8. 为什么 `.partial` segment 影响少数旧 timeline PITR，却不影响正常追随新 timeline？

## 21. 本节小结
本节核心链路：
```text
recovery_target_timeline
  -> recoveryTargetTLI
  -> expectedTLEs
  -> tliOfPointInHistory()
  -> START_REPLICATION ... TIMELINE
  -> historic streaming 到 switchpoint
  -> walreceiver 获取 history file
  -> rescanLatestTimeLine()
  -> 替换 expectedTLEs 并重启 streaming
```
`expectedTLEs` 是本节最重要状态。
它把目标 timeline 及其祖先压成一条允许恢复的连续历史。
walreceiver 负责搬运 WAL 和 history file。
walsender 负责按明确 timeline 请求发送 WAL。
startup process 负责判断本机是否允许进入新历史。
`recovery_target_timeline` 影响 PITR，
因为 PITR target 是沿选定 history replay 时被命中的。
time、LSN、restore point 和 XID 都不是脱离 timeline 的全局坐标。
错误路径也围绕同一个模型：
```text
上游落后:
  primaryTLI < startpointTLI。
上游不包含请求历史:
  START_REPLICATION TIMELINE 被拒绝。
本地不接受新历史:
  rescanLatestTimeLine() 拒绝非后代或过早分叉。
WAL 文件混入错误历史:
  ReadRecord() 报 unexpected timeline ID。
```
可观测入口分层：
```text
文件:
  pg_wal/*.history 和 WAL segment 文件名。
SQL:
  pg_stat_wal_receiver、pg_last_wal_receive_lsn()、
  pg_last_wal_replay_lsn()、pg_control_checkpoint()、pg_control_recovery()。
日志 / gdb:
  new target timeline、START_REPLICATION TIMELINE、
  sendTimeLineIsHistoric、expectedTLEs。
```
可迁移的系统规律：
```text
在分叉历史系统里，“最新”不是排序问题，而是路径合法性问题。
必须先定义当前节点允许进入哪条历史，
再解释位置、文件名、复制协议和恢复目标。
```
