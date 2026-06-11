# PostgreSQL recovery target、pause 与 promote 恢复位置控制
## 课程定位
前置知识：已经理解上一节 startup process 如何进入 crash recovery、archive recovery、standby mode 和 Hot Standby consistent state。
也已经知道 walreceiver 只负责接收并写入 WAL，真正决定 replay 位置的是 startup process。
本节唯一主问题：
```text
PITR 或 standby 调试时如何用 recovery target、pause、resume 和 promote 控制恢复位置，
目标时间、XID、name、LSN 与 timeline 如何共同决定停止点？
```
核心矛盾：恢复过程希望给运维人员一个可控的停止点。
但 WAL replay 是按 record 顺序推进的，不是按 SQL 语句、事务开始顺序或墙钟时间连续可切片。
一个恢复目标必须同时回答几个问题：
```text
沿哪条 timeline 读 WAL？
目标是时间、XID、name、LSN，还是 earliest consistent point？
目标记录应该被包含还是排除？
到达目标后是暂停、升主，还是关闭？
如果用户手动 pause / resume / promote，是否还能继续等待目标？
```
PostgreSQL 的解法不是在任意 LSN 上打断 recovery。
它把控制面拆成：
```text
启动时解析的 recovery target GUC
  -> startup process 本地的 timeline 和目标状态
  -> redo loop 对每条 WAL record 做 before / after 判定
  -> shared memory 中的 pause / promote 状态
  -> end-of-recovery 选择新 timeline 并写入持久边界
```
学完后应能判断：
```text
为什么 recovery_target_time 不是按每条 WAL record 的物理时间停止；
为什么 recovery_target_xid 只能按事务结束记录做等值判断；
为什么 recovery_target_lsn 的比较点是 WAL record 的起始位置；
为什么 recovery_target_name 必须在 replay restore point record 之后才成立；
为什么 recovery_target_timeline 先限定历史，再由 target 决定停止点；
为什么 pause 不是立刻冻结 walreceiver，而是让 startup process 在安全点停止 replay；
为什么 promote 会解除 pause，并把恢复结束转换成新 timeline。
```
本课基于本地源码 `/home/nail/postgres`，分支 `master`，提交 `0e1f1ed157e90741e12a3715909e1b2d71ff9344`。
## 1. 本节在总主线中的位置
前几节已经建立了物理复制和恢复的基础链路：
```text
walsender 发送 WAL
  -> walreceiver 接收并 flush 到 pg_wal
  -> startup process 从 pg_wal / archive / stream 选择下一条 WAL
  -> redo record 修改数据页和共享恢复状态
```
上一节重点是：
```text
数据库什么时候到达 consistent state，
什么时候允许 Hot Standby 只读查询。
```
这一节继续沿 startup process 往后走。
问题从“能不能开放查询”变成：
```text
已经能查询以后，如何精确控制 replay 停在什么位置？
```
这个问题同时覆盖两个真实场景。
第一类是 PITR。
你从 base backup 加 archive 恢复，希望回到某个误操作之前或某个 restore point。
这时使用 `recovery.signal`、`restore_command` 和 recovery target 参数。
第二类是 standby 调试或故障切换演练。
你希望暂停 replay 看当前只读视图，或者确认某个 LSN 已经 replay，再 resume 或 promote。
这时常用 SQL 函数：
```text
pg_wal_replay_pause()
pg_wal_replay_resume()
pg_promote()
pg_is_wal_replay_paused()
pg_get_wal_replay_pause_state()
pg_last_wal_replay_lsn()
```
这两类场景共享同一个核心控制面：
```text
startup process 的 redo loop 是唯一能安全改变 replay 位置的地方。
```
walreceiver 可以继续接收 WAL。
用户会话可以设置 pause 请求。
postmaster 可以转发 promote 信号。
但真正停下、继续、结束恢复和切 timeline，都必须落回 startup process 的 replay 主循环。
## 2. 核心矛盾与一句话运行模型
一句话运行模型：
```text
InitWalRecovery() 解析 signal 文件和 recovery target；
validateRecoveryParameters() 选定目标 timeline；
PerformWalRecovery() 每读到一条 WAL record，
先检查手动 pause，再用 recoveryStopsBefore() 判断是否应在 record 前停，
必要时等待 recovery_min_apply_delay，
然后 ApplyWalRecord()，
再用 recoveryStopsAfter() 判断是否应在 record 后停；
到达目标后按 recoveryTargetAction 选择 pause、promote 或 shutdown。
```
这里的 tension 是：
```text
用户想用业务概念描述恢复点
  vs
WAL replay 只能在 record 边界、timeline 历史和一致性边界上做决定
```
业务概念可能是：
```text
恢复到 10:30 之前
恢复到事务 123456
恢复到名为 before_drop_table 的 restore point
恢复到某个 LSN
恢复到 base backup 完成后最早一致点
```
源码中真正可判断的对象却是：
```text
XLogReaderState 当前 record
record->ReadRecPtr
record 的 rmgr 和 info
commit / abort record 中的 timestamp 和 xid
XLOG_RESTORE_POINT record 中的 name
reachedConsistency
expectedTLEs 中允许读取的 timeline history
```
所以 recovery target 不是一个万能断点。
它是一个把用户目标压缩到 WAL record 边界上的判定器。
pause / resume 也不是数据接收开关。
它们控制的是 replay。
standby 暂停时，walreceiver 仍可能继续接收并 flush WAL。
因此诊断时必须分开看：
```text
pg_last_wal_receive_lsn()
pg_last_wal_replay_lsn()
pg_get_wal_replay_pause_state()
pg_stat_recovery.pause_state
```
promote 也不是“继续 replay 到所有可用 WAL 末尾后自然结束”的同义词。
promotion 是一个请求：
```text
停止继续等待 upstream 新 WAL，
完成当前恢复边界，
写 end-of-recovery 记录或 checkpoint，
选择新 timeline，
把系统切到可写 primary。
```
## 3. 核心文件分工与阅读顺序
本节阅读顺序按控制面推进，不按文件名排序。
| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/access/xlogrecovery.h` | `RecoveryTargetType`、`RecoveryTargetAction`、`RecoveryPauseState`、`XLogRecoveryCtlData`。 |
| 2 | `src/backend/utils/misc/guc_parameters.dat` | recovery target、timeline、action、inclusive、`restore_command`、`primary_conninfo` 的 GUC 入口。 |
| 3 | `src/backend/access/transam/xlogrecovery.c` | `InitWalRecovery()`、`validateRecoveryParameters()`、`PerformWalRecovery()`、`recoveryStopsBefore()`、`recoveryStopsAfter()`、pause/promote 状态。 |
| 4 | `src/backend/access/transam/xlogfuncs.c` | `pg_wal_replay_pause()`、`pg_wal_replay_resume()`、`pg_promote()`、`pg_last_wal_replay_lsn()`。 |
| 5 | `src/backend/access/transam/xlog.c` | `StartupXLOG()`、end-of-recovery、新 timeline、`PerformRecoveryXLogAction()`。 |
| 6 | `src/backend/postmaster/startup.c` | startup process 的 promote 信号标记和 `WakeupRecovery()` 协作。 |
| 7 | `src/backend/replication/walreceiver.c` | streaming timeline 与 history file 拉取，promotion 后 timeline 避免冲突。 |
| 8 | `src/backend/catalog/system_views.sql` | `pg_stat_recovery` 如何暴露 replay、timeline、pause 和 promote 状态。 |
推荐读法是：
```text
先读 enum 和 shared state
  -> 再读 GUC 如何设置 startup process 本地变量
  -> 再读 redo loop 的 before / after 判断
  -> 再读 SQL 函数如何改变 shared pause / promote 请求
  -> 最后读 end-of-recovery 如何选择新 timeline
