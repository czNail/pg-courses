# PostgreSQL checkpoint lifecycle 与 redo pointer

## 课程定位

上一组课程已经讲过 WAL record、WAL flush、segment、full-page writes 和 rmgr redo contract。
这一节把这些线索收束到 checkpoint。
checkpoint 不是简单的“把脏页刷到磁盘”。
它是一个跨 WAL 插入、buffer pool、SLRU、fsync request、control file、recovery 和 timeline 的协议。

本节唯一主问题：
checkpoint 对外发布新的恢复起点时，PostgreSQL 怎样保证这个起点既足够靠后以缩短 recovery，又不会越过任何尚未安全落盘的数据文件状态？

本节围绕的核心矛盾：
系统希望频繁推进 redo pointer 来限制 WAL replay 距离和 WAL 保留；但一旦 `pg_control` 指向新的 checkpoint record，crash recovery 就会相信从 `CheckPoint.redo` 开始 replay 已经足够。checkpoint 必须在延迟、I/O 抖动和 recovery safety 之间建立一个原子可见边界。

读完本节，你应该能回答：
- `RequestCheckpoint()` 做了什么，没做什么。
- `CheckpointerMain()` 怎样把请求、定时、WAL 消耗合并成一次 checkpoint 或 restartpoint。
- `CreateCheckPoint()` 为什么在线 checkpoint 要写两个 WAL record。
- `XLOG_CHECKPOINT_REDO` 和 `XLOG_CHECKPOINT_ONLINE` 分别表示什么。
- shutdown checkpoint 为什么只需要一个 checkpoint record。
- `CheckPoint.redo` 为什么通常不是 checkpoint record 自己的位置。
- redo pointer 在 WAL 插入端如何影响 full-page image。
- checkpoint 的脏页刷盘顺序和 checkpoint WAL record 顺序是什么。
- 为什么数据页写出前必须先把对应 WAL flush 到页 LSN。
- `pg_control` 在 checkpoint 完成后按什么顺序更新。
- crash recovery 为什么从 checkpoint record 指向的 redo pointer 开始读。
- restartpoint 和 checkpoint 的相同点、不同点、限制条件。
- checkpoint cause 在源码中具体有哪些 bit。
- 哪些错误会导致 checkpoint 失败，哪些会导致 PANIC/FATAL。
- timeline history 为什么也属于 checkpoint/recovery 的安全边界。

源码基线：本课使用当前实际源码路径 `/home/highgo/postgres`，branch `master`，commit `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；核心源码分工见第 3 节。

## 1. 本节在总主线中的位置

前面几节分别讲了 WAL record、WAL flush、full-page writes、segment write 和 fsync queue。本节把这些机制合到 checkpoint 生命周期里：checkpoint 不是一个单纯刷脏页动作，而是发布 crash recovery 新入口的跨模块协议。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
在线 checkpoint 先写 XLOG_CHECKPOINT_REDO 固定 redo pointer，再写出并 fsync 本轮需要覆盖的数据状态，最后插入并 flush checkpoint record、更新 pg_control，让 crash recovery 能从 CheckPoint.redo 开始重放。
```

核心矛盾是：系统想推进 redo pointer 来缩短恢复和释放 WAL，但 control file 一旦指向新 checkpoint，recovery 就会相信从该 redo 起点开始足够恢复；checkpoint 必须把性能窗口和恢复安全边界同时固定住。

## 3. 核心文件分工与阅读顺序

本节把源码基线、重点入口和辅助核对路径集中放在这里，避免课程定位之后再漂一个未编号大节。

源码仓库：

```text
/home/highgo/postgres
```

基线：

```text
branch: master
commit: bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8
```

本节重点阅读：

```text
src/backend/access/transam/xlog.c
src/backend/access/transam/xlogrecovery.c
src/backend/postmaster/checkpointer.c
src/include/access/xlog.h
src/include/access/xlog_internal.h
src/backend/access/transam/timeline.c
```

为讲清 checkpoint record 和 dirty page flush，本节还辅助核对：

```text
src/include/catalog/pg_control.h
src/common/controldata_utils.c
src/backend/storage/buffer/bufmgr.c
src/backend/access/transam/xloginsert.c
src/backend/storage/smgr/md.c
```

行号来自：

```text
nl -ba <source-file>
```

## 4. 运行模型：checkpoint record 与 redo pointer

PostgreSQL 的在线 checkpoint 是一个两阶段 WAL 标记。
第一阶段是 checkpoint 开始时插入 `XLOG_CHECKPOINT_REDO`。
这条 record 的起始 LSN 成为新的 redo pointer。
第二阶段是刷完 checkpoint 需要覆盖的脏页和 fsync request 后，插入并 flush `XLOG_CHECKPOINT_ONLINE`。
这个 online checkpoint record 的 payload 是 `CheckPoint`。
其中 `CheckPoint.redo` 指回第一阶段的 redo pointer。
最后，`pg_control` 才被更新为：

```text
ControlFile->checkPoint = checkpoint record 的起始 LSN
ControlFile->checkPointCopy = CheckPoint payload 的拷贝
```

所以在线 checkpoint 有两个重要 LSN：

```text
redo pointer      = XLOG_CHECKPOINT_REDO record 的起始 LSN
checkpoint record = XLOG_CHECKPOINT_ONLINE record 的起始 LSN
```

这两个 LSN 不能混为一谈。
crash recovery 先用 `pg_control` 找到最新 checkpoint record。
读出 checkpoint record 后，再从 `CheckPoint.redo` 指向的位置开始 replay。
如果 `CheckPoint.redo` 早于 checkpoint record，那么那里的第一条 record 必须是 `XLOG_CHECKPOINT_REDO`。
`xlogrecovery.c:1662-1678` 明确校验了这一点。

shutdown checkpoint 不需要两个 record。
因为 shutdown checkpoint 时不应再有并发 WAL 插入。
`CreateCheckPoint()` 对 shutdown checkpoint 计算下一条 WAL record 的位置作为 `checkPoint.redo`。
随后插入 `XLOG_CHECKPOINT_SHUTDOWN`。
源码还要求：

```text
shutdown && checkPoint.redo == ProcLastRecPtr
```

如果不相等，说明 shutdown 期间仍有并发 WAL activity，直接 PANIC。
见 `xlog.c:7770-7776`。

redo pointer 的核心含义是：

```text
从这个 WAL 位置开始 replay，足以把数据文件恢复到 checkpoint 之后的一致状态。
```

它不是“最近一条 checkpoint record 的位置”。
`xlog.c:265-271` 的注释说得很清楚：`RedoRecPtr` 几乎但不完全等同于最近 checkpoint record 的指针。
在线 checkpoint 下，它指向 `XLOG_CHECKPOINT_REDO`。
shutdown checkpoint 下，它才等于 checkpoint record。

redo pointer 还直接影响 WAL 插入端的 full-page image 决策。
`xloginsert.c:678-695` 里，如果一个被注册进 WAL record 的 page 的 page LSN 小于等于当前 `RedoRecPtr`，并且需要 full-page writes，就要把 full-page image 带进 WAL。
原因是：checkpoint 从 redo pointer 之后开始恢复。
如果某个 page 在 redo pointer 时刻以前已经存在，之后第一次修改它时，需要 FPI 来防 torn page。
这就是 redo pointer 与 full-page writes 的交点。

checkpoint 的刷盘顺序可以概括为：
1. 在线 checkpoint 开始时写 `XLOG_CHECKPOINT_REDO`，推进 redo pointer。
2. 收集 checkpoint payload 需要的事务、OID、MultiXact、时间线等状态。
3. 等待会影响 checkpoint 边界的事务关键区。
4. `CheckPointGuts()` 写出 SLRU 和 buffer pool 的脏数据，并处理 fsync request。
5. 写并 flush `XLOG_CHECKPOINT_ONLINE` 或 `XLOG_CHECKPOINT_SHUTDOWN`。
6. 更新并 fsync `pg_control`。
7. 移除或回收不再需要的 WAL segment。

注意第 4 和第 5 的顺序。
在线 checkpoint 的 completion record 是刷完数据之后才写的。
如果崩溃发生在刷脏页期间，但 checkpoint completion record 还没有持久化，`pg_control` 不会指向这个未完成 checkpoint。
恢复会从更早的 checkpoint/restartpoint 开始。
这就是 checkpoint 的原子可见性边界。

数据页自己的 WAL-before-data 规则仍然更底层。
`bufmgr.c:4553-4571` 在写 permanent buffer 之前，先 `XLogFlush(page_lsn)`。
这保证了即使 checkpoint 正在刷数据页，也不会把一个 WAL 还没落盘的数据页变化先写出去。

