# PostgreSQL Timeline ID 与 history 文件
## 课程定位
前置知识：已经理解 startup process 如何在 crash recovery、archive recovery、standby recovery 之间切换，知道 WAL record 用 LSN 串成顺序，物理复制中 walsender 发送 WAL、walreceiver 接收 WAL。
本节唯一主问题：
```text
为什么 promotion 后必须产生新的 timeline，.history 文件如何记录父 timeline、分叉 LSN 和恢复路径，
避免两个不同历史被误认为同一条 WAL 流？
```
核心矛盾：同一个 PostgreSQL system identifier 下，多个从同一 base backup 或同一 standby 分叉出来的节点可以共享早期 WAL 前缀；但 promotion 以后，它们会从某个 LSN 开始写出不同未来。如果只用 system identifier 和 LSN 判断 WAL 身份，两个不同历史会在同一个地址空间里发生碰撞。
PostgreSQL 的解法不是把 LSN 重新编号。
LSN 仍然表示一条抽象 WAL 地址轴。
真正区分历史分叉的是 `TimeLineID` 和 timeline history。
学完后应能判断：一个 WAL segment 文件名中的前 8 位 TLI、`.history` 文件中的 switchpoint、recovery target timeline、walsender 的 historic timeline 发送边界，以及 redo 中的 timeline switch 校验分别解决哪个 correctness 问题。
本课基于本地源码 `/home/nail/postgres`，分支 `master`，提交 `0e1f1ed157e90741e12a3715909e1b2d71ff9344`。
## 1. 本节在总主线中的位置
前面几节已经建立了物理复制和恢复的主线：
```text
primary 写 WAL
  -> walsender 按 LSN 发送 WAL
  -> walreceiver 写入 pg_wal
  -> startup process 读取 WAL 并 redo
  -> promotion 让 standby 退出 recovery 并开始本地写 WAL
```
这条线还有一个没有解释的断点：
```text
standby promotion 之后，它不再只是复制旧 primary 的未来；
它开始创造自己的未来。
```
如果旧 primary 之后也继续写 WAL，或者另一个 standby 也被提升，系统会出现：
```text
相同 system identifier
相同早期 WAL 前缀
相同 LSN 地址空间
不同分叉点之后的 WAL 内容
```
本节只围绕这个断点展开。
不讲每类 WAL record 的 redo 语义。
不讲选主协议。
不讲同步复制确认策略。
我们只追踪一个状态：
```text
一个节点如何知道某个 LSN 应该从哪条 timeline 上读取？
```
这会把下面几个模块连成一条线：
```text
timeline.c:
  读写 .history 文件，构造 TimeLineHistoryEntry 区间链

xlog.c:
  recovery 结束时选择 newTLI，写 history，初始化新 WAL segment

xlogrecovery.c:
  根据 recoveryTargetTLI 和 expectedTLEs 选择 WAL source，并校验 timeline switch

walsender.c:
  根据客户端请求 timeline 和 switchpoint 限制历史 WAL 发送范围

walreceiver.c:
  从 upstream 拉取缺失的 timeline history 文件
```
本节的 runtime 现象很具体：
```text
promotion 后 pg_wal 中出现 0000000N.history；
新 WAL segment 文件名前 8 位 TLI 增大；
一个 standby 如果缺少 history 文件，可能无法判断请求点是否属于上游历史；
两个节点 system_identifier 相同，不代表它们在同一条 WAL 流上。
```
下一节如果继续进入恢复和复制故障定位，这个模型会变成判断 “missing WAL” 和 “wrong timeline” 的基础。
## 2. 核心矛盾与一句话运行模型
一句话运行模型：
```text
promotion 结束 archive recovery 时，PostgreSQL 选择一个未被使用的新 TimeLineID，
在新 TLI 下复制或创建切换点所在 WAL segment，
写入 <newTLI>.history 记录 parentTLI 和 switchpoint，
后续恢复、复制和 redo 都用该 history 构造 expectedTLEs，
从而把同一个 LSN 地址映射到唯一的历史区间。
```
这里的 tension 是：
```text
LSN 必须保持全局单调地址语义，方便 WAL record、checkpoint、minRecoveryPoint 和复制协议使用；
但 promotion 会让同一地址之后出现多个合法未来，必须给每个未来一个不同身份。
```
如果只用 LSN，不加 timeline，下面两个 WAL record 无法区分：
```text
节点 A:
  systemid = S
  LSN = 0/7000100
  内容 = 在旧 primary 上提交事务 x

节点 B:
  systemid = S
  LSN = 0/7000100
  内容 = standby promotion 后提交事务 y
```
它们都可能来自同一个 base backup。
它们的 system identifier 也相同。
错误地把它们看成一条 WAL 流，会导致 standby 在 redo 中把不属于自己历史的记录应用到数据目录。
PostgreSQL 用三层身份避免这个错误：
```text
system identifier:
  证明是不是同一个集群血统

TimeLineID:
  证明 WAL 文件属于哪条历史分支

timeline history:
  证明某个 TimeLineID 如何从父 timeline 分叉，以及某个 LSN 位于哪条区间
```
这三个层次不能互相替代。
`system_identifier` 相同只是物理复制的基本前提。
`TimeLineID` 相同也不够，因为某个 TLI 的可用范围有边界。
只有 `TimeLineID + switchpoint + history chain` 才能回答：
```text
这个 LSN 在这台服务器认可的恢复路径上吗？
```
## 3. 核心文件分工与阅读顺序
| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/access/timeline.h` | `TimeLineHistoryEntry` 的区间语义，`readTimeLineHistory()`、`tliOfPointInHistory()`、`tliSwitchPoint()` 接口。 |
| 2 | `src/include/access/xlog_internal.h` | WAL 文件名、timeline history 文件名和 `xl_end_of_recovery` 中的 TLI 字段。 |
| 3 | `src/backend/access/transam/timeline.c` | `.history` 文件格式、读写、archive fetch、`findNewestTimeLine()`、区间查询。 |
| 4 | `src/backend/access/transam/xlog.c` | `XLogInitNewTimeline()`、promotion 后 `newTLI` 选择、`writeTimeLineHistory()`、end-of-recovery 记录。 |
| 5 | `src/backend/access/transam/xlogrecovery.c` | `recoveryTargetTLI`、`expectedTLEs`、`rescanLatestTimeLine()`、`checkTimeLineSwitch()`、按 history 读取 WAL。 |
| 6 | `src/backend/access/transam/xlogarchive.c` | `RestoreArchivedFile()`、`KeepFileRestoredFromArchive()`、history 文件 archive notify 的优先级。 |
| 7 | `src/backend/replication/walsender.c` | `IDENTIFY_SYSTEM`、`TIMELINE_HISTORY`、`START_REPLICATION ... TIMELINE` 和 historic timeline 发送边界。 |
| 8 | `src/backend/replication/walreceiver.c` | `WalRcvFetchTimeLineHistoryFiles()` 如何从 upstream 获取缺失 history 文件。 |
阅读时不要从 `timeline.c` 顶部背到尾。
更有效的顺序是沿一个分叉事件读：
```text
standby 结束 recovery
  -> 选择 newTLI
  -> 创建新 TLI 的起始 WAL segment
  -> 写 <newTLI>.history
  -> archive / stream history file
  -> 下游读取 history 构造 expectedTLEs
  -> redo / walsender 用 switchpoint 限制 WAL 范围