```
不要从 `recovery_target_time`、`recovery_target_xid` 等参数逐个背诵。
它们只是不同输入形式。
真正的主线是：
```text
目标被解析成状态；
状态被绑定到 timeline；
redo loop 在 record 边界判断；
目标命中后 action 改变恢复生命周期。
```
## 4. 控制面的状态边界
### 4.1 recovery target 是 startup process 本地决策
`xlogrecovery.h` 中的目标类型很小：
```text
RECOVERY_TARGET_UNSET
RECOVERY_TARGET_XID
RECOVERY_TARGET_TIME
RECOVERY_TARGET_NAME
RECOVERY_TARGET_LSN
RECOVERY_TARGET_IMMEDIATE
```
这些值通过 GUC hook 间接设置到全局变量：
```text
recoveryTarget
recoveryTargetInclusive
recoveryTargetAction
recoveryTargetXid
recoveryTargetTime
recoveryTargetName
recoveryTargetLSN
```
它们不是共享内存状态。
核心消费方是 startup process。
普通 backend 不会拿这些变量判断自己能否查询。
这很重要。
`recovery_target_time` 不是一个运行时可频繁调整的开关。
在 `guc_parameters.dat` 中，recovery target 相关参数都是 `PGC_POSTMASTER`。
这意味着：
```text
目标点在服务器启动时确定；
到达目标后如果发现不合适，应关闭、改配置、重新启动恢复。
```
`recovery_target_action` 的默认值是 `RECOVERY_TARGET_ACTION_PAUSE`。
这不是为了让生产 standby 长期停住，而是为了 PITR 验证：
```text
先停在目标点；
允许只读查询确认数据状态；
确认正确后执行 pg_wal_replay_resume()，
让恢复结束并进入 promotion 路径。
```
如果 `hot_standby` 没有启用，源码会把 `pause` 改成 `shutdown`。
因为没有 Hot Standby 查询入口时，pause 不能让用户检查目标点。
### 4.2 pause / promote 是共享恢复状态
pause 和 promote 不是本地 GUC。
它们在 `XLogRecoveryCtlData` 中：
```text
SharedPromoteIsTriggered
recoveryPauseState
recoveryWakeupLatch
recoveryNotPausedCV
lastReplayedReadRecPtr
lastReplayedEndRecPtr
lastReplayedTLI
replayEndRecPtr
replayEndTLI
recoveryLastXTime
currentChunkStartTime
```
这块共享内存回答两个问题。
第一，startup process 当前 replay 到哪里了。
普通 backend 通过 `GetXLogReplayRecPtr()`、`pg_last_wal_replay_lsn()` 或 `pg_stat_recovery` 看到这类状态。
第二，用户或 postmaster 是否请求了控制动作。
`pg_wal_replay_pause()` 会调用 `SetRecoveryPause(true)`。
`pg_wal_replay_resume()` 会调用 `SetRecoveryPause(false)`。
`pg_promote()` 创建 `promote` signal file 并通知 postmaster。
startup process 后续用 `CheckForStandbyTrigger()` 把 signal file 转成 `SharedPromoteIsTriggered`。
共享状态用 `info_lck` spinlock 保护。
pause 的等待用 condition variable。
唤醒 startup process 用独立的 `recoveryWakeupLatch`。
源码注释特意说明：
```text
startup process 等 recovery pause / WAL arrival 的 latch，
和等待 recovery conflict 的 procLatch 不是同一个语义。
```
### 4.3 timeline 是目标空间，不是停止动作
recovery target 还需要一个 timeline 空间。
`xlogrecovery.c` 中相关状态是：
```text
recoveryTargetTimeLineGoal
recoveryTargetTLIRequested
recoveryTargetTLI
expectedTLEs
curFileTLI
```
`recovery_target_timeline` 支持三种目标：
```text
current
latest
numeric timeline ID
```
`current` 在源码中对应 `RECOVERY_TARGET_TIMELINE_CONTROLFILE`。
它沿 `pg_control` 中已有 timeline 恢复。
`latest` 对应 `RECOVERY_TARGET_TIMELINE_LATEST`。
`validateRecoveryParameters()` 会从当前 control file timeline 开始调用 `findNewestTimeLine()`。
standby streaming 中如果后来发现了更新 timeline，`rescanLatestTimeLine()` 可以更新 `recoveryTargetTLI`。
numeric timeline 对应 `RECOVERY_TARGET_TIMELINE_NUMERIC`。
它先检查 timeline history 是否存在。
除了 timeline 1，其他 numeric timeline 必须能找到 history file。
timeline 选择先于 stop target 判断。
如果 base backup checkpoint 或 `minRecoveryPoint` 不在请求 timeline 的历史中，恢复会 FATAL。
这说明：
```text
你不能先选一个时间点，再从任意 timeline 上找看起来最接近的记录。
必须先证明这份数据目录的历史包含目标 timeline。
```
## 5. 启动时如何解析目标
恢复控制从 `StartupXLOG()` 进入。
主链路是：
```text
StartupProcessMain()
  -> StartupXLOG()
     -> InitWalRecovery(ControlFile, ...)
        -> readRecoverySignalFile()
        -> validateRecoveryParameters()
     -> PerformWalRecovery()
     -> FinishWalRecovery()
     -> end-of-recovery actions in xlog.c
```
`readRecoverySignalFile()` 先看 signal 文件：
```text
standby.signal
recovery.signal
```
如果两者都存在，`standby.signal` 优先。
源码设置：
```text
standby.signal:
  StandbyModeRequested = true
  ArchiveRecoveryRequested = true