restartpoint 是 recovery 期间的 checkpoint 类似物。
它不会产生新的 checkpoint WAL record。
它只能基于已经 replay 到的 checkpoint record。
startup process 在 replay 到 online/shutdown checkpoint record 时调用 `RecoveryRestartPoint()`，把该 checkpoint 存到共享内存。
checkpointer 之后调用 `CreateRestartPoint()`，如果这个 checkpoint 比 `pg_control` 里的旧 restartpoint 更新，就刷出恢复过程中产生的脏页，并让 `pg_control` 指向那条已经存在的 checkpoint record。
这能缩短下次 recovery 的 replay 起点。
但 restartpoint 不能越过还没有 replay 到的 checkpoint。
也不能在存在 unresolved invalid page references 时建立。

checkpoint cause 在本基线里只有两个“cause bit”：

```text
CHECKPOINT_CAUSE_XLOG
CHECKPOINT_CAUSE_TIME
```

它们定义在 `xlog.h:160-162`。
手动 CHECKPOINT、shutdown、end-of-recovery、force、fast、wait、flush-unlogged 是请求或行为标志，不是独立 cause bit。
`CheckpointFlagsString()` 会把这些 bit 拼到日志里，见 `xlog.c:7145-7165`。

## 5. 核心结构和名词

先读 `src/include/access/xlog.h`。
`xlog.h:144-162` 定义 checkpoint request flag。
这些 flag 可以 OR 在一起。
源码注释要求 flag 的语义必须适合 OR 合并。
原因是多个 backend 可以同时请求 checkpoint。
`RequestCheckpoint()` 不覆盖已有请求，而是把新 flag OR 到共享内存。

直接影响 `CreateCheckPoint()` 行为的 flag 有：
- `CHECKPOINT_IS_SHUTDOWN`
- `CHECKPOINT_END_OF_RECOVERY`
- `CHECKPOINT_FAST`
- `CHECKPOINT_FORCE`
- `CHECKPOINT_FLUSH_UNLOGGED`

`RequestCheckpoint()` 自己关心的 flag 有：
- `CHECKPOINT_WAIT`
- `CHECKPOINT_REQUESTED`

用于日志和统计的 cause bit 有：
- `CHECKPOINT_CAUSE_XLOG`
- `CHECKPOINT_CAUSE_TIME`

`CHECKPOINT_IS_SHUTDOWN` 表示数据库关闭时的 checkpoint。
`CHECKPOINT_END_OF_RECOVERY` 表示 recovery 结束时的 checkpoint。
源码注释说它像 shutdown checkpoint，但在 WAL recovery 结束时发出。
这很重要。
end-of-recovery checkpoint 写的是“真实 checkpoint”，不是 restartpoint。

`CHECKPOINT_FAST` 会忽略 `checkpoint_completion_target` 的平滑写入节奏。
`CheckpointWriteDelay()` 只在没有 fast request、没有 shutdown request、且进度符合计划时 sleep。
见 `checkpointer.c:787-859`。

`CHECKPOINT_FORCE` 表示即使没有新的重要 WAL activity，也要做。
没有 force 的普通 online checkpoint 在系统空闲时可能被跳过。
`CreateCheckPoint()` 在 `xlog.c:7481-7495` 用 `last_important_lsn == ControlFile->checkPoint` 判断 idle skip。

`CHECKPOINT_WAIT` 不改变 checkpoint 的物理动作。
`CreateCheckPoint()` 的注释在 `xlog.c:7374-7376` 明说，它是同步函数，不关心 `CHECKPOINT_WAIT`。
等待逻辑在 `RequestCheckpoint()`。

checkpoint record 的主体结构不在 `xlog.h`。
它定义在 `src/include/catalog/pg_control.h:31-69`。
名字是 `CheckPoint`。
它被放在 `pg_control.h`，因为 `pg_control` 中保存了最新 checkpoint record 的一份拷贝。
源码注释还说，改变这个结构需要 bump `PG_CONTROL_VERSION`。

`CheckPoint` 的字段很多。
本节最关键的是：
- `redo`
- `ThisTimeLineID`
- `PrevTimeLineID`
- `fullPageWrites`
- `wal_level`
- `logicalDecodingEnabled`
- `nextXid`
- `nextOid`
- `nextMulti`
- `oldestXid`
- `oldestMulti`
- `time`
- `dataChecksumState`

`redo` 的注释在 `pg_control.h:37-38`。
它是开始创建 checkpoint 时的下一个可用 RecPtr，也就是 REDO start point。
这句话是本节核心。
它不是 checkpoint record 的“结束位置”。
它也不一定是 checkpoint record 的“开始位置”。

`src/include/catalog/pg_control.h:71-87` 定义 XLOG rmgr 的 info 值。
与本节最相关的是：
- `XLOG_CHECKPOINT_SHUTDOWN`
- `XLOG_CHECKPOINT_ONLINE`
- `XLOG_CHECKPOINT_REDO`
- `XLOG_END_OF_RECOVERY`
- `XLOG_FPW_CHANGE`
- `XLOG_PARAMETER_CHANGE`

`src/include/access/xlog_internal.h:313-318` 定义 `xl_checkpoint_redo`。
这不是完整 checkpoint record。
它只是 `XLOG_CHECKPOINT_REDO` 的 payload。
本基线中只包含：
- `wal_level`
- `data_checksum_version`

所以本节会区分两种 record：

```text
XLOG_CHECKPOINT_REDO    payload = xl_checkpoint_redo
XLOG_CHECKPOINT_ONLINE  payload = CheckPoint
XLOG_CHECKPOINT_SHUTDOWN payload = CheckPoint
```

`pg_control` 的核心结构在 `pg_control.h:112-180`。
与 checkpoint 直接相关的字段是：
- `state`
- `time`
- `checkPoint`
- `checkPointCopy`
- `unloggedLSN`
- `minRecoveryPoint`
- `minRecoveryPointTLI`
- `backupStartPoint`
- `backupEndPoint`
- `backupEndRequired`

`ControlFile->checkPoint` 是 checkpoint record 的位置。
`ControlFile->checkPointCopy` 是 checkpoint record payload 的拷贝。
`ControlFile->checkPointCopy.redo` 才是 redo pointer。
`pg_controldata` 输出里常见的 checkpoint location 和 REDO location，就是这两类信息。

## 6. `RequestCheckpoint()`：请求不是执行

入口在 `src/backend/postmaster/checkpointer.c:1051-1196`。
函数注释说它由 backend process 调用，用于请求 checkpoint。
它不是 checkpoint 执行器。
在 postmaster 环境下，它只做四件事。

第一，如果不是 postmaster 环境，standalone backend 直接自己做。
`checkpointer.c:1075-1089` 调用 `CreateCheckPoint(flags | CHECKPOINT_FAST)`，然后 `smgrdestroyall()`。
standalone 没有其它 backend，所以没有必要慢慢 checkpoint。

第二，在共享内存中 OR 上请求 flag。
`checkpointer.c:1092-1107` 先记录旧的 `ckpt_failed` 和 `ckpt_started`。
然后：

```text
ckpt_flags |= flags | CHECKPOINT_REQUESTED
```

这里不能赋值覆盖。
例如一个 backend 请求 fast，另一个 backend 请求 wait。
OR 合并后，checkpointer 看到的是更强的合并请求。
这就是 `xlog.h` 注释强调 flag 必须适合 OR 的原因。

第三，唤醒 checkpointer。
`checkpointer.c:1109-1143` 查 `ProcGlobal->checkpointerProc`。
如果可用，就 set checkpointer 的 latch。
如果 checkpointer 还没启动，最多重试 600 次，每次 0.1 秒。
如果调用者没有 `CHECKPOINT_WAIT`，通知失败只 LOG。
如果有 wait，通知失败可以 ERROR。

第四，如果请求带 `CHECKPOINT_WAIT`，等待 checkpoint start 和 done。
等待 start 使用 `start_cv`。
等待 done 使用 `done_cv`。
`ckpt_started` 变化表示请求 flag 已经被 checkpointer 看见。
`ckpt_done` 到达对应值表示这次 checkpoint 完成。
如果 `ckpt_failed` 变化，等待者报 ERROR。
见 `checkpointer.c:1145-1195`。

这说明 `CHECKPOINT_WAIT` 的边界是“等待 checkpointer 完成请求”。
它不是“让 `CreateCheckPoint()` 同步执行”的开关。
`CreateCheckPoint()` 本来就是同步函数。
差别在于请求者是否等 checkpointer 的执行结果。

