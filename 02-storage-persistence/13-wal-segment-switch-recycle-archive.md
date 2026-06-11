# PostgreSQL WAL segment、switch、recycle 与 archive 边界

## 课程定位

前置知识：已理解 WAL byte stream、`XLogRecPtr`、record end LSN、checkpoint redo pointer、archive 和 replication slot 的基本作用。

本节唯一主问题：

```text
一个 WAL segment 什么时候可以从当前写入对象变成可归档、可回收、可删除，或者仍必须保留？
```

核心矛盾：系统希望尽快复用和删除旧 WAL 以控制磁盘；但 crash recovery、PITR、归档、timeline 和复制 slot 都可能仍然需要同一段 WAL。

一句话运行模型：

```text
WAL 逻辑上是一条连续 byte stream，pg_wal segment 只是固定大小切片；switch 推进 segment 边界，XLogWrite() 完成 segment 并通知归档，checkpoint/restartpoint 再综合 redo、slot、wal_keep_size、archive gate 和 timeline 决定保留、回收或删除。
```

学完后应能判断：`XLByteToSeg()` 与 `XLByteToPrevSeg()` 的边界差异；switch record 为什么消耗剩余空间；`.ready/.done` 谁创建和消费；slot / checkpoint / archive 如何共同推迟删除。

源码基线：本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`。

## 1. 本节在总主线中的位置

前面课程主要关注一条 WAL record 如何产生、插入和 flush。本节把视角扩大到 WAL 文件生命周期：连续 WAL byte stream 如何落到 `pg_wal` 的 segment 文件，又如何在写完后进入归档、回收、删除或恢复读取路径。

它不讲 rmgr 语义，也不重新展开 insertion lock。我们只盯住 segment 边界和保留条件。

## 2. 核心矛盾与一句话运行模型

WAL segment 管理的难点在于，一个旧文件对当前写入者可能已经无用，但对 crash recovery、PITR、standby、replication slot、archive_command 或 timeline history 仍可能是必需品。删除和回收必须在所有这些边界之后发生。

最短模型如下：

```text
LSN -> segment/offset
  -> switch 推进边界
  -> XLogWrite() 完成 segment
  -> archive_status 标记 ready/done
  -> checkpoint/restartpoint 计算保留边界
  -> recycle/remove 或 recovery 继续读取