recovery.signal:
  StandbyModeRequested = false
  ArchiveRecoveryRequested = true
```
如果都不存在，则不进入 archive recovery 控制面。
这时 recovery target 参数不会参与 crash recovery 停止点判断。
`recoveryStopsBefore()` 和 `recoveryStopsAfter()` 开头都会检查 `ArchiveRecoveryRequested`。
`validateRecoveryParameters()` 接着做几件事。
第一，确认 WAL 来源配置。
standby mode 下，`primary_conninfo` 和 `restore_command` 都为空只会 WARNING。
这种 standby 会轮询 `pg_wal`，等待外部放入 WAL。
非 standby 的 archive recovery 下，如果没有 `restore_command`，会 FATAL。
PITR 不是长期等待 upstream 的模式，它需要明确知道缺失 WAL 从哪里取。
第二，处理 `recovery_target_action=pause` 与 `hot_standby` 的关系。
如果不能让用户连接查询，pause 没有可用价值。
源码把 action 改成 shutdown。
第三，最终解析 `recovery_target_time`。
`check_recovery_target_time()` 只做语法级检查。
真正依赖时区的 `timestamptz_in` 转换推迟到 `validateRecoveryParameters()`。
第四，解析目标 timeline。
`latest` 要等 archive recovery 条件已成立后才能找 history file。
因为 timeline history 可能需要从 archive 中恢复。
### 5.1 多个 recovery target 为什么直接报错
`xlogrecovery.c` 中 GUC assign hook 有一个共同约束：
```text
recoveryTarget != RECOVERY_TARGET_UNSET
并且不是当前同一种 target
  -> error_multiple_recovery_targets()
```
这禁止同时设置：
```text
recovery_target_time
recovery_target_xid
recovery_target_name
recovery_target_lsn
recovery_target = 'immediate'
```
原因不是语法洁癖。
不同 target 的 stop predicate 不在同一个语义空间。
例如一个事务可能 XID 更大但更早提交。
一个 LSN 可能落在某个 commit record 之前。
一个 restore point name 可以重复。
如果允许多个目标并存，就必须定义复杂的优先级或交叉语义。
PostgreSQL 选择把目标空间收敛成一个。
同一个参数可以多次设置。
也可以先把某个目标置空，再设置另一个目标。
这就是 assign hook 中“同类允许、空字符串 unset”的原因。
### 5.2 target 不是 timeline
`recovery_target_timeline` 不算那几个互斥 target。
它回答“从哪条历史读 WAL”。
`recovery_target_time` / `xid` / `name` / `lsn` 回答“在这条历史上停在哪个 record 边界”。
因此合法组合可以是：
```text
recovery_target_name = 'before_drop'
recovery_target_timeline = 'latest'
recovery_target_action = 'pause'
```
源码会先确定 `recoveryTargetTLI` 和 `expectedTLEs`。
随后 `ReadRecord()` / `WaitForWALToBecomeAvailable()` 只在这些 expected timeline 上找 WAL。
最后 redo loop 再检查 restore point name。
## 6. 主流程源码 walkthrough
本节主流程从 redo loop 开始。
`PerformWalRecovery()` 中的主体是：
```text
record = ReadRecord(...)
while record != NULL:
  ProcessStartupProcInterrupts()
  if recoveryPauseState != RECOVERY_NOT_PAUSED:
      recoveryPausesHere(false)
  if recoveryStopsBefore(record):
      reachedRecoveryTarget = true
      break
  if recoveryApplyDelay(record):
      if recoveryPauseState != RECOVERY_NOT_PAUSED:
          recoveryPausesHere(false)
  ApplyWalRecord(record)
  WaitLSNWakeup(...)
  if recoveryStopsAfter(record):
      reachedRecoveryTarget = true
      break
  record = ReadRecord(...)
```
这段顺序是本节最重要的源码事实。
pause 请求先于 target before 判断。
这意味着用户手动 `pg_wal_replay_pause()` 后，startup process 会在下一次检查点尽快进入 pause。
它可能晚一条 WAL record 才看到请求。
源码注释明确说这是可接受的异步控制，不值得为每条 record 增加额外 spinlock 成本。
`recoveryStopsBefore()` 在应用当前 record 之前判断。
它用于：
```text
exclusive LSN
exclusive XID
time target 的 before 边界
immediate target 在 reachedConsistency 后的一个安全点
```
`ApplyWalRecord()` 真正执行 resource manager redo。
它更新数据页，也会间接推进 Hot Standby、KnownAssignedXids、locks、replay LSN 等状态。
`WaitLSNWakeup()` 在 record replay 后唤醒等待 standby replay/write/flush LSN 的进程。
对本节来说，这解释了为什么 `pg_last_wal_replay_lsn()` 和 LSN wait 看到的是 replay 后的位置。
`recoveryStopsAfter()` 在应用当前 record 后判断。
它用于：
```text
inclusive LSN
inclusive XID
restore point name
immediate target 在 reachedConsistency 后的 after 边界
```
如果命中了目标，后面按 `recoveryTargetAction` 收尾：
```text
shutdown:
  proc_exit(3)
pause:
  SetRecoveryPause(true)
  recoveryPausesHere(true)
  fall through to promote
promote:
  break out and finish recovery
```
这个 fall through 很关键。
`recovery_target_action=pause` 并不是永远停在 recovery。
用户执行 `pg_wal_replay_resume()` 后，pause 解除，控制流继续落到 promote 分支，恢复结束并成为可写实例。
## 7. 不同 target 如何决定 before / after
### 7.1 `RECOVERY_TARGET_IMMEDIATE`
`recovery_target = 'immediate'` 的目标是最早 consistent point。
它不是“什么 WAL 都不 replay”。
源码条件是：
```text
recoveryTarget == RECOVERY_TARGET_IMMEDIATE && reachedConsistency
```
`reachedConsistency` 来自恢复一致性判断。
它通常要等 base backup 结束点或 `minRecoveryPoint` 被 replay 过。
因此 immediate target 的真实含义是：
```text
先 replay 到数据库物理一致；
一旦一致，就在最近的安全 record 边界结束恢复。
```
这也是为什么 target before / after 两个函数都检查 immediate。
一致性可能在不同位置被发现。
源码把它压成“到达一致后停止”这个语义，而不是绑定到某一种 record 类型。
如果 target 在 consistent point 之前命中，`PerformWalRecovery()` 会 FATAL：
```text
requested recovery stop point is before consistent recovery point
```
这条错误保护了一个不变量：
```text
PostgreSQL 不允许用户停在数据目录还不一致的位置。
```
### 7.2 `RECOVERY_TARGET_LSN`
LSN target 用 `record->ReadRecPtr` 比较。
这表示当前 WAL record 的起始位置。
exclusive 逻辑在 `recoveryStopsBefore()`：
```text
!recoveryTargetInclusive
record->ReadRecPtr >= recoveryTargetLSN
  -> stop before current record