手动 SQL `CHECKPOINT` 的入口是 `ExecCheckpoint()`。
它在 `checkpointer.c:1000-1049`。
它解析 `mode=spread|fast` 和 `flush_unlogged`。
权限要求是 `pg_checkpoint`。
最后调用：

```text
RequestCheckpoint(CHECKPOINT_WAIT | ... | CHECKPOINT_FORCE)
```

如果正在 recovery，手动 CHECKPOINT 不加 force。
因为 recovery 中请求的是 restartpoint。
restartpoint 是否能建立取决于是否已经 replay 到新的安全 checkpoint record。

## 7. `CheckpointerMain()`：合并请求、时间和 recovery 状态

`CheckpointerMain()` 在 `checkpointer.c:205-664`。
它是 checkpointer 进程主循环。
本节只关心 checkpoint 相关路径。

启动时，checkpointer 把自己的 pid 写入共享内存：

```text
CheckpointerShmem->checkpointer_pid = MyProcPid
```

见 `checkpointer.c:214`。

共享内存结构在 `checkpointer.c:118-143`。
关键字段有：
- `ckpt_started`
- `ckpt_done`
- `ckpt_failed`
- `ckpt_flags`
- `start_cv`
- `done_cv`
- fsync request ring buffer

`ckpt_flags` 受 `ckpt_lck` 保护。
fsync request ring buffer 受 `CheckpointerCommLock` 保护。
这两个锁保护的对象不同。

主循环每轮先 reset latch。
然后调用 `AbsorbSyncRequests()`。
再处理信号。
见 `checkpointer.c:371-390`。

`AbsorbSyncRequests()` 不是 checkpoint 自身的全部 sync。
它把 backend 转发来的 fsync request 从共享队列搬进本地 sync 机制。
它会在主循环、checkpoint write delay、以及 `CreateCheckPoint()` 等待事务关键区时多次调用。

主循环判断是否需要 checkpoint 有两个来源。
第一个来源是共享内存中 `ckpt_flags` 非零。
`checkpointer.c:393-402` 不加锁做快速检查。
第二个来源是时间。
`checkpointer.c:404-418` 如果距离 `last_checkpoint_time` 超过 `CheckPointTimeout`，设置 `CHECKPOINT_CAUSE_TIME`。

如果要做 checkpoint，checkpointer 先判断当前是否在 recovery。
`checkpointer.c:428-430`：

```text
do_restartpoint = RecoveryInProgress()
```

然后在 `ckpt_lck` 下原子取出 flags。
它把共享内存里的 `ckpt_flags` 合并到本地 `flags`。
把共享内存 `ckpt_flags` 清零。
递增 `ckpt_started`。
再广播 `start_cv`。
见 `checkpointer.c:431-442`。

如果 flags 包含 `CHECKPOINT_END_OF_RECOVERY`，即使 `RecoveryInProgress()` 为 true，也必须做真实 checkpoint。
`checkpointer.c:444-449` 把 `do_restartpoint` 改成 false。
这是 end-of-recovery checkpoint 与普通 recovery restartpoint 的分界。

统计上，time-driven checkpoint 和 requested checkpoint 分开计数。
如果是 recovery，则计入 restartpoint 对应统计。
见 `checkpointer.c:451-467`。

如果不是 restartpoint，且 flags 包含 `CHECKPOINT_CAUSE_XLOG`，并且离上次 checkpoint 太近，checkpointer 会打“checkpoints are occurring too frequently”日志。
见 `checkpointer.c:469-484`。
注意它只对 WAL 消耗触发的普通 checkpoint 做这个 warning。

真正执行在 `checkpointer.c:498-504`：

```text
if (!do_restartpoint)
    CreateCheckPoint(flags)
else
    CreateRestartPoint(flags)
```

执行完后，checkpointer 调 `smgrdestroyall()`。
然后更新 `ckpt_done`，广播 `done_cv`。
等待 `CHECKPOINT_WAIT` 的 backend 就在这里被释放。

如果普通 checkpoint 执行了，`last_checkpoint_time` 记录 checkpoint 开始时间，不是结束时间。
这让 time-driven checkpoint 的间隔更可预测。
见 `checkpointer.c:523-533`。

如果 restartpoint 没做成，通常是没有 replay 到新的 checkpoint record。
checkpointer 把下一次尝试推到约 15 秒后。
见 `checkpointer.c:547-557`。

主循环外还有 shutdown 路径。
收到 shutdown XLOG 请求后，`CheckpointerMain()` 调 `ShutdownXLOG()`。
`ShutdownXLOG()` 会创建 shutdown checkpoint 或 shutdown restartpoint。
见 `checkpointer.c:620-639` 和 `xlog.c:7101-7142`。

## 8. `CreateCheckPoint()`：在线 checkpoint 的主流程

`CreateCheckPoint()` 在 `xlog.c:7398-7896`。
函数注释在 `xlog.c:7361-7396`，值得完整读。
它明确说：
在线 checkpoint 先插入 `XLOG_CHECKPOINT_REDO`。
刷完所有东西后，再插入 `XLOG_CHECKPOINT_ONLINE`。
后者指回前者。

函数一开始判断 shutdown。
如果 flags 包含 `CHECKPOINT_IS_SHUTDOWN` 或 `CHECKPOINT_END_OF_RECOVERY`，就按 shutdown checkpoint 处理。
见 `xlog.c:7413-7420`。

如果当前还在 recovery，却没有 `CHECKPOINT_END_OF_RECOVERY`，不能创建普通 checkpoint。
`xlog.c:7422-7424` 报 ERROR。
recovery 中普通周期性动作应该是 restartpoint。

checkpoint 统计初始化在 `xlog.c:7426-7435`。
`SyncPreCheckpoint()` 在 `xlog.c:7436-7442`。
源码要求它在 critical section 外，并且在确定 REDO pointer 之前执行。
它不能做需要 checkpoint 回滚的事情。

`START_CRIT_SECTION()` 在 `xlog.c:7448-7451`。
这里之后的关键状态更新失败会导致系统 panic。
但后面真正刷大量脏页前会退出 critical section。
这个边界很重要。

如果是 shutdown checkpoint，先把 control file 状态写成 `DB_SHUTDOWNING`。
见 `xlog.c:7453-7459`。
如果崩溃发生在 shutdown checkpoint 中途，下次启动可以看到 shutdown interrupted。

然后初始化 `CheckPoint` payload。
`checkPoint.time` 用当前时间。
Hot Standby 需要的 `oldestActiveXid` 在固定 redo pointer 之前计算。
见 `xlog.c:7461-7473`。

接着读取 `last_important_lsn`。
这是为了判断非 force、非 shutdown、非 end-of-recovery 的 checkpoint 是否可以跳过。
如果从上次 checkpoint 以来没有重要 WAL activity，就不插入重复 checkpoint。
见 `xlog.c:7475-7495`。

end-of-recovery checkpoint 发生在系统还不允许普通 backend 写 WAL 的阶段。
为了写 checkpoint record，`xlog.c:7498-7504` 临时允许本地 WAL insert。

时间线字段接着写入 checkpoint payload。
`ThisTimeLineID` 来自 `XLogCtl->InsertTimeLineID`。
如果是 end-of-recovery checkpoint，`PrevTimeLineID` 是旧 timeline。
否则 `PrevTimeLineID == ThisTimeLineID`。
见 `xlog.c:7506-7510`。

随后持有所有 WAL insertion locks。
`xlog.c:7512-7516` 的注释说：检查 insert state 时必须阻塞并发插入。
此时读取 authoritative 的 `Insert->fullPageWrites`。
还读取 checkpoint 时刻的数据 checksum state。
见 `xlog.c:7517-7526`。

shutdown checkpoint 的 redo pointer 在这里计算。
`xlog.c:7528-7547` 用当前 WAL insert position，必要时跳过 page header。
得到的位置是下一条 WAL record 的位置。
因为 shutdown checkpoint 期间不应再有并发 WAL 插入，所以这个位置也会成为 checkpoint record 自己的开始位置。

然后更新 shared `RedoRecPtr`。
`xlog.c:7548-7560` 的注释解释了为什么即使 checkpoint 之后失败，也可以先推进。
如果 checkpoint 失败，`RedoRecPtr` 可能比必要位置更靠后。
后果只是后续 WAL insert 可能多写一些 full-page image。
但不能等刷完脏页再推进。
因为 checkpoint 刷页期间发生的新 WAL insert 必须假设自己的页面修改不属于当前 checkpoint。

释放 WAL insertion locks 后，普通事务可以继续插 WAL。
见 `xlog.c:7563-7567`。

