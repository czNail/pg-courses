# PostgreSQL Crash restart：从 control file 到一致性点
## 课程定位
本节主题：PostgreSQL 进程或主机 crash 后，startup process 如何从 `pg_control` 找到恢复入口，并把数据目录推进到可以再次接受读写的一致状态。
前置知识：
- 已理解 WAL record、rmgr redo contract、full-page writes、WAL-before-data。
- 已理解 checkpoint record 和 redo pointer 的区别。
- 已知道 relation fork、segment file、smgr/md 存储层的基本边界。
本节唯一主问题：
crash restart 时，PostgreSQL 为什么不能直接相信磁盘上的数据文件，又为什么可以从 `pg_control` 指向的 checkpoint record 和它的 `redo` pointer 开始，最终判断系统已经回到一致状态？
本节围绕的核心矛盾：
系统需要用一个很小的、可原子更新的 control file 快速定位恢复入口；但 crash 后真正可信的不是数据文件当前内容，而是 control file、checkpoint record、WAL 顺序校验、rmgr redo、invalid-page 检查和 cleanup 共同形成的协议。
学完本节，你应该能独立判断：
- `pg_control` 中哪些字段能作为启动事实，哪些字段只是 checkpoint payload 的拷贝。
- clean shutdown、crash recovery、archive recovery、backup recovery 对“一致点”的定义有什么不同。
- 为什么在线 checkpoint 的 `checkPoint` 和 `checkPointCopy.redo` 不是同一个 LSN。
- startup process 如何校验 checkpoint record 和 redo record。
- WAL reader 在 redo 中校验了哪些边界，哪些错误会终止恢复。
- crash recovery 为什么必须 replay 到本地可用 WAL 的末尾。
- `minRecoveryPoint` 为什么主要服务 archive/base-backup recovery，而不是普通 crash recovery。
- redo 中遇到不存在的页为什么先记 invalid-page，而不是立即静默忽略。
- unlogged relation 和 temp file 为什么不靠 WAL redo 恢复。
- 哪些现象能从日志、`pg_controldata`、`pg_waldump`、wait event 和断点中看到。
## 源码基线
源码仓库：
```text
/home/nail/postgres-lab
```
基线：
```text
branch: master
commit: bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8
```
本节重点阅读：
```text
src/backend/access/transam/xlogrecovery.c
src/backend/access/transam/xlog.c
src/backend/access/transam/xlogreader.c
src/backend/access/transam/xlogutils.c
src/include/catalog/pg_control.h
src/common/controldata_utils.c
src/backend/storage/file/reinit.c
src/backend/storage/smgr/md.c
```
为讲清 temp cleanup 和 smgr object cleanup，本节还少量引用：
```text
src/backend/storage/file/fd.c
src/backend/storage/smgr/smgr.c
src/backend/postmaster/postmaster.c
```
行号来自：
```text
nl -ba <source-file>
```
---
## 1. 本节在总主线中的位置
前几节已经把 WAL 和 checkpoint 的写入侧讲清楚：
修改数据页前先写 WAL；
checkpoint 完成后把新的恢复起点发布到 `pg_control`；
full-page writes 保护 checkpoint 后第一次修改的 torn page 风险。
本节换到读入侧。
问题不再是“怎样写出一个可恢复的历史”，而是：
```text
crash 后只剩数据目录和 WAL，
startup process 怎样决定从哪里开始相信历史，
又怎样知道 replay 到哪里才可以对外服务。
```
这一节不要把 recovery 理解成“把 WAL 再执行一遍”。
真正的主线是状态边界：
```text
pg_control
  -> checkpoint record
  -> checkpoint.redo
  -> WAL reader 校验
  -> rmgr redo
  -> consistency check
  -> cleanup
  -> DB_IN_PRODUCTION
```
每一步都在缩小不确定性。
`pg_control` 只给出候选入口。
checkpoint record 校验这个入口确实是 checkpoint。
`checkpoint.redo` 给出需要 replay 的历史下界。
WAL reader 判断后续 WAL 是否形成连续可信序列。
rmgr redo 把变化应用到数据文件和 shared state。
一致性检查确认没有悬空的 invalid page 或未完成 backup 边界。
cleanup 把 WAL 不覆盖的临时状态、unlogged 状态、smgr 缓存状态收尾。
---
## 2. 核心矛盾与一句话运行模型
一句话运行模型：
```text
PostgreSQL crash restart 先用 pg_control 定位 checkpoint record，
再从 checkpoint.redo 开始按 WAL 顺序 replay，
直到普通 crash recovery 读完所有可用 WAL，
或 archive/base-backup recovery 达到 minRecoveryPoint 与 backup end，
然后清理 WAL 无法保证的临时和 unlogged 文件状态，
最后把 control file 与 shared recovery state 切到 production。
```
这里的 tension 有三层。
第一层是入口可信度。
`pg_control` 必须小、可校验、可 fsync。
但它不能包含所有恢复事实。
所以它保存的是：
- 当前 database state。
- 最新 checkpoint record 的 LSN。
- checkpoint record payload 的拷贝。
- archive/base-backup recovery 的最小恢复边界。
- 与 WAL 兼容性相关的参数。
- CRC。
第二层是数据文件可信度。
crash 后，数据文件可能已经包含一些未 fsync 的写入，也可能丢掉一些已经写到 OS cache 但没落盘的写入。
PostgreSQL 不把“当前磁盘页内容”当成完整事实。
它通过 page LSN、FPI、record CRC 和 rmgr redo 判断每个 WAL record 是否需要应用。
第三层是一致点定义。
普通 crash recovery 没有“中途一致点”。
它要 replay 到本地可用 WAL 的末尾。
archive/base-backup recovery 则需要达到 `minRecoveryPoint`、backup end 和 hot standby snapshot 等条件。
源码里 `CheckRecoveryConsistency()` 对这两类路径的处理完全不同。
本节最重要的边界：
```text
control file 给出入口，
checkpoint record 给出 redo 起点，
WAL reader 给出可信顺序，
rmgr redo 给出状态推进，
cleanup 给出 WAL 不负责对象的重置，
DB_IN_PRODUCTION 才是对普通 backend 开放写 WAL 的边界。
```
---
## 3. 核心文件分工与阅读顺序
建议阅读顺序不是按文件名，而是按 crash restart 时间线。
1. `pg_control.h:35-69,112-249`：先读 `CheckPoint` 和 `ControlFileData`，确认 `checkPoint`、`checkPointCopy.redo`、`minRecoveryPoint`、backup 边界和 CRC 的语义。
2. `controldata_utils.c:52-178,189-284`：再读 control file 读写，理解 frontend retry、backend PANIC、CRC 和 fsync。
3. `xlog.c:4406-4627,5270-5277,5846-6702`：读 `ReadControlFile()`、早期 local copy 和 `StartupXLOG()` 主流程。
4. `xlogrecovery.c:456-974,1612-1877,2151-2246`：读 `InitWalRecovery()`、`PerformWalRecovery()`、`CheckRecoveryConsistency()`。
5. `xlogreader.c:232-244,530-967,1235-1371`：读起点定位、record/header/CRC 校验、page header 校验。
6. `xlogutils.c:56-64,101-161,234-262,340-427,630-663`：读 invalid-page deferred validation、redo 读页和 drop/truncate 消解。
7. `reinit.c:47-100,161-367` 与 `md.c:222-274,337-476,1288-1385`：读 unlogged reset 和 redo 中 create/unlink/truncate 的幂等边界。
---
## 4. 关键数据结构与状态
先看 `CheckPoint`。
不要把它理解成“checkpoint 发生时所有状态的全量快照”。
它只保存 crash restart 必须从一个 checkpoint 接续系统状态所需的字段。
最关键字段是 `redo`、`ThisTimeLineID`、`PrevTimeLineID`、`fullPageWrites`、`wal_level`、`logicalDecodingEnabled`、`nextXid`、`nextOid`、MultiXact next/oldest 边界、oldest XID horizon、`oldestActiveXid` 和 `dataChecksumState`。
`redo` 是恢复下界。
它表示“从这里开始 replay 足够恢复到 checkpoint 之后的一致状态”。
在线 checkpoint 下，它通常早于 checkpoint record 本身。
shutdown checkpoint 下，它等于 checkpoint record。
`ThisTimeLineID` 和 `PrevTimeLineID` 不只是日志展示字段。
recovery 需要用它验证 checkpoint 所属 timeline 是否在目标 timeline 的历史中。
`xlogrecovery.c:791-825` 对 checkpoint location 和 minRecoveryPoint 的 timeline 都做了历史检查。
`fullPageWrites` 会影响 standby backup 和后续 FPI 判断。
本节不展开写入侧，但要记住：
恢复入口如果跨过了需要 FPI 保护的窗口，数据页 torn write 就无法被单纯的 incremental redo 修复。
`nextXid`、`nextOid`、MultiXact、oldest horizon 字段不是“业务数据恢复”。
它们恢复的是事务 ID、OID、MultiXact 和 truncation horizon 的分配边界。
`StartupXLOG()` 在 `xlog.c:5995-6004` 用 checkpoint payload 初始化这些 shared variables。
`oldestActiveXid` 只服务 hot standby 从 online checkpoint 初始化运行事务状态。
`pg_control.h:59-65` 明确说它只在 replica wal_level 的 online checkpoint 有意义。
再看 `ControlFileData`。
本节最重要字段是 `state`、`time`、`checkPoint`、`checkPointCopy`、`minRecoveryPoint`、`minRecoveryPointTLI`、`backupStartPoint`、`backupEndPoint`、`backupEndRequired`、WAL/Hot Standby 参数、`xlog_seg_size`、`data_checksum_version` 和 `crc`。
`checkPoint` 是 checkpoint record 的起始 LSN。
`checkPointCopy.redo` 是 redo 起始 LSN。
它们不是同一个概念。
`InitWalRecovery()` 从 `ControlFile->checkPoint` 读 checkpoint record，再从 `checkPoint.redo` 读 redo 起点。
`state` 是启动分支的第一层判断。
枚举在 `pg_control.h:97-106`：
```text
DB_STARTUP
DB_SHUTDOWNED
DB_SHUTDOWNED_IN_RECOVERY
DB_SHUTDOWNING
DB_IN_CRASH_RECOVERY
DB_IN_ARCHIVE_RECOVERY
DB_IN_PRODUCTION
```
`DB_SHUTDOWNED` 意味着上次干净 shutdown。
但如果有 recovery signal file，仍可能强制进入 archive recovery。
`DB_IN_PRODUCTION` 则说明 crash 前系统在正常生产态运行，启动时通常要做 crash recovery。
`minRecoveryPoint` 是 archive recovery 的安全边界。
注释在 `pg_control.h:148-156`：
如果 archive recovery 中已经把某个 WAL record 的数据变化 flush 到磁盘，就不能下次从更早位置启动。
普通 crash recovery 下它是 invalid，意思是必须 replay 所有本地可用 WAL。
`backupStartPoint`、`backupEndPoint`、`backupEndRequired` 是 base backup recovery 的边界。
如果恢复自 online backup，就不能在 backup end 之前宣布一致。
`crc` 是 control file 自身的完整性边界。
`ReadControlFile()` 在 `xlog.c:4469-4479` 重新计算并校验它。
frontend 工具通过 `get_controlfile()` 也会计算 CRC，并在并发读到不稳定内容时短暂重试。
最后看 recovery 局部状态：
核心局部状态包括 `InRecovery`、`InArchiveRecovery`、`ArchiveRecoveryRequested`、`StandbyModeRequested`、`RedoStartLSN`、`RedoStartTLI`、`CheckPointLoc`、`CheckPointTLI`、`minRecoveryPoint`、`backupEndRequired`、`reachedConsistency` 和 `invalid_page_tab`。
`InRecovery` 在 `xlogutils.c:36-50` 的注释很关键。
它主要表示“当前 startup process 正在 replay WAL record”。
不要用它替代所有进程可见的 `RecoveryInProgress()`。
`reachedConsistency` 在 `xlogrecovery.c:291-303` 附近定义。
它只在 archive/base-backup recovery 里有中途变为 true 的意义。
普通 crash recovery 下，`CheckRecoveryConsistency()` 因为 `minRecoveryPoint` invalid 直接返回。
`invalid_page_tab` 是 recovery-local hash table。
它不是 shared memory 状态。
只有 startup process replay WAL 时使用。
---
## 5. 主流程源码 walkthrough
### 5.1 postmaster 先清掉旧 temp file
在正常 postmaster 启动中，`postmaster.c:1329-1333` 会调用 `RemovePgTempFiles()`。
注释强调此时没有其它 PostgreSQL 进程使用这个 data directory，所以删除旧 temp file 是安全的。
post-backend-crash 的重启周期也可能删除 temp file。
`postmaster.c:3235-3242` 中，所有非 syslogger 子进程退出后，如果 `remove_temp_files_after_crash` 打开，就调用 `RemovePgTempFiles()`。
这一步不是 WAL recovery。
它处理的是 WAL 不承诺恢复的临时文件和临时 relation 文件。
如果删除失败，`fd.c` 里通常只记 LOG 并继续。
这是故意的：
temp cleanup 不是启动正确性的硬边界。
### 5.2 shared memory 建立前读取 pg_control
`LocalProcessControlFile(false)` 在 shared memory 可用前读 `pg_control`。
见 `xlog.c:5257-5277`。
原因是 shared memory 大小和某些配置可能依赖 control file 内容。
所以启动早期先把 control file 读到 local memory：
```text
LocalControlFile = palloc_object(ControlFileData)
ControlFile = LocalControlFile
ReadControlFile()
SetLocalDataChecksumState(...)
```
随后 `XLOGShmemInit()` 把 local copy 移到 shared memory。
见 `xlog.c:5371-5379`。
这里的 ownership 很清楚：
- 早期 owner 是 startup/postmaster local memory。
- shared memory 初始化后 owner 变成 `Control File` shared memory struct。
- 后续更新需要 `ControlFileLock`，或在还没有其它进程时由 startup process 独占更新。
### 5.3 ReadControlFile() 的第一道防线
`ReadControlFile()` 从 `global/pg_control` 读取 `sizeof(ControlFileData)`。
`xlog.c:4416-4437` 中，打开、短读、读失败都会报错。
然后它做严格校验：
`PG_CONTROL_VERSION`、CRC、`CATALOG_VERSION_NO`、`MAXALIGN`、浮点格式、`BLCKSZ`、`RELSEG_SIZE`、`SLRU_PAGES_PER_SEGMENT`、`XLOG_BLCKSZ`、`NAMEDATALEN`、`INDEX_MAX_KEYS`、TOAST chunk size、large object block size 和 WAL segment size。
这些校验不是形式主义。
recovery 会直接解释 WAL record、tuple、page、SLRU segment 和 relation segment。
如果这些编译期或 initdb-time 参数不一致，继续 redo 只会扩大破坏面。
`ControlFileData` 被限制在 512 bytes 以内。
`pg_control.h:251-272` 的注释说明原因：
active data 要尽量落在常见磁盘 sector 可原子写范围内。
物理文件大小仍然固定为 `PG_CONTROL_FILE_SIZE` 8192 bytes。
这样未来格式不兼容时更容易读出错误版本，而不是短读。
### 5.4 StartupXLOG() 先解释 DBState
`StartupXLOG()` 是 startup process 的主入口。
见 `xlog.c:5846`。
第一步检查 `ControlFile->checkPoint` 是否是有效 record offset。
`xlog.c:5877-5880` 不接受无效 checkpoint location。
接着按 `ControlFile->state` 打日志：
`DB_SHUTDOWNED` 表示 clean shutdown；`DB_SHUTDOWNED_IN_RECOVERY` 表示 recovery 中 clean shutdown；`DB_SHUTDOWNING`、`DB_IN_CRASH_RECOVERY`、`DB_IN_ARCHIVE_RECOVERY`、`DB_IN_PRODUCTION` 分别表示 shutdown、crash recovery、archive recovery 或生产态被中断。
这组日志是现场诊断第一入口。
例如普通 crash 常见：
```text
database system was interrupted; last known up at ...
database system was not properly shut down; automatic recovery in progress
```
注意：
`state` 只是入口判断。
它不是最终恢复成功的证明。
真正能否恢复，还要看 checkpoint record、WAL 序列和 cleanup。
### 5.5 crash 后先清临时 WAL 并同步数据目录
如果 control file state 不是 `DB_SHUTDOWNED` 或 `DB_SHUTDOWNED_IN_RECOVERY`，`StartupXLOG()` 认为存在 crash。
见 `xlog.c:5973-5981`。
它先执行：
```text
RemoveTempXlogFiles()
SyncDataDirectory()
didCrash = true
```
`RemoveTempXlogFiles()` 在 `xlog.c:3883-3903`。
它扫描 `pg_wal`，删除 `xlogtemp.*`。
这些临时 WAL segment 是创建新 segment 时留下的中间状态。
`SyncDataDirectory()` 的目的不是把数据恢复正确。
它的目的是处理 OS/文件系统层面的持久化风险：
crash 前可能有一些写入已经发出但尚未 fsync。
如果紧接着再次掉电，早先未 fsync 的写可能丢失，而 recovery 后的新写却保留。
启动时同步整个 data directory，是为了把这个危险窗口收紧。
这一步会很贵。
在大实例、慢盘、很多 relations 或大量 dirty metadata 的场景，startup 时间可能主要花在这里，而不是 WAL replay。
### 5.6 InitWalRecovery() 决定入口
`StartupXLOG()` 接着调用：
```text
InitWalRecovery(ControlFile, &wasShutdown, &haveBackupLabel, &haveTblspcMap)
```
见 `xlog.c:5991-5992`。
`InitWalRecovery()` 的注释在 `xlogrecovery.c:437-454`。
它做三件事：
分析 control file、`backup_label`、`tablespace_map`，并决定是否要 crash/archive recovery 以及 replay 到哪里才一致。
它先读 recovery signal files。
`readRecoverySignalFile()` 在 `xlogrecovery.c:983-1065`。
`standby.signal` 优先于 `recovery.signal`。
老的 `recovery.conf` 直接 FATAL。
它再分两条入口。
第一条是有 `backup_label`。
`read_backup_label()` 在 `xlogrecovery.c:1147-1299`。
如果存在 `backup_label`，恢复从 label 指定的 checkpoint 开始，而不是 `pg_control`。
这是 online backup 的关键安全规则。
因为备份中的 `pg_control` 可能比 backup start 晚几个 checkpoint。
第二条是没有 `backup_label`。
这就是普通 crash restart 的主路径。
`xlogrecovery.c:718-724` 从 control file 取：
```text
CheckPointLoc = ControlFile->checkPoint
CheckPointTLI = ControlFile->checkPointCopy.ThisTimeLineID
RedoStartLSN = ControlFile->checkPointCopy.redo
RedoStartTLI = ControlFile->checkPointCopy.ThisTimeLineID
```
然后调用 `ReadCheckpointRecord()` 读取 `CheckPointLoc` 上的 checkpoint record。
### 5.7 ReadCheckpointRecord() 校验 checkpoint record
`ReadCheckpointRecord()` 在 `xlogrecovery.c:4060-4104`。
它不是“读到 record 就行”。
它要求：
- `RecPtr` 是合法 record offset。
- `ReadRecord()` 能读到 record。
- `xl_rmid == RM_XLOG_ID`。
- `xl_info` 是 `XLOG_CHECKPOINT_SHUTDOWN` 或 `XLOG_CHECKPOINT_ONLINE`。
- record 长度正好是 checkpoint payload 的长度。
如果读不到，普通 control-file 路径直接 FATAL。
`xlogrecovery.c:731-741` 说明现在不再退回 secondary checkpoint。
这也是一个重要版本点。
有些旧资料会提 secondary checkpoint fallback。
本基线中这条 fallback 已经不存在。
### 5.8 校验 redo location 存在
读到 checkpoint record 后，`InitWalRecovery()` 把 payload copy 到 `checkPoint`。
如果 `checkPoint.redo < CheckPointLoc`，说明这是 online checkpoint。
此时还要验证 redo location 本身能读到 record。
见 `xlogrecovery.c:746-753`。
为什么要验证？
因为 recovery 真正要从 `checkpoint.redo` replay。
checkpoint record 可读不代表 redo 起点所在 WAL 仍然存在。
如果 WAL 被错误删除、归档缺失或 backup label 错误，继续启动会越过必须 replay 的历史。
在 backup_label 路径中，缺 redo location 会给更具体的 hint。
`xlogrecovery.c:588-599` 提示：
- 如果是恢复备份，应创建 recovery signal 并配置恢复选项。
- 如果不是恢复备份，可能是遗留 `backup_label`。
- 但删除 `backup_label` 可能导致从备份恢复的 cluster 损坏。
这类 hint 很有工程价值：
同一个错误可能是运维误删文件，也可能是恢复流程入口错了。
### 5.9 判断是否需要 recovery
`InitWalRecovery()` 在 `xlogrecovery.c:857-875` 决定 `InRecovery`。
主要条件：
```text
checkPoint.redo < CheckPointLoc
ControlFile->state != DB_SHUTDOWNED
ArchiveRecoveryRequested
```
第一条很容易忽略。
即使 control file state 看起来 clean，如果 checkpoint 是 online checkpoint，`redo < checkpoint` 也意味着必须 replay。
shutdown checkpoint 则不允许 `redo < checkpoint`。
如果 shutdown checkpoint 中 redo 更早，源码 PANIC。
当进入 recovery 时，内存中的 control file 会被更新：
```text
state = DB_IN_CRASH_RECOVERY 或 DB_IN_ARCHIVE_RECOVERY
checkPoint = CheckPointLoc
checkPointCopy = checkPoint payload
minRecoveryPoint 可能初始化
```
注意 `xlogrecovery.c:877-884`：
这里还不写磁盘。
因为 `StartupXLOG()` 还要初始化一批子系统。
真正 `UpdateControlFile()` 在 `xlog.c:6132-6140`。
### 5.10 初始化事务、SLRU、slot 和 relcache 状态
回到 `StartupXLOG()`。
`xlog.c:5995-6004` 从 checkpoint 初始化事务相关 shared state：
- `nextXid`。
- `nextOid`。
- MultiXact next id 和 offset。
- clog oldest。
- xid/multixact limit。
- commit timestamp limit。
然后它无条件删除 relcache init files。
`xlog.c:6006-6018` 的注释很直接：
如果做 WAL replay，旧 relcache init file 很可能和数据库现实不一致；
即使 clean shutdown 理论上可以保留，也统一删除更安全。
接着初始化 replication slots、logical decoding status、reorder buffer、CLOG、MultiXact、CommitTs、replication origin。
这些不是本节主角，但要知道为什么 control file 的 checkpoint payload 不只是 storage 状态。
recovery 要把事务分配器、SLRU 和逻辑复制相关状态放到能 replay 后续 WAL 的位置。
### 5.11 写入“正在恢复”的 pg_control
如果 `InRecovery` 为 true，`StartupXLOG()` 在 `xlog.c:6132-6140` 调用 `UpdateControlFile()`。
这个更新很关键。
如果 recovery 自己中途再次 crash，下次启动能看到：
```text
DB_IN_CRASH_RECOVERY
```
或：
```text
DB_IN_ARCHIVE_RECOVERY
```
同时 control file 也记录本次选择的 checkpoint。
这样下次 restart 不会误以为之前已经正常完成。
`UpdateControlFile()` 最终调用 `update_controlfile(DataDir, ControlFile, true)`。
`controldata_utils.c:197-205` 更新时间并重算 CRC。
`controldata_utils.c:238-252` 写满 `PG_CONTROL_FILE_SIZE`。
`controldata_utils.c:257-270` 在 `do_sync` 为 true 时 fsync。
backend 中这些失败会 PANIC。
control file 更新失败不能降级。
因为如果系统继续运行，下一次 crash restart 会从错误状态恢复。
### 5.12 recovery 前清理 unlogged 旧内容
`StartupXLOG()` 在 redo 前调用：
```text
ResetUnloggedRelations(UNLOGGED_RELATION_CLEANUP)
```
见 `xlog.c:6193-6199`。
为什么在 redo 前？
因为 recovery 中 unlogged relation 的非 init fork 可能是 crash 前遗留垃圾。
如果 hot standby 之后允许只读连接读取它们，会暴露错误内容。
所以必须在允许 Hot Standby 前清掉。
`reinit.c:38-45` 定义两种操作：
- `UNLOGGED_RELATION_CLEANUP`：删除有 init fork 的 unlogged relation 的其它 fork。
- `UNLOGGED_RELATION_INIT`：把 init fork copy 成 main fork。
cleanup 的具体实现是两遍扫描。
`reinit.c:170-185` 先用 hash table 记录有 init fork 的 relfilenumber。
`reinit.c:230-265` 第二遍删除同 relfilenumber 的其它 fork，但保留 init fork。
这一步不依赖 WAL。
unlogged relation 本来就不承诺 crash 后保留数据。
它承诺的是 crash restart 后 relation 回到 init fork 定义的空状态。
### 5.13 ordinary crash recovery 的 minRecoveryPoint 是 invalid
`StartupXLOG()` 在 `xlog.c:6169-6188` 初始化 local min recovery point。
如果是 archive recovery：
```text
LocalMinRecoveryPoint = ControlFile->minRecoveryPoint
LocalMinRecoveryPointTLI = ControlFile->minRecoveryPointTLI
```
如果是 crash recovery：
```text
LocalMinRecoveryPoint = InvalidXLogRecPtr
LocalMinRecoveryPointTLI = 0
```
这影响 `CheckRecoveryConsistency()`。
`xlogrecovery.c:2156-2161` 写得很明确：
普通 crash recovery 中，没有 valid `minRecoveryPoint`，所以不会通过这个函数中途达到 consistent state。
它必须 replay all WAL。
这也是本节最容易混淆的点：
日志里的 `consistent recovery state reached at ...` 通常是 archive recovery / standby 场景。
普通 crash restart 的一致性边界更接近：
```text
redo loop 读到本地 WAL 末尾
FinishWalRecovery()
end-of-recovery cleanup
ControlFile->state = DB_IN_PRODUCTION
SharedRecoveryState = RECOVERY_STATE_DONE
```
### 5.14 PerformWalRecovery() 初始化 replay progress
进入 `PerformWalRecovery()`。
见 `xlogrecovery.c:1612-1877`。
它先初始化 `XLogRecoveryCtl` 里的 replay progress。
`xlogrecovery.c:1618-1641` 设置：
- `lastReplayedReadRecPtr`。
- `lastReplayedEndRecPtr`。
- `lastReplayedTLI`。
- `replayEndRecPtr`。
- recovery pause state。
如果 `RedoStartLSN < CheckPointLoc`，说明 redo 起点在 checkpoint record 之前。
此时还没有 replay 任何实际 record，`lastReplayedReadRecPtr` 设为 invalid，`lastReplayedEndRecPtr` 设成 redo start。
如果是 shutdown checkpoint，redo 起点就是 checkpoint record。
此时 replay progress 可以从 checkpoint record 位置开始。
### 5.15 先做一次 consistency check
`PerformWalRecovery()` 在读第一条 redo record 前调用 `CheckRecoveryConsistency()`。
见 `xlogrecovery.c:1653-1656`。
这服务 archive recovery：
如果从一个 shutdown checkpoint 启动，并且 `minRecoveryPoint` 已经满足，理论上可以很快进入 consistent 状态。
普通 crash recovery 下它会直接返回。
因为 `minRecoveryPoint` invalid。
### 5.16 找到第一条要 replay 的 record
`PerformWalRecovery()` 接着决定第一条 record。
在线 checkpoint：
```text
RedoStartLSN < CheckPointLoc
XLogPrefetcherBeginRead(..., RedoStartLSN)
ReadRecord(..., PANIC, ...)
record 必须是 XLOG_CHECKPOINT_REDO
```
见 `xlogrecovery.c:1662-1678`。
这一步把上一节 checkpoint 课的结论反向验证：
在线 checkpoint 的 `checkpoint.redo` 指向 `XLOG_CHECKPOINT_REDO`。
如果那里不是这类 record，说明 control file/checkpoint/WAL 组合不自洽，FATAL。
shutdown checkpoint：
```text
RedoStartLSN >= CheckPointLoc
从 CheckPointLoc 后读取下一条 record
```
见 `xlogrecovery.c:1680-1686`。
如果没有后续 record，普通 crash recovery 会记录：
```text
redo is not required
```
### 5.17 WAL reader 的可信顺序
`ReadRecord()` 是 recovery 层包装。
它调用 `XLogPrefetcherReadRecord()`，底层依赖 `xlogreader.c`。
`XLogBeginRead()` 在 `xlogreader.c:232-244` 设置起始 LSN。
第一次读时，`NextRecPtr` 和 `EndRecPtr` 都指向调用者给的 LSN。
`XLogDecodeNextRecord()` 做关键解析：
- 读包含 record header 的 WAL page。
- 校验 page header。
- 处理 page header 大小。
- 读取 `xl_tot_len`。
- 校验 record header。
- 处理跨页 continuation record。
- 校验 continuation record 长度。
- 校验 record CRC。
- decode block references 和 main data。
record header 校验在 `xlogreader.c:1138-1191`。
它检查：
- `xl_tot_len >= SizeOfXLogRecord`。
- rmgr id 合法。
- random access 时 `xl_prev < RecPtr`。
- sequential read 时 `xl_prev == PrevRecPtr`。
CRC 校验在 `xlogreader.c:1204-1227`。
注释强调：
除了为了知道要读多少字节的最小信息外，不相信 record 内容，直到 CRC 通过。
page header 校验在 `xlogreader.c:1235-1371`。
它检查：
- `xlp_magic`。
- page flags。
- long header 中的 system identifier。
- WAL segment size。
- `XLOG_BLCKSZ`。
- `xlp_pageaddr`。
- timeline 不倒退。
这就是为什么 `pg_control` 中的 `system_identifier` 和 `xlog_seg_size` 必须早期读出。
WAL 文件不仅要“存在”，还必须属于同一个 cluster、同一 WAL layout。
### 5.18 WAL 来源 fallback 与终止规则
`ReadRecord()` 的行为依赖 recovery 模式。
见 `xlogrecovery.c:3107-3244`。
如果 record 无效：
- crash recovery 不在 standby mode，通常返回 NULL。
- 如果 `emode` 是 PANIC，错误不会返回。
- standby mode 会循环等待或切换来源。
- archive recovery 可从 archive、`pg_wal`、stream 之间切换。
`WaitForWALToBecomeAvailable()` 的状态机在 `xlogrecovery.c:3533-3568` 注释中：
```text
archive 或 pg_wal
promotion trigger check
stream
rescan timelines
sleep and retry
```
普通 crash restart 不等远端 WAL。
它只 replay 本地 `pg_wal` 中可用的连续 WAL。
当读到无效 record 或 WAL 末尾，redo loop 结束。
但有两个区别要记住：
第一，读 checkpoint record 或 redo 起点时，缺失是硬错误。
这些是必须存在的入口。
第二，如果 archive recovery 被请求，但前面还在 crash recovery 阶段，读完本地 WAL 后可以切换到 archive recovery。
`xlogrecovery.c:3202-3228` 会设置 `InArchiveRecovery`、`minRecoveryPoint`，再检查 consistency。
### 5.19 redo apply loop
主循环在 `xlogrecovery.c:1707-1806`。
每条 record 的时间线是：
```text
ProcessStartupProcInterrupts()
recovery pause / apply delay / target stop 检查
ApplyWalRecord()
Wake LSN waiters
recovery target after 检查
ReadRecord() 读下一条
```
普通 crash recovery 没有 hot standby 查询需要服务，但同一套逻辑也覆盖 standby。
所以你会看到 pause、delay、target stop 等代码。
不要把这些分支误解成普通 crash restart 必经路径。
### 5.20 ApplyWalRecord() 的状态推进
`ApplyWalRecord()` 在 `xlogrecovery.c:1883-2040`。
它先设置 error context。
如果 rmgr redo 报错，日志能带上：
```text
WAL redo at <LSN> for <rmgr description and block info>
```
然后它推进 `nextXid`。
`xlogrecovery.c:1894-1897` 保证事务分配器超过 record 的 xid。
接着处理 timeline switch。
`xlogrecovery.c:1907-1939` 对 `XLOG_CHECKPOINT_SHUTDOWN` 和 `XLOG_END_OF_RECOVERY` 做特殊判断。
如果 record 切换 timeline，先校验再更新 `replayTLI`。
之后，它在真正 redo 前更新：
```text
XLogRecoveryCtl->replayEndRecPtr = record->EndRecPtr
XLogRecoveryCtl->replayEndTLI = replayTLI
```
见 `xlogrecovery.c:1942-1949`。
这个顺序服务 archive recovery 中的 `XLogFlush()` / `minRecoveryPoint` 更新。
redo 中如果某个路径需要确保数据 flush 边界，它要知道当前 record 的 end LSN。
再之后：
- hot standby 记录 known assigned xids。
- `xlogrecovery_redo()` 处理 recovery 直接关心的 XLOG rmgr record。
- `GetRmgr(record->xl_rmid).rm_redo(xlogreader)` 调用真正 rmgr redo。
- 如有一致性检查标志，校验 backup page consistency。
- 成功后更新 `lastReplayedReadRecPtr`、`lastReplayedEndRecPtr`、`lastReplayedTLI`。
- 唤醒 walsender 或 walreceiver reply。
- 调用 `CheckRecoveryConsistency()`。
- timeline switch 后清掉非父 timeline 的 WAL 文件。
这里最重要的不变量：
```text
replayEndRecPtr 表示正在 replay 的 record end。
lastReplayedEndRecPtr 只有 record 成功 redo 后才推进。
```
不要用一个 LSN 同时表达“正在处理”和“已经完成”。
---
## 6. consistent state 到底是什么
### 6.1 普通 crash recovery 的一致点
普通 crash recovery 中：
```text
minRecoveryPoint = InvalidXLogRecPtr
ArchiveRecoveryRequested = false
```
`CheckRecoveryConsistency()` 在 `xlogrecovery.c:2156-2161` 直接返回。
注释写得很清楚：
```text
During crash recovery, we don't reach a consistent state until we've replayed all the WAL.
```
所以普通 crash restart 的一致点不是某条 checkpoint record。
也不是 `checkpoint.redo`。
而是：
```text
从 checkpoint.redo replay 到本地连续 WAL 末尾，
完成 end-of-recovery 动作，
清理 unlogged/temp/reader 状态，
更新 control file 为 DB_IN_PRODUCTION，
把 SharedRecoveryState 改成 RECOVERY_STATE_DONE。
```
这就是用户看到数据库可以重新接受连接和写 WAL 的边界。
### 6.2 archive/base-backup recovery 的一致点
archive/base-backup recovery 有中途一致点。
`CheckRecoveryConsistency()` 的核心条件在 `xlogrecovery.c:2204-2206`：
```text
!reachedConsistency
!backupEndRequired
minRecoveryPoint <= lastReplayedEndRecPtr
```
在此之前，如果还有 `backupEndRequired`，需要看到 `XLOG_BACKUP_END`。
`xlogrecovery_redo()` 在 `xlogrecovery.c:2077-2096` 处理 backup end record，把 `backupEndPoint` 设为当前 LSN。
下一次 consistency check 会调用 `ReachedEndOfBackup()`。
到达一致点时还要做两个检查：
- `XLogCheckInvalidPages()`。
- `CheckTablespaceDirectory()`。
如果通过，才：
```text
reachedConsistency = true
SendPostmasterSignal(PMSIGNAL_RECOVERY_CONSISTENT)
LOG "consistent recovery state reached at ..."
```
见 `xlogrecovery.c:2207-2225`。
Hot Standby 还需要有效 starting snapshot。
`xlogrecovery.c:2228-2245` 中，只有 `standbyState == STANDBY_SNAPSHOT_READY` 且 `reachedConsistency` 后，才发 `PMSIGNAL_BEGIN_HOT_STANDBY`。
### 6.3 为什么普通 crash recovery 不宣布中途一致
普通 crash recovery 的目标是恢复 crash 前本地已经持久化的 WAL 历史。
本地 `pg_wal` 中最后一条完整可信 record 之前的所有 record 都可能已经影响过数据页、SLRU、relmap、fsm、visibility map、事务状态或 catalog。
如果中途开放：
- 后续 WAL 可能包含事务 commit/abort。
- 后续 WAL 可能删除或 truncate 前面提到的 invalid page。
- 后续 WAL 可能推进 CLOG/MultiXact/relmap。
- 后续 WAL 可能完成 relation drop 或 database drop。
所以 crash recovery 的一致性不是“达到 checkpoint”。
checkpoint 是恢复入口，不是恢复终点。
---
## 7. invalid pages：延迟证明而不是静默忽略
`xlogutils.c` 的 invalid page 机制是本节的关键异常路径。
场景：
redo 某条 WAL record 时，它引用了一个 page。
但磁盘上这个 page 不存在，或者读出来是 all-zero page。
直接 PANIC 太激进。
因为 WAL 后面可能有 drop/truncate record，说明这个 page 在 crash 前确实已经被删除。
直接忽略也不安全。
因为如果后面没有 drop/truncate，这就是缺页或损坏。
所以 PostgreSQL 做 deferred validation：
```text
先记 invalid_page_tab
后续 drop/truncate/database drop 可以 forget
达到一致点或一致点之后仍存在则报错
```
`xlogutils.c:56-64` 注释说明：
这种情况主要在 `full_page_writes = off` 时可能出现。
如果 full-page writes 开启，第一次引用 page 通常应该带 full-page rewrite。
`XLogReadBufferExtended()` 在 `xlogutils.c:431-458` 解释模式差异。
`RBM_NORMAL` 下：
- page 不存在则返回 `InvalidBuffer`。
- page 是 all-zero 则返回 `InvalidBuffer`。
- 同时记录 invalid page。
具体代码：
- `xlogutils.c:502-507` 记录不存在的 page。
- `xlogutils.c:523-538` 记录 all-zero page。
记录函数 `log_invalid_page()` 在 `xlogutils.c:101-161`。
如果 `reachedConsistency` 已经为 true，再遇到 invalid page 直接 WARNING + PANIC。
原因是 consistent state 之后 invalid-page table 应该为空并保持为空。
后续消解入口：
- `XLogDropRelation()` 调 `forget_invalid_pages(..., 0)`。
- `XLogDropDatabase()` 调 `forget_invalid_pages_db()`。
- `XLogTruncateRelation()` 调 `forget_invalid_pages(..., nblocks)`。
见 `xlogutils.c:623-663`。
到一致点时，`XLogCheckInvalidPages()` 扫表。
见 `xlogutils.c:234-262`。
它先把所有 unresolved entries 以 WARNING 打出，再 PANIC。
如果 `ignore_invalid_pages` 打开，则降级为 WARNING。
`ignore_invalid_pages` 是逃生口，不是修复。
它会允许 recovery 继续，但可能丢掉本该 replay 的 page change。
课程实验中只能在 throwaway copy 上使用。
---
## 8. unlogged、temp 与 md 层 cleanup
### 8.1 unlogged relation 两阶段 reset
unlogged relation 的数据 fork 不写 WAL。
所以 crash restart 不能靠 redo 恢复它的旧内容。
正确策略是恢复到 init fork 定义的初始状态。
startup 中有两次 reset：
```text
redo 前：ResetUnloggedRelations(UNLOGGED_RELATION_CLEANUP)
redo 后：ResetUnloggedRelations(UNLOGGED_RELATION_INIT)
```
第一次在 `xlog.c:6193-6199`。
目标是删除旧 main/fsm/vm 等 fork，避免 recovery 中或 hot standby 中读取 crash 遗留垃圾。
第二次在 `xlog.c:6340-6348`。
注释说明：
要在 recovery 完成后做，这样 recovery 期间创建的 unlogged relations 也能包含进来；
但要在 recovery 被标记为成功完成前做，这样如果后续失败，下次 startup 会重试。
`reinit.c:47-100` 的总入口会扫描：
- `base`。
- `pg_tblspc/<oid>/<TABLESPACE_VERSION_DIRECTORY>`。
cleanup 阶段：
- 找到 init fork。
- 记录 relfilenumber 到 hash。
- 删除同 relfilenumber 的非 init fork。
init 阶段：
- 对每个 init fork 构造 main fork 路径。
- `copy_file(srcpath, dstpath)`。
- 再单独 fsync main fork。
- 最后 fsync database directory。
见 `reinit.c:280-365`。
这也是为什么 `xlogutils.c:397-404` 里 redo 恢复 init fork FPI 后会 `FlushOneBuffer()`。
恢复结束会不经过 shared buffers 复制 init fork 到 main fork。
因此 init fork 的磁盘状态必须和 buffer 中一致。
### 8.2 temp file 与 temp relation cleanup
temporary file 和 temporary relation 不是 WAL recovery 的对象。
它们的生命周期绑定 backend 或 postmaster crash cleanup。
`RemovePgTempFiles()` 在 `fd.c:3323-3365`。
它处理：
- `base/pgsql_tmp`。
- default tablespace 下的 temp relation 文件。
- 非 default tablespace 下的 `pgsql_tmp`。
- 非 default tablespace 下的 temp relation 文件。
`RemovePgTempFilesInDir()` 在 `fd.c:3383-3439`。
它删除 `pgsql_tmp` 下匹配 temp prefix 的文件或递归目录。
遇到系统调用错误通常 LOG 并继续。
`RemovePgTempRelationFilesInDbspace()` 在 `fd.c:3471-3495`。
它按 temp relation 文件名模式删除。
这种 cleanup 的正确性边界是：
没有其它进程正在使用这些 temp 文件。
postmaster 初始启动时满足这个条件。
post-backend-crash 重启时，要等所有相关子进程退出后再做。
### 8.3 md.c 在 redo 中的宽容边界
`md.c` 是 smgr 的 filesystem 实现。
它在 recovery 中有一些“看似宽容”的行为，但都是有边界的。
`mdcreate()` 在 `md.c:222-274`。
如果 `isRedo` 为 true，relation 已经存在时可以打开已有文件。
这支持 redo 幂等性：
crash 前某个 create 已经落盘，redo 再执行不能因为 EEXIST 中止。
`mdunlink()` 的注释在 `md.c:276-336`。
普通运行中，main fork 第一段通常会 truncate 为 0 并登记 checkpoint 后删除。
这样避免 relfilenumber 复用和 WAL-skipped 文件之间的危险交互。
但 `isRedo` 为 true 时不同。
`md.c:326-329` 说明 redo 中文件已经不存在是正常的；
并且可以立即删除，因为 redo 期间不会创建冲突 relation。
`mdtruncate()` 在 `md.c:1288-1385`。
如果请求 truncate 到的 `nblocks` 大于当前 `curnblk`：
- 普通运行报 ERROR。
- `InRecovery` 下直接 return。
这不是掩盖错误。
redo 中 relation 大小可能已经因为前序 crash 落盘状态、后续 WAL 或 replay 幂等性而和原始执行时不同。
truncate 超出当前大小没有更多工作可做。
`smgrdestroyall()` 在 `smgr.c:380-406`。
`xlog.c` 在 replay checkpoint record 后会调用它。
见 `xlog.c:8948-8955` 和 `xlog.c:9009-9015`。
原因是 startup process 不处理 shared invalidation，也不会像普通 backend 一样在事务结束调用 `AtEOXact_SMgr()`。
如果不定期销毁 unpinned smgr objects，被 drop 的 relation 的 smgr 状态会长期留着。
---
## 9. FinishWalRecovery 与回到 production
redo loop 结束后，`StartupXLOG()` 调用 `FinishWalRecovery()`。
见 `xlog.c:6283-6290`。
`FinishWalRecovery()` 做几件事。
第一，停止 WAL receiver 和 slot sync worker。
见 `xlogrecovery.c:1424-1456`。
结束 recovery 前不能让 WAL receiver 继续写入旧流。
第二，重新读取最后一个 valid/applied record。
见 `xlogrecovery.c:1458-1483`。
这用于精确计算：
- `lastRec`。
- `endOfLog`。
- 最后一页 partial block。
- incomplete continuation record 的位置。
如果 WAL 结尾有不完整跨页 record，`xlogreader.c:928-949` 会设置：
```text
abortedRecPtr
missingContrecPtr
```
普通 crash recovery 结束后，后续会写 `XLOG_OVERWRITE_CONTRECORD`，告诉下游 WAL 读者这段缺失是有意覆盖。
见 `xlog.c:6543-6548`。
第三，archive recovery 下关闭 archive fetching 相关状态，并为新 timeline 做准备。
回到 `StartupXLOG()` 后，先检查是否达到必要恢复边界。
`xlog.c:6298-6338` 处理：
- WAL ends before end of online backup。
- WAL ends before consistent recovery point。
然后做 `ResetUnloggedRelations(UNLOGGED_RELATION_INIT)`。
见 `xlog.c:6340-6348`。
接着预扫描 prepared transactions，启用 WAL segment creation，决定是否切 timeline。
普通 crash recovery 继续使用原 timeline。
archive recovery 会分配新 timeline。
见 `xlog.c:6363-6416`。
然后初始化 WAL insertion pointers。
`xlog.c:6444-6495` 把 `EndOfLog` 放入 WAL insert shared state 和 flush/write positions。
关键边界在 `xlog.c:6503-6506`：
```text
InRecovery = false
```
但这还不是普通 backend 可写 WAL 的最终边界。
随后还要：
- startup subtrans。
- trim CLOG/MultiXact。
- recover prepared transactions。
- shutdown xlogreader。
- 仅对 startup backend 允许本地 WAL insert。
- 写 overwrite contrecord。
- 更新 full_page_writes 状态。
- 写 end-of-recovery record 或请求 shutdown checkpoint。
- cleanup archive recovery signal files。
- 完成 commit timestamp 初始化。
- 更新 logical decoding status。
- 处理 data checksum in-progress 状态。
最后才在 `xlog.c:6651-6660`：
```text
LWLockAcquire(ControlFileLock, LW_EXCLUSIVE)
ControlFile->state = DB_IN_PRODUCTION
ControlFile->data_checksum_version = XLogCtl->data_checksum_version
XLogCtl->SharedRecoveryState = RECOVERY_STATE_DONE
UpdateControlFile()
LWLockRelease(ControlFileLock)
```
这才是对其它 backend 语义上开放的核心边界。
注释也承认：
存在很小窗口，backend 已可写 WAL，而 on-disk control file 还没显示 `DB_IN_PRODUCTION`。
但 shared recovery state 和 control file shared copy 在 `ControlFileLock` 下保持一致。
如果是 promotion，`xlog.c:6694-6701` 会请求一次 checkpoint。
这不是 correctness 必需。
它是为了避免后续 crash 从太老 restartpoint replay 太久。
---
## 10. 正确性机制层次
本节至少有七层正确性机制。
第一层：control file CRC 与版本校验。
它保证 startup 不会用错误格式解释 control file。
它不保证 checkpoint record 仍在 WAL 中。
第二层：checkpoint record 类型和长度校验。
`ReadCheckpointRecord()` 保证 `ControlFile->checkPoint` 指向的 record 确实是 checkpoint。
它不保证 redo 起点之后 WAL 连续完整。
第三层：redo location 验证。
`InitWalRecovery()` 确保 `checkPoint.redo` 可读。
在线 checkpoint 下，`PerformWalRecovery()` 还要求那里是 `XLOG_CHECKPOINT_REDO`。
第四层：WAL page/record 校验。
`xlogreader.c` 校验 page magic、system identifier、segment size、page address、timeline、prev-link、CRC。
它保证 record 序列可信。
它不保证每个数据页已经在磁盘上处于最新状态。
第五层：rmgr redo contract。
每个 rmgr 用 page LSN/FPI/record payload 决定是否应用。
它保证自身资源管理器的恢复语义。
它不能替其它 rmgr 修复状态。
第六层：invalid-page deferred validation。
redo 中缺页先记录，drop/truncate 可消解，一致点前必须清空。
它处理“后续 WAL 证明前面缺页合理”的情况。
第七层：end-of-recovery cleanup。
unlogged、temp、relcache init file、smgr object、exported snapshots、partial WAL continuation 等都不完全靠普通 rmgr redo 收尾。
它们在 startup 的固定位置被清理或重建。
一个可迁移的判断：
```text
crash safety 不是单点机制。
它通常是 durable metadata + append-only history + idempotent replay + deferred validation + cleanup phase 的组合。
```
---
## 11. 错误路径 / fallback
### 11.1 control file 错误
backend `ReadControlFile()` 中：
- 打不开或短读 `pg_control`：PANIC。
- control file version 不匹配：FATAL。
- CRC 错：FATAL。
- catalog/layout 参数不匹配：FATAL。
frontend `get_controlfile()` 的行为略不同。
`controldata_utils.c:147-162` 中，如果 CRC 不对，可能是并发读取到 server 正在写的中间状态。
frontend 会短暂重试，直到同一个坏 CRC 连续出现或重试次数耗尽。
backend startup 不用这个 retry。
它自己控制启动时序，不能在不可信 control file 上继续。
### 11.2 checkpoint record 缺失
本基线中，control-file 指向的 checkpoint record 读不到就是 FATAL。
`xlogrecovery.c:731-741` 明确不再尝试 secondary checkpoint。
诊断重点：
- `pg_control` 是否来自错误数据目录。
- `pg_wal` 是否缺 segment。
- 是否把 base backup 和不同时间点的 `pg_wal` 混用。
- 是否遗留错误 `backup_label`。
- timeline history 是否完整。
### 11.3 redo location 缺失
checkpoint record 可读，但 `checkPoint.redo` 缺失，也 FATAL。
这是 archive/backup 问题中很常见的现场。
对 online checkpoint 而言，`checkPoint.redo` 早于 checkpoint record。
只保留 checkpoint record 所在 segment 不够。
必须保留从 redo pointer 开始的所有 WAL。
### 11.4 WAL corruption 或不完整 record
WAL reader 发现 page header、record header、prev-link、CRC 错误时，把错误交给 recovery 层。
普通 crash recovery：
- checkpoint/redo 起点附近错误会 FATAL/PANIC。
- replay 到最后遇到“没有下一条完整 record”通常表示 WAL 末尾，redo loop 结束。
- 跨页 incomplete record 会记录 aborted/missing contrecord，结束后写 overwrite-contrecord。
standby/archive recovery：
- 可从 archive、`pg_wal`、stream 切换来源。
- standby mode 可等待新 WAL 到达。
- promotion trigger 会使等待结束。
### 11.5 recovery target 太早
如果 archive recovery 配了 target，但 target 在 consistent point 之前，`PerformWalRecovery()` 会 FATAL。
见 `xlogrecovery.c:1812-1817`。
这不是任性限制。
在 consistent point 之前，base backup 可能还没 replay 到 backup end，invalid pages 也可能还没被后续 drop/truncate 消解。
### 11.6 WAL ends before backup end
`StartupXLOG()` 在 `xlog.c:6316-6337` 检查：
如果还没达到 `LocalMinRecoveryPoint` 或 `backupStartPoint` 仍 valid，就说明 WAL 不够。
常见错误：
- 从 online backup 恢复，但没有归档所有 backup 期间生成的 WAL。
- 调了 `pg_backup_start()`，没拿到或没保留 `pg_backup_stop()` 对应 WAL。
- standby backup 中没有保证 `pg_control` 最后备份。
### 11.7 参数不足
`CheckRequiredParameterValues()` 在 `xlog.c:5801-5840`。
archive recovery 中，如果 WAL 是 `wal_level=minimal`，不能继续恢复。
Hot Standby 还要求本机参数不低于 primary 记录在 control file 中的值：
- `max_connections`。
- `max_worker_processes`。
- `max_wal_senders`。
- `max_prepared_transactions`。
- `max_locks_per_transaction`。
如果 Hot Standby 已经 active，`RecoveryRequiresIntParameter()` 会先 pause 并给用户机会调整。
最终仍会 FATAL。
### 11.8 invalid pages unresolved
`XLogCheckInvalidPages()` 会先输出每个 unresolved page，再 PANIC。
`ignore_invalid_pages` 可以降级为 WARNING。
这个 fallback 只能用于救援或只读提取数据的场景。
不能把它当成正常恢复开关。
### 11.9 unlogged cleanup 失败
`ResetUnloggedRelations()` 中删除、copy、fsync 失败会 ERROR。
因为 unlogged reset 是恢复完成前的硬边界。
如果不能把 unlogged relation 重置到 init fork，就不能安全开放系统。
temp cleanup 不同。
`RemovePgTempFiles()` 通常 LOG 后继续。
临时文件不构成持久化数据一致性的硬边界。
---
## 12. 成本、资源与跨模块传播
成本一：启动前全目录 fsync。
如果上次不是 clean shutdown，`SyncDataDirectory()` 可能扫描并 fsync 大量文件。
它的成本跟 relation 数、tablespace 数、文件系统 metadata 状态和存储延迟相关。
成本二：WAL replay 量。
从 `checkpoint.redo` 到 WAL 末尾的字节数越大，恢复越久。
checkpoint 频率、`max_wal_size`、写入压力、full-page writes、bulk load 都会影响 replay 距离。
成本三：WAL reader source fallback。
archive/standby recovery 中，找 WAL 可能经过 archive、`pg_wal`、stream、sleep retry。
每次失败都有系统调用、restore_command、walreceiver 状态切换或 timeline scan 成本。
成本四：rmgr redo 的 data page I/O。
如果 page 不在 buffer pool，redo 要读或扩展数据文件。
FPI 可直接 restore page，但也会产生写放大。
incremental redo 需要读旧页并比较 page LSN。
成本五：invalid-page hash。
正常情况下很小。
但如果 full_page_writes 关闭、WAL 缺失、文件损坏或 drop/truncate 密集，invalid_page_tab 可能积累大量 entries。
最终 consistency check 要逐条输出 warning。
成本六：unlogged reset。
它扫描所有 database directories 和 tablespaces。
大量 unlogged relation 或很多 segment 文件时，cleanup 和 init copy/fsync 都会拖慢 startup。
成本七：temp cleanup。
`RemovePgTempFiles()` 扫描 `pgsql_tmp` 和 temp relation 文件。
大量 spill 文件残留会让 postmaster 启动或 post-crash reinitialization 变慢。
但失败一般不阻止启动。
跨模块传播：
- `pg_control` 连接 xlog、checkpoint、backup、参数兼容性。
- `xlogreader` 连接 WAL segment 文件、timeline、system identifier。
- `xlogutils` 连接 rmgr redo、buffer manager、smgr、invalid page。
- `reinit.c` 连接 unlogged relation fork 语义、tablespace 目录、文件 fsync。
- `md.c` 连接 relation segment 生命周期、pending fsync、redo 幂等性。
- postmaster/fd 连接 backend crash、temp file 生命周期和启动时清理。
---
## 13. 观测与诊断入口
### 13.1 `pg_controldata`
`pg_controldata` 是 control file 观测入口。
它基于 `get_controlfile()`，会检查 CRC。
重点看：
- Database cluster state。
- Latest checkpoint location。
- Latest checkpoint's REDO location。
- Latest checkpoint's TimeLineID。
- Minimum recovery ending location。
- Backup start location。
- Backup end location。
- wal_level。
- max_connections 等参数。
- WAL block size 和 segment size。
诊断方式：
```text
pg_controldata $PGDATA
```
把 `Latest checkpoint location` 和 `Latest checkpoint's REDO location` 区分开。
如果 redo location 所在 WAL segment 不存在，restart 可能失败。
### 13.2 server log
普通 crash restart 常见日志链：
```text
database system was interrupted; last known up at ...
database system was not properly shut down; automatic recovery in progress
redo starts at ...
redo done at ... system usage: ...
database system is ready to accept connections
```
archive/base-backup recovery 还可能看到：
```text
starting archive recovery
starting backup recovery with redo LSN ...
consistent recovery state reached at ...
archive recovery complete
selected new timeline ID: ...
```
invalid pages 会看到：
```text
page <n> of relation <path> does not exist
WAL contains references to invalid pages
```
checkpoint/redo 缺失会看到：
```text
could not locate a valid checkpoint record at ...
could not find redo location ... referenced by checkpoint record at ...
```
### 13.3 `pg_waldump`
`pg_waldump` 能从 redo pointer 附近验证 WAL 序列。
示例：
```bash
pg_waldump -p "$PGDATA/pg_wal" -s <redo-lsn> -n 20
```
你应该检查：
- redo pointer 处是否是 `XLOG_CHECKPOINT_REDO`。
- checkpoint record 处是否是 `CHECKPOINT_ONLINE` 或 `CHECKPOINT_SHUTDOWN`。
- 后续 record 的 rmgr、LSN 是否连续。
注意：
`pg_waldump` 能证明 WAL record 可读。
它不能证明 rmgr redo 后的数据文件状态已经正确。
### 13.4 wait event 与统计
启动过程中常见 wait event：
- `ControlFileRead`。
- `ControlFileWriteUpdate`。
- `ControlFileSyncUpdate`。
- `WALRead`。
- `DataFileRead`。
- `DataFileWrite`。
- `DataFileSync`。
- `RecoveryApplyDelay`。
- `RecoveryPause`。
`pg_stat_io` 和 `pg_stat_wal` 是数据库起来后的累计视图。
它们不能完整还原 startup 期间每一条 redo 的因果。
但结合日志和时间戳，可以判断恢复慢主要是 WAL read、data file read/write、fsync 还是 unlogged/temp cleanup。
`pg_stat_recovery_prefetch` 在 recovery 中更有价值。
`ShutdownWalRecovery()` 在 `xlogrecovery.c:1571-1572` 会做 final update。
### 13.5 gdb 断点
源码跟读时建议断点：
```text
StartupXLOG
ReadControlFile
InitWalRecovery
ReadCheckpointRecord
PerformWalRecovery
ApplyWalRecord
CheckRecoveryConsistency
XLogCheckInvalidPages
ResetUnloggedRelations
FinishWalRecovery
UpdateControlFile
```
重点观察变量：
```text
ControlFile->state
ControlFile->checkPoint
ControlFile->checkPointCopy.redo
CheckPointLoc
RedoStartLSN
InRecovery
InArchiveRecovery
LocalMinRecoveryPoint
reachedConsistency
XLogRecoveryCtl->lastReplayedEndRecPtr
XLogRecoveryCtl->replayEndRecPtr
```
---
## 14. 常见误区
误区一：
把 `pg_control` 当作完整恢复事实。
正确理解：
`pg_control` 是入口元数据。
它必须被 checkpoint record 和 WAL reader 继续验证。
误区二：
把 checkpoint record 的 LSN 当作 redo 起点。
正确理解：
在线 checkpoint 下 redo 起点是 `checkPointCopy.redo`。
checkpoint record 只是发布这个 redo pointer 的 completion record。
误区三：
看到 clean shutdown 就认为一定不读 WAL。
正确理解：
如果有 recovery signal file，或 checkpoint redo/checkpoint location 关系表明需要 recovery，仍可能进入 recovery。
误区四：
把 `consistent recovery state reached` 当成所有 crash restart 都会有的日志。
正确理解：
普通 crash recovery 必须 replay 到 WAL 末尾。
`CheckRecoveryConsistency()` 的中途 consistent state 主要服务 archive/base-backup/hot standby。
误区五：
认为 invalid page 一出现就说明数据不可恢复。
正确理解：
它可能会被后续 drop/truncate record 消解。
一致点仍未消解才是严重问题。
误区六：
认为 unlogged relation crash 后会被 WAL 恢复。
正确理解：
unlogged relation 的数据 fork 不 WAL-log。
restart 后通过 init fork 重置。
误区七：
把 temp file cleanup 失败和数据一致性失败混为一谈。
正确理解：
temp cleanup 失败通常只 LOG。
它可能浪费磁盘空间或影响后续 temp file 命名碰撞处理，但不是持久化数据恢复的硬边界。
误区八：
把 `InRecovery` 当成所有进程可见的 recovery 状态。
正确理解：
`InRecovery` 是 startup process replay WAL 的局部语义。
普通 backend 应使用 `RecoveryInProgress()` 观察 shared recovery state。
---
## 15. 课堂实验
### 实验 1：观察普通 crash restart 的主链路
准备一个 throwaway cluster。
启用较详细日志：
```conf
log_min_messages = debug1
log_startup_progress_interval = 1s
```
步骤：
```bash
pg_ctl -D "$PGDATA" start
psql -c "create table crash_demo(id int primary key, payload text);"
psql -c "insert into crash_demo select g, repeat('x', 200) from generate_series(1, 50000) g;"
psql -c "checkpoint;"
psql -c "insert into crash_demo select g, repeat('y', 200) from generate_series(50001, 100000) g;"
kill -9 <postmaster-pid>
pg_ctl -D "$PGDATA" start
```
观察：
- 日志中的 `database system was interrupted`。
- `automatic recovery in progress`。
- `redo starts at`。
- `redo done at`。
- 是否有 `consistent recovery state reached`。
预期：
普通 crash recovery 不一定出现 `consistent recovery state reached`。
它通过 replay 到 WAL 末尾并完成 end-of-recovery 进入 production。
回源码解释：
- `StartupXLOG()` 根据 `DB_IN_PRODUCTION` 判断 crash。
- `InitWalRecovery()` 从 `pg_control` 找 checkpoint。
- `PerformWalRecovery()` replay。
- `FinishWalRecovery()` 找 end of WAL。
- `ControlFile->state` 最后更新为 `DB_IN_PRODUCTION`。
### 实验 2：用 `pg_controldata` 和 `pg_waldump` 对齐 checkpoint 与 redo
在 throwaway cluster 上：
```bash
pg_controldata "$PGDATA" | egrep 'Latest checkpoint|REDO|Database cluster state|TimeLine'
```
记录：
- Latest checkpoint location。
- Latest checkpoint's REDO location。
- TimeLineID。
然后：
```bash
pg_waldump -p "$PGDATA/pg_wal" -s "<redo-lsn>" -n 20
```
观察：
- redo pointer 处是否能读到 record。
- 如果是在线 checkpoint，起点附近是否有 checkpoint redo marker。
- checkpoint record 的位置是否晚于 redo pointer。
讨论：
为什么 control file 同时保存 `checkPoint` 和 `checkPointCopy.redo`？
如果只保存一个 LSN，会丢掉哪条恢复语义？
### 实验 3：验证 unlogged 与 temp cleanup
创建 unlogged table 和大 temp spill。
```sql
create unlogged table u_demo(id int, payload text);
insert into u_demo select g, repeat('u', 100) from generate_series(1, 10000) g;
create table t_demo(id int, payload text);
insert into t_demo select g, repeat('p', 100) from generate_series(1, 10000) g;
```
制造 temp 文件可以用小 `work_mem` 的排序或 hash aggregate。
然后强杀 postmaster。
重启后检查：
```sql
select count(*) from u_demo;
select count(*) from t_demo;
```
预期：
- permanent table 通过 WAL recovery 保留。
- unlogged table 回到 init fork 状态，通常为空。
- temp files 不出现在 SQL 层，目录残留由 `RemovePgTempFiles()` 尝试清理。
回源码解释：
- `ResetUnloggedRelations(UNLOGGED_RELATION_CLEANUP)` 在 redo 前。
- `ResetUnloggedRelations(UNLOGGED_RELATION_INIT)` 在 redo 后。
- temp cleanup 在 postmaster/fd 层，不是 WAL rmgr redo。
## 16. 讨论题
1. 为什么 `pg_control` 只保存 checkpoint record LSN 和 checkpoint payload copy，而不是保存所有需要恢复的状态？
2. 在线 checkpoint 下，为什么 `checkPoint` 和 `checkPointCopy.redo` 分离？
如果 crash recovery 从 checkpoint record 后面开始，会漏掉什么？
3. 普通 crash recovery 为什么不能在 `minRecoveryPoint` 达到后宣布 consistent？
这个问题本身哪里混淆了 crash recovery 和 archive recovery？
4. WAL reader 校验 system identifier、segment size、page address 和 CRC 分别防哪类错误？
哪些错误仍然要等 rmgr redo 才会暴露？
5. invalid page 为什么采用 deferred validation？
如果直接忽略或直接 PANIC，各自会破坏什么场景？
6. unlogged relation 为什么 redo 前 cleanup、redo 后 init？
如果顺序反过来，会出现哪些风险？
7. temp file cleanup 失败为什么通常不阻止启动，而 unlogged init copy/fsync 失败会阻止启动？
这反映了哪类对象的持久化承诺差异？
8. `ControlFile->state = DB_IN_PRODUCTION` 和 `SharedRecoveryState = RECOVERY_STATE_DONE` 为什么要在 `ControlFileLock` 下相邻更新？
如果二者被其它 backend 看到不一致，会有什么问题？
---
## 17. 本节小结
本节核心链路：
```text
ReadControlFile()
  -> StartupXLOG()
  -> InitWalRecovery()
  -> ReadCheckpointRecord()
  -> checkPoint.redo
  -> PerformWalRecovery()
  -> ApplyWalRecord()
  -> CheckRecoveryConsistency()
  -> FinishWalRecovery()
  -> ResetUnloggedRelations()
  -> PerformRecoveryXLogAction()
  -> ControlFile->state = DB_IN_PRODUCTION
```
核心边界：`ControlFile->checkPoint` 是 checkpoint record 位置，`ControlFile->checkPointCopy.redo` 是 replay 起点，`ControlFile->state` 是启动分支输入而不是恢复完成证明，`minRecoveryPoint` 服务 archive/base-backup recovery，`lastReplayedEndRecPtr` 只有 record 成功 redo 后才推进。
ownership / cleanup：control file 早期在 local memory，shared memory 初始化后复制到 shared `ControlFile`；更新时重算 CRC 并 fsync；WAL reader 属于 startup process；invalid-page table 是 recovery-local hash；unlogged relation 通过 init fork 重建；temp file 由 postmaster/fd 清理；checkpoint redo 后用 `smgrdestroyall()` 清理 unpinned smgr entries。
错误路径收尾：control file 格式/CRC 错、checkpoint record 缺失、redo location 缺失都会中止；WAL reader 在末尾可结束普通 crash recovery，但在必需位置会报错；standby/archive recovery 可切换来源或等待；recovery target 早于 consistent point 会 FATAL；invalid pages 未消解会 PANIC 或在 `ignore_invalid_pages` 下 WARNING；unlogged reset 失败不能开放系统，temp cleanup 失败通常只 LOG。
观测边界：`pg_controldata` 看 control file、checkpoint、redo、minRecoveryPoint 和 backup flags；server log 看 recovery 分支、redo 起止、一致点、timeline 和错误原因；`pg_waldump` 验证 WAL 从 redo pointer 起是否可读；wait event 帮助判断 control file、WAL read、data file sync/read/write 是否拖慢启动。单个 rmgr redo 的完整因果、invalid-page 的根因和 startup I/O 因果通常要结合断点、WAL_DEBUG 或完整 WAL 序列推断。
可迁移系统规律：crash restart 的一致性不是“读取一个 metadata 文件”，而是小型 durable metadata、可校验 append-only log、幂等 redo、延迟异常验证和 end-of-recovery cleanup 的组合协议。
当你分析其它存储系统的 crash recovery 时，也应该先找：
```text
入口元数据在哪里
入口如何校验
redo/undo 从哪里开始
日志如何证明顺序可信
什么时候宣布 consistent
哪些对象不在日志承诺范围内
异常状态如何延迟验证或清理
```
仍然依赖环境的判断：startup 慢要结合日志、I/O、文件数量和存储延迟；invalid pages 是否可由后续 WAL 消解要看完整 WAL 序列；archive recovery 的一致点依赖备份方式、`minRecoveryPoint`、timeline history 和 WAL 可用性；`ignore_invalid_pages` 只适合救援判断，不是正常运行配置。