```
inclusive 逻辑在 `recoveryStopsAfter()`：
```text
recoveryTargetInclusive
record->ReadRecPtr >= recoveryTargetLSN
  -> stop after current record
```
诊断时要记住：
```text
LSN target 是 WAL record 边界判断，
不是任意字节偏移处都能切开 redo。
```
如果你用 `pg_current_wal_lsn()` 得到一个业务操作后的 LSN，通常可用于“恢复到至少包含该操作之后的 WAL 边界”。
但源码上的比较点仍然是 record start。
不要把它理解成数据页上的物理 offset。
### 7.3 `RECOVERY_TARGET_XID`
XID target 只看事务结束 record。
`recoveryStopsBefore()` 和 `recoveryStopsAfter()` 都先要求：
```text
XLogRecGetRmid(record) == RM_XACT_ID
```
然后只处理：
```text
XLOG_XACT_COMMIT
XLOG_XACT_COMMIT_PREPARED
XLOG_XACT_ABORT
XLOG_XACT_ABORT_PREPARED
```
prepared transaction 的 XID 要从 parsed commit / abort record 中取 `twophase_xid`。
普通事务用 `XLogRecGetXid(record)`。
exclusive 逻辑：
```text
recordXid == recoveryTargetXid
  -> stop before commit / abort record
```
inclusive 逻辑：
```text
recordXid == recoveryTargetXid
  -> ApplyWalRecord()
  -> stop after commit / abort record
```
源码注释强调不能用 `recordXid >= targetXid`。
XID 按事务开始顺序分配。
事务结束顺序可以不同。
一个更大的 XID 可能先提交。
因此 XID target 只能是等值命中目标事务的结束 record。
这也解释了一个诊断边界：
```text
recovery_target_xid 控制的是目标事务的结束是否被包含，
不是恢复到“所有 XID 小于等于目标”的集合。
```
### 7.4 `RECOVERY_TARGET_TIME`
time target 也只看事务结束 record。
原因是只有 commit / abort record 能提供事务完成时间。
普通 heap insert、btree split、visibility map 等 WAL record 没有可用于 PITR 的业务完成时间。
源码会调用 `getRecordTimestamp(record, &recordXtime)`。
然后：
```text
inclusive:
  stop before first record whose recordXtime > recoveryTargetTime
exclusive:
  stop before first record whose recordXtime >= recoveryTargetTime
```
这听起来反直觉，但结果符合语义。
inclusive 为 true 时，要包含所有时间等于目标的事务结束记录。
所以看到第一个大于目标时间的 commit / abort，才停在它之前。
inclusive 为 false 时，目标时间相等的事务也不应包含。
所以看到第一个大于等于目标时间的 commit / abort，就停在它之前。
这里有两个边界。
第一，很多事务可能共享同一个 commit timestamp。
inclusive 的实现就是为了“包含同一时间点的所有事务”。
第二，standby 机器本地时钟不决定 target time。
比较的是 WAL record 中的事务时间。
`recovery_min_apply_delay` 才会使用 standby 当前时间去延迟 apply。
### 7.5 `RECOVERY_TARGET_NAME`
name target 对应 named restore point。
它在 `recoveryStopsAfter()` 中判断：
```text
rmid == RM_XLOG_ID
info == XLOG_RESTORE_POINT
strcmp(rp_name, recoveryTargetName) == 0
```
命中后停止在 restore point record 之后。
`recovery_target_inclusive` 不影响 name target。
源码注释说明：
```text
多个 restore point 可以使用同一个名字；
recovery 会停在第一个匹配的 restore point。
```
这意味着 restore point name 是人为标签，不是全局唯一 ID。
如果生产中重复使用同名 restore point，PITR 的目标可能比你预期更早。
## 8. timeline 如何限制停止点
timeline 是恢复 target 的坐标系。
同一个 LSN 数值在不同 timeline 上可以代表不同历史。
所以 PostgreSQL 必须先确定合法 timeline history，再寻找目标 record。
`validateRecoveryParameters()` 处理初始 timeline：
```text
numeric:
  检查 history file，设置 recoveryTargetTLI
latest:
  findNewestTimeLine(recoveryTargetTLI)
current:
  使用 control file 中已有 recoveryTargetTLI
```
随后 `InitWalRecovery()` 会检查 checkpoint 位置和 `minRecoveryPoint`。
它调用 `tliOfPointInHistory()`。
如果当前数据目录不在目标 timeline 的历史中，会 FATAL：
```text
requested timeline ... is not a child of this server's history
requested timeline ... does not contain minimum recovery point
```
`expectedTLEs` 保存目标 timeline 及其父 timeline history。
读取 WAL 时，`XLogPageRead()` / `WaitForWALToBecomeAvailable()` 会在这些 timeline 中寻找 segment。
`curFileTLI` 还限制顺序扫描时不能倒退到更旧 timeline。
这避免在同一个 recovery 过程中不受控地从一条历史跳回另一条历史。
### 8.1 `latest` 在 standby 中为什么还会变化
`recovery_target_timeline='latest'` 的特殊性在 standby 中最明显。
walreceiver 连接 primary 后会拉取缺失 timeline history files。
`walreceiver.c` 中即使当前还不关心 primary 的最新 timeline，也会尝试获取 history。
注释给出的理由是避免以后 promotion 时选中已经被 primary 用过的 timeline ID。
streaming 时，walreceiver 会按 startup process 指定的 `startpointTLI` 开始。
如果 upstream 已经切到更新 timeline，旧 timeline 流可能很快结束。
startup process 之后会重新扫描 `pg_wal` / archive 中的新 history file。
`rescanLatestTimeLine()` 做三件事：
```text
findNewestTimeLine()
readTimeLineHistory(newtarget)
确认当前 recovery timeline 是新 timeline 的父历史，
并且 fork point 不早于当前 replay LSN
```
如果合法，它更新：
```text
recoveryTargetTLI = newtarget
expectedTLEs = newExpectedTLEs
```
这解释了为什么 target timeline 是运行中的约束，而不是启动时一个永不变化的常量。
只有 `latest` 允许这种向前更新。
numeric timeline 则是固定目标。
### 8.2 end-of-recovery 为什么总要新 timeline
`StartupXLOG()` 在 archive recovery 结束后选择新 timeline。
逻辑在 `xlog.c`：
```text
if ArchiveRecoveryRequested:
  newTLI = findNewestTimeLine(recoveryTargetTLI) + 1
  XLogInitNewTimeline(...)
  writeTimeLineHistory(newTLI, recoveryTargetTLI, EndOfLog, reason)
