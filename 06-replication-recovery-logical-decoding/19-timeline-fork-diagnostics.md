# PostgreSQL timeline fork 故障诊断：归档缺失、恢复目标错误与错误上游
## 课程定位
前置知识：已经理解物理复制 handshake、walreceiver 写入、startup process recovery state machine、timeline ID 与 history 文件、promotion end-of-recovery checkpoint。
本节唯一主问题：
```text
遇到 requested timeline、missing history file、
WAL segment belongs to a different timeline 这类错误时，
如何判断问题是归档缺失、恢复目标错误，还是 standby 接到了错误上游？
```
核心矛盾：同一个集群血统里的多个节点可以共享早期 WAL 前缀，但 promotion 后会形成不同未来；恢复过程必须尽量从 archive、pg_wal、streaming 多个来源继续推进，又绝不能把另一个未来的 WAL 当成自己的历史。
PostgreSQL 的诊断入口不是单个错误字符串。真正要判断的是：
```text
本机想恢复到哪条 target timeline；
这条 target timeline 的 history 是否完整；
请求 LSN 在 history 区间上应该属于哪个 TLI；
当前 WAL 来源提供的文件是否就是这个 TLI；
streaming 上游是否知道并拥有这条 history。
```
学完后应能独立判断：
```text
restore_command 找不到 0000000N.history 是 archive gap 还是正常等待；
recovery_target_timeline 指向不存在或不包含本机 checkpoint 时为什么会 FATAL；
primary_conninfo 指到错误分支时为什么 system identifier 可能仍然相同；
pg_waldump 和 pg_controldata 分别能验证哪一层事实。
```
本课基于本地源码 `/home/nail/postgres`，分支 `master`，提交 `0e1f1ed157e90741e12a3715909e1b2d71ff9344`。
## 1. 本节在总主线中的位置
第 16 节建立了 timeline 的基本语义：
```text
TimeLineID + .history switchpoint
  -> 把同一个 LSN 地址映射到一条唯一历史区间
```
第 17 节说明 promotion 如何创建新历史：
```text
standby promotion
  -> 选 newTLI
  -> 写 <newTLI>.history
  -> 写 end-of-recovery record/checkpoint
  -> 进入可写 primary
```
本节不再重复 promotion 流程。本节只看故障现场：
```text
恢复或级联复制卡住；
日志里出现 requested timeline；
restore_command 反复找 history 或 WAL segment；
walreceiver 说 primary 没有 requested timeline 上的更多 WAL；
pg_waldump 读某个 segment 时发现 timeline 或 page header 不符合预期。
```
这些现象表面很像：
```text
都是缺 WAL；
都是 timeline 不对；
都可能发生在 failover 之后。
```
但处理方向完全不同：
```text
archive gap:
  补 archive / history 文件，恢复路径本身没有错。
wrong recovery target:
  改 recovery_target_timeline 或恢复目标 LSN/time/name，
  当前数据目录不属于你要求的那条 history。
wrong upstream:
  改 primary_conninfo 或重建 standby，
  上游 system identifier 可能相同，但未来分支不相同。
```
本节围绕一个对象推进：
```text
expectedTLEs
```
它不是持久化文件。它是 startup process 根据 `.history` 文件构造出的内存区间链。
只要能解释 `expectedTLEs` 怎么来、怎么被查询、怎么约束 WAL source，就能把上面的错误分开。
## 2. 核心矛盾与一句话运行模型
一句话运行模型：
```text
恢复开始时，startup process 根据 recoveryTargetTLI 读取 timeline history，
构造 newest-to-oldest 的 expectedTLEs；
之后每次读 WAL segment、启动 walreceiver、校验 checkpoint 或响应 START_REPLICATION，
都用 tliOfPointInHistory() / tliSwitchPoint() 判断某个 LSN 或请求 TLI
是否属于这条历史。
```
这条模型背后的 tension 是：
```text
为了可用性，恢复要从 archive、pg_wal、streaming 中不断重试；
为了正确性，任何一个来源都不能越过 history switchpoint 提供另一条分支的 WAL。
```
如果系统只按文件名找 WAL：
```text
000000020000000000000013
```
它只能知道：
```text
这是 timeline 2 的第 0/13 段。
```
它还不知道：
```text
timeline 2 是否是本机 checkpoint 所属历史的后代；
0/13000000 这个 LSN 是否还在 timeline 2 的合法区间；
本机想追的 latest 是否已经变成 timeline 3；
上游是否有 timeline 2 的 history 文件。
```
`.history` 文件回答父链和 switchpoint。`expectedTLEs` 把这些父链变成查询结构。WAL source state machine 决定从哪里拿文件。
诊断要把这三层分开。常见错误字符串只是在不同层次露出的症状。例如：
```text
requested timeline %u is not in this server's history
```
通常不是“缺一个 WAL record”。它是在说：
```text
tliSwitchPoint() 在当前 history 链里找不到这个 TLI。
```
又如：
```text
according to history file, WAL location ... belongs to timeline X,
but previous recovered WAL file came from timeline Y
```
这不是简单网络断开。它是在说：
```text
同一条恢复流中，实际读到的前一个 WAL 文件 TLI
已经比 history 计算出的当前 LSN 所属 TLI 更新；
如果继续读，就可能跨回父 timeline 或读错分支。
```
因此本节的诊断原则是：
```text
不要从错误字符串直接跳到修复动作；
先重建本机认可的 target history，
再判断缺的是文件、目标还是上游。
```
## 3. 三类故障表面相似，但状态不同
先把三类问题放在同一条时间线上。假设原 primary 是 P。P 在 timeline 1 写到 `0/6000000`。
standby S1 在 `0/5000000` promotion，生成 timeline 2。另一个 standby S2 还没切过去。之后可能出现三类现场。
第一类是 archive gap：
```text
S2 想追 timeline 2。
S2 的 target history 正确。
但 archive 里缺 00000002.history，
或者缺 000000020000000000000005 之后的 segment。
```
这时系统想要的历史没错。缺的是实现这条历史的文件。第二类是 recovery target 错误：
```text
S2 的 pg_control / backup_label 表示它的 checkpoint 在 timeline 1 的 0/5800000。
管理员设置 recovery_target_timeline = 2。
但 timeline 2 在 0/5000000 就已经从 timeline 1 分叉。
```
这时 timeline 2 的 history 存在。但本机 checkpoint 已经位于分叉点之后的 timeline 1。
这个数据目录不能恢复到 timeline 2。第三类是错误上游：
```text
S2 设置 primary_conninfo 指向 P。
管理员以为 P 已经切到新主。
实际上 P 仍在 timeline 1，或者 P 是另一个分支。
```
这时 system identifier 可能完全一致。`IDENTIFY_SYSTEM` 只能证明集群血统相同。
它不能证明上游拥有目标 timeline 的未来。walreceiver 连接成功，不等于 upstream 正确。它必须还满足：
```text
primaryTLI >= startpointTLI；
能够返回缺失的 timeline history；
START_REPLICATION startpoint on timeline 能进入 COPY mode；
或者明确告诉本 timeline 已经没有更多 WAL。
```
这三类问题的第一个区别是：
```text
archive gap:
  expectedTLEs 是对的，source 找不到文件。
wrong target:
  expectedTLEs 本身和 pg_control / backup_label 冲突。
wrong upstream:
  本机 expectedTLEs 合理，但 streaming 端的 history / current TLI 不支持它。
```
第二个区别是修复顺序：
```text
archive gap:
  先补 history，再补 WAL segment。
wrong target:
  先修改 recovery_target_timeline 或重做 base backup/PITR 目标。
wrong upstream:
  先断开错误 upstream，改 primary_conninfo 到目标分支的节点。
```
## 4. 核心源码文件与阅读顺序
阅读顺序按诊断链走，不按文件名排序。
| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/access/timeline.h` | `TimeLineHistoryEntry` 的 `[begin, end)` 区间语义。 |
| 2 | `src/backend/access/transam/timeline.c` | `readTimeLineHistory()`、`existsTimeLineHistory()`、`tliOfPointInHistory()`、`tliSwitchPoint()`。 |
| 3 | `src/backend/access/transam/xlogrecovery.c` | `recoveryTargetTLI`、`expectedTLEs`、`WaitForWALToBecomeAvailable()`、`XLogFileReadAnyTLI()`、`rescanLatestTimeLine()`。 |
| 4 | `src/backend/access/transam/xlogarchive.c` | `RestoreArchivedFile()`、`KeepFileRestoredFromArchive()`、history 文件归档优先级。 |
| 5 | `src/backend/replication/walreceiver.c` | `WalRcvFetchTimeLineHistoryFiles()`、streaming 启动前的 systemid/TLI 校验。 |
| 6 | `src/backend/replication/libpqwalreceiver/libpqwalreceiver.c` | `libpqrcv_startstreaming()`、`libpqrcv_readtimelinehistoryfile()`。 |
| 7 | `src/backend/replication/walsender.c` | `SendTimeLineHistory()`、`StartReplication()`、historic timeline 发送边界。 |
| 8 | `src/backend/access/transam/xlogreader.c` | WAL page header 中 timeline/pageaddr 的局部一致性校验。 |
| 9 | `src/bin/pg_waldump/pg_waldump.c` | `-t/--timeline` 如何让离线 dump 按指定 TLI 读 segment。 |
| 10 | `src/bin/pg_controldata/pg_controldata.c` | checkpoint TLI、PrevTLI、minRecoveryPointTLI 的可观测入口。 |
推荐先读这几个小函数：
```text
readTimeLineHistory()
  -> 把 .history 文件变成 list