在线 checkpoint 的 redo pointer 在 `xlog.c:7569-7601` 确定。
流程是：
1. 读取当前 `wal_level` 和 checksum state。
2. `XLogBeginInsert()`。
3. 注册 `xl_checkpoint_redo`。
4. `XLogInsert(RM_XLOG_ID, XLOG_CHECKPOINT_REDO)`。
5. 从 backend-local `RedoRecPtr` 拷贝到 `checkPoint.redo`。

为什么 `XLogInsert()` 会更新 `RedoRecPtr`？
因为 `XLogInsertRecord()` 对 `XLOG_CHECKPOINT_REDO` 有特殊处理。
`xlog.c:802-809` 把它分类成 `WALINSERT_SPECIAL_CHECKPOINT`。
`xlog.c:927-941` 持有所有 WAL insertion locks，reserve record 空间后，把 `RedoRecPtr = Insert->RedoRecPtr = StartPos`。
也就是说，`XLOG_CHECKPOINT_REDO` record 的起始位置就是新的 redo pointer。

随后 `xlog.c:7603-7606` 更新 `XLogCtl->RedoRecPtr` 这份受 `info_lck` 保护的副本。
这份副本可以被不持 WAL insertion lock 的代码读到，例如 `GetRedoRecPtr()`。

到这里，在线 checkpoint 的“开始边界”已经写入 WAL buffer。
但 `pg_control` 还没有更新。
数据页也还没有全刷完。
所以这个 checkpoint 还不可作为启动 recovery 的最新 checkpoint。

## 9. checkpoint record：不是 redo record

`XLOG_CHECKPOINT_REDO` 是 checkpoint lifecycle 的开始标记。
它的 payload 很小。
本基线只有 wal level 和 checksum state。
redo 时，`xlog_redo()` 在 `xlog.c:9168-9184` 处理它。
它更新 `XLogCtl->data_checksum_version`，必要时触发 checksum barrier。

完整 checkpoint record 是 `XLOG_CHECKPOINT_ONLINE` 或 `XLOG_CHECKPOINT_SHUTDOWN`。
它的 payload 是 `CheckPoint`。
`xlog.c:7622-7653` 收集 `nextXid`、`oldestXid`、commit timestamp、OID、MultiXact、logical decoding 状态等。

在线 checkpoint 的 payload 中，`checkPoint.redo` 已经指向 `XLOG_CHECKPOINT_REDO`。
shutdown checkpoint 的 payload 中，`checkPoint.redo` 指向 shutdown checkpoint record 自己。

`CheckPoint` 里的时间线字段有两个。
`ThisTimeLineID` 表示当前 timeline。
`PrevTimeLineID` 只有在这条 checkpoint record 开启新 timeline 时不同。
end-of-recovery checkpoint 会使用这个差异。

checkpoint record 的写入发生在 `CheckPointGuts()` 之后。
`xlog.c:7742-7753`：
1. 进入 critical section。
2. `XLogBeginInsert()`。
3. 注册 `CheckPoint` payload。
4. `XLogInsert(RM_XLOG_ID, XLOG_CHECKPOINT_SHUTDOWN 或 XLOG_CHECKPOINT_ONLINE)`。
5. `XLogFlush(recptr)`。

`recptr` 是 checkpoint record 的结束位置。
`ProcLastRecPtr` 是 checkpoint record 的开始位置。
`xlog.c:7770-7773` 的注释明确了这两个值。

checkpoint record flush 是非常关键的边界。
只有 checkpoint record 被 flush 后，`pg_control` 才能安全指向它。
否则 `pg_control` 指向一个可能不存在或不完整的 WAL record，启动恢复会失败。

shutdown checkpoint 还有额外保护。
`xlog.c:7774-7776` 要求 `checkPoint.redo == ProcLastRecPtr`。
如果 shutdown 时还有并发 WAL activity，PANIC。
这保护了“shutdown checkpoint 自己就是 redo start”这个假设。

redo 侧，`xlog_redo()` 对 shutdown checkpoint 和 online checkpoint 的处理不同。
`xlog.c:8852-8955` 是 shutdown checkpoint。
它相信 checkpoint 中的计数器精确值。
例如 nextXid、nextOid、MultiXact。
因为 shutdown checkpoint 表示主库此刻没有普通事务在跑。

`xlog.c:8956-9016` 是 online checkpoint。
它把 `nextXid` 当成下界。
OID 计数器则更信任后续 `XLOG_NEXTOID` record。
原因是 online checkpoint 的这些值来自 checkpoint 早期，后续 WAL 可能已经推进。

两种 checkpoint redo 结束时都会调用 `RecoveryRestartPoint()`。
也就是说，recovery replay 到 checkpoint record 后，才可能把它作为 restartpoint 的候选。

## 10. redo pointer：三层含义

第一层含义是 recovery start。
`pg_control` 记录的 checkpoint record 只是入口。
真正 replay 从 `ControlFile->checkPointCopy.redo` 开始。
`InitWalRecovery()` 在 `xlogrecovery.c:718-724` 从 control file 取：

```text
CheckPointLoc = ControlFile->checkPoint
RedoStartLSN = ControlFile->checkPointCopy.redo
```

如果从 backup label 启动，入口可以来自 `backup_label`。
`xlogrecovery.c:537-599` 读 backup label 指定的 checkpoint，再校验 redo location 是否存在。
这解释了为什么在线备份会把 checkpoint location 和 start WAL location 都写进 label。

第二层含义是 WAL insertion 的 FPI 边界。
`xloginsert.c:678-695` 根据 page LSN 和 `RedoRecPtr` 判断是否需要 full-page image。
如果 page LSN `<= RedoRecPtr`，说明这可能是 checkpoint 后第一次修改这个 page。
为了防止 torn page，WAL record 要带 FPI。

这个判断不是只做一次。
`xloginsert.c:522-533` 组装 record 前先用本地缓存读 `RedoRecPtr` 和 `doPageWrites`。
但这些缓存可能过期。
`XLogInsertRecord()` 在持有 WAL insertion lock 后再次检查。
`xlog.c:878-895` 如果发现 redo pointer 或 full-page write 状态变化导致需要 FPI，就返回 `InvalidXLogRecPtr`。
调用方重新组装 WAL record。

第三层含义是 WAL segment 保留边界。
checkpoint 完成后，旧 WAL 的保留计算从 redo pointer 出发。
`xlog.c:7845-7864` 在 checkpoint 后从 `RedoRecPtr` 算 segment，经过 `KeepLogSeg()` 调整，再 `RemoveOldXlogFiles()`。
restartpoint 也类似，见 `xlog.c:8301-8346`。

所以 redo pointer 同时影响：
- recovery 从哪里开始。
- full-page image 从哪里开始重新需要。
- 旧 WAL segment 最早能删到哪里。

不要把它理解成单纯的“checkpoint record LSN”。

## 11. dirty page flush 与 WAL record 顺序

`CreateCheckPoint()` 在构造 checkpoint payload 后退出 critical section。
见 `xlog.c:7655-7663`。
原因是接下来要做大量 I/O。
这些 I/O 可能失败。
源码注释说，如果失败，checkpoint 可以不完成，没有必要强制系统 panic。

退出 critical section 后，先等待 `DELAY_CHKPT_START`。
`xlog.c:7665-7712` 的注释用事务提交举例。
如果一个事务在 redo point 之前插入了 commit record，但还没完成 pg_xact 更新，那么从 redo point 开始 recovery 不会 replay 那条 commit record。
checkpoint 必须保证对应的 pg_xact 更新已经被刷下去。
所以 checkpoint 要等待处于相关 critical section 的事务离开。

等待过程中还会调用 `AbsorbSyncRequests()`。
这是为了避免等待对象正好要往 checkpointer fsync 队列里放请求而卡住。

随后调用 `CheckPointGuts(checkPoint.redo, flags)`。
见 `xlog.c:7714`。
这是普通 checkpoint 和 recovery restartpoint 共享的刷盘逻辑。

`CheckPointGuts()` 在 `xlog.c:8048-8075`。
顺序是：
1. `CheckPointRelationMap()`。
2. `CheckPointReplicationSlots()`。
3. `CheckPointSnapBuild()`。
4. `CheckPointLogicalRewriteHeap()`。
5. `CheckPointReplicationOrigin()`。
6. 写出 SLRU 和 buffer pool。
7. `ProcessSyncRequests()` 执行 queued fsync。
8. `CheckPointTwoPhase(checkPointRedo)`。

SLRU 部分包括：
- `CheckPointCLOG()`
- `CheckPointCommitTs()`
- `CheckPointSUBTRANS()`
- `CheckPointMultiXact()`
- `CheckPointPredicate()`