```
即使 PITR 恢复到了 archive 的末尾，也会选择新 timeline。
这是为了避免覆盖已归档的 WAL segment。
也为了让后续历史分叉可追踪。
timeline history 中会写入 `getRecoveryStopReason()` 生成的说明。
例如：
```text
after transaction 123
before LSN 0/...
at restore point "..."
reached consistency
```
这不是日志装饰。
它会成为后续 PITR、standby 跟随和 timeline 诊断的一部分。
## 9. pause / resume 的真实语义
`pg_wal_replay_pause()` 在 `xlogfuncs.c` 中做的事很少：
```text
确认 RecoveryInProgress()
确认 PromoteIsTriggered() 为 false
SetRecoveryPause(true)
WakeupRecovery()
```
它不直接修改 replay LSN。
也不直接暂停 walreceiver。
它只是把 `recoveryPauseState` 从 `RECOVERY_NOT_PAUSED` 改成 `RECOVERY_PAUSE_REQUESTED`。
startup process 在 redo loop 中看到状态不是 `RECOVERY_NOT_PAUSED` 后，调用：
```text
recoveryPausesHere(false)
```
`recoveryPausesHere()` 先检查两个边界：
```text
!LocalHotStandbyActive:
  return
LocalPromoteIsTriggered:
  return
```
第一条说明 pause 只对可连接的 Hot Standby 状态有意义。
SQL 函数本身只能在 recovery 中由可连接 backend 执行。
但 end-of-recovery 的 `recovery_target_action=pause` 也要尊重这个边界。
第二条说明 promotion 触发后不再停在 pause。
promotion 的语义优先于继续等待用户 resume。
进入 pause loop 后，startup process 会反复：
```text
ProcessStartupProcInterrupts()
CheckForStandbyTrigger()
ConfirmRecoveryPaused()
ConditionVariableTimedSleep(..., WAIT_EVENT_RECOVERY_PAUSE)
```
`ConfirmRecoveryPaused()` 把状态从 `RECOVERY_PAUSE_REQUESTED` 改成 `RECOVERY_PAUSED`。
所以用户看到的状态可能经历：
```text
not paused
pause requested
paused
```
`pg_is_wal_replay_paused()` 返回的是：
```text
GetRecoveryPauseState() != RECOVERY_NOT_PAUSED
```
因此它在 `pause requested` 时已经返回 true。
如果需要确认 startup process 已经进入 pause loop，应看：
```text
pg_get_wal_replay_pause_state() = 'paused'
```
`pg_wal_replay_resume()` 调用 `SetRecoveryPause(false)`。
这会设置 `RECOVERY_NOT_PAUSED`，并 `ConditionVariableBroadcast()` 唤醒 startup process。
### 9.1 target action pause 与手动 pause 的差异
手动 pause 发生在恢复进行中。
resume 后继续 replay 后续 WAL。
`recovery_target_action=pause` 发生在目标已经命中后。
`PerformWalRecovery()` 的 action switch 里：
```text
SetRecoveryPause(true)
recoveryPausesHere(true)
pg_fallthrough
```
也就是说目标点 pause 的 resume 不是“继续追 WAL”。
它是“确认这个目标点就是最终点，继续完成 end-of-recovery”。
日志 hint 也不同。
普通 pause 的 hint 是继续：
```text
Execute pg_wal_replay_resume() to continue.
```
目标点 pause 的 hint 是 promote：
```text
Execute pg_wal_replay_resume() to promote.
```
这条差异在事故恢复中很重要。
如果你在目标点发现数据不对，不应该 resume。
你应该关闭实例，修改 recovery target 到更晚或更早的点，再重新启动恢复。
## 10. promote 请求如何穿过 pause
`pg_promote(wait, wait_seconds)` 的入口在 `xlogfuncs.c`。
它做几个动作：
```text
确认 RecoveryInProgress()
创建 PROMOTE_SIGNAL_FILE
kill(PostmasterPid, SIGUSR1)
如果 wait=true，循环等待 RecoveryInProgress() 变 false
```
postmaster 收到信号后会唤醒 startup process。
startup process 的 `StartupProcTriggerHandler()` 设置：
```text
promote_signaled = true
WakeupRecovery()
```
startup process 在恢复循环或等待 WAL 的路径里调用 `CheckForStandbyTrigger()`。
该函数检查：
```text
IsPromoteSignaled()
CheckPromoteSignal()
```
如果 promote signal file 存在，源码会：
```text
LOG: received promote request
RemovePromoteSignalFiles()
ResetPromoteSignaled()
SetPromoteIsTriggered()
```
`SetPromoteIsTriggered()` 很关键。
它设置 `SharedPromoteIsTriggered = true`。
然后调用：
```text
SetRecoveryPause(false)
```
所以 promotion 会解除 pause。
这避免 `pg_get_wal_replay_pause_state()` 在 promotion 已经开始后还长期显示 `paused`。
也避免 startup process 卡在 pause loop，无法完成升主。
### 10.1 promote 不是立刻丢弃本地 WAL
`WaitForWALToBecomeAvailable()` 中有一个细节。
当当前来源是 archive 或 `pg_wal`，读取失败后才检查 promotion：
```text
when currentSource is archive / pg_wal:
  after source failed:
    if StandbyMode && CheckForStandbyTrigger():
        XLogShutdownWalRcv()
        return fail
```
源码注释说明：
```text
promote 时仍会尽量 replay archive 和 pg_wal 中已经可用的 WAL，
然后才 failover。
```
这不是强制等待 upstream 所有未来 WAL。
它只是避免本地已经拿到的 WAL 被无谓跳过。
streaming 来源失败时，startup process 会退回 archive / pg_wal 重试。
如果 `recovery_target_timeline='latest'`，还会 rescan 新 timeline。
promotion 触发后，不再继续依赖 walreceiver 等待 primary。
### 10.2 target action promote 与 pg_promote 的差异
`recovery_target_action=promote` 是 target 命中后的动作。
它不需要用户再执行 SQL。
redo loop 命中目标后直接结束 recovery。
`pg_promote()` 是外部触发。
它可以在普通 standby replay 中发生。
也可以在 pause 状态中发生。
两者最终都进入 `StartupXLOG()` 的 end-of-recovery 路径。
区别在于 `PromoteIsTriggered()` 会影响 `PerformRecoveryXLogAction()`：
```text
ArchiveRecoveryRequested && IsUnderPostmaster && PromoteIsTriggered()
  -> CreateEndOfRecoveryRecord()