```

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节读什么 |
| --- | --- | --- |
| 1 | `src/include/access/xlog_internal.h` | `XLogRecPtr` 到 segment number、file name、offset 的宏和边界。 |
| 2 | `src/backend/access/transam/xlog.c` | switch、`XLogWrite()`、segment create/preallocate/recycle/remove、checkpoint 清理入口、`KeepLogSeg()`。 |
| 3 | `src/backend/access/transam/xlogarchive.c` | `XLogArchiveNotify()`、`XLogArchiveCheckDone()`、`.ready/.done` gate。 |
| 4 | `src/backend/postmaster/pgarch.c` | archiver 如何消费 `.ready` 并生成 `.done`。 |
| 5 | `src/backend/access/transam/xlogrecovery.c` | recovery 读取 WAL segment、来源状态机和 record 合法性检查。 |
| 6 | `src/backend/access/transam/timeline.c` | timeline history 如何限制恢复读取。 |
| 7 | `src/include/postmaster/pgarch.h` | archiver 边界的头文件入口。 |

## 4. 关键结论：segment 边界

WAL 在逻辑上是一条单调增长的 byte stream。
`XLogRecPtr` 是这条 byte stream 上的位置。
`pg_wal` 里的 segment 文件只是把同一条 byte stream 按固定 `wal_segment_size` 切片后的外部形态。
核心换算在 `xlog_internal.h:99-120`。
`XLByteToSeg()` 回答：
`从这个 LSN 开始读，属于哪个 segment？`
`XLByteToPrevSeg()` 回答：
`写到这个 end pointer 为止，刚完成的是哪个 segment？`
只有当 LSN 正好在 segment 边界上时，两者不同。
`XLogSegmentOffset()` 回答 LSN 在 segment 内的 offset。
源码用：
`lsn & (wal_segment_size - 1)`
这要求 `wal_segment_size` 是 2 的幂。
`xlog_internal.h:86-97` 限制 WAL segment size 必须是 1MB 到 1GB 之间的 2 的幂。
`XLogFileName()` 在 `xlog_internal.h:164-170`。
它生成 24 个十六进制字符：
`TTTTTTTTLLLLLLLLSSSSSSSS`
第一段是 timeline ID。
第二段是 segment number 除以每 4GB 可容纳的 segment 数。
第三段是 segment number 在这个 4GB 组内的余数。
默认 16MB segment 时，每 4GB 有 256 个 segment。
所以 timeline 1 上：

```text
segno 0   -> 000000010000000000000000
segno 1   -> 000000010000000000000001
segno 255 -> 0000000100000000000000FF
segno 256 -> 000000010000000100000000
```

segment switch 的本质不是“创建一个新文件”。
它是把 WAL insert position 推到下一个 segment 的开头。
如果当前位置已经在 segment 开头，`ReserveXLogSwitch()` 返回 false，不需要写真实 switch record。
见 `xlog.c:1205-1258`。
如果当前位置不在开头，switch record 会占用一个 WAL record header，然后把当前 segment 剩余空间也当作已保留空间消耗掉。
见 `xlog.c:1347-1406`。
`XLogWrite()` 真正把 WAL buffers 写入 segment 文件。
当它完成一个 segment，会 fsync 该 segment，唤醒 walsender，必要时创建 `.ready`，记录最后 switch 时间和 LSN，并可能请求 checkpoint。
见 `xlog.c:2325-2601`。
旧 WAL 不在 switch 时删除。
它在 checkpoint 或 restartpoint 清理阶段成为候选。
候选边界先来自 checkpoint redo，再被 replication slot、`wal_keep_size`、WAL summarization、`max_slot_wal_keep_size` 调整，最后还要通过 archive gate。
普通 checkpoint 的清理入口见 `xlog.c:7830-7871`。
`KeepLogSeg()` 的保留边界逻辑见 `xlog.c:8497-8563`。
归档状态是另一条边界。
`XLogArchiveNotify()` 创建 `.ready`。
archiver 归档成功后把 `.ready` 改成 `.done`。
checkpoint 删除或回收前调用 `XLogArchiveCheckDone()`，看到 `.done` 才能继续。
见 `xlogarchive.c:445-607` 和 `pgarch.c:392-505`、`pgarch.c:813-838`。
恢复侧不能只看文件名。
它要检查 checkpoint redo、timeline history、WAL page header、record 合法性和 WAL 来源状态机。
相关代码在 `xlogrecovery.c:456-966`、`xlogrecovery.c:3108-3498`、`xlogrecovery.c:3533-4375`。

## 5. 核心名词

`XLogRecPtr`：WAL byte stream 上的位置，通常显示为 `X/Y`。
`wal_segment_size`：每个 WAL segment 文件的大小。它是集群级属性，不是普通在线可改 GUC。
`XLogSegNo`：segment number，从 byte stream 开头按 `wal_segment_size` 编号。
`TimeLineID`：timeline。它参与文件名，同一个 segment number 在不同 timeline 上可以有不同文件。
`XLogFileName()`：把 `tli + segno + wal_segment_size` 转成 24 字符 WAL 文件名。
`XLogFromFileName()`：从文件名反推出 timeline 和 segment number，见 `xlog_internal.h:198-205`。
`XLogSegmentOffset()`：返回 LSN 在当前 segment 内的 offset。
`XLByteToSeg()`：把 LSN 归到当前位置所在 segment；边界 LSN 归到后一个 segment。
`XLByteToPrevSeg()`：把边界 LSN 归到前一个 segment；适合“刚写完哪里”的问题。
`archive_status`：`pg_wal/archive_status` 下的状态目录。
`.ready`：这个 WAL 相关文件需要 archiver 归档。
`.done`：archiver 已成功归档。
`wal_recycle`：允许把旧 segment 重命名成未来 segment，而不是直接删除。
`replication slot restart_lsn`：slot 仍需要的最老 WAL 位置，会推后删除边界。

## 6. byte stream 到文件名和 offset

先读 `xlog_internal.h:86-120`。
这里把 WAL 地址体系压缩成几个宏：
- `IsValidWalSegSize`
- `XLogSegmentsPerXLogId`
- `XLogSegNoOffsetToRecPtr`
- `XLogSegmentOffset`
- `XLByteToSeg`
- `XLByteToPrevSeg`
`XLogSegmentsPerXLogId(wal_segment_size)` 表示 4GB 地址空间内有多少 segment。
默认 16MB 时是 256。
从 LSN 找文件名和 offset 的过程是：

```text
XLByteToSeg(lsn, segno, wal_segment_size)
XLogFileName(fname, tli, segno, wal_segment_size)
offset = XLogSegmentOffset(lsn, wal_segment_size)
```

从文件名找 byte stream 范围的过程是：

```text
XLogFromFileName(fname, &tli, &segno, wal_segment_size)
start = segno * wal_segment_size
end = start + wal_segment_size
```

手工例子，假设 `wal_segment_size = 16MB = 0x01000000`。
LSN：
`0/02000188`
byte position 是 `0x02000188`。
segment number 是 `2`。
segment offset 是 `0x188`。
timeline 1 上文件名是：
`000000010000000000000002`
边界 LSN 更容易出错。
LSN：
`0/02000000`
`XLByteToSeg()` 得到 segment 2。
`XLByteToPrevSeg()` 得到 segment 1。
如果问题是“从这里开始读”，读 segment 2。
如果问题是“写到这里为止刚完成哪个 segment”，答案是 segment 1。
这个差异会在 `XLogWrite()` 中直接出现。
当 `LogwrtResult.Write` 正好是下一个 segment 开头，写入路径仍然要 fsync 刚完成的前一个 segment。
所以代码使用 previous segment 语义，见 `xlog.c:2381-2397`。

## 7. segment 不是 WAL page

segment 是外部文件边界。
WAL page 是 segment 内部的页边界。
`CopyXLogRecordToWAL()` 写 record 时会跨 WAL page 拷贝，见 `xlog.c:1266-1344`。
record 跨页时，下一页需要 continuation 信息。
segment 第一页使用 long page header，其他页使用 short page header。
所以 `XLogSegmentOffset()` 只回答“在 segment 文件中的 offset”。
它不回答“当前 WAL page 剩多少 payload 空间”。
理解时可以建立两层模型：

```text
XLogRecPtr byte stream
  -> 按 XLOG_BLCKSZ 分 WAL page
  -> 按 wal_segment_size 分 segment file