```
每个函数都回答一个很窄的问题：
```text
谁声明新历史存在？
谁把父链变成可查询区间？
谁确保新的 WAL 不覆盖旧 archive？
谁拒绝错误分叉？
谁把 history 文件传播给下游？
```
这些问题连起来，才是 timeline 机制。
## 4. WAL 流身份不是 system identifier 加 LSN
`IDENTIFY_SYSTEM` 在 `walsender.c` 中返回四列：
```text
systemid
timeline
xlogpos
dbname
```
`systemid` 来自 `GetSystemIdentifier()`。
它的作用是防止把完全无关的集群接到一起。
如果 system identifier 不同，物理复制应该直接拒绝。
但是 system identifier 相同只说明：
```text
这些节点来自同一个 initdb 或 base backup 血统。
```
它不能说明：
```text
它们 promotion 之后是否继续沿同一条历史前进。
```
一个典型分叉过程是：
```text
primary P 在 timeline 1 写到 0/6000000
standby S1 replay 到 0/5000000 后 promotion
S1 选择 timeline 2，从 0/5000000 之后写新 WAL
P 仍然在 timeline 1 上写到 0/7000000
```
此时 P 和 S1 的 system identifier 相同。
它们在 `0/0` 到 `0/5000000` 之前共享历史。
它们在 `0/5000000` 之后的 WAL 内容不同。
如果一个下游只说：
```text
我需要 systemid S 的 WAL，从 0/6000000 开始。
```
这个请求是不完整的。
它必须再说明：
```text
我需要 timeline 1 上的 0/6000000，
还是 timeline 2 上的 0/6000000？
```
这就是 `TimeLineID` 的最小必要性。
但 `TimeLineID` 也不是单独可用的全局标签。
因为 timeline 2 在 `0/5000000` 之前并没有独立 WAL。
timeline 2 的起始 segment 可能复制了 timeline 1 在 switchpoint 之前的前缀。
因此还要知道：
```text
timeline 2 是从 timeline 1 的哪个 LSN 分叉出来的？
```
这个 LSN 就是 switchpoint。
`.history` 文件把这个 switchpoint 持久化。
## 5. TimeLineID 与 WAL 文件名
`TimeLineID` 是 WAL 文件身份的一部分。
`xlog_internal.h` 中的 `XLogFileName()` 生成 24 字节 WAL segment 文件名：
```text
%08X%08X%08X
  TLI
  log
  seg
```
例如：
```text
000000010000000000000013
000000020000000000000013
```
这两个文件的 log/seg 相同，但 TLI 不同。
它们不表示同一个 WAL segment。
它们只表示：
```text
同一个 LSN 地址区间，在两条不同 timeline 上的物理文件。
```
如果 switchpoint 落在 segment 0x13 中间，两个文件的前半段可以完全相同。
`XLogInitNewTimeline()` 会把旧 timeline 已使用部分复制到新 TLI 的起始 segment。
但这不代表两个文件可以合并。
segment 文件名中的 TLI 是恢复路径的一部分。
恢复读取时必须知道当前 LSN 属于哪条 timeline。
否则同一个 `0000000000000013` 地址区间会落到错误文件。
history 文件名由 `TLHistoryFileName()` 生成：
```text
%08X.history
```
例如：
```text
00000002.history
```
注意 history 文件属于 child timeline。
`00000002.history` 描述的是 timeline 2 如何从父历史走来。
timeline 1 没有 history 文件。
源码中 `readTimeLineHistory(1)` 直接构造一条无父 entry。
这是一个重要边界：
```text
没有 history 文件不一定是错误；
timeline 1 没有 history 文件是正常状态。
```
但 timeline 2 及以后如果缺少 history 文件，语义就要看上下文。
在某些读取路径中，PostgreSQL 会临时假设它没有父 timeline。
在恢复目标校验或 walsender 历史校验中，这通常会很快暴露为 history mismatch。
## 6. `.history` 文件格式不是日志摘要
`timeline.c` 顶部定义了 history 文件的文本格式：
```text
<parentTLI> <switchpoint> <reason>
```
实际分隔符是 tab。
`switchpoint` 用 `XLogRecPtr` 的常见格式：
```text
%X/%08X
```
一个最小例子：
```text
1	0/5000000	no recovery target specified
```
这行如果出现在 `00000002.history` 中，含义是：
```text
timeline 2 的父 timeline 是 1；
timeline 2 从 LSN 0/5000000 开始分叉；
0/5000000 之前的历史属于 timeline 1；
0/5000000 及之后属于 timeline 2，直到再分叉。
```
如果后来 timeline 3 从 timeline 2 分叉，`00000003.history` 会先复制父 history 内容，再追加一行：
```text
1	0/5000000	no recovery target specified
2	0/9000000	recovery target reached
```
这说明 history 文件不是只记录直接父节点。
它记录从 timeline 1 到当前 timeline 的完整父链。
`writeTimeLineHistory()` 创建新 history 文件时，会先读取 parent timeline 的 history 文件并原样复制。
然后再追加当前分叉行。
这让读取一个 child 的 history 文件就能知道完整恢复路径。
不需要递归打开每个父文件。
`reason` 是人可读信息。
它来自 `endOfRecoveryInfo->recoveryStopReason`。
它对诊断有帮助，但不是 correctness 字段。
redo 和 walsender 的判断依赖的是：
```text
parentTLI
switchpoint
child timeline 文件名
```
不是 reason 字符串。
因此不要把 `.history` 文件理解成审计日志。
它首先是 WAL 路径的持久化索引。
## 7. `TimeLineHistoryEntry` 把文本父链变成区间链
`timeline.h` 中的核心结构是：
```text
typedef struct
{
    TimeLineID tli;
    XLogRecPtr begin;  /* inclusive */
    XLogRecPtr end;    /* exclusive, InvalidXLogRecPtr means infinity */
} TimeLineHistoryEntry;
```
源码注释给出最关键的语义：
```text
每个 entry 表示 history 中一段 WAL 属于哪个 timeline。
所有 entry 的 begin 和 end 组成一条从起点到 infinity 的连续线。
```
`readTimeLineHistory()` 读取文件时，先按文件中从旧到新的行解析。
每读到一个 switchpoint，就构造父 timeline 的有效区间。
然后用 `lcons()` 把 entry 放到 list 头部。
所以结果 list 是：
```text
newest first
```
以 `00000003.history` 为例：
```text
1	0/5000000	...
2	0/9000000	...
```
`readTimeLineHistory(3)` 得到的区间链是：
```text
tli = 3, begin = 0/9000000, end = infinity
tli = 2, begin = 0/5000000, end = 0/9000000
tli = 1, begin = invalid,   end = 0/5000000
```
这里 `begin` 是 inclusive。
`end` 是 exclusive。
所以 switchpoint 本身属于新 timeline。
这点非常重要。
promotion 后第一条新 WAL record 可能正好写在 switchpoint。
读取逻辑必须把这个位置归给 child timeline。
`tliOfPointInHistory(ptr, history)` 做的就是区间查找：
```text
if begin invalid or begin <= ptr
and end invalid or ptr < end
  return tli