else
  -> RequestCheckpoint(CHECKPOINT_END_OF_RECOVERY | ...)
```
promotion 场景写轻量 end-of-recovery record。
非 promotion 的 archive recovery 收尾会做 end-of-recovery checkpoint。
## 11. end-of-recovery 的持久边界
当 `PerformWalRecovery()` 退出 redo loop 后，控制回到 `StartupXLOG()`。
`FinishWalRecovery()` 返回 `EndOfWalRecoveryInfo`。
其中包含：
```text
lastRec
lastRecTLI
endOfLog
endOfLogTLI
abortedRecPtr
missingContrecPtr
recoveryStopReason
standby_signal_file_found
recovery_signal_file_found
```
`StartupXLOG()` 随后决定新 timeline。
如果是 archive recovery：
```text
newTLI = findNewestTimeLine(recoveryTargetTLI) + 1
XLogInitNewTimeline(EndOfLogTLI, EndOfLog, newTLI)
durable_unlink(STANDBY_SIGNAL_FILE / RECOVERY_SIGNAL_FILE)
writeTimeLineHistory(newTLI, recoveryTargetTLI, EndOfLog, recoveryStopReason)
```
signal 文件移除也在这里。
`recovery_target_action=shutdown` 是例外。
它在目标命中时 `proc_exit(3)`。
这时 `recovery.signal` 不会被移除。
文档和源码设计都允许下次启动继续以 recovery 配置进入。
之后 `xlog.c` 会：
```text
LocalSetXLogInsertAllowed()
CreateOverwriteContrecordRecord() if needed
UpdateFullPageWrites()
PerformRecoveryXLogAction()
XLogReportParameters()
CleanupAfterArchiveRecovery()
UpdateLogicalDecodingStatusEndOfRecovery()
ControlFile->state = DB_IN_PRODUCTION
XLogCtl->SharedRecoveryState = RECOVERY_STATE_DONE
ShutdownRecoveryTransactionEnvironment()
WalSndWakeup(true, true)
RequestCheckpoint(CHECKPOINT_FORCE) if promoted
```
这些动作说明 recovery target 不是单独的 stop predicate。
它会改变后续实例身份。
一旦 `SharedRecoveryState` 切到 done，普通 backend 看到的 `RecoveryInProgress()` 就变 false。
此时 `pg_last_wal_replay_lsn()` 不再是继续推进的 recovery 位置。
实例开始写新 timeline 上的 WAL。
## 12. 生命周期 / ownership / cleanup
### 谁创建
recovery target 状态由 GUC 系统在 postmaster 启动阶段解析。
具体 hook 在 `xlogrecovery.c`。
表项在 `guc_parameters.dat`。
signal 文件由外部工具或 SQL 函数创建：
```text
recovery.signal
standby.signal
promote signal file
```
pause 请求由普通 backend 通过 SQL 函数设置到 shared memory。
promote 请求由 `pg_promote()` 写 signal file，再通过 postmaster 唤醒 startup process。
### 谁持有
startup process 持有恢复主状态。
例如：
```text
ArchiveRecoveryRequested
InArchiveRecovery
StandbyModeRequested
StandbyMode
recoveryTargetTLI
expectedTLEs
curFileTLI
```
普通 backend 不持有这些状态。
它们只能通过共享状态和 SQL 函数观察 replay 进度、pause 状态和 promotion 状态。
共享状态由 `XLogRecoveryCtl` 持有。
它的生命周期是 shared memory 生命周期。
字段更新用 `info_lck` 保护。
### 谁释放或失效
archive recovery 成功结束时，`StartupXLOG()` 会 durable unlink signal 文件。
新 timeline history 被写出后，旧 recovery target 不再是运行时控制目标。
pause 状态由 `SetRecoveryPause(false)` 清除。
这可以来自 `pg_wal_replay_resume()`。
也可以来自 `SetPromoteIsTriggered()`。
promote signal file 被 `CheckForStandbyTrigger()` 确认后移除。
`SharedPromoteIsTriggered` 一旦变 true，不需要再变回 false。
promotion 不可撤销。
### ERROR / abort 时怎么办
startup process 不是普通事务 backend。
严重恢复错误通常是 FATAL 或 PANIC 级别，导致 postmaster 处理启动失败或实例退出。
几类关键错误路径：
```text
多个 recovery target:
  ERROR / startup failure
非 standby archive recovery 没有 restore_command:
  FATAL
target 在 consistent point 之前:
  FATAL
archive recovery 到 WAL 末尾仍未命中 target:
  FATAL
requested timeline 不在本数据目录历史中:
  FATAL
hot_standby 关闭但 action=pause:
  自动改成 shutdown
```
这些错误看起来严格。
但它们服务同一个不变量：
```text
不能让用户以为已经精确恢复到了目标点，
实际却停在不一致、错误 timeline 或未命中目标的位置。
```
## 13. 正确性机制层次
### WAL record 边界
恢复只能在 WAL record 边界判断 stop。
`ApplyWalRecord()` 是资源管理器 redo 的完整单位。
不能在 record 中间切断。
这保证了：
```text
每个 rmgr 看到的是完整 record；
WAL-before-data 和 redo idempotence 不被 target 控制破坏；
replay LSN 代表已完整应用的 record 边界。
```
### consistent state 边界
`reachedConsistency` 是所有 target 的下限。
即使用户要求更早的位置，也不能停在 consistent point 之前。
这连接了上一节：
```text
base backup / minRecoveryPoint / backup end
  -> reachedConsistency
  -> Hot Standby 可查询
  -> recovery target pause 才能用于验证