tliOfPointInHistory()
  -> 给 LSN，回答这个点属于哪个 TLI
tliSwitchPoint()
  -> 给 TLI，回答这条 history 在哪里从它分叉
XLogFileReadAnyTLI()
  -> 按 expectedTLEs 找一个 segment 应该用哪个 TLI 文件
StartReplication()
  -> walsender 判断客户端请求的 timeline 是否在本机 history 中
```
不要先从 `WaitForWALToBecomeAvailable()` 的大循环读起。那个循环是诊断现场，但它依赖前面的 history 查询模型。
## 5. 核心状态：raw field 不是语义
### 5.1 `TimeLineHistoryEntry`
`src/include/access/timeline.h` 中的结构很小：
```text
tli
begin
end
```
但语义不是三个字段相加。它表示：
```text
在这一条 target history 中，
所有 begin <= LSN < end 的 WAL 都属于 tli。
```
`begin` 是 inclusive。`end` 是 exclusive。`end = InvalidXLogRecPtr` 表示这条 target timeline 的 tip 还没有上界。
列表顺序是 newest-to-oldest。这点诊断时很重要。如果 history 是：
```text
tli 3: begin 0/9000000, end infinity
tli 2: begin 0/5000000, end 0/9000000
tli 1: begin invalid, end 0/5000000
```
那么 `0/8000000` 属于 timeline 2。`0/4000000` 属于 timeline 1。`0/A000000` 属于 timeline 3。
### 5.2 `expectedTLEs`
`expectedTLEs` 是 `xlogrecovery.c` 里的恢复期内存状态。它来自：
```text
readTimeLineHistory(recoveryTargetTLI)
```
它表达的是：
```text
本次恢复目标 timeline 所认可的完整父链。
```
它不是：
```text
pg_wal 目录里实际存在的所有 timeline；
archive 中所有 timeline；
上游 primary 当前 timeline；
任意可选择的 timeline 集合。
```
诊断时如果把 `expectedTLEs` 理解成“发现了哪些文件”，会误判。它是“应该接受哪些历史区间”。
文件只是满足这个 history 的候选来源。
### 5.3 `recoveryTargetTLI`
`recoveryTargetTLI` 是 startup process 当前决定追的目标 TLI。它可能来自三种路径：
```text
control file 默认值；
recovery_target_timeline = numeric；
recovery_target_timeline = latest 的 findNewestTimeLine() 结果。
```
numeric 目标会调用 `existsTimeLineHistory()` 验证。`latest` 会从 control file 的 timeline 起向后探测 history 文件。
这个差别决定了诊断策略：
```text
numeric:
  目标是管理员显式指定，缺 history 会直接说明目标不存在。