```
如果找不到，源码会报：
```text
timeline history was not contiguous
```
这不是普通文件缺失。
这是内存中的 history 区间链不连续，表示 history 文件或解析结果已经不满足核心不变量。
## 8. `tliSwitchPoint()` 的问题和 `tliOfPointInHistory()` 不同
`tliOfPointInHistory()` 回答：
```text
某个 LSN 属于哪条 timeline？
```
`tliSwitchPoint()` 回答：
```text
某条 timeline 在当前服务器认可的 history 中，到哪个 LSN 为止？
如果它不是 tip，下一条 timeline 是谁？
```
这是 walsender 和 cascading standby 的关键问题。
如果一个下游请求从历史 timeline 4 读取 WAL，而上游当前在 timeline 5，上游可以发送 timeline 4 的 WAL，但只能发送到 4 分叉到 5 的 switchpoint。
超过 switchpoint 之后，timeline 4 的未来不再是这台上游的历史。
`tliSwitchPoint(tli, history, &nextTLI)` 遍历 newest-first list。
当它找到 `tle->tli == tli` 时，返回这个 entry 的 `end`。
如果 `end` 是 `InvalidXLogRecPtr`，说明请求的 timeline 是当前 tip。
如果找不到，报错：
```text
requested timeline %u is not in this server's history
```
这条错误常出现在错误上游、缺失 history 文件、或请求了 sibling timeline 的场景。
它不是“WAL 文件不存在”的同义词。
它表示：
```text
按这台服务器的 history chain，这条 timeline 根本不是祖先或当前 tip。
```
这比文件存在性更强。
即使 archive 中碰巧有一个同名 WAL segment，如果 timeline 不在 history 中，也不能把它当成合法恢复来源。
## 9. promotion 为什么必须产生新的 timeline
`xlog.c` 在结束恢复时有一段直接解释新 timeline 的必要性。
逻辑可以压成两类场景：
```text
archive recovery / standby promotion:
  必须选择新 TLI

normal crash recovery:
  可以继续原 TLI
```
normal crash recovery 的前提是：
```text
本节点只是在同一条历史上从 crash 前最后一致位置继续写。
```
它没有引入新的分叉。
所以可以延续 `endOfRecoveryInfo->lastRecTLI`。
archive recovery 或 standby promotion 不同。
它可能停在旧 WAL 的中间位置。
即使它刚好 replay 到旧 WAL 末尾，仍然不安全地继续旧 TLI。
源码注释给出两个原因：
```text
如果 recovery 停在 WAL 末尾之前，显然会生成新历史。

即使跑到末尾，继续修改当前最后 segment 也可能覆盖已经归档的旧副本，
而 PostgreSQL 鼓励 archive_command 拒绝覆盖。
```
因此 promotion 后必须让新的 active segment 拥有新的 TLI。
这不是为了美观。
这是为了让 archive、standby 和后续恢复都能同时保存：
```text
旧 timeline 的同号 segment
新 timeline 的同号 segment
```
没有 TLI，两个 segment 会争夺同一个文件名。
有了 TLI，它们可以同时存在：
```text
000000010000000000000013
000000020000000000000013
```
这就是 “same system identifier 下的分叉” 能被持久化的根本原因。
## 10. promotion 主流程源码 walkthrough
promotion 后创建 timeline 的核心路径在 `xlog.c`。
简化时间线是：
```text
StartupXLOG()
  -> FinishWalRecovery()
     -> 计算 EndOfLog / EndOfLogTLI
     -> 如果 ArchiveRecoveryRequested:
          newTLI = findNewestTimeLine(recoveryTargetTLI) + 1
          XLogInitNewTimeline(EndOfLogTLI, EndOfLog, newTLI)
          durable_unlink standby.signal / recovery.signal
          writeTimeLineHistory(newTLI, recoveryTargetTLI, EndOfLog, reason)
     -> 设置 XLogCtl->InsertTimeLineID / PrevTimeLineID
     -> PerformRecoveryXLogAction()
          promotion: CreateEndOfRecoveryRecord()
          non-promotion: CHECKPOINT_END_OF_RECOVERY
     -> CleanupAfterArchiveRecovery()
     -> 允许普通 backend 写 WAL
```
`newTLI = findNewestTimeLine(recoveryTargetTLI) + 1` 是一个选择唯一 ID 的动作。
`findNewestTimeLine()` 从当前 target 往上探测 history 文件是否存在。
只要发现 `probeTLI.history`，就认为该 TLI 已经被占用。
遇到第一个不存在的 history 文件后停止。
源码注释说这是 heuristic，但它保证：
```text
result + 1 不是已知 timeline。
```
这就足以避免在本节点已知的 archive / pg_wal 范围内重用 timeline ID。
选择 `newTLI` 后，`XLogInitNewTimeline()` 初始化新 timeline 的起始 WAL segment。
如果 switchpoint 在 segment 中间：
```text
XLogFileCopy(newTLI, endLogSegNo, endTLI, endLogSegNo, used_bytes)
```
如果 switchpoint 在 segment 边界：
```text
XLogFileInit(startLogSegNo, newTLI)
```
前者保留旧 timeline 已经 replay 过的 WAL 前缀。
后者直接创建新 timeline 的第一个空 segment。
随后删除 `standby.signal` 或 `recovery.signal`。
这防止 crash 后再次进入 archive recovery。
然后写 `.history`。
源码中特别提醒：
```text
history 文件一旦被归档，standby 就会认为这个 timeline 已经被占用。
```
所以 `writeTimeLineHistory()` 和写 end-of-recovery record 之间的窗口要尽量短。
这是一个非常 PostgreSQL 的工程细节：
```text
先公开新历史身份
  -> 尽快写出能证明切换的 WAL record