```
### timeline history 边界
`expectedTLEs` 限定哪些 timeline 的 WAL 合法。
`tliOfPointInHistory()` 和 `tliInHistory()` 防止把不属于当前历史的 WAL 混入恢复。
这比单纯比较 LSN 更重要。
LSN 是位置。
timeline 才是历史身份。
### shared memory 并发边界
pause / promote 状态被普通 backend 和 startup process 共同访问。
`info_lck` 保证读写一致性。
condition variable 保证 resume 能唤醒 pause loop。
`recoveryWakeupLatch` 保证 pause、SIGHUP、WAL arrival、promotion 等控制事件能唤醒 startup process。
### postmaster / startup 协作边界
`pg_promote()` 不直接给 startup process 发信号。
它写 signal file 并通知 postmaster。
startup process handler 只设置轻量 flag。
真正文件检查和状态转换在安全位置完成。
这是 PostgreSQL 常见模式：
```text
signal handler 只做异步安全的标记；
主循环在可控位置执行复杂逻辑。
```
## 14. 异常路径与 fallback
### 14.1 目标未命中
如果配置了 recovery target，但 WAL 结束前没有命中，`PerformWalRecovery()` 在 redo loop 之后报：
```text
recovery ended before configured recovery target was reached
```
这通常来自：
```text
restore point name 拼错或重复理解错误；
archive 不完整；
选错 recovery_target_timeline；
目标时间晚于可用 WAL；
目标 XID 不在这条历史上提交或回滚；
restore_command 取不到后续 WAL。
```
诊断时不要只看最后一个 WAL 文件是否存在。
要同时看：
```text
target timeline history 是否包含目标；
server log 中是否有 source fallback；
pg_last_wal_replay_lsn() 最终到哪里；
restore_command 日志是否持续失败；
archive 中是否有 timeline history 文件。
```
### 14.2 pause 请求迟迟不变成 paused
`pg_is_wal_replay_paused()` 返回 true 可能只是 `pause requested`。
如果 `pg_get_wal_replay_pause_state()` 长时间停在 `pause requested`，说明 startup process 还没有走到 pause 检查点。
可能原因：
```text
startup process 正在执行一个耗时 redo record；
正在等待 recovery conflict；
正在 restore_command；
正在处理 startup interrupt；
实例尚未到 Hot Standby active；
promotion 已经触发，pause 被跳过或解除。
```
这个现象不能只用 SQL 视图解释。
需要结合 server log 和 wait event。
### 14.3 standby promotion 与 pause 交互
如果 standby 已经 paused，执行 `pg_promote()` 后：
```text
SetPromoteIsTriggered()
  -> SharedPromoteIsTriggered = true
  -> SetRecoveryPause(false)