buffer pool 部分是 `CheckPointBuffers(flags)`。
它在 `bufmgr.c:4433-4444` 只是调用 `BufferSync(flags)`。

`BufferSync()` 在 `bufmgr.c:3551-3826`。
它先扫描所有 shared buffer。
把 checkpoint 开始时已经 dirty 且需要 checkpoint 的 buffer 标记为 `BM_CHECKPOINT_NEEDED`。
见 `bufmgr.c:3585-3630`。
这点很关键。
checkpoint 要写的是 checkpoint 开始时的脏页集合。
checkpoint 进行期间新变脏的页不会自动纳入本次 checkpoint。
这些新修改会被 redo pointer 之后的 WAL 覆盖。

`BufferSync()` 之后排序这些 buffer。
排序是为了减少随机 I/O，并在 tablespace 之间平衡写入。
见 `bufmgr.c:3643-3650` 和 `bufmgr.c:3719-3737`。

写出时，`BufferSync()` 调 `SyncOneBuffer()`。
每处理一批进度，会调用 `CheckpointWriteDelay(flags, progress)`。
见 `bufmgr.c:3740-3807`。
这就是普通 checkpoint 能按 `checkpoint_completion_target` 平滑写的原因。

`SyncOneBuffer()` 在 `bufmgr.c:4124-4198`。
最重要的注释在 `bufmgr.c:4149-4157`。
它说：只要访问方法在 `XLogInsert()` 之前先 mark page dirty，那么 checkpoint 在检查 dirty bit 时不拿 content lock 也是安全的。
如果有人在 checkpoint 检查之后才把 buffer mark dirty，不用担心。
因为 checkpoint 的 redo pointer 在那条即将产生的 WAL record 之前。
那次修改会通过 WAL replay 覆盖。

真正写 shared buffer 的是 `FlushBuffer()`。
它在 `bufmgr.c:4496-4628`。
注释在 `bufmgr.c:4499-4503` 说，写 buffer 只是把内容交给 kernel。
真正持久化要等 fsync。
checkpoint WAL 之前必须 force 数据文件 changes 到磁盘。

`FlushBuffer()` 最核心的 WAL-before-data 规则在 `bufmgr.c:4553-4571`。
它取 page LSN。
如果 buffer 是 permanent relation，就先 `XLogFlush(page_lsn)`。
然后才 `smgrwrite()` 写数据页。

所以顺序是：

```text
page modification -> WAL record insert -> page LSN set
data page flush   -> XLogFlush(page LSN) -> write page
checkpoint finish -> ProcessSyncRequests -> checkpoint record insert+flush -> pg_control update
```

注意这里有两个 WAL flush。
第一个是每个数据页写出前按 page LSN flush。
第二个是 checkpoint record 自己写出后按 checkpoint record end LSN flush。
前者保护数据页。
后者保护 `pg_control` 可以指向 checkpoint record。

`ProcessSyncRequests()` 在 `CheckPointGuts()` 中发生在 buffer write 之后。
它处理 `FlushBuffer()` 和 smgr 路径登记的 fsync request。
`md.c:1510-1528` 显示 relation segment 被标记需要 fsync 时，会通过 `RegisterSyncRequest()` 尽量交给 checkpointer。
如果队列交不出去，backend 本地 fsync。

`AbsorbSyncRequests()` 的注释在 `checkpointer.c:1421-1428`。
它必须在 `CreateCheckPoint()` 开始 fsync 前吸收所有 pending request。
否则 checkpoint 可能漏掉某个 backend 已经写过、但尚未登记到本地 sync 机制的文件。

等 `CheckPointGuts()` 返回，`CreateCheckPoint()` 再等待 `DELAY_CHKPT_COMPLETE`。
见 `xlog.c:7716-7729`。
这处理另一类必须落在 checkpoint record 同侧的动作。

如果 Hot Standby 需要，在线 checkpoint 还会写 running xacts snapshot。
见 `xlog.c:7731-7740`。
shutdown checkpoint 或 crash recovery 结束时不需要。

最后才写 checkpoint record 并 flush。
再最后才更新 control file。

## 12. control file 更新顺序

`pg_control` 更新有三类场景。
本节重点是 checkpoint 完成时。

先看工具函数。
`xlog.c:4630-4637` 的 `UpdateControlFile()` 只是包装：

```text
update_controlfile(DataDir, ControlFile, true)
```

真正写文件在 `src/common/controldata_utils.c:181-284`。
它会：
1. 更新 `ControlFile->time`。
2. 重算 CRC。
3. 把 `ControlFileData` 拷贝到固定大小 buffer。
4. 写 `global/pg_control`。
5. 如果 `do_sync` 为 true，fsync。
6. close。

backend 内所有写错、fsync 错、close 错都是 PANIC 级别。
这很合理。
如果 control file 状态不可信，启动恢复的入口就不可信。

在线 checkpoint 完成时，更新顺序在 `xlog.c:7778-7804`。
先记住旧 checkpoint 的 redo pointer，用于距离估算。
然后持有 `ControlFileLock`。
如果是 shutdown，设置 `ControlFile->state = DB_SHUTDOWNED`。
然后写：

```text
ControlFile->checkPoint = ProcLastRecPtr
ControlFile->checkPointCopy = checkPoint
ControlFile->minRecoveryPoint = InvalidXLogRecPtr
ControlFile->minRecoveryPointTLI = 0
ControlFile->unloggedLSN = current unlogged LSN
UpdateControlFile()
```

这里的 `ProcLastRecPtr` 是 checkpoint record 的开始位置。
不是 checkpoint record 的结束位置。
`checkPoint.redo` 是 replay 起点。

为什么要在 checkpoint record flush 后才更新 `pg_control`？
因为启动时 `ReadCheckpointRecord()` 会从 `ControlFile->checkPoint` 读取 WAL record。
如果 `pg_control` 先更新，崩溃后可能指向一条没 flush 完的 checkpoint record。
这会导致 recovery 找不到有效 checkpoint。

为什么 `minRecoveryPoint` 在普通 checkpoint 后清零？
`xlog.c:7792-7794` 注释说 crash recovery 应该总是恢复到 WAL 末尾。
`minRecoveryPoint` 是 archive recovery/base backup 语义。
普通生产状态下不应保留旧 archive recovery 的最小恢复点。

启动时也会更新 control file。
`StartupXLOG()` 在 `xlog.c:5983-5993` 调 `InitWalRecovery()`。
`InitWalRecovery()` 只更新内存中的 `ControlFile`。
随后 `StartupXLOG()` 初始化各子系统，并在 `xlog.c:6132-6140` 调 `UpdateControlFile()`。
这次更新把 `pg_control` 标记为正在 recovery，并记录选择的 checkpoint。

recovery 完成进入生产状态时也更新。
`xlog.c:6638-6660` 持有 `ControlFileLock`，把 `state` 改为 `DB_IN_PRODUCTION`，更新 checksum state，把 shared recovery state 改成 done，然后 `UpdateControlFile()`。
源码注释承认有一个很小窗口：backend 可以写 WAL，而 on-disk control file 还没显示 production。
但 shared state 和 control file 内存更新在 lock 下协调，避免 backend 看到不一致的内存状态。

shutdown checkpoint 还有开始状态更新。
`CreateCheckPoint()` 在 `xlog.c:7453-7459` 先把 state 写成 `DB_SHUTDOWNING`。
成功完成后再写成 `DB_SHUTDOWNED`。
这让下次启动能区分 clean shutdown 和 shutdown interrupted。

## 13. checkpoint cause 与日志

checkpoint request flag 的定义在 `xlog.h:144-162`。
源码把 cause bit 单独列在 `xlog.h:160-162`：

```text
CHECKPOINT_CAUSE_XLOG
CHECKPOINT_CAUSE_TIME
```

`CHECKPOINT_CAUSE_TIME` 由 `CheckpointerMain()` 设置。
`checkpointer.c:404-418` 判断当前时间距离 `last_checkpoint_time` 是否超过 `CheckPointTimeout`。
如果超过，设置 cause time。

`CHECKPOINT_CAUSE_XLOG` 来自 WAL 消耗。
普通运行中，`xlog.c:2513-2525` 在完成 WAL segment 相关写入后检查 `XLogCheckpointNeeded(openLogSegNo)`。
如果本地 redo pointer 副本显示可能需要 checkpoint，会先 `GetRedoRecPtr()` 更新一次，再检查。
如果仍需要，就 `RequestCheckpoint(CHECKPOINT_CAUSE_XLOG)`。