```
如果中间 crash，下游可能已经看见新 history。
系统通过后续 recovery 继续收敛，但窗口越短越好。
## 11. `writeTimeLineHistory()` 如何保证文件落地
`writeTimeLineHistory()` 在 `timeline.c` 中。
它不是简单 `fopen("00000002.history", "w")`。
它采用类似 WAL 文件初始化的谨慎流程：
```text
生成 pg_wal/xlogtemp.<pid>
unlink 旧 temp
OpenTransientFile(O_CREAT | O_EXCL)
如果 parent history 存在，复制 parent 内容
追加当前 parentTLI / switchpoint / reason
pg_fsync(temp)
CloseTransientFile(temp)
durable_rename(temp, pg_wal/<newTLI>.history)
如果 archiving active，XLogArchiveNotify(<newTLI>.history)
```
这里的 correctness 边界有三个。
第一，先写 temp 再 rename。
这避免下游或 archive 看到半个 history 文件。
第二，`pg_fsync()` 在 rename 前完成。
history 文件不是可重建的缓存。
promotion 后它是恢复路径的一部分。
第三，文件创建后立即通知 archiver。
`xlogarchive.c` 中 `XLogArchiveNotify()` 对 timeline history 文件有特殊处理：
```text
if IsTLHistoryFileName(xlog):
  PgArchForceDirScan()
```
注释解释了原因：
```text
timeline history files are given the highest archival priority
```
目标是降低 promoted standby 选择一个 archive 中已经被占用 timeline 的概率。
这个优先级不是性能优化。
它是 distributed recovery metadata 的传播延迟控制。
history 文件越晚被其他节点看到，越容易出现多个节点选择同一个新 TLI 的风险。
## 12. `readTimeLineHistory()` 的正常路径和宽松路径
`readTimeLineHistory(targetTLI)` 的第一条特殊路径是 timeline 1：
```text
targetTLI == 1
  -> 返回一条 tli=1, begin=end=InvalidXLogRecPtr 的 entry
```
timeline 1 没有父历史。
第二条路径取决于是否 archive recovery：
```text
ArchiveRecoveryRequested:
  RestoreArchivedFile(path, "<targetTLI>.history", "RECOVERYHISTORY", 0, false)

否则:
  TLHistoryFilePath(path, targetTLI)
```
这里 `RECOVERYHISTORY` 是 restore_command 的临时目标名。
`RestoreArchivedFile()` 成功后，`readTimeLineHistory()` 最后会调用：
```text
KeepFileRestoredFromArchive(path, histfname)
```
把 history 文件保存到 `pg_wal`。
如果 open history 文件时得到 `ENOENT`，`readTimeLineHistory()` 会返回一条只有 targetTLI 的 list。
源码注释写得很明确：
```text
assume that the timeline has no parents
```
这个宽松路径容易误解。
它不是说缺失 history 永远安全。
它只是让某些路径可以继续尝试从本地或 stream 获取信息。
比如 `XLogFileReadAnyTLI()` 的注释说明：
```text
如果既没有 timeline history file，也没有 archive WAL segment，
但配置了 streaming replication，
不要过早保存 dummy history；
后续开始 streaming 后可能从 primary 收到真正 history 文件。
```
所以诊断时要区分：
```text
临时缺失:
  系统仍在尝试 archive / stream 获取 history

语义缺失:
  目标 timeline 校验、START_REPLICATION 或 redo timeline switch 已经无法证明 history