```
因此 pause state 会回到 not paused。
startup process 继续完成 promotion。
如果 target action 是 pause，但 promotion 已经 ongoing，源码注释和文档语义都是：
```text
pause behaves like promote
```
这避免 failover 被一个旧的 recovery target pause 阻塞。
### 14.4 recovery_min_apply_delay 与 pause
`recoveryApplyDelay()` 在 target before 判断之后、`ApplyWalRecord()` 之前。
它只延迟 commit record。
并且只在：
```text
recovery_min_apply_delay > 0
reachedConsistency
ArchiveRecoveryRequested
record is COMMIT / COMMIT_PREPARED
```
delay wait 中也会再次检查 pause。
源码注释说明，用户设置 delayed apply 可能就是为了有机会 pause。
所以暂停请求不能只在 delay 前检查一次。
promotion 触发时，delay wait 也会跳出。
因此延迟 apply 不应该阻塞 failover。
### 14.5 参数不足导致 Hot Standby 不可能
`RecoveryRequiresIntParameter()` 处理一种特殊异常：如果 standby replay 发现某些参数低于 primary，Hot Standby 不可能安全继续。
Hot Standby 已经 active 时，源码会 WARNING、`SetRecoveryPause(true)`，并提示 unpause 后 server will shut down。
如果用户尝试 promote，日志会继续提示 promotion is not possible because of insufficient parameter settings，最后仍然 FATAL。
这说明 pause 不总是用户自愿调试工具，也可以是恢复错误前给 DBA 一个观察和修正窗口。
## 15. 成本、资源与跨模块传播
recovery target 本身不是高 CPU hot path；每条 WAL record 多做几个分支判断，真正的成本和风险来自周边资源。
第一，timeline history 查找依赖 archive 和文件系统；`latest` 会扫描更高 timeline，archive 不稳定、history file 缺失或 `restore_command` 慢时，恢复可能在 WAL source fallback 中反复等待。
第二，pause 会积累 replay lag；walreceiver 可能继续接收，`pg_wal` 会增长，replication slot 的 restart LSN 可能阻止 WAL 回收，上游 primary 也可能因为物理 slot 或 archive 保留产生磁盘压力。
第三，延迟 apply 和 pause 会改变 Hot Standby 冲突窗口；queries 在 standby 上看到的是 replay LSN 之前的世界，primary 上已经发生的 DDL、vacuum cleanup、lock record 可能还没 replay。
第四，promotion 会传播到多个模块：`StartupXLOG()` 切 `SharedRecoveryState`，`WalSndWakeup(true, true)` 唤醒级联 standby 的 walsender，checkpointer 会被请求做 checkpoint，logical decoding 状态也会在 end-of-recovery 更新。
因此恢复控制的成本模型不是：
```text
设置一个 GUC，停一下。
```
更准确的模型是：
```text
replay 停止或分叉会改变 WAL 保留、timeline 历史、下游复制、Hot Standby 可见性和故障切换边界。
```
## 16. 观测与诊断入口
本节锚定的 runtime truth 是：pause 控制 replay，不控制 receive；target 控制 WAL record 边界，不控制任意业务时间片；timeline 控制可读历史，不是停止动作。
SQL 侧先看 `pg_is_in_recovery()`、`pg_last_wal_receive_lsn()`、`pg_last_wal_replay_lsn()`、`pg_is_wal_replay_paused()`、`pg_get_wal_replay_pause_state()` 和 `pg_last_xact_replay_timestamp()`。
当前 master 还有 `pg_stat_recovery`，来自 `pg_stat_get_recovery()`，能看到 `promote_triggered`、`last_replayed_read_lsn`、`last_replayed_end_lsn`、`last_replayed_tli`、`replay_end_lsn`、`replay_end_tli`、`recovery_last_xact_time`、`current_chunk_start_time` 和 `pause_state`。
`last_replayed_read_lsn` 是最后成功 replay record 的起始 LSN；`last_replayed_end_lsn` 是 end+1；`replay_end_lsn` 在 redo 正在进行时指当前 record 的 end+1，否则等于 last replayed end。
`pg_last_wal_replay_lsn()` 调用 `GetXLogReplayRecPtr()`，返回 `lastReplayedEndRecPtr`，适合判断只读查询最多能看到哪条已 replay WAL 之后的状态。
日志侧看三类信息：启动目标日志，例如 `starting point-in-time recovery to XID ...`、`to WAL location (LSN) ...`、`to "name"`、`to earliest consistent point` 或 `entering standby mode`；命中目标日志，例如 `recovery stopping before/after commit of transaction ...`、`recovery stopping before/after WAL location ...`、`recovery stopping at restore point ...`、`pausing at the end of recovery`；promotion 日志，例如 `received promote request`、`selected new timeline ID: ...` 和 `archive recovery complete`。
如果看到 `recovery ended before configured recovery target was reached`，优先排查 target 是否在目标 timeline 上、archive 是否完整、restore point name 是否拼错或重复、`recovery_target_time` 时区是否正确、`recovery_target_xid` 是否确实在这条历史结束、`restore_command` 是否返回成功但没有真正放好文件。
能直接看到的是 recovery 是否仍在进行、receive / replay LSN、pause state、promotion triggered、last replayed timeline、last replayed commit / abort timestamp，以及 server log 中的 target stop reason。
只能推断的是某个业务变更是否已经包含在 replay 视图中、target time 是否命中了所有同 timestamp 事务、LSN target 是否落在期望 WAL record 边界、timeline history 是否和外部集群管理期望一致。
几乎看不到的是 startup process 当前处于 `recoveryStopsBefore()` 还是 `ApplyWalRecord()`、某条具体 WAL record 内部 redo 到哪一步、pause requested 后还会多 replay 几条 record、`restore_command` 外部系统的真实延迟和一致性。
## 17. 课堂实验
实验 1：准备 streaming standby，在 standby 上执行 `SELECT pg_wal_replay_pause();`，primary 上写入一批数据并取 `pg_current_wal_lsn()`；standby 上比较 `pg_last_wal_receive_lsn()`、`pg_last_wal_replay_lsn()` 和 `pg_get_wal_replay_pause_state()`。预期是 receive LSN 继续前进、replay LSN 停住、状态最终从 `pause requested` 变成 `paused`；源码解释是 `pg_wal_replay_pause()` 调用 `SetRecoveryPause(true)` 和 `WakeupRecovery()`，startup process 在 `PerformWalRecovery()` 中进入 `recoveryPausesHere(false)`。
实验 2：从 primary 取一个写入后的 LSN，用 base backup 创建两个 restore 节点；一个配置 `recovery_target_lsn='...'`、`recovery_target_inclusive=on`、`recovery_target_action=pause`，另一个使用同一 LSN 但 `recovery_target_inclusive=off`。观察日志中 `recovery stopping after WAL location (LSN) ...` 与 `recovery stopping before WAL location (LSN) ...` 的差异，再用 `pg_last_wal_replay_lsn()` 对齐 replay 位置；源码解释是 exclusive 路径命中 `recoveryStopsBefore()`，inclusive 路径命中 `recoveryStopsAfter()`。
实验 3：primary 上执行 `SELECT pg_create_restore_point('course_target');`，切 WAL 并确保归档完成；从 backup 恢复时配置 `recovery_target_name='course_target'`、`recovery_target_timeline='latest'`、`recovery_target_action='pause'`。观察 `starting point-in-time recovery to "course_target"`、`recovery stopping at restore point "course_target"` 和 `pausing at the end of recovery`，随后执行 `SELECT pg_wal_replay_resume();`，观察恢复结束和新 timeline 日志；源码解释是 `recoveryStopsAfter()` 识别 `XLOG_RESTORE_POINT` 并设置 `recoveryStopName`，`StartupXLOG()` 后续 `writeTimeLineHistory()`。
## 18. 常见误区
误区 1：把 pause 当成停止接收 WAL；实际 pause 停的是 replay，receive / flush 可以继续推进。
误区 2：把 `pg_is_wal_replay_paused()` 当成已完全暂停；它在 `pause requested` 时也返回 true，确认 paused 要看 `pg_get_wal_replay_pause_state()`。
误区 3：把 XID target 当成“小于等于 XID 全部恢复”；源码按目标事务结束 record 等值判断，XID 分配顺序不是提交顺序。
误区 4：把 time target 当成 standby 本地时间；time target 比较 WAL 中 commit / abort timestamp，本地当前时间只影响 `recovery_min_apply_delay`。
误区 5：忽略 timeline；同一个 LSN 在不同 timeline 上不是同一段历史，必须先验证 `recovery_target_timeline` 和 history file。
误区 6：目标点 pause 后执行 resume 继续找更晚目标；target action pause 后 resume 会完成恢复并 promote，目标不对时应 shutdown、修改 target、重新恢复。
## 19. 讨论题
1. 为什么 `recovery_target_time` 的 inclusive=true 要在第一个大于目标时间的事务前停止，而不是在等于目标时间的事务后立即停止？
2. 为什么 `recovery_target_xid` 不能用 `recordXid >= targetXid` 判断？
3. 如果 `pg_is_wal_replay_paused()` 返回 true，但 `pg_last_wal_replay_lsn()` 还前进了一点，可能有哪些源码级原因？
4. 为什么 `recovery_target_name` 不受 `recovery_target_inclusive` 影响？
5. `recovery_target_timeline='latest'` 为什么需要在 streaming 过程中重新扫描 timeline history？
6. 如果 target 命中前 WAL 用完，为什么 PostgreSQL 选择 FATAL，而不是停在最后可用 WAL？
## 20. 本节小结
本节的核心链路是：
```text
GUC / signal file
  -> InitWalRecovery()
  -> validateRecoveryParameters()
  -> timeline history
  -> PerformWalRecovery()
  -> recoveryStopsBefore() / ApplyWalRecord() / recoveryStopsAfter()
  -> recoveryTargetAction
  -> end-of-recovery timeline fork
```
核心状态分两层。
recovery target 和 timeline 主要是 startup process 的本地恢复决策。
pause、promote、replay LSN 和 replay timeline 是 `XLogRecoveryCtl` 中可被其他 backend 观察或请求改变的共享状态。
停止点不是任意字节。
它是 WAL record 边界。
time 和 XID target 绑定事务结束 record。
name target 绑定 restore point record。
LSN target 比较 record 起始位置。
immediate target 绑定 consistent state。
timeline 先限定历史。
target 再在这条历史上选择停止点。
end-of-recovery 会写新 timeline history，让未来恢复和级联 standby 能理解这次分叉。
pause 是 replay 控制，不是 receive 控制。
resume 在普通 pause 中表示继续 replay。
在 target action pause 中表示确认目标点并完成 promotion。
promote 会解除 pause，并让 startup process 不再等待 upstream 新 WAL。
诊断时要把四类证据分开：
```text
server log 的 target stop reason
pg_last_wal_receive_lsn() 与 pg_last_wal_replay_lsn()
pg_get_wal_replay_pause_state() / pg_stat_recovery.pause_state
timeline history file 和 recoveryTargetTLI
```
可迁移的系统规律是：
```text
当用户接口给出业务级停止条件时，
内核必须把它压缩到一个可证明正确的执行边界；
在 PostgreSQL recovery 中，这个边界就是 timeline history 上的完整 WAL record。
```