```

switch 会同时触碰这两层：它要把 byte stream 推到下一个 segment，同时还要让剩余 WAL page 进入可写出状态。

## 8. segment switch

`RequestXLogSwitch()` 在 `xlog.c:8605-8617`。
它发起一条 `RM_XLOG_ID / XLOG_SWITCH` WAL insert。
真正特殊处理在 `ReserveXLogSwitch()` 和 `CopyXLogRecordToWAL()`。
`ReserveXLogSwitch()` 在 `xlog.c:1205-1258`。
它先读取当前 WAL insert byte position。
如果当前位置换算成 `XLogRecPtr` 后 offset 为 0，说明已经在 segment 开头。
这时函数把 `StartPos` 和 `EndPos` 都设为当前 LSN，并返回 false。
这解释了一个常见现象：
`连续执行 pg_switch_wal()，第二次可能不会生成新 segment。`
如果当前位置不在 segment 开头，它先预留 switch record header。
然后计算 `EndPos` 到 segment 末尾还剩多少。
如果还有剩余空间，就把 `EndPos` 推到下一个 segment 开头。
最后断言 `EndPos` 的 segment offset 为 0。
这一步只是 reservation。
实际写 WAL buffer 在 `CopyXLogRecordToWAL()`。
当 `isLogSwitch` 为 true 时，代码确认 switch record 没有额外 payload，然后把当前 segment 余下空间推进到 `EndPos`。
见 `xlog.c:1347-1406`。
剩余 WAL page 的 page header 处理还有一个压缩友好性考虑。
源码会让 switch 后余下部分尽量保持连续零，而不是每个 WAL page 都留下不同 header 值。
所以被 switch 出来的 WAL segment 后半段有大片零，不表示损坏。
它表示这个 segment 被人为关闭。
switch 的关键结论：

```text
switch 关闭当前 segment 的 WAL address space。
它不等于一定新建文件。
它不等于旧 segment 可以删除。
它不等于 archive 已完成。
```

## 9. `XLogWrite()` 和 segment 完成点

`XLogWrite()` 在 `xlog.c:2325-2601`。
它负责把 WAL buffers 写入 segment 文件，并按请求 fsync 到指定 LSN。
循环开始时，它从 `LogwrtResult.Write` 所在 WAL buffer page 写起。
每个待写页都有一个 `EndPtr`。
如果写位置不再属于当前 `openLogSegNo`，它关闭旧文件，按 `XLByteToPrevSeg()` 打开或创建要写的 segment。
见 `xlog.c:2381-2397`。
文件内 offset 通过 `XLogSegmentOffset(LogwrtResult.Write - XLOG_BLCKSZ, wal_segment_size)` 计算。
见 `xlog.c:2410-2417`。
当本次写入完成了 segment 最后一页，`finishing_seg` 为 true。
这个分支做六件重要事情，见 `xlog.c:2485-2525`：
- fsync 当前 WAL segment。
- 请求稍后唤醒 walsender。
- 把 flush position 推到 segment 末尾。
- 如果 archiving active，调用 `XLogArchiveNotifySeg()` 创建 `.ready`。
- 记录 `lastSegSwitchTime` 和 `lastSegSwitchLSN`。
- 如果距离 checkpoint redo 太远，请求 checkpoint。
普通事务提交也可能要求 WAL flush。
但普通 flush 只保证写到某个 LSN 并 fsync，不会创建 `.ready`。
创建 `.ready` 的时机是完整 segment 完成。
因此：

```text
commit flush 解决事务持久性。
finishing_seg 解决 segment 完整性和归档通知。
```

这两个边界不能混用。

## 10. WAL 文件创建和预创建

WAL segment 文件创建入口之一是 `XLogFileInit()`，内部调用 `XLogFileInitInternal()`。
关键代码在 `xlog.c:3243-3415`。
它先尝试打开目标文件。
如果文件已经由 checkpoint 预创建或 recycle 得到，就直接返回 fd。
如果不存在，它创建临时文件：
`pg_wal/xlogtemp.<pid>`
如果 `wal_init_zero` 开启，写满 `wal_segment_size` 的零。
如果关闭，只在文件末尾写一个零字节。
前者确保空间真实分配。
后者减少初始化写入。
随后 fsync 临时文件，再调用 `InstallXLogFileSegment()` 放入最终名字。
`InstallXLogFileSegment()` 在 `xlog.c:3613-3663`。
它有两个模式。
`find_free = false` 时，安装到指定 segment，必要时删除原文件。
`find_free = true` 时，从传入 segno 向后找空位，不能超过 `max_segno`。
这个函数受 `XLogCtl->InstallXLogFileSegmentActive` 控制。
recovery 或 walreceiver 某些阶段会关闭安装能力，避免本地预创建和恢复/流复制写入互相踩踏。
预创建逻辑在 `PreallocXlogFiles()`，见 `xlog.c:3741-3763`。
它非常保守：只有当前 end pointer 已经超过当前 segment 的 75%，才确保下一个 segment 存在。
checkpoint 尾部会在清理旧 WAL 后调用它，见 `xlog.c:7866-7871`。
这样可以优先利用刚 recycle 出来的文件。
重要边界：

```text
文件存在不等于文件里有有效 WAL 到末尾。
预创建文件也可能还没承载真实 WAL record。
恢复必须验证 page header、record、timeline。
```

## 11. recycle、remove 与 `XLOGfileslop()`

旧 WAL segment 的处理入口是 checkpoint 或 restartpoint。
普通 checkpoint 的路径在 `xlog.c:7830-7871`。
大致顺序是：

```text
SyncPostCheckpoint()
UpdateCheckPointDistanceEstimate()
XLByteToSeg(RedoRecPtr)
KeepLogSeg()
InvalidateObsoleteReplicationSlots()
KeepLogSeg() again if needed
RemoveOldXlogFiles()
PreallocXlogFiles()
```

`RedoRecPtr` 是 crash recovery 所需的基本下界。
早于这个 redo 边界的 WAL 才可能成为删除候选。
但它只是第一层边界。
`KeepLogSeg()` 会把边界继续往前退。
replication slots、`wal_keep_size`、WAL summarization 都可能要求保留更旧 segment。
真正处理候选文件的是 `RemoveOldXlogFiles()`，见 `xlog.c:3916-3972`。
它扫描 `pg_wal`。
只处理普通 WAL segment 文件名和 `.partial` 文件名。
它构造一个“最后可处理 segment”的文件名，并比较文件名的 segment 部分。
比较时忽略 timeline 的前 8 个字符。
这样能避免过早删除 parent timeline 上仍可能需要的 segment。
候选文件还要通过 archive gate：
`XLogArchiveCheckDone(xlde->d_name)`
见 `xlog.c:3959-3967`。
通过后才调用 `RemoveXlogFile()`。
`XLOGfileslop()` 在 `xlog.c:2250-2289`。
它决定旧文件是否值得 recycle 成未来 segment。
它综合：
- `min_wal_size_mb`
- `max_wal_size_mb`
- `CheckPointDistanceEstimate`
- `checkpoint_completion_target`
`RemoveXlogFile()` 在 `xlog.c:4059-4133`。
如果 `wal_recycle` 开启、未来 segment 仍在可回收上限内、文件是普通文件、当前允许安装 segment，并且 `InstallXLogFileSegment()` 成功，就 recycle。
否则删除。
无论 recycle 还是 remove，成功后都会调用 `XLogArchiveCleanup(segname)` 清理 `.done` 和残留 `.ready`。
见 `xlog.c:4133` 和 `xlogarchive.c:713-724`。
所以清理链条是：

```text
checkpoint 选候选
  -> archive gate
  -> recycle 或 remove
  -> cleanup archive_status