recovery 中也有类似请求。
`xlogrecovery.c:3293-3311` 在读取 WAL segment 切换时，如果 archive recovery 已 replay 太多 WAL，也会 `RequestCheckpoint(CHECKPOINT_CAUSE_XLOG)`。
在 recovery 中，这个请求会由 checkpointer 变成 restartpoint。

手动 SQL `CHECKPOINT` 不是 cause bit。
它通过 `ExecCheckpoint()` 传 `CHECKPOINT_WAIT`，通常还传 `CHECKPOINT_FORCE`，以及 fast/spread 和 flush_unlogged 行为。
见 `checkpointer.c:1000-1049`。

shutdown 也不是 cause bit。
`ShutdownXLOG()` 直接调用 `CreateCheckPoint(CHECKPOINT_IS_SHUTDOWN | CHECKPOINT_FAST)` 或 `CreateRestartPoint(CHECKPOINT_IS_SHUTDOWN | CHECKPOINT_FAST)`。
见 `xlog.c:7128-7142`。

promotion 后会请求一次 checkpoint。
`xlog.c:6694-6701` 说这不是一致性必需，但如果不做，最后 restartpoint 可能很早，之后崩溃恢复时间太长。
它调用 `RequestCheckpoint(CHECKPOINT_FORCE)`。

end-of-recovery checkpoint 由 recovery 结束路径请求。
`xlog.c:6818-6822` 调：

```text
RequestCheckpoint(CHECKPOINT_END_OF_RECOVERY | CHECKPOINT_FAST | CHECKPOINT_WAIT)
```

日志字符串由 `CheckpointFlagsString()` 拼接。
它在 `xlog.c:7145-7165`。
输出中可能出现：
- `shutdown`
- `end-of-recovery`
- `fast`
- `force`
- `wait`
- `wal`
- `time`
- `flush-unlogged`

`LogCheckpointStart()` 在 `xlog.c:7167-7183`。
普通 checkpoint 日志是 `checkpoint starting:...`。
restartpoint 日志是 `restartpoint starting:...`。

`LogCheckpointEnd()` 在 `xlog.c:7185-7287`。
它记录 wrote buffers、SLRU buffers、WAL files added/removed/recycled、write/sync/total time、sync files、distance、estimate、checkpoint lsn 和 redo lsn。
对理解 checkpoint 很有帮助。

## 14. crash recovery 从 checkpoint 开始

启动入口在 `StartupXLOG()`。
本节关注 `xlog.c:5868-6199` 和 `xlogrecovery.c:456-970`。

`StartupXLOG()` 先验证 control file 中 checkpoint location 合法。
`xlog.c:5874-5880` 如果 `ControlFile->checkPoint` 不是合法 record offset，FATAL。

然后根据 `ControlFile->state` 打日志。
如果是 `DB_SHUTDOWNED`，说明 clean shutdown。
如果是 `DB_IN_PRODUCTION`、`DB_SHUTDOWNING`、`DB_IN_CRASH_RECOVERY` 等状态，就需要考虑 recovery。
见 `xlog.c:5882-5939`。

如果上次不是 clean shutdown，`StartupXLOG()` 会删除临时 WAL 文件并 `SyncDataDirectory()`。
见 `xlog.c:5959-5979`。
注释解释了原因：之前可能有写入准备 fsync 但没 fsync，系统要避免之后的持久化顺序让旧未刷写入丢失。

然后调用 `InitWalRecovery()`。
它分析 control file、backup label、recovery signal，决定是否需要 crash recovery 或 archive recovery。
见 `xlog.c:5983-5993` 和 `xlogrecovery.c:436-456`。

没有 backup label 时，`InitWalRecovery()` 从 control file 取最新 checkpoint：

```text
CheckPointLoc = ControlFile->checkPoint
CheckPointTLI = ControlFile->checkPointCopy.ThisTimeLineID
RedoStartLSN = ControlFile->checkPointCopy.redo
RedoStartTLI = ControlFile->checkPointCopy.ThisTimeLineID
```

见 `xlogrecovery.c:718-724`。

随后调用 `ReadCheckpointRecord()`。
`ReadCheckpointRecord()` 在 `xlogrecovery.c:4056-4105`。
它校验：
- checkpoint location offset 合法。
- 能读到 record。
- `xl_rmid == RM_XLOG_ID`。
- `xl_info` 是 shutdown 或 online checkpoint。
- record length 正好等于固定 header 加 `CheckPoint` payload。

如果 checkpoint record 读不到，普通 control file 路径直接 FATAL。
见 `xlogrecovery.c:731-741`。

读出 `CheckPoint` 后，`InitWalRecovery()` 校验 redo location 是否存在。
如果 `checkPoint.redo < CheckPointLoc`，说明是在线 checkpoint。
它会从 `checkPoint.redo` 开始读一条 record，确认可读。
见 `xlogrecovery.c:746-754`。

再之后是 timeline 校验。
`xlogrecovery.c:786-812` 确认 checkpoint record 所在 timeline 是目标 timeline 的历史。
`xlogrecovery.c:814-825` 确认 `minRecoveryPoint` 也在目标 timeline 历史中。

然后校验 redo pointer。
`xlogrecovery.c:852-855` 不允许 `checkPoint.redo > CheckPointLoc`。
因为 redo pointer 不能在 checkpoint record 之后。
`xlogrecovery.c:862-867` 如果 shutdown checkpoint 的 redo pointer 早于 checkpoint record，也 PANIC。
shutdown checkpoint 必须自洽。

如果 `checkPoint.redo < CheckPointLoc`，必然需要 recovery。
如果 control file state 不是 `DB_SHUTDOWNED`，也需要 recovery。
如果存在 recovery signal，也强制 recovery。
见 `xlogrecovery.c:857-875`。

如果需要 recovery，`InitWalRecovery()` 更新内存里的 control file：
- state 改为 `DB_IN_CRASH_RECOVERY` 或 `DB_IN_ARCHIVE_RECOVERY`。
- `checkPoint` 改为选中的 checkpoint location。
- `checkPointCopy` 改为读出的 checkpoint payload。
- archive recovery 下初始化或推进 `minRecoveryPoint`。
见 `xlogrecovery.c:877-947`。

`StartupXLOG()` 随后把 checkpoint payload 初始化到 shared memory。
例如 nextXid、nextOid、MultiXact、oldest xid。
见 `xlog.c:5995-6004`。

然后设置 redo pointer：

```text
RedoRecPtr = XLogCtl->RedoRecPtr = XLogCtl->Insert.RedoRecPtr = checkPoint.redo
doPageWrites = lastFullPageWrites
```

见 `xlog.c:6116-6119`。

如果 `InRecovery`，`StartupXLOG()` 会把 control file 写回磁盘，标记正在 recovery。
见 `xlog.c:6132-6140`。
这发生在真正 replay 前。
如果 recovery 中再次崩溃，下次启动能知道上次是在 recovery 中断。

真正 replay 在 `PerformWalRecovery()`。
它在 `xlogrecovery.c:1606-1875`。
如果 `RedoStartLSN < CheckPointLoc`，它从 `RedoStartLSN` 开始读。
并要求第一条 record 是 `XLOG_CHECKPOINT_REDO`。
见 `xlogrecovery.c:1658-1678`。

如果 `RedoStartLSN >= CheckPointLoc`，说明 shutdown checkpoint 或其它不需要回退的情况。
它从 checkpoint record 后的下一条 record 读。
见 `xlogrecovery.c:1680-1686`。

主循环在 `xlogrecovery.c:1707-1806`。
每条 record：
1. 检查中断。
2. 检查 pause。
3. 检查 recovery target before。
4. 可能等待 recovery apply delay。
5. `ApplyWalRecord()`。
6. 唤醒等待 replay/write/flush LSN 的进程。
7. 检查 recovery target after。
8. 读下一条 record。

`ApplyWalRecord()` 在 `xlogrecovery.c:1882-2040`。
它先处理 timeline switch。
然后在 redo 之前设置 `XLogRecoveryCtl->replayEndRecPtr = record->EndRecPtr`。
注释说这是为了让 `XLogFlush()` 正确更新 `minRecoveryPoint`。
见 `xlogrecovery.c:1942-1949`。

如果 record 属于 XLOG rmgr，先调用 `xlogrecovery_redo()` 做 recovery 相关特殊处理。
然后调用通用 rmgr redo：

```text
GetRmgr(record->xl_rmid).rm_redo(xlogreader)
```

见 `xlogrecovery.c:1958-1966`。

redo 成功后，`ApplyWalRecord()` 才更新 `lastReplayedReadRecPtr`、`lastReplayedEndRecPtr` 和 replay TLI。
见 `xlogrecovery.c:1979-1987`。
这就是 `GetXLogReplayRecPtr()` 返回“已经成功 replay 的位置”的原因。