latest:
  目标取决于当前可见的 history 文件集合，
  archive gap 可能让 latest 停在旧 timeline。
```
### 5.4 `curFileTLI` 和 read source
`xlogrecovery.c` 在读 WAL 时还维护：
```text
curFileTLI
currentSource
readSource
XLogReceiptSource
```
`curFileTLI` 是当前或前一个成功读到的 WAL 文件 TLI。它不是 target TLI。它也不是上游 current TLI。
它用来防止恢复过程中 timeline 倒退。
例如从 timeline 2 的文件读过后，不能在同一条顺序恢复流中又回到 timeline 1 的较新 segment。
`currentSource` 在这些来源之间切换：
```text
XLOG_FROM_ARCHIVE
XLOG_FROM_PG_WAL
XLOG_FROM_STREAM
```
这解释了一个常见现场：
```text
同一个缺 WAL 位置，
日志可能先显示 restore_command 失败，
再启动 walreceiver，
再回到 archive retry。
```
这不是三个独立问题。这是 `WaitForWALToBecomeAvailable()` 的状态机。
## 6. `readTimeLineHistory()` 的延迟暴露特性
`readTimeLineHistory(targetTLI)` 是 timeline 诊断的第一站。它的关键行为是：
```text
targetTLI == 1:
  直接返回 timeline 1 的单 entry。
ArchiveRecoveryRequested:
  先尝试 RestoreArchivedFile(<targetTLI>.history, RECOVERYHISTORY)。
文件存在:
  解析每一行 parentTLI + switchpoint。
文件不存在且 errno == ENOENT:
  不立刻 FATAL；
  假设该 timeline 没有父 timeline，
  返回只有 targetTLI 的单 entry。
```
这个行为很容易误导排障。非 timeline 1 的 `.history` 缺失，不一定在 `readTimeLineHistory()` 这一刻报错。
它可能先形成一个“孤立 timeline”的假 history。之后在这些地方暴露：
```text
checkpoint 不在 target history 中；
minRecoveryPoint 不在 target history 中；
walsender 查找 requested timeline 的 switchpoint；
XLogFileReadAnyTLI 找不到合适 segment；
streaming 上游返回不了 TIMELINE_HISTORY。
```
为什么源码允许这个退化？因为恢复早期有一个现实问题：
```text
如果 archive 里还没有 history 文件，
但 streaming 上游稍后能发送它，
startup process 不应该太早把恢复路径判死。
```
`XLogFileReadAnyTLI()` 的注释也保留了这个 awkwardness：
```text
如果 expectedTLEs 还没初始化，
会临时 readTimeLineHistory(recoveryTargetTLI)，
但只有真正找到 valid segment 时才保存到 expectedTLEs。
```
这样一来，如果 archive 和 pg_wal 都没有 segment，后续 streaming 还有机会把 history 文件带过来。诊断含义是：
```text
missing .history 不是一个单点结论。
要看它是 numeric target 验证阶段缺，
还是 source retry 阶段缺，
还是从错误 upstream 请求 TIMELINE_HISTORY 时缺。
```
## 7. `tliOfPointInHistory()` 与 `tliSwitchPoint()` 如何判案
`tliOfPointInHistory(ptr, history)` 做区间查询。伪代码语义是：
```text
for tle in history:
  if begin invalid or begin <= ptr:
    if end invalid or ptr < end:
      return tle->tli
```
它回答的是：
```text
在这条 target history 中，这个 LSN 应该读哪个 TLI 的 WAL？
```
它不检查文件是否存在。它不连接 upstream。它只使用 history 区间。`tliSwitchPoint(tli, history, &nextTLI)` 做另一个查询：
```text
这个 tli 是否在 history 中；
如果在，history 在哪个 LSN 从它切到下一条 timeline；
如果不在，报 requested timeline is not in this server's history。
```
`tliSwitchPoint()` 的错误非常适合判断 wrong upstream 或 wrong target。因为它意味着：
```text
调用方请求了一个 TLI；
但本机当前 history 链上根本没有这个 TLI。
```
在 `walsender.c` 的 `StartReplication()` 中，这个查询用于：
```text
客户端 START_REPLICATION ... TIMELINE N；
walsender 读取本机 FlushTLI 的 history；
如果 N 不是本机 history 的一部分，就拒绝。
```
在 `xlogrecovery.c` 的启动校验中，这个查询用于：
```text
checkpoint record 所在的 TLI 与 target history 区间不一致；
系统需要解释本机 checkpoint 与 requested timeline 的分叉关系。
```
因此，排障时看到 requested timeline，要先问：
```text
是谁在 request？
```
如果是 startup process：
```text
大概率是 recovery_target_timeline 与 pg_control / backup_label 冲突。
```
如果是 standby 的 walreceiver 对 upstream 发起 replication：
```text
大概率是 primary_conninfo 指向的 upstream 不在目标历史上。
```
如果是人工 `pg_receivewal` 或复制客户端：
```text
大概率是客户端指定了上游不认可的 TIMELINE。
```
同一句错误，调用方不同，诊断方向不同。
## 8. 恢复启动阶段：先验证本机能否属于 target history
恢复启动时，`xlogrecovery.c` 会读取 checkpoint。checkpoint 来源可能是：
```text
backup_label；
pg_control。
```
读取出两个关键事实：
```text
CheckPointLoc
CheckPointTLI
```
如果开启 archive recovery，它会根据恢复配置决定 `recoveryTargetTLI`。然后做第一个 correctness check：
```text
tliOfPointInHistory(CheckPointLoc, expectedTLEs) == CheckPointTLI
```
如果不成立，就说明：
```text
这个数据目录的最新 checkpoint 不属于 requested target timeline 的历史。
```
源码报：
```text
requested timeline %u is not a child of this server's history
```
detail 会说明：
```text
Latest checkpoint in file "backup_label" or "pg_control"
is at LSN on timeline X,
but in the history of requested timeline,
the server forked off from that timeline at switchpoint.
```
这是 wrong recovery target 的典型证据。它不是 archive gap。
因为系统已经读到了 history，足以知道目标 timeline 和本机 checkpoint 不兼容。紧接着还会校验：
```text
ControlFile->minRecoveryPoint
ControlFile->minRecoveryPointTLI
```
如果 min recovery point 不属于 requested timeline：
```text
requested timeline %u does not contain minimum recovery point ...
```
这同样偏向 wrong target。`minRecoveryPoint` 的含义是：
```text
这个数据目录至少要恢复到这里，才能保证一致性。
```
如果 requested target 在这之前已经从另一个 timeline 分叉，就不可能用这个数据目录恢复到那个目标。诊断动作：
```text
pg_controldata $PGDATA
  -> 看 Latest checkpoint location
  -> 看 Latest checkpoint's TimeLineID
  -> 看 Latest checkpoint's PrevTimeLineID
  -> 看 Minimum recovery ending location
  -> 看 Min recovery ending loc's timeline