```

## 12. `archive_status` 与 archiver 边界

`XLogArchiveNotify()` 在 `xlogarchive.c:445-487`。
它创建：
`pg_wal/archive_status/<name>.ready`
状态文件内容为空。
文件名就是通知。
如果通知对象是 timeline history 文件，它会强制 archiver 下次重新扫描目录。
然后在 postmaster 下唤醒 archiver。
`XLogArchiveNotifySeg()` 在 `xlogarchive.c:493-501`。
它先用 `XLogFileName()` 从 segno 和 timeline 生成 WAL 文件名，再调用 notify。
写 WAL 完整 segment 的路径在 `xlog.c:2507-2508` 调用它。
timeline history 写完也会 notify，见 `timeline.c:448-452`。
`XLogArchiveCheckDone()` 在 `xlogarchive.c:565-607`。
它用于删除或回收前检查。
逻辑是：

```text
archiving off -> true
archive recovery 且不是 archive_mode=always -> true
存在 .done -> true
存在 .ready -> false
两者都没有 -> 补建 .ready，然后 false
```

补建 `.ready` 很重要。
如果第一次归档通知创建失败，后续 checkpoint 还会重试通知，而不是冒险删除。
`XLogArchiveIsBusy()` 在 `xlogarchive.c:619-651`。
它用于“等待归档完成”的场景，不会补建 `.ready`。
`XLogArchiveIsReadyOrDone()` 在 `xlogarchive.c:665-685`。
它主要给 recovery 路径判断一个文件是否已经或即将归档。
`KeepFileRestoredFromArchive()` 在 `xlogarchive.c:359-414`。
恢复从 archive 拉回 WAL 后，如果不是 `archive_mode = always`，它强制创建 `.done`，避免恢复出来的旧 segment 再被当作新 WAL 归档。
如果是 `archive_mode = always`，它创建 `.ready`，允许 standby/archive recovery 也归档这些文件。
archiver 本身在 `pgarch.c`。
`pgarch_readyXlog()` 扫描 `pg_wal/archive_status`，只取 `.ready` 文件，见 `pgarch.c:647-762`。
主循环拿到文件名后归档，见 `pgarch.c:392-505`。
成功后 `pgarch_archiveDone()` 把 `.ready` rename 成 `.done`，见 `pgarch.c:820-836`。
这个 rename 不是 durable rename。
源码要求 archive command 或 archive library 能容忍 crash 后重复归档。
最重要边界：

```text
WAL writer 只负责创建 .ready。
archiver 成功后才创建 .done。
checkpoint cleanup 看到 .done 才能删除或回收旧 WAL。
```

## 13. checkpoint、slot 与保留边界

`CalculateCheckpointSegments()` 在 `xlog.c:2191-2218`。
它根据 `max_wal_size_mb` 和 `checkpoint_completion_target` 估算触发 checkpoint 的 WAL 距离。
`XLogCheckpointNeeded()` 在 `xlog.c:2300-2310`。
它从当前 `RedoRecPtr` 所在 segment 算起，判断刚完成或刚读过的 segment 是否已经离 redo 太远。
正常写 WAL 完成 segment 时会检查它，见 `xlog.c:2520-2524`。
recovery 读到新 segment 时也会检查它，见 `xlogrecovery.c:3301-3311`。
slot 相关共享状态是 `XLogCtl->replicationSlotMinLSN`，见 `xlog.c:465`。
读取函数在 `xlog.c:2700-2707`。
`KeepLogSeg()` 会读取最老 slot LSN，见 `xlog.c:8506-8511`。
如果 slot 需要的 LSN 早于当前 `recptr`，保留边界会退到该 LSN 所在 segment。
这就是 slot 会撑大 `pg_wal` 的原因。
但 slot 不是无限保留。
如果 `max_slot_wal_keep_size` 非负，`KeepLogSeg()` 会限制 slot 最多向后保留多少 segment。
见 `xlog.c:8512-8527`。
checkpoint 随后会调用 `InvalidateObsoleteReplicationSlots()`。
如果有 slot 被失效，就重新计算保留边界。
见 `xlog.c:7851-7861`。
`wal_keep_size` 是另一条保留规则。
它不绑定具体 consumer，只保证最近至少保留一定量 WAL。
见 `xlog.c:8544-8557`。
WAL summarization 也可能推后删除。
见 `xlog.c:8530-8542`。
`GetWALAvailability()` 在 `xlog.c:8412-8477`。
它把某个 target LSN 的可用性分成：
- `WALAVAIL_RESERVED`
- `WALAVAIL_EXTENDED`
- `WALAVAIL_UNRESERVED`
- `WALAVAIL_REMOVED`
- `WALAVAIL_INVALID_LSN`
其中 `UNRESERVED` 表示不再被保留，但文件可能还没被 checkpoint 删除。
所以：

```text
不再 reserved 不等于立刻 removed。
max_wal_size 不是硬删除上限。
archive 未完成也会阻止删除。
```

## 14. crash recovery 边界

`InitWalRecovery()` 在 `xlogrecovery.c:456-966`。
它分析 control file、recovery signal、backup label，并决定是否需要 recovery、从哪里开始读 WAL。
如果有 `backup_label`，恢复从 backup label 指向的 checkpoint 开始，见 `xlogrecovery.c:532-648`。
如果没有 backup label，就从 `pg_control` 记录的 checkpoint 开始，见 `xlogrecovery.c:718-755`。
它必须能读到 checkpoint record。
如果 checkpoint redo 早于 checkpoint record，还必须确认 redo 位置存在。
见 `xlogrecovery.c:582-600` 和 `xlogrecovery.c:746-754`。
如果数据库干净关闭且没有 recovery signal，可能不进入 recovery。
如果 checkpoint redo 小于 checkpoint record，或者 control file 状态不是 shutdown，就进入 recovery。
见 `xlogrecovery.c:857-875`。
`PerformWalRecovery()` 在 `xlogrecovery.c:1611-1828`。
它从 checkpoint redo 或 checkpoint 后的下一条 record 开始主 redo loop。
主循环做：

```text
检查 pause
检查 recovery target before
必要时 apply delay
ApplyWalRecord()
唤醒等待 replay/write/flush LSN 的进程
检查 recovery target after
ReadRecord() 下一条
```

见 `xlogrecovery.c:1707-1806`。
crash recovery 的结束不是“读完某个文件名”。
它是“从选定 redo 点开始，按 WAL record 连续 replay，直到没有下一条有效 record 或达到 recovery target”。

## 15. recovery 读取 WAL segment

`ReadRecord()` 在 `xlogrecovery.c:3108-3240`。
它调用 WAL reader/prefetcher 读取下一条 record。
如果读到 record，会检查 WAL page 的 timeline 是否在 expected history 中。
见 `xlogrecovery.c:3169-3189`。
如果读不到 record，它会设置 `lastSourceFailed`。
一个关键分支是：如果请求了 archive recovery，但当前还在 crash recovery，并且不是在取 checkpoint，那么读到本地 `pg_wal` 末尾后会切入 archive recovery。
见 `xlogrecovery.c:3202-3236`。
`XLogPageRead()` 在 `xlogrecovery.c:3276-3498`。
它根据 `targetPagePtr` 计算 segment number 和 segment 内 offset，见 `xlogrecovery.c:3290-3292`。
如果目标页不在当前打开 segment 中，会关闭旧文件。
archive recovery 中，如果读过太多 WAL，还可能请求 restartpoint，见 `xlogrecovery.c:3301-3311`。
真正读文件时，offset 来自 `XLogSegmentOffset(targetPagePtr, wal_segment_size)`，见 `xlogrecovery.c:3378-3412`。
standby 模式下，如果 segment 第一页 header 无效，会立刻换来源重试，见 `xlogrecovery.c:3425-3471`。
这是为了处理本地 `pg_wal` 中存在同名但内容已经 recycle 的文件。
`WaitForWALToBecomeAvailable()` 在 `xlogrecovery.c:3533-3829`。
它实现 WAL 来源状态机：

```text
archive 或 pg_wal
promotion trigger check
stream
rescan timelines
sleep and retry
```

不在 archive recovery 时，读 `pg_wal`。
在 archive recovery 时，通常优先 archive。
standby 模式下，archive/pg_wal 失败后可以转向 stream。
stream 失败后会停止或重置 walreceiver，再回到 archive/pg_wal。
`XLogFileRead()` 在 `xlogrecovery.c:4201-4276`。
archive 来源会调用 `RestoreArchivedFile()`。
`pg_wal` 或 stream 来源直接构造 `pg_wal/<filename>` 路径。
`XLogFileReadAnyTLI()` 在 `xlogrecovery.c:4283-4375`。
它按 expected timeline history 遍历候选 timeline，不会随便读取任意同名 segment。
所以 recovery 读取 segment 的判断是：
`segment number + timeline history + source priority + WAL page/record validation`
不是“目录里存在这个文件就读”。

## 16. archive restore 边界

`RestoreArchivedFile()` 在 `xlogarchive.c:54-283`。
它只在 `ArchiveRecoveryRequested` 且存在 `restore_command` 时尝试 restore。
见 `xlogarchive.c:68-77`。
archive recovery 时，即使 `pg_wal` 里已有同名文件，也优先 archive copy。
原因是 `pg_wal` 里的文件可能来自 base backup，可能是旧的、未填满的或部分填充的。
见 `xlogarchive.c:79-100`。
restore 成功后会检查文件存在。
如果传入 expected size，还会检查大小。
见 `xlogarchive.c:185-227`。
restore 失败不一定表示 corruption。
PITR 的常见模式就是一直 roll forward，直到 restore command 找不到下一段 WAL。
源码在 `xlogarchive.c:241-270` 明确把这种失败视为恢复流程的一部分。
但信号、硬 shell 错误、错误大小等情况会升级为更高错误级别。
所以 archive restore 的边界是：

```text
找不到下一个 WAL 可以是正常结束。
找到了但大小不对、命令异常或信号中断，可能是错误。
```

## 17. timeline history 边界

timeline history 由 `timeline.c` 处理。
文件格式在 `timeline.c:1-22`。
文件名类似：
`00000005.history`
每行记录：
`parentTLI    switchpoint    reason`
`readTimeLineHistory()` 在 `timeline.c:76-217`。
timeline 1 没有 history 文件。
archive recovery 时，history 文件可以从 archive restore。
见 `timeline.c:88-104`。
解析结果是 newest first 的 timeline 链。
每个 entry 有 `tli`、`begin`、`end`。
`tliOfPointInHistory()` 在 `timeline.c:544-563`，回答某个 LSN 属于 history 中哪个 timeline。
`tliSwitchPoint()` 在 `timeline.c:572-592`，回答某个 timeline 在哪里分叉。
recovery 初始化会用这些函数确认 checkpoint 和 min recovery point 属于目标 timeline history。
见 `xlogrecovery.c:786-825`。
promotion 或 end-of-recovery 会创建新 timeline history。
`writeTimeLineHistory()` 在 `timeline.c:304-454`。
它写临时文件，复制 parent history，追加本次 switchpoint，fsync，再 rename 到最终文件名。
如果 archiving active，会立即通知 archiver，见 `timeline.c:448-452`。
`RemoveNonParentXlogFiles()` 在 `xlog.c:3990-4044`。
切到新 timeline 时，它清理不属于新 timeline history 的未来 WAL segment，避免旧 timeline 上预创建或 recycle 出来的垃圾文件被错误归档。

## 18. 成本、资源与常见误区：四个常见误读

误读一：
`switch 就是创建新 WAL 文件。`
不准确。switch 是 WAL insert position 推进到下一个 segment。目标文件可能已经预创建，也可能 recycle 得来，也可能稍后才创建。
误读二：
`.ready 表示已经归档。`
错误。`.ready` 表示等待归档，`.done` 才表示 archiver 成功。
误读三：
`max_wal_size 到了就强行删除旧 WAL。`
不准确。`max_wal_size` 影响 checkpoint 触发和回收估算，但 slot、`wal_keep_size`、summarization、archive status 都会影响最终删除边界。
误读四：
`文件存在就能用于 recovery。`
错误。文件可能是预创建、recycle 后残留、来自 base backup 的部分文件，或属于另一个 timeline。recovery 必须验证 page header、record、timeline 和来源状态。

## 19. 源码跟读练习一：LSN 到文件

目标：
`给一个 LSN，手算文件名和 offset。`
步骤：
1. 读 `xlog_internal.h:86-120`。
2. 找出 `XLogSegmentsPerXLogId()`。
3. 对比 `XLByteToSeg()` 和 `XLByteToPrevSeg()`。
4. 读 `xlog_internal.h:164-205`。
5. 用默认 16MB segment 手算 `0/02000188`。
预期：

```text
segno = 2
offset = 0x188
timeline 1 file = 000000010000000000000002
```

边界题：

```text
0/02000000
XLByteToSeg -> 2
XLByteToPrevSeg -> 1
```

解释时必须说明“从这里读”和“刚写完哪里”是两个问题。

## 20. 源码跟读练习二：switch

目标：
`解释 pg_switch_wal() 为什么有时不产生新 segment。`
步骤：
1. 读 `xlog.c:8605-8617`。
2. 跳到 `xlog.c:1205-1258`。
3. 找到 offset 为 0 时返回 false 的路径。
4. 继续读 `xlog.c:1347-1406`。
5. 画出 `StartPos`、switch record header、`EndPos` 的关系。
检查点：

```text
ReserveXLogSwitch() 返回 false 不是错误；
它表示当前 WAL insert position 已经在 segment 边界。
```

## 21. 源码跟读练习三：segment 完成

目标：
`找出完整 segment 写完时发生的副作用。`
步骤：
1. 读 `xlog.c:2325-2433`。
2. 看 `XLogWrite()` 如何判断当前打开文件。
3. 读 `xlog.c:2485-2525`。
4. 列出 `finishing_seg` 分支中的动作。
5. 对照 `xlogarchive.c:493-501`。
必须列出：

```text
fsync
walsender wakeup
flush position advance
archive notify
last switch time/LSN
maybe checkpoint request
```

## 22. 源码跟读练习四：checkpoint 删除边界

目标：
`解释 checkpoint 为什么不能只按 RedoRecPtr 删除。`
步骤：
1. 读 `xlog.c:7830-7871`。
2. 找到 `XLByteToSeg(RedoRecPtr)`。
3. 读 `xlog.c:8497-8563`。
4. 标出 slot、summarization、`wal_keep_size` 三个分支。
5. 回到 `xlog.c:7851-7861`。
6. 解释 slot invalidation 后为什么要重新计算。
一句话答案：

```text
RedoRecPtr 是 crash recovery 下界；
KeepLogSeg() 把它调整为所有保留需求中的最老边界。
```

## 23. 源码跟读练习五：archive_status

目标：
`区分 .ready、.done、删除前检查和 archiver 完成。`
步骤：
1. 读 `xlogarchive.c:445-487`。
2. 确认 `XLogArchiveNotify()` 创建 `.ready`。
3. 读 `xlogarchive.c:565-607`。
4. 确认 `XLogArchiveCheckDone()` 的 true/false 条件。
5. 读 `pgarch.c:647-762`。
6. 确认 archiver 扫描 `.ready`。
7. 读 `pgarch.c:820-836`。
8. 确认成功后 rename 为 `.done`。
9. 读 `xlog.c:4133` 和 `xlogarchive.c:713-724`。
10. 确认 recycle/remove 后清理状态文件。
一句话答案：
`.ready 是任务队列，.done 是完成标记，checkpoint 是删除候选的最终消费者。`

## 24. 源码跟读练习六：恢复读 segment

目标：
`解释 recovery 为什么不能只按文件名读 pg_wal。`
步骤：
1. 读 `xlogrecovery.c:456-966`。
2. 找到 crash recovery 和 archive recovery 的初始化差异。
3. 读 `xlogrecovery.c:3108-3240`。
4. 看 `ReadRecord()` 如何处理 invalid record 和 archive transition。
5. 读 `xlogrecovery.c:3276-3498`。
6. 看 `XLogPageRead()` 如何计算 offset 和验证 page header。
7. 读 `xlogrecovery.c:3533-3829`。
8. 画出 archive、pg_wal、stream 的状态机。
9. 读 `timeline.c:76-217` 和 `timeline.c:544-592`。
10. 说明 timeline history 如何限制候选文件。
结论：

```text
文件名只给出候选 segment；
recovery 还要验证 timeline、page、record、source 和 target。
```

## 25. 实验一：观察 LSN、文件名、offset

在测试实例中执行：

```sql
select pg_current_wal_lsn();
select pg_walfile_name(pg_current_wal_lsn());
select * from pg_walfile_name_offset(pg_current_wal_lsn());
```

练习：
1. 记录 LSN。
2. 确认实例实际 `wal_segment_size`。
3. 手工计算 segment number 和 offset。
4. 对比 SQL 输出。
5. 回到 `xlog_internal.h:99-120` 核对公式。
注意：
`不要假设所有实例都是 16MB segment。`

## 26. 实验二：观察 switch

执行：

```sql
select pg_current_wal_lsn();
select pg_switch_wal();
select pg_current_wal_lsn();
select pg_switch_wal();
```

观察：

```text
$PGDATA/pg_wal
$PGDATA/pg_wal/archive_status
```

如果 archiving 没开，可能没有 `.ready`。
如果第二次 switch 时已经在 segment 开头，可能不会产生新的 WAL segment。
对应源码：

```text
xlog.c:1205-1258
xlog.c:1347-1406
xlog.c:8605-8617
```

## 27. 实验三：观察 archive_status

配置测试归档：

```text
archive_mode = on
archive_command = 'test ! -f /tmp/pg-archive/%f && cp %p /tmp/pg-archive/%f'
```

创建目录：

```sh
mkdir -p /tmp/pg-archive
```

重启实例后执行：

```sql
select pg_switch_wal();
```

观察：

```sh
ls -l "$PGDATA/pg_wal/archive_status"
ls -l /tmp/pg-archive
```

成功时 `.ready` 会变成 `.done`。
执行：

```sql
checkpoint;
```

再次观察 `.done` 是否被清理。
对应源码链：

```text
XLogWrite()
XLogArchiveNotifySeg()
pgarch_readyXlog()
pgarch_archiveDone()
RemoveXlogFile()
XLogArchiveCleanup()
```

## 28. 诊断矩阵：segment 没有按预期出现、归档或删除

排查 WAL segment 问题时，先把现象归到具体边界，不要直接从文件名推断结论。

现象一：
`pg_switch_wal()` 返回了 LSN，但目录里没有你期待的新文件。
先看返回前的位置是否已经在 segment 边界。
`ReserveXLogSwitch()` 在边界上可能不真正推进到下一个 segment。
再看目标 segment 是否已经预创建或由 recycle 得来。
文件存在与否不是 switch 的唯一结果。

现象二：
segment 文件存在，但 `archive_status` 没有 `.ready`。
先确认该 segment 是否已经被完整写完。
普通 commit flush 只保证某个 LSN 之前的 WAL durable。
`.ready` 是完整 segment 归档任务，不是每次 flush 都创建。
断点优先放在 `XLogWrite()` 的 `finishing_seg` 分支和 `XLogArchiveNotifySeg()`。

现象三：
`.ready` 长期不变成 `.done`。
这通常不是 checkpoint 删除逻辑的问题。
先看 archiver 是否启动、`archive_command` 是否返回 0、目标目录是否已有同名文件。
`pgarch_readyXlog()` 取任务，`pgarch_archiveXlog()` 执行命令，成功后 `pgarch_archiveDone()` 写 `.done`。
只有 `.done` 才表示归档完成。

现象四：
旧 WAL 一直不删除。
把保留边界按从老到新列出来：
checkpoint redo pointer、replication slot restart LSN、`wal_keep_size`、archive status、summarization 或 recovery 需要。
最终必须保留到最老的消费边界。
`max_wal_size` 主要影响 checkpoint pressure 和 recycle 估算，不是越界删除许可。

现象五：
recovery 读到文件名匹配的 segment 仍然失败。
文件名只说明候选 timeline、log、seg 编号。
recovery 还要验证 WAL page header、record CRC、continuation、timeline history 和 source state。
如果来自 archive、`pg_wal`、stream 的切换边界不清楚，先跟 `XLogPageRead()`，再看 `ReadRecord()` 对 invalid record 的处理。

## 29. 讨论题

1. 为什么 segment switch 的本质是推进 WAL insert position，而不是“创建一个新文件”？
2. `XLByteToSeg()` 和 `XLByteToPrevSeg()` 只在边界 LSN 上不同，这个差异分别服务“从哪里读”和“刚写完哪里”两个问题，为什么不能混用？
3. 如果 `archive_command` 持续失败，checkpoint 为什么不能因为磁盘压力就直接删除带 `.ready` 的旧 segment？
4. replication slot 的 `restart_lsn` 为什么会推后 WAL 删除边界？`max_slot_wal_keep_size` 到底是在保护 slot，还是在保护磁盘？
5. `wal_keep_size`、slot、archive status 和 checkpoint redo pointer 同时存在时，哪个边界最老，为什么就必须保留到哪里？
6. archive recovery 读不到 archive 中的下一段 WAL 时，什么时候可以 fallback 到 `pg_wal` 或等待 stream，什么时候意味着恢复结束或错误？
7. timeline history 为什么也是 recovery contract 的一部分，而不只是文件名装饰？
8. 本节的可迁移规律是什么：文件生命周期什么时候不能由“文件是否存在”决定，而必须由多方消费进度共同决定？

## 30. 调试断点建议

优先断这些函数：

```text
RequestXLogSwitch
ReserveXLogSwitch
CopyXLogRecordToWAL
XLogWrite
XLogFileInitInternal
InstallXLogFileSegment
PreallocXlogFiles
RemoveOldXlogFiles
RemoveXlogFile
XLogArchiveNotify
XLogArchiveCheckDone
KeepLogSeg
ReadRecord
XLogPageRead
WaitForWALToBecomeAvailable
XLogFileReadAnyTLI
readTimeLineHistory
writeTimeLineHistory
```

建议先跟 switch。
再跟 checkpoint cleanup。
最后跟 recovery。
不要一开始就在全量 recovery 上打太多断点，WAL replay 调用频率很高。

## 31. 本节小结

WAL segment 是 WAL byte stream 的文件化切片。
`XLogFileName()` 把 timeline 和 segment number 编码成 24 字符文件名。
`XLogSegmentOffset()` 给出 LSN 在 segment 内的 offset。
`XLByteToSeg()` 和 `XLByteToPrevSeg()` 只在边界 LSN 上不同。
这个差异对应“从哪里读”和“刚写完哪里”两个问题。
segment switch 是 WAL insert position 的推进，不是简单创建新文件。
如果当前位置已在 segment 开头，switch 请求可以不写真实 record。
如果不在开头，switch record 会消耗当前 segment 剩余空间。
`XLogWrite()` 是完整 segment 的完成点。
它在完成 segment 时 fsync、唤醒 walsender、通知 archiver，并可能请求 checkpoint。
WAL 文件可能被创建、预创建、回收或删除。
这些动作都不等于 WAL record 语义本身。
checkpoint 清理旧 WAL 时，先从 redo pointer 出发，再被 replication slot、`wal_keep_size`、summarization 和 `max_slot_wal_keep_size` 调整。
最后还要通过 archive gate。
`.ready` 表示等待 archiver。
`.done` 表示 archiver 成功。
checkpoint 看到 `.done` 后才可以删除或回收对应旧 WAL。
recycle 是把旧文件重命名成未来 segment。
remove 才是真删除。
两者成功后都会清理 archive status。
恢复侧不信任文件名本身。
它按 checkpoint redo、recovery target、timeline history、page header、record 校验和 WAL 来源状态机决定能否继续读。
archive recovery 通常优先 archive。
`pg_wal` 和 stream 是不同状态下的 fallback 或等待来源。
timeline history 文件本身也要归档。
promotion 后的新 timeline 依赖 history 文件让后续恢复判断哪些 WAL segment 合法。
本节最重要的一句话：

```text
WAL segment 的正确性不在单个文件名里，而在 byte stream 位置、timeline、
flush/archive 状态、checkpoint 保留边界和 recovery 验证共同形成的边界里。
```