错误上下文在 `xlogrecovery.c:2248-2267`。
redo 出错时，日志会带上：

```text
WAL redo at <ReadRecPtr> for <record desc and block info>
```

## 15. restartpoint：recovery 中的 checkpoint 类似物

restartpoint 用于 recovery 期间缩短下次恢复时间。
它的入口是 `CreateRestartPoint()`。
源码在 `xlog.c:8116-8387`。

restartpoint 和 checkpoint 的共同点：
- 都调用 `CheckPointGuts()`。
- 都把 dirty shared buffers、SLRU、fsync request 推到稳定状态。
- 都更新 `pg_control`。
- 都可能触发旧 WAL 清理和预分配。
- 都记录 checkpoint/restartpoint 统计日志。

restartpoint 和 checkpoint 的不同点更重要。
第一，restartpoint 不写新的 checkpoint WAL record。
它只能使用已经 replay 到的 checkpoint record。
startup process 在 redo 到 checkpoint record 时调用 `RecoveryRestartPoint()`。
`RecoveryRestartPoint()` 在 `xlog.c:8078-8114`。
它把 `lastCheckPointRecPtr`、`lastCheckPointEndPtr` 和 `lastCheckPoint` 拷贝到 shared memory。

第二，如果存在 unresolved invalid pages，不能建立 restartpoint。
`xlog.c:8090-8103` 直接返回。
原因是如果从这个 restartpoint 重启 recovery，会跳过之前的 invalid page references。
那样就失去了后续 drop relation 是否解释这些 invalid references 的交叉检查。

第三，`CreateRestartPoint()` 只能在 checkpointer 或 standalone 中执行。
`xlog.c:8141-8142` 有断言。
实际 normal server 中由 checkpointer 执行。

第四，如果没有新的 safe checkpoint record，restartpoint 会跳过。
`xlog.c:8176-8192` 判断：

```text
!valid(lastCheckPointRecPtr) ||
lastCheckPoint.redo <= ControlFile->checkPointCopy.redo
```

这时它仍会调用 `UpdateMinRecoveryPoint(InvalidXLogRecPtr, true)`。
如果是 shutdown restartpoint，还会把 state 写成 `DB_SHUTDOWNED_IN_RECOVERY`。

第五，restartpoint 推进 shared redo pointer 到 `lastCheckPoint.redo`。
见 `xlog.c:8194-8210`。
注释说 recovery 中没有普通 WAL insert，这个锁更多是形式上保持与 checkpoint 一致。

第六，restartpoint 调 `CheckPointGuts(lastCheckPoint.redo, flags)`。
见 `xlog.c:8228`。
这会刷出 recovery replay 产生的脏页。

第七，restartpoint 更新 `pg_control` 时使用的是 replay 到的 checkpoint record。
`xlog.c:8248-8292`：

```text
ControlFile->checkPoint = lastCheckPointRecPtr
ControlFile->checkPointCopy = lastCheckPoint
```

如果仍处于 archive recovery，它还确保 `minRecoveryPoint >= lastCheckPointEndPtr`。
源码注释解释：备份在 recovery 中执行时，必须至少包含 checkpoint record 所在 WAL 文件。
普通 recovery restart 也没有必要把最小恢复点放在 checkpoint record 之前。

第八，restartpoint 的 WAL 删除边界还要考虑 receive/replay end。
`xlog.c:8305-8314` 先从 redo pointer 算 segment。
然后取 walreceiver flush ptr 和 replay ptr 的较大值作为 endptr，交给 `KeepLogSeg()`。
这是 standby/recovery 与 primary checkpoint 的差异。

第九，restartpoint 结束后可以执行 `archive_cleanup_command`。
见 `xlog.c:8378-8384`。
普通 checkpoint 没这个动作。

所以 restartpoint 可以总结为：

```text
把已经 replay 到的 checkpoint record 变成新的 control-file recovery 起点，
但不创造新的 checkpoint record。
```

## 16. timeline 边界

checkpoint record 里有 timeline。
recovery 不能只看 LSN。
同一个 LSN 可以在不同 timeline 上有不同历史。

`timeline.c` 负责 timeline history 文件读写和查询。
`readTimeLineHistory()` 在 `timeline.c:77` 开始。
如果找不到 history file，源码认为这个 timeline 没有 parents。
它返回一个 begin/end 都 invalid 的单元素 history。
见 `timeline.c:106-118`。

解析 history file 时，每行包含 parent TLI 和 switchpoint。
`timeline.c:158-183` 把它们变成 `TimeLineHistoryEntry`。
每个 entry 有：
- `tli`
- `begin`
- `end`

`tliOfPointInHistory()` 在 `timeline.c:545-564`。
它遍历 history。
如果 `begin <= ptr < end`，就返回该 ptr 所在 timeline。
如果找不到，说明 history 不连续，ERROR。

`tliSwitchPoint()` 在 `timeline.c:573-593`。
它返回从某条 timeline 分叉出去的位置。
如果请求的 timeline 不在 history 中，ERROR。

`InitWalRecovery()` 用这些函数做 checkpoint timeline 校验。
`xlogrecovery.c:791-812` 确认 checkpoint record 的 timeline 在请求 timeline 的 history 中。
`xlogrecovery.c:818-825` 确认 minRecoveryPoint 也在 history 中。

`ApplyWalRecord()` 在 replay 前检查 timeline switch。
`checkTimeLineSwitch()` 在 `xlogrecovery.c:2347-2391`。
当前版本中，timeline 只能在 shutdown checkpoint 处改变。
它校验：
- record 中的 previous TLI 等于当前 replay TLI。
- new TLI 在 expected history 中。
- new TLI 不能倒退。
- 不能在达到 minRecoveryPoint 之前切到更高 timeline 而错过 minRecoveryPoint。

这解释了为什么 checkpoint record 中的 `ThisTimeLineID` 和 `PrevTimeLineID` 是 recovery correctness 的一部分。

## 17. 正确性与错误边界

checkpoint/recovery 的错误边界可以按阶段看。

第一类是请求阶段错误。
`RequestCheckpoint()` 如果找不到 checkpointer 且调用者要求 wait，可能 ERROR。
如果不 wait，只 LOG。
见 `checkpointer.c:1119-1131`。
如果等待的 checkpoint 失败，等待者报 ERROR。
见 `checkpointer.c:1191-1194`。

第二类是 checkpointer 自身 ERROR 恢复。
`CheckpointerMain()` 在 `checkpointer.c:285-345` 用 `sigsetjmp` 做顶层错误恢复。
发生 ERROR 后释放 LWLock、取消 condition variable sleep、清理 buffer 和 smgr 状态。
如果当时有 active checkpoint，它递增 `ckpt_failed`，把 `ckpt_done` 推到 `ckpt_started`，广播 `done_cv`。
等待者能收到失败。

第三类是 checkpoint critical section。
`CreateCheckPoint()` 在确定 WAL/control file 关键状态时使用 critical section。
`xlog.c:7448-7451` 进入。
刷大量脏页前 `xlog.c:7655-7663` 退出。
写 checkpoint record、flush checkpoint record、更新 control file 时 `xlog.c:7742-7810` 又进入 critical section。
这些阶段出错通常意味着系统状态可能半更新，所以要 PANIC。

第四类是 checkpoint I/O。
`CheckPointGuts()` 刷 SLRU、buffer pool、sync request。
`CreateCheckPoint()` 特意在这之前退出 critical section。
注释说这些 I/O 可能失败，失败时 checkpoint 不完成即可，不必强制 panic。
见 `xlog.c:7655-7663`。

第五类是 WAL-before-data 错误。
`FlushBuffer()` 写 permanent buffer 前必须 `XLogFlush(page_lsn)`。
如果 WAL flush 失败，数据页不能继续安全写出。
这是基本 WAL rule 的执行点。
见 `bufmgr.c:4553-4571`。

第六类是 fsync request 队列错误。
`AbsorbSyncRequests()` 的注释在 `checkpointer.c:1446-1458`。
一旦请求从共享队列清掉，如果之后不能吸收进本地 sync 机制，系统不能安全运行，因此必须 PANIC。

第七类是 control file 错误。
`ReadControlFile()` 对读不到、CRC 错、版本错、编译参数不匹配等情况报 PANIC 或 FATAL。
见 `xlog.c:4405-4627`。
`update_controlfile()` 写、fsync、close 失败在 backend 中都是 PANIC。
见 `controldata_utils.c:217-283`。