```
然后手工读目标 `.history`。判断：
```text
checkpoint LSN 在 history 区间中应属于哪个 TLI？
minRecoveryPoint - 1 在 history 区间中应属于哪个 TLI？
```
如果算出来的 TLI 与 `pg_controldata` 不同，先不要补 WAL。先修正 target 或换 base backup。
## 9. WAL source state machine：缺文件如何表现
`WaitForWALToBecomeAvailable()` 的状态机按这个顺序推进：
```text
archive / pg_wal
  -> 检查 promotion trigger
  -> streaming
  -> rescan latest timeline
  -> sleep wal_retrieve_retry_interval
  -> 回到 archive / pg_wal
```
正常缺归档时，你会看到类似模式：
```text
restore_command 尝试某个 0000000N.history 或 WAL segment；
命令失败或返回非 0；
startup process 再尝试 pg_wal；
standby mode 下尝试 walreceiver；
如果所有来源都没有，打印 waiting for WAL to become available at LSN。
```
`XLogFileReadAnyTLI()` 负责按 `expectedTLEs` 搜索 segment。它不是只找 `recoveryTargetTLI`。它会遍历 history 中的 TLI：
```text
newest-to-oldest
  -> 跳过比 curFileTLI 更旧的 TLI
  -> 根据 begin segment 判断这个 segment 是否可能属于该 timeline
  -> 先 archive，再 pg_wal
```
这解释了两个重要边界。第一，恢复到 timeline 3 时，早期 segment 仍可能来自 timeline 1 或 2。
所以 archive 里只有最新 TLI 的文件是不够的。还要保留父 timeline 上分叉点之前需要的 WAL。
第二，`curFileTLI` 防止恢复倒退。
如果前一个成功读取的文件已经来自 timeline 2，后面同一顺序流不应该又读 timeline 1 的更后段。
因此 `XLogFileReadAnyTLI()` 会在 `tli < curFileTLI` 时停止扫描。archive gap 的证据通常是：
```text
expectedTLEs 与 pg_controldata 兼容；
target history 存在且 switchpoint 合理；
但 RestoreArchivedFile() / pg_wal / streaming 都找不到所需 segment；
日志反复 waiting for WAL to become available。
```
这时补文件才有意义。如果 target history 本身不兼容，补再多 segment 也不会改变 `tliOfPointInHistory()` 的判断。
## 10. `restore_command` 与 history 文件缺失
`RestoreArchivedFile()` 有几个诊断上必须记住的行为。第一，非 archive recovery 会忽略 `restore_command`。
也就是说 crash recovery 下缺文件，不要从 `restore_command` 方向查。第二，standby mode 中 `restore_command` 可以为空。
这时缺 archive 不一定是配置错误。系统还可能通过 streaming 获得 WAL 和 history。
第三，archive recovery 优先用 archive 版本，即使 `pg_wal` 有同名文件。
源码注释说这是为了避免 base backup 里残留的旧、未填满或部分填充 segment 被误用。
第四，history 文件有更高归档优先级。`XLogArchiveNotify()` 发现 `IsTLHistoryFileName()` 会 `PgArchForceDirScan()`。
这是为了降低 promotion 后新 `.history` 迟迟未归档导致下游选择冲突 timeline 的概率。但这不是强一致保证。
外部归档系统仍然可能漏传、延迟或被清理。判断 missing `.history` 的关键是上下文。如果日志在配置校验阶段报：
```text
recovery target timeline N does not exist
```
且你显式设置了：
```text
recovery_target_timeline = 'N'
```
这表示 `existsTimeLineHistory(N)` 没找到。可能是 archive gap，也可能是你指定了根本不存在的 N。
下一步要去 archive、上游 `pg_wal`、目标 primary 的 `pg_wal` 查：
```text
0000000N.history
```
如果目标 primary 也没有，说明目标错。如果目标 primary 有，archive 没有，说明 archive gap。
如果日志是在 walreceiver 获取 history 时出现：
```text
could not receive timeline history file from the primary server
```
则更偏向 wrong upstream 或 upstream 缺本地 history 文件。因为此时请求通过 replication protocol 的：
```text
TIMELINE_HISTORY N
```
发给了 upstream。正确的 upstream 应该能从 `pg_wal/0000000N.history` 读出内容。
## 11. `primary_conninfo` 指错上游的源码症状
walreceiver 连接 upstream 后，先做 identity 检查。它会比较：
```text
primary system identifier
standby system identifier
```
如果不同，直接报：
```text
database system identifier differs between the primary and standby
```
这个错误容易诊断。更危险的是 system identifier 相同。同一个 base backup 分叉出的多个节点都会相同。
这时还要检查 timeline。walreceiver 会确认：
```text
primaryTLI >= startpointTLI
```
如果 primary 最高 timeline 比本机请求的低：
```text
highest timeline %u of the primary is behind recovery timeline %u
```
这是明确的 wrong upstream。但还有更隐蔽的情况：
```text
primaryTLI 足够高，
但它的 history 不是本机目标 history 的后代。
```
这时可能在后续阶段暴露。`WalRcvFetchTimeLineHistoryFiles(startpointTLI, primaryTLI)` 会请求缺失 history 文件。
如果 upstream 没有本机需要的 history，`TIMELINE_HISTORY` 失败。
如果 upstream 能返回某个 history，但它的 switchpoint 与本机 expectedTLEs 不兼容，startup process 后续校验会失败。
`libpqrcv_startstreaming()` 的注释还说明了一个重要返回：
```text
返回 false 表示服务器成功执行 START_REPLICATION，
但没有进入 copy-both mode；
含义是 requested timeline/startpoint 上没有 WAL，
因为服务器在该点或之前已经切到了另一条 timeline。
```
walreceiver 会记录：
```text
primary server contains no more WAL on requested timeline %u
```
这句话不总是错误。
如果本机设置 `recovery_target_timeline = latest`，startup process 可能随后发现新 history，切换 target timeline 并要求 walreceiver 重启。
但如果它反复出现在同一个 requested timeline，并且没有新 target timeline 日志，就要怀疑：
```text
primary_conninfo 指到的节点不再提供这条历史；
或者本机 recoveryTargetTLI 没有跟上 upstream 的新 timeline。
```
诊断动作：
```text
在 standby 日志中找 started/restarted WAL streaming at LSN on timeline N；
找 fetching timeline history file for timeline N from primary server；
找 primary server contains no more WAL on requested timeline N；
对 upstream 执行 IDENTIFY_SYSTEM，确认 timeline 和 xlogpos；
在 upstream pg_wal 中确认 0000000N.history 是否存在。
```
如果你有多个候选 upstream，要逐个确认 `.history` 父链。不要只看谁的 LSN 最大。
LSN 大但在错误分支上，是更危险的上游。
## 12. walsender 如何拒绝不属于自己历史的请求
错误上游的另一面是 walsender。客户端发起：
```text
START_REPLICATION ... TIMELINE N
```
`walsender.c` 的 `StartReplication()` 先选择本机当前 flush timeline：
```text
primary:
  GetFlushRecPtr(&FlushTLI)