```
后者才是故障。
## 13. `.history` 文件解析的硬约束
`readTimeLineHistory()` 解析每一行时忽略空行和 `#` 注释。
有效行必须能解析出：
```text
timeline id
switchpoint high/low
```
否则 FATAL。
它还检查 timeline ID 单调增加：
```text
if result && tli <= lasttli:
  FATAL "Timeline IDs must be in increasing sequence."
```
最后检查：
```text
targetTLI > lasttli
```
如果 history 文件中某行的 parent TLI 大于等于 child 文件名代表的 target TLI，也会 FATAL。
这些约束保证：
```text
history 文件从旧父节点走到新 child；
不会出现回退 timeline；
不会出现 child 指向自己或未来 timeline。
```
注意解析时忽略每行 switchpoint 后面的剩余内容。
也就是说 reason 字段可以包含更多可读内容。
但 parser 的 correctness 输入在前两列已经结束。
解析后补一个 tip entry：
```text
entry->tli = targetTLI
entry->begin = prevend
entry->end = InvalidXLogRecPtr
```
这个 tip entry 不在文件中。
它由文件名和最后一个 switchpoint 推导出来。
所以 `.history` 文件内容和文件名必须一起解释。
只看文件内容无法知道当前 child timeline 是谁。
## 14. recovery target timeline 如何进入主循环
`xlogrecovery.c` 中有几组状态：
```text
recoveryTargetTLIRequested
recoveryTargetTLI
expectedTLEs
```
`recoveryTargetTLIRequested` 是用户配置解析出的数字值。
`recoveryTargetTLI` 是当前理解的目标 timeline。
`expectedTLEs` 是 `recoveryTargetTLI` 及其父 timeline 的 `TimeLineHistoryEntry` list，newest first。
`validateRecoveryParameters()` 处理 `recovery_target_timeline`。
如果用户指定数字：
```text
if rtli != 1 && !existsTimeLineHistory(rtli):
  FATAL "recovery target timeline %u does not exist"
recoveryTargetTLI = rtli
```
如果指定 `latest`：
```text
recoveryTargetTLI = findNewestTimeLine(recoveryTargetTLI)
```
这里必须等 archive recovery 状态和 restore_command 已经处理完。
因为寻找最新 timeline 可能需要从 archive 拉取 history 文件。
恢复主循环在读 WAL 前后会用 `expectedTLEs` 回答两个问题：
```text
这个 WAL segment 可以从哪些 TLI 尝试打开？
这个 record 的 LSN 按 history 应该属于哪个 TLI？
```
`XLogFileReadAnyTLI()` 会遍历 history 中的 TLI。
它还维护 `curFileTLI` 不后退。
注释说明原因：
```text
防止 parent timeline 在更高 segment number 上的文件被误选，
而我们真正想读的是 child timeline。
```
也就是说恢复不是按文件名 glob 随便找一个能打开的 segment。
它按 expected history 扫描，并防止 timeline 倒退。
## 15. `tliOfPointInHistory()` 在恢复读取中的位置
在 `WaitForWALToBecomeAvailable()` 的 stream 分支中，startup process 请求 walreceiver 时会先算当前请求点属于哪条 TLI：
```text
tli = tliOfPointInHistory(tliRecPtr, expectedTLEs)
RequestXLogStreaming(tli, ptr, PrimaryConnInfo, ...)
```
这里的 `tliRecPtr` 是 record begin position。
源码注释强调：
```text
Use the record begin position to determine the TLI, rather than the position we're reading.
```
原因是一个 WAL record 可能跨 page。
读的位置和 record 起点不完全相同。
timeline 归属应以 record 起点为准。
如果算出的 `tli` 小于已经读过的 `curFileTLI`，会报错：
```text
according to history file, WAL location ... belongs to timeline ...,
but previous recovered WAL file came from timeline ...
```
这条错误说明 history 归属和实际读取顺序冲突。
它不是普通缺文件。
它意味着恢复路径已经不再是一条单调 history。
当 stream 有数据时，startup process 会在必要时读取 history：
```text
if !expectedTLEs:
  expectedTLEs = readTimeLineHistory(recoveryTargetTLI)
readFile = XLogFileRead(readSegNo, receiveTLI, XLOG_FROM_STREAM, false)
```
源码注释提醒：
```text
readTimeLineHistory 要基于 recoveryTargetTLI，而不是 receiveTLI。
```
因为 `recovery_target_timeline = latest` 时，archive 可能已经让 `recoveryTargetTLI` 更新到更高 timeline。
这说明 `receiveTLI` 是当前收到 WAL 的事实。
`recoveryTargetTLI` 是恢复路径的目标语义。
两者不能混为一个字段。
## 16. redo 侧如何拒绝错误 timeline switch
redo 不只相信文件名。
WAL 记录本身也携带 timeline 切换信息。
`xlog_internal.h` 中 `xl_end_of_recovery` 保存：
```text
ThisTimeLineID
PrevTimeLineID
```
checkpoint record 中也有 `ThisTimeLineID` 和 `PrevTimeLineID`。
promotion 场景中，`CreateEndOfRecoveryRecord()` 写入：
```text
xlrec.ThisTimeLineID = XLogCtl->InsertTimeLineID
xlrec.PrevTimeLineID = XLogCtl->PrevTimeLineID
```
`CreateCheckPoint()` 在 `CHECKPOINT_END_OF_RECOVERY` 时也把：
```text
checkPoint.ThisTimeLineID = XLogCtl->InsertTimeLineID
checkPoint.PrevTimeLineID = XLogCtl->PrevTimeLineID
```
写入 checkpoint。
recovery replay 每条 record 前会检查是否造成 timeline switch。
路径在 `xlogrecovery.c`：
```text
if RM_XLOG_ID and XLOG_CHECKPOINT_SHUTDOWN:
  newReplayTLI = checkPoint.ThisTimeLineID
  prevReplayTLI = checkPoint.PrevTimeLineID

if RM_XLOG_ID and XLOG_END_OF_RECOVERY:
  newReplayTLI = xlrec.ThisTimeLineID
  prevReplayTLI = xlrec.PrevTimeLineID

if newReplayTLI != *replayTLI:
  checkTimeLineSwitch(recordEnd, newReplayTLI, prevReplayTLI, *replayTLI)
  *replayTLI = newReplayTLI
```
`checkTimeLineSwitch()` 有三层硬校验。
第一：
```text
prevTLI must equal replayTLI
```
如果 WAL record 说自己从 timeline 2 切来，但 startup 当前正在 replay timeline 1，这是不一致。
第二：
```text
newTLI >= replayTLI
newTLI in expectedTLEs
```
新 timeline 必须在目标 history 中。
不能突然跳到一个不属于恢复路径的 timeline。
第三：
```text
如果还没到 minRecoveryPoint，
不能切到一个大于 minRecoveryPointTLI 的 timeline，
否则之后不可能再到达正确 timeline 上的 minRecoveryPoint。
```
这些校验说明 `.history` 文件不是软提示。
它参与 redo correctness。
## 17. walsender 如何处理历史 timeline
`START_REPLICATION` 可以带 timeline。
如果客户端没有指定，walsender 使用当前 flush position 所在 timeline。
如果客户端指定了 timeline，`walsender.c` 的 `StartReplication()` 会判断它是当前 timeline 还是 historic timeline。
当前 timeline：
```text
sendTimeLine = FlushTLI
sendTimeLineIsHistoric = false
sendTimeLineValidUpto = InvalidXLogRecPtr
```
历史 timeline：
```text
timeLineHistory = readTimeLineHistory(FlushTLI)
switchpoint = tliSwitchPoint(cmd->timeline, timeLineHistory, &sendTimeLineNextTLI)
sendTimeLineIsHistoric = true
sendTimeLineValidUpto = switchpoint
```
如果 requested startpoint 超过该 historic timeline 在本服务器历史中的 switchpoint，报错：
```text
requested starting point ... on timeline ... is not in this server's history
```
这个错误的细节会说明：
```text
This server's history forked from timeline ... at ...
```
这正是 `.history` 文件提供的信息。
walsender 发送 historic timeline 时，只能发送到 `sendTimeLineValidUpto`。
到达后，它发送 `CopyDone`，然后返回一行结果：
```text
next_tli
next_tli_startpos
```
客户端据此切到下一条 timeline 继续请求。
这就是物理复制跨 timeline 追赶的基本协议。
它不是一次 COPY BOTH 永远发送到底。
timeline 边界会把流切成多个阶段。
## 18. walsender 为什么有时读新 TLI 的同号 segment
`walsender.c` 的 segment open 逻辑有一个看起来反直觉的注释。
当上游当前在 timeline 5，而客户端请求 timeline 4 的历史 WAL 时，如果 switchpoint 落在 segment 0x13 中间，可能出现：
```text
000000040000000000000012
000000040000000000000013
000000050000000000000013
000000050000000000000014
```
请求 timeline 4 的 segment 0x13 时，walsender 可能打开：
```text
000000050000000000000013
```
原因是 timeline switch 时，新 TLI 的同号 segment 复制了旧 timeline 到 switchpoint 为止的已用前缀。
如果 archive recovery 优先恢复较新 timeline 的文件，旧 timeline 的同号文件可能不在本地。
但在 switchpoint 之前，两者内容相同。
所以在 historic timeline 发送到 switchpoint 的最后一个 segment 时，读取新 TLI 文件可以是合法的。
这个细节说明：
```text
文件名 TLI 是身份；
文件内容前缀可以共享；
是否能共享读取，要由 switchpoint 限定。
```
没有 switchpoint，这个优化就不安全。
## 19. walreceiver 如何传播缺失 history 文件
walreceiver 在 `walreceiver.c` 中有 `WalRcvFetchTimeLineHistoryFiles(first, last)`。
当 upstream 告诉它需要切到新 timeline，或者本地缺少某些 history 文件时，它会逐个检查：
```text
if tli != 1 && !existsTimeLineHistory(tli):
  walrcv_readtimelinehistoryfile(wrconn, tli, &fname, &content, &len)
  TLHistoryFileName(expectedfname, tli)
  if fname mismatch:
    ERROR protocol violation
  writeTimeLineHistoryFile(tli, content, len)
```
`TIMELINE_HISTORY %u` 命令由 walsender 的 `SendTimeLineHistory()` 处理。
它返回两列：
```text
filename
content
```
walreceiver 收到后用 `writeTimeLineHistoryFile()` 写入 `pg_wal`。
这个函数也走 temp file、fsync、durable rename。
写完后根据 archive mode 处理 archive status：
```text
archive_mode != always:
  XLogArchiveForceDone(fname)

archive_mode == always:
  XLogArchiveNotify(fname)
```
这反映一个边界：
```text
从 upstream stream 下来的 history 文件，是本地恢复需要的 metadata；
是否再次归档，取决于本节点 archive_mode 语义。
```
如果 archive mode 不是 always，本地不需要把它重新归档。
如果是 always，本节点作为 archive 生产者，也要通知 archiver。
## 20. `findNewestTimeLine()` 的边界
`findNewestTimeLine(startTLI)` 的算法很简单：
```text
newestTLI = startTLI
for probeTLI = startTLI + 1; ; probeTLI++:
  if existsTimeLineHistory(probeTLI):
    newestTLI = probeTLI
  else:
    break
return newestTLI
```
它不是遍历一棵 timeline DAG。
它只是按数字向上探测 history 文件。
源码注释也承认：
```text
somewhat heuristic
```
但是它提供一个实际需要的保证：
```text
result + 1 is not a known timeline
```
这保证当前节点选择 `findNewestTimeLine(recoveryTargetTLI) + 1` 时，不会和已知 history 文件冲突。
为什么不要求全局强一致分配？
因为 timeline ID 是通过 archive / replication metadata 传播的。
PostgreSQL 没有一个全局 timeline allocator。
它依赖：
```text
history 文件尽快归档
standby 启动或 latest target 时尽快发现更高 TLI
promotion 后新 TLI 文件名避免覆盖旧 WAL
```
如果多个隔离节点同时从同一 timeline promotion，并且彼此看不到对方 history，它们理论上可能选择相同 newTLI。
这不是单机代码能完全解决的问题。
生产系统要靠故障切换编排、archive 可见性和避免 split brain 解决。
课程里的核心结论是：
```text
TimeLineID 是恢复路径身份，不是选主共识协议。
```
## 21. `rescanLatestTimeLine()` 如何追上新分叉
当 `recovery_target_timeline = latest` 时，恢复过程中可能发现 archive 中出现了新的 timeline。
`rescanLatestTimeLine(replayTLI, replayLSN)` 做这件事。
它先调用：
```text
newtarget = findNewestTimeLine(recoveryTargetTLI)
```
如果没有更高 TLI，返回 false。
如果发现更高 TLI，就读取它的 history：
```text
newExpectedTLEs = readTimeLineHistory(newtarget)
```
然后检查当前 recovery target 是否在新 history 中。
如果不在，日志：
```text
new timeline %u is not a child of database system timeline %u
```
返回 false。
即使当前 target 在新 history 中，还要检查分叉点：
```text
currentTle->end < replayLSN
```
如果新 timeline 从当前 timeline 分叉的位置早于当前 replay 位置，不能切过去。
日志会说：
```text
new timeline %u forked off current database system timeline %u before current recovery point ...
```
这条边界非常重要。
一个 standby 已经 replay 过某些 WAL。
如果新 timeline 在更早位置分叉，那么切过去等于要求撤销已经 redo 的未来。
PostgreSQL 不做这种回滚。
它只能在还没越过 switchpoint 时切到新 target。
通过这个检查后，才更新：
```text
recoveryTargetTLI = newtarget
expectedTLEs = newExpectedTLEs
restoreTimeLineHistoryFiles(oldtarget + 1, newtarget)
```
然后记录：
```text
new target timeline is %u
```
## 22. `RemoveNonParentXlogFiles()` 的角色
当系统切换 timeline 时，`RemoveNonParentXlogFiles(switchpoint, newTLI)` 会清理 `pg_wal` 中不属于新 history 的未来 WAL segment。
注释说得很直接：
```text
extra WAL files might be leftover pre-allocated or recycled WAL segments on the old timeline,
and contain garbage.
If we just leave them in pg_wal, they will eventually be archived,
and we can't let that happen.
```
这个函数不是删除所有旧 timeline WAL。
它删除的是：
```text
timeline 比 newTLI 老
segment number 在新 timeline 起始段之后
不属于 parent history 的未来文件
```
已经成功 replay、属于父历史的旧文件仍然有效。
不属于新 history 的未来文件可能是：
```text
旧 primary 后来产生的 WAL
预分配但没真正使用的 WAL
recycled 文件中的垃圾内容
```
如果这些文件被归档，下游可能在错误时间看到它们。
所以 timeline switch 后的 cleanup 也是 correctness 的一部分。
promotion 后如果旧 timeline 的最后一个 segment 是 partial，并且 archive active，`CleanupAfterArchiveRecovery()` 会把它改名成 `.partial` 后归档。
注释解释了折中：
```text
archive recovery 不会自动读取 .partial；
但 DBA 在特殊 PITR 或调试场景可以手动使用。
```
这避免了把旧 timeline 未完成 segment 当成正常可恢复 WAL。
## 23. 生命周期与 ownership
timeline history 的 owner 不是某个 backend。
它是文件系统状态，放在：
```text
pg_wal/<TLI>.history
```
并通过 archive 传播。
内存中的 `List *expectedTLEs` 是 startup process 的运行期解析结果。
它由 `readTimeLineHistory()` 通过 `palloc_object(TimeLineHistoryEntry)` 分配。
在目标 timeline 更新时，代码会：
```text
list_free_deep(expectedTLEs)
expectedTLEs = newExpectedTLEs
```
文件由谁创建：
```text
promotion / archive recovery 结束:
  writeTimeLineHistory()

streaming replication 收到 upstream history:
  writeTimeLineHistoryFile()

archive recovery 从 archive 拉取:
  RestoreArchivedFile() + KeepFileRestoredFromArchive()
```
文件由谁消费：
```text
startup process:
  选择 WAL source，校验 recovery target

walsender:
  判断 START_REPLICATION timeline 是否在 history

walreceiver:
  判断缺失哪些 history 并写入本地

backup / manifest / replication slot 展示:
  需要把 LSN 映射回 timeline
```
文件何时释放：
```text
通常不主动释放。
```
history 文件很小，但语义长期有效。
清理旧 WAL segment 和清理 history file 不是同一件事。
删除 history 文件可能让后续 PITR、standby reconnect 或 cascading replication 无法证明恢复路径。
ERROR / abort 时如何兜底：
```text
writeTimeLineHistory() 写失败会删除 temp 并 ERROR
fsync 失败按 data_sync_elevel(ERROR) 处理
durable_rename 失败 ERROR
walreceiver 写入失败 ERROR，连接会重试或 recovery 停止
```
这些失败通常不能 silent ignore。
因为缺少 history 文件会让 timeline 身份不可传播。
## 24. 正确性机制层次
timeline correctness 不是一个字段保证的。
它是多层机制叠加。
第一层是文件名：
```text
WAL segment:
  TLI + log + seg

history file:
  TLI + ".history"
```
文件名防止两个历史写到同一个物理路径。
第二层是 history 文件：
```text
parentTLI + switchpoint
```
它证明 child timeline 从哪里开始。
第三层是内存区间链：
```text
TimeLineHistoryEntry(begin, end)
```
它让恢复能在任意 LSN 上做 timeline lookup。
第四层是 WAL record 内部 TLI：
```text
checkpoint.ThisTimeLineID
checkpoint.PrevTimeLineID
xl_end_of_recovery.ThisTimeLineID
xl_end_of_recovery.PrevTimeLineID
```
它让 redo 检查实际 WAL record 是否真的在当前 expected history 上切换。
第五层是 archive / streaming 传播：
```text
XLogArchiveNotify()
PgArchForceDirScan()
TIMELINE_HISTORY command
WalRcvFetchTimeLineHistoryFiles()
```
它让下游及时知道新历史。
第六层是 cleanup：
```text
RemoveNonParentXlogFiles()
.partial handling
```
它减少错误未来 WAL 被归档或误读的机会。
这些机制各自只解决一部分问题。
`TimeLineID` 不保证 archive 中没有 split brain。
`.history` 不保证选主正确。
redo 校验不负责拉取缺失文件。
walsender 不负责决定 recovery target。
只有把它们串起来，才能得到 “同一个 LSN 在同一个 system identifier 下仍然有唯一历史身份” 的效果。
## 25. 错误路径一：missing history
missing history 的表象不止一种。
如果用户指定 numeric recovery target timeline，且 history 文件不存在，`validateRecoveryParameters()` 会 FATAL：
```text
recovery target timeline %u does not exist
```
如果 walsender 需要读取 requested timeline，但当前 history 中找不到，`tliSwitchPoint()` 会 ERROR：
```text
requested timeline %u is not in this server's history
```
如果 walreceiver 试图从 upstream 拉取 history 文件失败，libpqwalreceiver 会报告：
```text
could not receive timeline history file from the primary server
```
如果 `readTimeLineHistory()` 打开文件时遇到非 ENOENT 错误，会 FATAL：
```text
could not open file "...": ...
```
如果只是 ENOENT，它可能临时返回 dummy one-entry history。
所以诊断 missing history 时不要只看一条日志。
要问：
```text
当前代码路径是 target validation、WAL source lookup、walsender historic request，
还是 startup process 还在等待 stream 提供 history？
```
典型排查顺序：
```text
确认 pg_wal 中是否有 <targetTLI>.history
确认 archive 中是否有该 history
确认 restore_command 是否能取回它
确认 upstream walsender 是否能响应 TIMELINE_HISTORY
确认 recovery_target_timeline 是否误指向 sibling timeline
```
如果 system identifier 相同但报 history missing，不要先怀疑 sysid。
sysid 已经解决了 “是不是同一集群血统”。
当前问题是 “是不是同一恢复路径”。
## 26. 错误路径二：wrong timeline
wrong timeline 通常表现为 history 里能找到某些 TLI，但 switchpoint 和当前位置不匹配。
`StartReplication()` 里的典型错误：
```text
requested starting point ... on timeline ... is not in this server's history
```
detail：
```text
This server's history forked from timeline ... at ...
```
这说明客户端请求的位置已经超过该 historic timeline 在上游历史中的有效上限。
常见原因：
```text
standby 跟到了旧 primary 的 timeline，但现在连接到了 promoted sibling
replication slot 的 restart_lsn 位于一条上游不再承认的历史
手工指定 START_REPLICATION TIMELINE 错误
archive 中混入 sibling timeline 的 WAL 和 history
```
`rescanLatestTimeLine()` 的错误：
```text
new timeline ... forked off current database system timeline ... before current recovery point ...
```
说明 standby 已经 replay 过新 timeline 的分叉点。
它不能向后撤销。
这通常意味着：
```text
发现 latest timeline 太晚
archive / restore_command 可见性滞后
recovery_target_timeline 选择不符合当前 replay 进度
```
redo 中的 PANIC：
```text
unexpected previous timeline ID ...
unexpected timeline ID ... in checkpoint record
unexpected timeline ID ... in end-of-recovery record
```
说明 WAL record 自己携带的 TLI 和当前 expected history 不一致。
这类问题更严重。
优先怀疑：
```text
WAL 文件来自错误 timeline
history 文件和 WAL 文件不匹配
pg_wal 中残留了不属于当前 history 的未来 segment
archive 返回了错误文件
```
## 27. 同名 segment、成本与观测
promotion 后常见两个同号 segment：
```text
000000010000000000000013
000000020000000000000013
```
它们不是重复文件，而是同一 LSN 区间在不同 timeline 上的文件。
如果 switchpoint 在 segment 中间，新 TLI 文件前缀会复制旧 TLI 已使用部分；switchpoint 之后仍然是不同历史。
旧 timeline 的未完成 segment 可能以 `.partial` 归档，archive recovery 不会自动读取它，常规恢复应依赖新 TLI segment 和 `.history`。
timeline history 本身通常不在 WAL insert hot path 上，成本主要来自 archive 可见性、`restore_command` 探测、walsender historic streaming 边界、`pg_wal` cleanup 扫描，以及诊断时必须跨文件系统、日志和复制协议拼接因果。
可直接观察的入口包括日志中的 `selected new timeline ID: N`、`new target timeline is N`、`fetching timeline history file for timeline N from primary server`，文件系统中的 `pg_wal/*.history`，以及 `IDENTIFY_SYSTEM`、`TIMELINE_HISTORY <tli>`、`START_REPLICATION ... TIMELINE <tli>`。
SQL 只能看到部分事实：
```sql
SELECT pg_is_in_recovery();
SELECT pg_last_wal_replay_lsn();
SELECT pg_current_wal_lsn();
SELECT pg_walfile_name(pg_current_wal_lsn());
```
`pg_walfile_name()` 可以暴露当前插入 timeline 的文件名前缀；`pg_last_wal_replay_lsn()` 只有 LSN，不包含完整 history 区间。
wait event 可以辅助定位卡在哪里：
```text
TIMELINE_HISTORY_READ
TIMELINE_HISTORY_WRITE
TIMELINE_HISTORY_SYNC
TIMELINE_HISTORY_FILE_WRITE
TIMELINE_HISTORY_FILE_SYNC
WALSENDER_TIMELINE_HISTORY_READ
RESTORE_COMMAND
```
它们说明当前在读写 history、执行 restore_command 或由 walsender 读取 history，但不能单独证明 wrong timeline 的根因。
源码断点优先打在 `writeTimeLineHistory()`、`XLogInitNewTimeline()`、`readTimeLineHistory()`、`tliOfPointInHistory()`、`tliSwitchPoint()`、`checkTimeLineSwitch()`、`StartReplication()` 和 `WalRcvFetchTimeLineHistoryFiles()`。
在断点上看 `newTLI`、`parentTLI`、`switchpoint`、`recoveryTargetTLI`、`sendTimeLineValidUpto`，比只盯着 LSN 更接近本节问题。
## 28. 课堂实验
实验一：搭一个 primary 和 physical standby，触发 standby promotion。
观察日志 `selected new timeline ID: 2`，检查 `pg_wal/00000002.history`，再用 `pg_walfile_name(pg_current_wal_lsn())` 确认新 WAL 文件名前 8 位已经变成新 TLI。
把结果画成：
```text
timeline 1: [begin, switchpoint)
timeline 2: [switchpoint, infinity)
```
解释时必须说清楚 system identifier 没变，变的是同一 LSN 地址空间之后的历史身份。
实验二：手工读取一个 `.history` 文件，按 `readTimeLineHistory()` 的规则把它倒排成 newest-first 区间链，然后用几个 LSN 手算 `tliOfPointInHistory()` 会返回哪个 TLI。
重点检查：
```text
begin inclusive
end exclusive
switchpoint belongs to child timeline
```
实验三：让一个 standby 连接到 sibling promoted server，观察 `requested starting point ... on timeline ... is not in this server's history` 或 `primary server contains no more WAL on requested timeline ...`。
诊断时同时记录 `IDENTIFY_SYSTEM` 返回的 timeline、本地 `.history` 文件和请求的 `START_REPLICATION` timeline。
## 29. 常见误区
`system_identifier` 相同不代表一定能复制；它只证明同一集群血统，不能证明请求点属于同一恢复路径。
timeline 不是 WAL 文件名前缀装饰；promotion 后同一个 LSN 可以有多个合法未来，没有 TLI 和 switchpoint 无法唯一定位 WAL。
`.history` 不是只记录直接父节点；child history 会复制 parent history，再追加当前分叉行。
switchpoint 不属于 parent timeline；`TimeLineHistoryEntry` 的 `begin` inclusive、`end` exclusive，所以 switchpoint 是 child timeline 的起点。
缺少 history 文件不一定在第一次读取时 FATAL；某些路径会临时等待 archive 或 stream 提供真正 history，但 target validation、walsender 校验和 redo switch 会把语义错误暴露出来。
`TimeLineID` 不能防止 split brain；它是 WAL 历史身份，外部故障切换系统仍然要保证不会同时 promotion 多个写主。
## 30. 讨论题
1. 为什么 normal crash recovery 可以延续原 TLI，而 archive recovery / promotion 必须选择新 TLI？
2. 如果两个节点 system identifier 相同，但 `START_REPLICATION` 报 history mismatch，你会按什么顺序排查？
3. `.history` 文件为什么要复制 parent history，而不是只记录直接父 timeline？
4. `switchpoint` 为什么设计成 child timeline 的 begin，而不是 parent timeline 的最后一个 LSN？
5. `findNewestTimeLine()` 为什么只能提供已知范围内的唯一性，而不是全局 timeline 分配保证？
6. walsender 为什么在发送 historic timeline 的最后一个 segment 时，有时可以打开 next timeline 的同号 segment？
7. `checkTimeLineSwitch()` 为什么同时检查 `PrevTimeLineID` 和 `newTLI in expectedTLEs`？
## 31. 本节小结
本节只回答一个问题：promotion 后如何避免不同历史被误认为同一条 WAL 流。
核心链路是：archive recovery / promotion 选择 `findNewestTimeLine() + 1` 作为新 TLI，`XLogInitNewTimeline()` 在新 TLI 下创建起始 WAL segment，`writeTimeLineHistory()` 写 `<newTLI>.history` 记录 `parentTLI` 和 `switchpoint`，archive / walreceiver 传播 history 文件，`readTimeLineHistory()` 构造 `TimeLineHistoryEntry` 区间链，`tliOfPointInHistory()` 和 `tliSwitchPoint()` 把 LSN 映射到 timeline 与有效上界，最后 recovery、walsender 和 redo 用 `expectedTLEs` 拒绝错误历史。
核心状态是四个：`TimeLineID` 是 WAL 文件身份的一部分；`.history` 是 child timeline 的完整父链文件；`switchpoint` 是 child timeline 的 begin、parent timeline 的 end；`expectedTLEs` 是当前 recovery target 的可接受历史区间链。
ownership 边界也要记住：history 文件属于 `pg_wal` / archive 的持久状态，内存 list 属于 startup process 当前恢复目标，写入走 temp + fsync + durable rename，错误未来 WAL 由 `RemoveNonParentXlogFiles()` 清理。
诊断时按层次拆分：system identifier mismatch 是血统错误；missing history 是恢复路径 metadata 不完整；wrong timeline 是请求点不属于上游认可的 history；unexpected timeline ID 是 WAL record、history 和 replay state 已经互相矛盾。
可迁移规律是：当系统允许共享历史前缀后分叉，单调地址不能单独表达对象身份；必须把地址、分叉版本和父链区间一起持久化，并在读取、复制和恢复边界重新校验。
具体判断仍然依赖 archive 可见性、`restore_command` 行为、cascading replication 拓扑、promotion 编排和 PostgreSQL 版本细节；但稳定不变量不变：同一个 system identifier 下，LSN 只有带上 timeline history 才是可恢复的 WAL 位置。