第八类是 checkpoint record 无效。
`ReadCheckpointRecord()` 对无效 checkpoint location、无效 rmid、无效 xl_info、长度错误记录 LOG 并返回 NULL。
调用方在必须有 checkpoint 时 FATAL。
见 `xlogrecovery.c:4056-4105` 和 `xlogrecovery.c:731-741`。

第九类是 redo pointer 自相矛盾。
`xlogrecovery.c:852-855` 不允许 checkpoint redo 大于 checkpoint location。
`xlogrecovery.c:862-867` 不允许 shutdown checkpoint 的 redo 小于 checkpoint location。
这些是 PANIC。

第十类是 restartpoint 不安全。
`RecoveryRestartPoint()` 如果有 invalid pages，直接不记录候选 restartpoint。
这不是 ERROR。
它只是保守地不推进 recovery 起点。
见 `xlog.c:8090-8103`。

第十一类是 timeline 不匹配。
checkpoint 不属于请求 timeline history 时，`InitWalRecovery()` FATAL。
replay 中 checkpoint/end-of-recovery 记录的 timeline switch 不合法时，`checkTimeLineSwitch()` PANIC。
见 `xlogrecovery.c:786-825` 和 `xlogrecovery.c:2354-2388`。

第十二类是 redo routine 出错。
`ApplyWalRecord()` 安装 `rm_redo_error_callback()`。
错误上下文会说明 WAL record 的 ReadRecPtr、rmgr 描述、block references。
见 `xlogrecovery.c:1888-1892` 和 `xlogrecovery.c:2248-2267`。

## 18. 成本、资源与跨模块传播

checkpoint 的成本来自三段：写脏页、fsync 已写文件、写并 flush checkpoint record 和 control file。它随 dirty buffer 数、SLRU 脏页数、pending fsync entry 数、WAL 产生速度、磁盘 flush latency、replication/recovery timeline 状态扩张。

资源压力会跨模块传播。WAL 产生快会触发 WAL cause checkpoint；dirty buffer 多会拉长 write phase；fsync request 多或存储 flush 慢会拉长 sync phase；redo pointer 推进慢会增加 WAL 保留和 recovery replay 距离；checkpoint I/O burst 又会影响前台延迟和 bgwriter/checkpointer 调度。

shared state 的推进者包括 backend、checkpointer、bgwriter、walwriter、startup process 和 archiver。backend 推动 WAL insert 和 dirty page 产生；bgwriter 预写 dirty buffer；checkpointer 完成 checkpoint/restartpoint；walwriter 负责 WAL flush 背景推进；startup process 在 recovery 中 replay checkpoint record 并产生 restartpoint 候选；archiver 和 WAL recycling 受 redo horizon 约束。

## 19. 观测与诊断入口

本节的 runtime truth 是：一次 completed checkpoint 对外发布了一个 checkpoint record location 和一个 redo pointer，二者在线 checkpoint 中通常不同。

能直接观测的是 `log_checkpoints` 里的 starting/complete、`lsn` 和 `redo lsn`，`pg_controldata` 的 Latest checkpoint location 和 REDO location，`pg_stat_checkpointer`/`pg_stat_bgwriter` 累计，`pg_stat_wal` 的 WAL 量，以及 `pg_waldump` 中的 `CHECKPOINT_REDO`、`CHECKPOINT_ONLINE`、`CHECKPOINT_SHUTDOWN`。能间接推断的是 dirty page 集合、fsync pending 数和 full-page image 变化。几乎不可见的是 `RedoRecPtr` 更新瞬间、critical section 边界和 timeline history 校验内部状态，需要断点或源码插桩。

诊断 checkpoint 变慢时先区分 write time、sync time 和 total time；再判断是 dirty buffer、fsync latency、WAL pressure、control file/WAL flush、restartpoint 限制还是 timeline/recovery 边界。不要把 checkpoint complete 日志里的 `lsn` 当成 redo start；要同时看 `redo lsn`。

## 20. 常见误区

- checkpoint 不是单纯“刷脏页”；它还包含 fsync request、checkpoint record、control file 和 WAL recycling/restartpoint 边界。
- 在线 checkpoint 的 checkpoint record location 不等于 redo pointer；redo pointer 指向 `XLOG_CHECKPOINT_REDO`。
- shutdown checkpoint 不需要两条 record，是因为不应再有并发 WAL insert。
- `pg_control` 更新不是 checkpoint 开始的标志；它发生在 checkpoint record flush 之后。
- full-page writes 的判断和 redo pointer 相关，不是只看页面是否 dirty。
- restartpoint 不会创造新的 checkpoint record；它只能选择 recovery 已 replay 到的安全 checkpoint。
- checkpoint I/O 失败和 control-file/WAL 关键状态失败的错误级别不同，不能一概 PANIC 或一概重试。

## 21. 课堂实验

实验 1：画在线 checkpoint 时序图。

```bash
cd /home/highgo/postgres
nl -ba src/backend/access/transam/xlog.c | sed -n '7378,7810p'
```

在图上标出 `XLOG_CHECKPOINT_REDO`、`CheckPointGuts()`、`XLOG_CHECKPOINT_ONLINE` flush、`ControlFile->checkPoint` 更新，以及 `checkPoint.redo` 的来源。

实验 2：观察日志和 `pg_control`。

```sql
ALTER SYSTEM SET log_checkpoints = on;
SELECT pg_reload_conf();
CHECKPOINT;
```

查看日志中的 `lsn` 和 `redo lsn`。停止测试集群后执行：

```bash
pg_controldata "$PGDATA" | rg "Latest checkpoint|REDO|Database cluster state"
```

把 Latest checkpoint location 映射到 `ControlFile->checkPoint`，把 REDO location 映射到 `ControlFile->checkPointCopy.redo`。

实验 3：用 `pg_waldump` 验证 record。

```bash
pg_waldump -p "$PGDATA/pg_wal" <segment>
```

找到 `rmgr: XLOG` 下的 `CHECKPOINT_REDO`、`CHECKPOINT_ONLINE` 或 `CHECKPOINT_SHUTDOWN`。在线 checkpoint 中记录两条 record 的 LSN，确认 online checkpoint payload 指向 redo LSN。

## 22. 讨论题

1. 在线 checkpoint 为什么需要 `XLOG_CHECKPOINT_REDO` 和 `XLOG_CHECKPOINT_ONLINE` 两条 record？
2. `CheckPoint.redo` 为什么不能简单等于 checkpoint record location？
3. `pg_control` 为什么必须在 checkpoint record flush 后才更新？
4. dirty page 写出前的 `XLogFlush(page_lsn)` 和 checkpoint record flush 分别保护什么？
5. redo pointer 推进为什么会影响 full-page image 决策？
6. restartpoint 为什么可能因为 invalid pages 或 timeline/minRecoveryPoint 边界而不推进？
7. 哪些 checkpoint 错误可以放弃本次 checkpoint，哪些必须 PANIC/FATAL？
8. 日志中的 `lsn`、`redo lsn`、`pg_controldata` 的 REDO location 分别对应源码里的什么字段？

## 23. 本节小结

本节核心链路是：在线 checkpoint 开始时写 `XLOG_CHECKPOINT_REDO` 并形成新的 redo pointer；完成 dirty page、SLRU 和 fsync request 后，写并 flush `XLOG_CHECKPOINT_ONLINE`；最后更新并 fsync `pg_control`。shutdown checkpoint 因为没有并发 WAL insert，只需要一条 shutdown checkpoint record。

核心状态边界是：`ControlFile->checkPoint` 保存 checkpoint record location，`ControlFile->checkPointCopy.redo` 保存 recovery replay start。redo pointer 还影响 WAL insert 端是否需要 full-page image。

生命周期和 cleanup 由 checkpointer 主循环、`CreateCheckPoint()`、`CheckPointGuts()`、`SyncPreCheckpoint()`/`ProcessSyncRequests()`/`SyncPostCheckpoint()` 和 control file 更新共同完成。restartpoint 是 recovery 中对已 replay checkpoint 的推进，不创造新 checkpoint record。

异常路径的判断框架是：能安全放弃本次 checkpoint 的 I/O 失败，可以让 checkpoint 失败；已经发布或即将发布 recovery 起点的 WAL/control-file 关键状态失败，通常必须 PANIC/FATAL。timeline history、minRecoveryPoint 和 invalid page references 决定 recovery 中能否安全推进。

可观测入口是 `log_checkpoints`、`pg_controldata`、`pg_waldump`、`pg_stat_checkpointer`/`pg_stat_io` 和源码断点。可迁移规律是：恢复起点不是一个日志位置那么简单，它是“此前数据文件状态已经足够稳定，之后 WAL replay 足以补齐”的系统承诺。