cascading standby:
  GetStandbyFlushRecPtr(&FlushTLI)
```
如果客户端请求的 `cmd->timeline` 等于 `FlushTLI`，发送当前 timeline。如果不同，就认为客户端请求 historic timeline。
此时读取：
```text
readTimeLineHistory(FlushTLI)
```
然后查询：
```text
tliSwitchPoint(cmd->timeline, timeLineHistory, &sendTimeLineNextTLI)
```
如果 requested timeline 不在本机 history 中，就报：
```text
requested timeline N is not in this server's history
```
如果在，但客户端 startpoint 超过了本机从该 timeline 分叉的 switchpoint：
```text
requested starting point ... on timeline N is not in this server's history
```
detail 会说本机从 timeline N 的哪个 LSN 分叉。这给 wrong upstream 一个很强证据：
```text
standby 请求 N；
upstream 的 FlushTLI history 中没有 N，
或者 upstream 在请求 startpoint 之前已经从 N 分叉。
```
如果 upstream 是级联 standby，还要注意发送中的 timeline 可能变成 historic。`walsender.c` 后续会在发送循环中检测：
```text
本机 promotion；
或者 recovery 目标 timeline 改变；
```
然后重新读新 timeline history，设置：
```text
sendTimeLineValidUpto
sendTimeLineNextTLI
sendTimeLineIsHistoric
```
这保证 walsender 不会在旧 timeline 的 switchpoint 之后继续发送旧历史。
诊断时如果下游说 upstream 断流，upstream 可能不是坏了。
它可能只是正确地在 historic timeline 的 `sendTimeLineValidUpto` 停止。
## 13. WAL segment timeline mismatch 的两层含义
“WAL segment belongs to a different timeline” 在现场可能指两种不同层次。第一层是 history 层：
```text
根据 expectedTLEs，这个 LSN 应该属于 timeline X；
但前一个已恢复 WAL 文件来自 timeline Y。
```
`xlogrecovery.c` 的错误是：
```text
according to history file, WAL location ... belongs to timeline X,
but previous recovered WAL file came from timeline Y
```
这通常说明：
```text
archive / pg_wal / streaming 中混入了另一条分支的 segment；
或者 expectedTLEs 被错误 target history 引导；
或者 restore_command 用了不按 %f 精确取文件的错误脚本。
```
第二层是 WAL page header 层。`xlogreader.c` 会校验：
```text
xlp_pageaddr 是否等于预期 recptr；
页面 timeline 是否不会倒退。
```
可能出现：
```text
unexpected pageaddr ...
out-of-sequence timeline ID ...
```
这类错误更像：
```text
拿到了错误 segment 文件；
segment 被覆盖、截断、复制错目录；
用错误 wal_segment_size 或错误集群的文件；
从另一个 timeline 同名 log/seg 拿了文件。
```
注意 WAL segment 文件名的前 8 位 TLI 很重要：
```text
000000010000000000000013
000000020000000000000013
```
这两个不是同一个文件的两个名字。
如果 restore_command 忽略 `%f` 中的 TLI，只按后 16 位查找，就会把错误 timeline 的 segment 拿回来。
这种脚本 bug 会表现得像 archive gap 或 WAL corruption。真正诊断要看：
```text
恢复要求的文件名完整是什么；
archive 返回的文件名和内容是否匹配；
pg_waldump -t N 是否能按该 TLI 读出；
segment 内 page header 的 TLI 是否出现倒退。
```
## 14. `pg_controldata`：先定本机事实
`pg_controldata` 是 timeline 诊断的第一条命令。它来自 `src/bin/pg_controldata/pg_controldata.c`。本节关心这些输出：
```text
Database system identifier
Database cluster state
Latest checkpoint location
Latest checkpoint's REDO location
Latest checkpoint's REDO WAL file
Latest checkpoint's TimeLineID
Latest checkpoint's PrevTimeLineID
Minimum recovery ending location
Min recovery ending loc's timeline
Backup start location
Backup end location
```
`Latest checkpoint's TimeLineID` 回答：
```text
本机最新 checkpoint record 认为自己在哪条 timeline。
```
`Latest checkpoint's PrevTimeLineID` 回答：
```text
如果 checkpoint 是 end-of-recovery 相关边界，
它从哪个旧 timeline 分叉。
```
`Minimum recovery ending location` 和对应 timeline 回答：
```text
这个数据目录至少要恢复到哪条历史上的哪个 LSN 才一致。
```
诊断流程：
```text
1. 记录 system identifier。
2. 记录 checkpoint LSN + checkpoint TLI。
3. 记录 minRecoveryPoint + minRecoveryPointTLI。
4. 找 recovery_target_timeline 当前值。
5. 读取目标 <TLI>.history。
6. 用 history 区间计算 checkpoint LSN 应属 TLI。
7. 用 history 区间计算 minRecoveryPoint - 1 应属 TLI。
```
如果第 6 或第 7 步不一致：
```text
wrong recovery target 优先。
```
如果一致：
```text
再查 archive / streaming 文件来源。
```
不要跳过 `pg_controldata` 直接看 archive。
很多 failover 后的误操作，本质是用一个已经越过分叉点的数据目录，去追另一个早已分叉的 target timeline。
这种情况下 archive 里文件再完整，也不能恢复。
## 15. `.history` 文件：手工读法
history 文件每一行是：
```text
parentTLI    switchpoint    reason
```
例如 `00000003.history`：
```text
1    0/5000000    no recovery target specified
2    0/9000000    before 2026-06-11 10:00:00+08
```
它表示：
```text
timeline 3 的 history 是：
  timeline 1: begin invalid, end 0/5000000
  timeline 2: begin 0/5000000, end 0/9000000
  timeline 3: begin 0/9000000, end infinity
```
读 history 时不要把 switchpoint 理解成“父 timeline 最后一条 record 的 LSN”。它是区间边界：
```text
LSN < switchpoint:
  仍属于父 timeline。
LSN >= switchpoint:
  属于 child timeline。
```
源码中 `end` 是 exclusive。所以诊断 minRecoveryPoint 时源码用：
```text
ControlFile->minRecoveryPoint - 1
```
因为 min recovery point 本身是恢复结束边界。手工判断时也要注意这个 off-by-one。一个实用表格：
| 观察 | 可能含义 |
| --- | --- |
| target N 的 history 文件不存在，目标 primary 也没有 | 目标 TLI 可能写错。 |
| target N 的 history 文件不存在，目标 primary 有 | archive gap 或 history 未归档。 |
| history 存在，但 checkpoint LSN 在父 timeline 的分叉点之后 | 当前 base backup 不属于 target history。 |
| history 存在，checkpoint/minRecoveryPoint 都兼容，但 segment 缺 | WAL archive gap。 |
| history 存在，本机兼容，streaming 上游拒绝 requested timeline | primary_conninfo 指错分支。 |
## 16. `pg_waldump`：验证 segment 内容，而不是替代 history 判断
`pg_waldump` 适合回答：
```text
这个具体 WAL segment 能不能按某个 timeline 和 LSN 被读出？
```
它不替你判断 recovery target 是否正确。但它能验证 archive 返回的文件是否像你以为的那样。常用方式：
```bash
pg_waldump -p /path/to/archive -t 2 -s 0/5000000 000000020000000000000005
```
`pg_waldump.c` 的 `-t/--timeline` 含义是：
```text
timeline from which to read WAL records
```
如果命令行传入 STARTSEG，工具会从文件名解析 timeline。如果显式 `-t`，就按指定 timeline 配置 reader。
诊断时关注三类输出。第一，能正常读到 record：
```text
说明 segment 至少局部可读，
但还要回到 history 判断它是否属于目标路径。
```
第二，报 start WAL location 不在文件内：
```text
说明 LSN 与 segment 边界算错。
```
第三，报 pageaddr 或 timeline 相关 invalid record：
```text
说明文件内容与请求的 segment / timeline 序列不匹配，
要怀疑 archive 取错文件、压缩解压错误、restore_command 脚本忽略 TLI。
```
`pg_waldump` 的正确用法是和 `pg_controldata` 配对。先用 `pg_controldata` 确定本机要从哪个 LSN/TLI 继续。
再用 `.history` 确定这个 LSN 应属哪个 segment 文件名。最后用 `pg_waldump` 验证 archive 里的那个文件内容。
如果顺序反过来，很容易因为某个错误分支上的 WAL 可读，就误以为它可用于当前恢复。
## 17. 日志诊断：按调用方归类
同样的 timeline 字样，在不同进程里含义不同。
### 17.1 startup process
startup process 日志常见线索：
```text
starting archive recovery
entering standby mode
new target timeline is N
waiting for WAL to become available at X/Y
requested timeline N is not a child of this server's history
requested timeline N does not contain minimum recovery point ...
according to history file, WAL location ... belongs to timeline ...
```
这类日志先看本机：
```text
recovery_target_timeline
pg_control / backup_label
expected history
archive / pg_wal source
```
### 17.2 walreceiver
walreceiver 日志常见线索：
```text
started streaming WAL from primary at X/Y on timeline N
restarted WAL streaming at X/Y on timeline N
fetching timeline history file for timeline N from primary server
primary server contains no more WAL on requested timeline N
database system identifier differs between the primary and standby
highest timeline N of the primary is behind recovery timeline M
```
这类日志先看 upstream：
```text
primary_conninfo 实际连到谁；
upstream IDENTIFY_SYSTEM 的 systemid/timeline/xlogpos；
upstream pg_wal 中是否有请求的 .history；
upstream history 是否包含 standby 请求的 timeline。
```
### 17.3 walsender
walsender 日志或客户端错误常见线索：
```text
requested timeline N is not in this server's history
requested starting point X/Y on timeline N is not in this server's history
```
这类日志说明：
```text
客户端请求的 timeline/startpoint 与 walsender 本机 FlushTLI 的 history 不兼容。
```
如果客户端就是你的 standby，问题往往在 `primary_conninfo`。如果客户端是人工工具，检查工具传入的 `TIMELINE`。
## 18. 诊断主流程：从现场到结论
下面是一条可执行的线性流程。第一步，固定本机事实：
```bash
pg_controldata $PGDATA
```
记录：
```text
system identifier
Latest checkpoint location
Latest checkpoint's TimeLineID
Minimum recovery ending location
Min recovery ending loc's timeline
Database cluster state
```
第二步，固定目标：
```text
查看 postgresql.conf / auto.conf 中 recovery_target_timeline；
确认是 numeric、latest，还是默认 controlfile。
查看是否有 recovery_target_time、recovery_target_lsn、recovery_target_name。
```
第三步，固定 target history：
```text
找到 0000000N.history。
如果 N = 1，没有 history 文件是正常的。
如果 N > 1，分别检查：
  pg_wal；
  archive；
  预期 upstream 的 pg_wal。
```
第四步，做区间判断：
```text
checkpoint LSN 在 target history 中应属哪个 TLI？
minRecoveryPoint - 1 在 target history 中应属哪个 TLI？
```
如果与 `pg_controldata` 不一致：
```text
结论偏向 recovery_target_timeline 或恢复目标错误。
```
第五步，验证 WAL source：
```text
根据 history 计算当前缺失 LSN 应使用的 TLI；
拼出完整 WAL segment 文件名；
确认 archive / pg_wal 是否存在该文件；
用 pg_waldump -t 验证可读性。
```
如果 history 一致但文件缺：
```text
结论偏向 archive gap / WAL retention gap。
```
第六步，验证 upstream：
```text
确认 primary_conninfo 实际连接的 host/port；
在 upstream 执行 IDENTIFY_SYSTEM 或查看日志；
确认 upstream current timeline >= standby requested timeline；
确认 upstream 的 history 包含 standby 请求的 startpoint timeline；
确认 upstream 能返回 TIMELINE_HISTORY N。
```
如果本机 history 与目标一致，archive 也不是唯一来源，但 upstream 拒绝：
```text
结论偏向 primary_conninfo 指错上游。
```
第七步，决定修复动作：
```text
archive gap:
  补 history/WAL；
  修 restore_command；
  修 archive retention。
wrong target:
  改 recovery_target_timeline；
  改 PITR 目标；
  换 base backup。
wrong upstream:
  改 primary_conninfo；
  断开错误 cascading path；
  必要时重建 standby。
```
## 19. 案例一：显式 target timeline 但 history 缺失
现场：
```text
recovery_target_timeline = '5'
启动时报 recovery target timeline 5 does not exist
archive 中没有 00000005.history
```
源码路径：
```text
xlogrecovery.c
  -> recoveryTargetTimeLineGoal == NUMERIC
  -> existsTimeLineHistory(5)
     -> RestoreArchivedFile(00000005.history, RECOVERYHISTORY)
     -> AllocateFile()
  -> false
  -> FATAL
```
诊断不要立刻说“归档坏了”。先问：
```text
timeline 5 是否真的应该存在？
```
如果新 primary 的 `pg_wal` 里有 `00000005.history`：
```text
archive gap。
```
修复方向是补 history 文件并检查 archiver。如果所有候选 primary 都没有：
```text
recovery target 错。
```
也可能是管理员把另一套环境的 TLI 写进了配置。
如果存在 `00000004.history`，没有 `00000005.history`，且目标是“追最新”：
```text
考虑改成 recovery_target_timeline = 'latest'，
但前提是你的 base backup 属于这条 latest history。
```
## 20. 案例二：checkpoint 已越过分叉点
现场：
```text
pg_controldata:
  Latest checkpoint location: 0/7000000
  Latest checkpoint's TimeLineID: 1
00000002.history:
  1    0/5000000    ...
recovery_target_timeline = '2'
```
手工判断：
```text
timeline 2 在 0/5000000 从 timeline 1 分叉。
0/7000000 在 target history 中应该属于 timeline 2。
但 pg_control 说 checkpoint 在 timeline 1。
```
源码路径：
```text
tliOfPointInHistory(0/7000000, expectedTLEs) returns 2
CheckPointTLI is 1
```
结果：
```text
requested timeline 2 is not a child of this server's history
```
这不是缺 WAL。这是数据目录已经沿 timeline 1 走到了分叉点之后。它不能被带到 timeline 2 的未来。修复方向：
```text
使用分叉点之前的 base backup；
或者恢复到 timeline 1；
或者从 timeline 2 的 primary 重建 standby。
```
## 21. 案例三：standby 接到了旧 primary
现场：
```text
failover 后，新 primary 是 S1，timeline 2。
S2 的 primary_conninfo 仍指向旧 P。
P 的 system identifier 与 S2 相同。
P 仍在 timeline 1。
```
walreceiver 可能报：
```text
highest timeline 1 of the primary is behind recovery timeline 2
```
或者 upstream walsender 报：
```text
requested timeline 2 is not in this server's history
```
源码路径：
```text
walreceiver:
  IDENTIFY_SYSTEM
  -> systemid 相同
  -> primaryTLI < startpointTLI
  -> ERROR
walsender:
  StartReplication(TIMELINE 2)
  -> readTimeLineHistory(FlushTLI = 1)
  -> tliSwitchPoint(2, history_of_1)
  -> ERROR
```
判断标准：
```text
如果 standby 的 target history 合理，
但 upstream 的 current timeline/history 不包含 standby 请求，
就是 wrong upstream。
```
修复方向：
```text
把 primary_conninfo 指向 timeline 2 的新 primary；
确认 replication slot 是否也在新 primary 上存在；
如果 S2 已经从旧 P 拉入分叉点后的 timeline 1 WAL，要评估是否需要重建。
```
不要只修改 `restore_command`。错误上游会继续提供错误未来。
## 22. 案例四：restore_command 返回了错误 TLI 的 segment
现场：
```text
target history 判断当前缺失 LSN 应属于 timeline 2。
restore_command 请求 000000020000000000000013。
脚本实际从对象存储返回了 000000010000000000000013 的内容。
```
可能日志：
```text
unexpected pageaddr ...
out-of-sequence timeline ID ...
according to history file, WAL location ... belongs to timeline 2,
but previous recovered WAL file came from timeline 1
```
源码层次：
```text
XLogFileReadAnyTLI()
  -> 选择 timeline 2 的 segment 文件名
RestoreArchivedFile()
  -> 执行 restore_command
  -> 只知道命令返回成功和目标文件 stat/size
xlogreader.c
  -> 读取 page header
  -> 发现内容不符合预期顺序
```
这类问题最容易被误诊成 WAL 损坏。实际常见原因是脚本：
```text
只按 log/seg 后 16 位查对象；
没有把文件名前 8 位 TLI 纳入 key；
解压缓存复用了旧文件；
恢复到 RECOVERYXLOG 临时名时没有覆盖成功；
对象存储里同名别名指向错误内容。
```
诊断动作：
```text
把 restore_command 实际返回的文件复制出来；
比较文件名期望 TLI；
pg_waldump -t 2 读；
再 pg_waldump -t 1 读；
检查脚本日志中使用的是完整 %f 还是截断名称。
```
如果 `-t 1` 能读，`-t 2` 不能读，且目标 history 要求 timeline 2：
```text
archive 内容串线。
```
## 23. 成本、资源与跨模块传播
timeline 诊断看起来是控制面问题，但会影响资源。archive gap 会导致：
```text
startup process 反复执行 restore_command；
walreceiver 反复连接 / 重启；
日志和外部归档系统压力增加；
hot standby replay 停滞；
replication slot restart_lsn 不能前进；
上游 pg_wal 保留压力上升。
```
wrong target 会导致：
```text
实例启动 FATAL；
反复重启也不会推进；
如果自动化脚本只补 WAL，会浪费恢复窗口。
```
wrong upstream 会导致：
```text
walreceiver 连接成功但无法获得正确 WAL；
级联 standby 继续等待；
如果错误 upstream 仍可写，业务可能发生 split-brain 风险。
```
这就是为什么诊断要先分层。不同层次的错误修复成本差别很大。补一个 `.history` 文件是小动作。
重建 standby 是大动作。把 standby 接回错误 upstream 则可能造成数据安全事故。
## 24. 常见误区
误区一：
```text
system identifier 相同，所以 upstream 一定正确。
```
错误。system identifier 只证明同一集群血统，不证明 timeline history 一致。误区二：
```text
recovery_target_timeline = latest 总是最安全。
```
错误。`latest` 只在当前可见 history 文件集合里找最新。如果 archive 缺 history，它可能看不到真正最新。
如果本机 checkpoint 已经越过另一个分叉点，最新 timeline 也可能不包含本机。误区三：
```text
缺 0000000N.history 一定是 archive gap。
```
不一定。也可能 N 根本不存在。也可能 standby 连到的 upstream 不是目标 primary。误区四：
```text
WAL segment 文件能被 pg_waldump 读出，就能用于当前恢复。
```
不够。还要看它是否属于 target history 的当前 LSN 区间。误区五：
```text
只要补齐最新 timeline 的 WAL segment 即可。
```
错误。恢复到 child timeline 时，分叉点之前仍需要父 timeline 的 WAL。误区六：
```text
requested timeline 错误只和 recovery_target_timeline 有关。
```
不够。walsender 也会对客户端请求的 `START_REPLICATION ... TIMELINE` 报同类错误。必须看日志进程和调用方。
## 25. 课堂实验
实验一：手工构造 history 区间。
```text
给定：
  00000003.history:
    1    0/5000000
    2    0/9000000
判断：
  0/4000000 属于哪个 TLI？
  0/5000000 属于哪个 TLI？
  0/8FFFFFF 属于哪个 TLI？
  0/9000000 属于哪个 TLI？
```
预期答案：
```text
0/4000000 -> 1
0/5000000 -> 2
0/8FFFFFF -> 2
0/9000000 -> 3
```
实验二：用 `pg_controldata` 判断 target 是否可能。
```text
pg_controldata $PGDATA
cat $ARCHIVE/00000002.history
```
把 checkpoint LSN 放进 history 区间。如果算出的 TLI 与 checkpoint TLI 不同，写出原因。实验三：验证 archive 是否串线。
```text
pg_waldump -p $ARCHIVE -t 2 -s 0/13000000 000000020000000000000013
pg_waldump -p $ARCHIVE -t 1 -s 0/13000000 000000010000000000000013
```
比较：
```text
哪个能读；
错误是否是 pageaddr；
archive 中两个文件是否大小相同；
restore_command 是否按完整 %f 取文件。
```
实验四：验证 upstream history。
```text
psql "replication=database ..." -c "IDENTIFY_SYSTEM"
```
然后在 upstream 上检查：
```text
ls pg_wal/*.history
cat pg_wal/0000000N.history
```
判断 standby 请求的 startpoint timeline 是否在 upstream 当前 FlushTLI 的 history 里。实验五：阅读源码断点。建议断点：
```text
readTimeLineHistory
tliOfPointInHistory
tliSwitchPoint
WaitForWALToBecomeAvailable
XLogFileReadAnyTLI
WalRcvFetchTimeLineHistoryFiles
StartReplication
```
观察：
```text
recoveryTargetTLI；
expectedTLEs；
curFileTLI；
currentSource；
cmd->timeline；
sendTimeLineValidUpto。
```
## 26. 讨论题
1. 为什么 timeline fork 诊断不能只比较两个节点的 latest checkpoint LSN，而必须同时比较 `.history` 中的 fork point？
2. 如果 `restore_command` 能返回文件但 timeline 不匹配，你会先怀疑 archive 命名、recovery target，还是上游节点选错？对应源码入口分别在哪里？
3. walsender 拒绝 timeline 请求时，它保护的是“文件存在性”还是“历史合法性”？这两种解释会导致什么不同的修复动作？

## 27. 本节小结
timeline fork 故障不能按错误字符串直接修。
同一个 `requested timeline` 可以来自 startup process、walreceiver、walsender 或人工复制客户端。
同一个 missing `.history` 可以是 archive gap、目标 TLI 不存在，也可以是 upstream 不在目标分支。
同一个 WAL segment mismatch 可以是 restore_command 串线，也可以是 recovery target history 错。稳定诊断顺序是：
```text
pg_controldata 定本机事实
  -> recovery_target_timeline 定目标
  -> .history 定区间
  -> tliOfPointInHistory / tliSwitchPoint 定语义
  -> archive / pg_wal / streaming 定来源
  -> pg_waldump 定具体 segment 内容
```
可迁移规律：
```text
带分叉的日志系统中，位置不是身份。
LSN 只给出地址；
timeline history 才给出这段地址属于哪条可接受历史。
诊断必须先恢复 history 语义，再处理文件缺失和网络来源。
```
