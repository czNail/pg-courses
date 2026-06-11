# PostgreSQL REDO record 回放与一致性边界

## 课程定位
前置知识：已经理解 startup process 会在 crash recovery、archive recovery、streaming recovery 和 Hot Standby 之间推进；知道 WAL record 描述持久化修改，checkpoint 只是恢复边界的一部分，不是“所有页面都静止”的瞬间。
本节唯一主问题：
```text
startup process 如何按 LSN 顺序应用 WAL record，full-page image、
resource manager redo routine 和 checkpoint redo pointer
如何保证数据页回到一致状态？
```
核心矛盾：数据页可以在任意时刻被写出，也可能在 crash 时 torn；recovery 只能从 checkpoint 记录的某个 redo pointer 开始，顺序读取 WAL，并根据每页自己的 page LSN 判断是否还需要重做。
学完后应能判断：为什么 recovery 起点是 `checkPoint.redo` 而不是 checkpoint record 后；为什么 `ReadRecPtr` 是 record 起点而 `EndRecPtr` 是页面应用后的进度；为什么 FPI 主要覆盖 checkpoint 后首次修改页面的前态风险；为什么 rmgr redo routine 必须尊重 `BLK_RESTORED`、`BLK_DONE`、`BLK_NEEDS_REDO`；为什么 consistent state 是实例级门槛，不是单页 redo 成功。
本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e90741e12a3715909e1b2d71ff9344`。

## 1. 本节在总主线中的位置
上一节讲 startup process 如何选择恢复模式和推进 recovery 状态。本节只跟一条主链路：
```text
checkpoint record
  -> checkPoint.redo / RedoStartLSN
  -> ReadRecord() / XLogReader
  -> ApplyWalRecord()
  -> xlogrecovery_redo()
  -> GetRmgr(record->xl_rmid).rm_redo()
  -> XLogReadBufferForRedoExtended()
  -> RestoreBlockImage 或 page LSN delta redo
  -> PageSetLSN(page, record->EndRecPtr)
  -> lastReplayedEndRecPtr
  -> CheckRecoveryConsistency()
```
下一节会讲 Hot Standby snapshot；那里关心 standby 只读查询如何获得 MVCC 可见性。本节先回答更底层的问题：页面和控制状态如何被 WAL replay 推回一个可解释的物理一致边界。
这节课不展开 WAL insert lock、WAL buffer、archive command 的实现细节，也不把 logical decoding 混进来。logical decoding 会消费 WAL，但本节讲的是 startup process 的物理 REDO。

## 2. 核心矛盾与一句话运行模型
一句话模型：startup process 从 checkpoint record 的 `checkPoint.redo` 开始顺序读 WAL；`XLogReader` 校验并解码每条 record，给出 `ReadRecPtr` 和 `EndRecPtr`；`PerformWalRecovery()` 对每条 record 调 `ApplyWalRecord()`；`ApplyWalRecord()` 先更新正在 replay 的 `replayEndRecPtr`，再处理 recovery 专用 XLOG record，最后按 `record->xl_rmid` 分派到具体 rmgr redo routine；redo routine 对每个 block 先通过 `XLogReadBufferForRedoExtended()` 判断 FPI/page LSN，只有需要时才修改页面，并把 page LSN 推到 `record->EndRecPtr`。
这条链路的 tension 是：
```text
WAL 是全局线性顺序
  vs
数据页是分散落盘、各自带 page LSN、可能 torn 的局部对象。
```
PostgreSQL 没有用一个字段解决这个矛盾，而是把正确性拆成六层：`checkPoint.redo` 定义 replay horizon；`ReadRecord()` 和 `XLogReader` 保证 record 顺序和格式完整性；FPI 覆盖 checkpoint 后首次修改页面的 torn-page 前态风险；page LSN 提供幂等 replay；rmgr redo routine 保留 access method 和控制状态语义；`CheckRecoveryConsistency()` 决定实例何时可被只读解释。
所以 recovery 不是“把 WAL 重新执行一遍”这么简单。更准确的说法是：从一个可证明足够早的 WAL LSN 开始，按 record 顺序推进全局历史；对每个 block 用 FPI/page LSN 判断局部页面是否需要变化；再由 rmgr routine 执行具体语义。

## 3. 核心文件分工与阅读顺序
| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/access/transam/xlogrecovery.c` | checkpoint 选择、`PerformWalRecovery()`、`ReadRecord()`、`ApplyWalRecord()`、`xlogrecovery_redo()`、consistent state。 |
| 2 | `src/backend/access/transam/xlogreader.c` | `XLogReadRecord()`、跨页 record 组装、page/record 校验、`DecodeXLogRecord()`、`RestoreBlockImage()`。 |
| 3 | `src/include/access/xlogreader.h` | `XLogReaderState`、`ReadRecPtr`、`EndRecPtr`、decoded block accessor。 |
| 4 | `src/backend/access/transam/xlogutils.c` | `XLogReadBufferForRedoExtended()`、`BLK_*` action、FPI restore、page LSN 比较。 |
| 5 | `src/backend/access/transam/xloginsert.c` | WAL 插入端如何用 `RedoRecPtr` 和 page LSN 决定是否包含 FPI。 |
| 6 | `src/backend/access/transam/xlog.c` | checkpoint 生成、`GetRedoRecPtr()`、`XLogFlush()`、`xlog_redo()`。 |
| 7 | `src/backend/access/transam/rmgr.c` | `RmgrTable`、`RmgrStartup()`、`RmgrCleanup()`。 |
| 8 | `src/include/access/rmgrlist.h` | `xl_rmid` 到 redo routine 的稳定映射。 |
| 9 | `src/backend/storage/buffer/bufmgr.c` | `FlushBuffer()` 中 WAL-before-data。 |
| 10 | `src/backend/access/heap/heapam_xlog.c` | `heap_redo()`、`heap2_redo()` 的页面 redo 模式。 |
| 11 | `src/backend/access/nbtree/nbtxlog.c` | `btree_redo()` 的多页结构恢复。 |
推荐阅读路径不是按文件名，而是按状态推进：先读 `PerformWalRecovery()`，再读 `ReadRecord()` 和 `ApplyWalRecord()`，然后进入 `XLogReadBufferForRedoExtended()`，最后回到 `XLogRecordAssemble()` 和 `CreateCheckPoint()` 理解 FPI 和 redo pointer 的来源。
这个顺序能避免把 FPI、checkpoint redo pointer、page LSN 和 rmgr dispatch 当成四个互不相干的概念。它们都服务同一个目标：从安全 horizon 开始，把每个页面按 WAL 顺序推进到正确 LSN。

## 4. checkpoint redo pointer
checkpoint record 里的 `checkPoint.redo` 是 recovery 的 replay 起点。它不是 checkpoint record 自己的 LSN。
online checkpoint 的真实过程是：checkpoint 开始时确定 redo pointer；checkpointer 慢慢刷 dirty buffers；并发 backend 继续插入 WAL；checkpoint 完成时写 checkpoint record；checkpoint record 保存早先确定的 `checkPoint.redo`。
因此 recovery 必须从 `checkPoint.redo` 开始。如果从 checkpoint record 后开始，就可能漏掉 checkpoint 期间并发修改过、但数据页未完整落盘的页面变化。
shutdown checkpoint 是例外场景。因为没有并发 WAL 插入，redo pointer 可以是下一条 WAL record 的位置，checkpoint record 本身也能作为边界。
`xlog.c` 的 `CreateCheckPoint()` 对 online checkpoint 会先写 `RM_XLOG_ID + XLOG_CHECKPOINT_REDO`，这条 record 的位置成为新的 redo pointer。随后 checkpoint 刷页并写最终 checkpoint record。源码还会更新 `RedoRecPtr`、`XLogCtl->Insert.RedoRecPtr`、`XLogCtl->RedoRecPtr` 和 `checkPoint.redo`。
这里有一个保守性取舍：如果 checkpoint 最终没完成，而 `RedoRecPtr` 已经向前推进，最坏结果是后续 `XLogInsert` 多写一些 full-page image。多写 FPI 是成本问题；少写必要 FPI 是 correctness 问题。
恢复端会验证 checkpoint record 和 redo pointer。如果 `checkPoint.redo < CheckPointLoc`，startup process 会从 redo pointer 读一条 record，并要求它是 `XLOG_CHECKPOINT_REDO`。如果找不到，或者类型不对，系统拒绝继续恢复。
典型报错包括 `could not locate required checkpoint record`、`invalid checkpoint record`、`could not find redo location ... referenced by checkpoint record`、`unexpected record type found at redo point`。这些错误都发生在正常 redo loop 之前，说明 replay horizon 自身不可信。
redo pointer 只说明“从哪里开始 replay 足够”，不保证对应 WAL 一定还在本地 `pg_wal`、archive 或上游 primary。缺 WAL 时，recovery 仍要依赖 `WaitForWALToBecomeAvailable()`、archive recovery、walreceiver streaming 和 timeline history 检查。

## 5. `ReadRecPtr`、`EndRecPtr` 与 decoded record
`XLogReaderState` 中，`ReadRecPtr` 是当前返回 record 的起始 LSN，`EndRecPtr` 是当前返回 record 的结束后一字节 LSN。`XLogNextRecord()` 从 decoded queue 取出 record 后设置：
```text
state->ReadRecPtr = state->record->lsn
state->EndRecPtr = state->record->next_lsn
```
redo routine 通常取 `XLogRecPtr lsn = record->EndRecPtr`，然后在修改页面后执行 `PageSetLSN(page, lsn)`。原因是页面应用完当前 record 后，语义上已经包含当前 record 的效果；这个完成边界是 `EndRecPtr`，不是 `ReadRecPtr`。
`ReadRecPtr` 更适合诊断。`rm_redo_error_callback()` 报出的 `WAL redo at <LSN> for ...` 指向的就是出问题 record 的起始位置。用 `pg_waldump -s <LSN>` 对齐时，也通常从 `ReadRecPtr` 开始。
`XLogReader` 不只是返回固定 header。`DecodeXLogRecord()` 会把 record 解成 main data、block references、block data、optional full-page image、origin、top-level xid 等 decoded representation。rmgr redo routine 通过 accessor 读这些内容：
```text
XLogRecGetData(record)
XLogRecGetBlockData(record, block_id, &len)
XLogRecGetBlockTag(record, block_id, ...)
XLogRecHasBlockImage(record, block_id)
XLogRecBlockImageApply(record, block_id)
```
这条边界很重要。`XLogReader` 负责 WAL record 的格式完整性，rmgr 负责业务语义。reader 通过 CRC 不代表 heap offset、btree sibling link 或 visibility map 状态一定满足 rmgr 预期。
`XLogReadRecord()` 返回的指针也有生命周期边界：它指向 reader 内部 buffer，只保证到下一次 `XLogReadRecord()` 前有效。rmgr routine 必须在当前调用内消费 decoded data，不能把 `XLogRecGetBlockData()` 返回的指针保存到更长生命周期。

## 6. `ReadRecord()` 与 `XLogReader` 主流程
`ReadRecord()` 是 `xlogrecovery.c` 的 recovery wrapper。它把 `fetching_ckpt`、`emode`、`randAccess`、`replayTLI` 传给 page read callback，然后循环调用 `XLogPrefetcherReadRecord(xlogprefetcher, &errormsg)`。
读到 record 就返回。读不到时，要根据恢复模式处理：crash recovery 可能到本地 WAL 末尾就结束或报错；archive recovery 可以从 `pg_wal` 切到 archive source；standby mode 如果没有 promotion trigger，会继续等待或重试。
所以 standby apply LSN 不动时，不能马上归因到 redo routine 慢。startup process 可能仍在 `ReadRecord()`/`XLogPageRead()` 等下一段 WAL，或在 restore archive，或在等 walreceiver flush streaming WAL。
`XLogReadRecord()` 和 `XLogDecodeNextRecord()` 负责从 `NextRecPtr` 继续读下一条 record，跨 WAL page 组装 continuation record，校验 WAL page header、record header、record CRC，处理 `XLOG_SWITCH` 等特殊边界，最后调 `DecodeXLogRecord()` 生成 decoded record。
reader 层常见错误包括 `invalid magic number`、`invalid info bits`、`unexpected pageaddr`、`out-of-sequence timeline ID`、`incorrect resource manager data checksum`、`invalid contrecord length`、`record with invalid length`。
这些错误发生在 rmgr redo 前，含义是系统还没有资格解释这条 record 的 heap/btree/xact 语义。standby mode 下可能换 WAL 来源或继续等；如果已确定 required WAL 损坏或缺失，recovery 不能跳过坏 record 继续，因为后续字节没有可信 record boundary。

## 7. `PerformWalRecovery()` 主循环
`PerformWalRecovery()` 开始时先初始化 `XLogRecoveryCtl` 里的 progress。如果 redo start 早于 checkpoint record，`lastReplayedReadRecPtr = InvalidXLogRecPtr`，`lastReplayedEndRecPtr = RedoStartLSN`，`lastReplayedTLI = RedoStartTLI`。否则使用当前 reader 已读 checkpoint record 的位置初始化。
随后设置 `replayEndRecPtr = lastReplayedEndRecPtr` 和 `replayEndTLI = lastReplayedTLI`。这表示 recovery 站在 redo 起点之前，下一条成功 replay 的 record 会推进 `lastReplayedEndRecPtr`。
主循环形态可以压缩成：
```text
do
{
    ProcessStartupProcInterrupts();
    recovery pause / target / delay checks;
    ApplyWalRecord(xlogreader, record, &replayTLI);
    WaitLSNWakeup(... lastReplayedEndRecPtr ...);
    record = ReadRecord(...);
} while (record != NULL);
```
顺序点是：先判断是否应该在当前 record 前停止，再 replay 当前 record，再判断是否应该在当前 record 后停止，最后读下一条 record。PITR 的 before/after target 语义就是围绕 record 粒度实现的。
如果 checkpoint 之后没有 WAL，日志会显示 `redo is not required`。如果有 WAL，redo loop 前会调用 `RmgrStartup()`，结束后调用 `RmgrCleanup()`。某些 rmgr 可以在 replay 期间建立或清理临时状态，但这不替代每条 record 的 buffer release。

## 8. `ApplyWalRecord()` 的固定顺序
`ApplyWalRecord()` 是单条 record 的状态推进器。它先安装 `rm_redo_error_callback`，让 rmgr redo 内部报错时日志带上 `WAL redo at <ReadRecPtr> for <rmgr-specific description>`。
然后它执行 `AdvanceNextFullTransactionIdPastXid(record->xl_xid)`，保证 recovery 中看到的 xid 不会让本地 `nextXid` 落后。
接着检查 timeline switch record。对 `XLOG_CHECKPOINT_SHUTDOWN` 和 `XLOG_END_OF_RECOVERY`，如果当前 record 会切换 timeline，`replayTLI` 必须在 redo 前更新。原因是如果 recovery 在当前 record 后停止，`minRecoveryPoint` 的 timeline 必须已经指向新 timeline。
真正调用 rmgr 前，`ApplyWalRecord()` 先写 `XLogRecoveryCtl->replayEndRecPtr = xlogreader->EndRecPtr` 和 `replayEndTLI`。源码注释说明这是为了让 `XLogFlush` 正确更新 `minRecoveryPoint`。但此时还不能更新 `lastReplayedEndRecPtr`，因为当前 record 还没有成功 replay。
这里要分清两种进度：`replayEndRecPtr` 是当前正在 replay 或即将完成的位置；`lastReplayedEndRecPtr` 是已经成功 replay 完成的位置。诊断 standby apply progress 时，后者才表示“已完成”。
如果 record 是 `RM_XLOG_ID`，`ApplyWalRecord()` 先调用 `xlogrecovery_redo(xlogreader, replayTLI)`。当前源码中它处理 `XLOG_OVERWRITE_CONTRECORD` 和 `XLOG_BACKUP_END`。前者验证并跳过被覆盖的 continuation record；后者在匹配 `backupStartPoint` 时把 `backupEndPoint` 设成当前 `EndRecPtr`。
这不是替代 `xlog_redo()`。后面仍然执行 `GetRmgr(RM_XLOG_ID).rm_redo(xlogreader)`，也就是 `xlog_redo()`。边界是：`xlogrecovery_redo()` 处理 startup process 恢复状态机需要的特殊语义；`xlog_redo()` 处理 XLOG rmgr record 自己的 redo 语义，例如 FPI、FPW change、parameter change、checkpoint redo record。
rmgr redo 成功后，`ApplyWalRecord()` 才写 `lastReplayedReadRecPtr`、`lastReplayedEndRecPtr`、`lastReplayedTLI`。之后它唤醒等待 standby replay/write/flush LSN 的进程，必要时唤醒 walsender/walreceiver，最后调用 `CheckRecoveryConsistency()`。

## 9. resource manager redo dispatch
`rmgrlist.h` 定义 WAL record 中 `xl_rmid` 到 rmgr 的稳定映射。典型映射包括 `RM_XLOG_ID -> xlog_redo`、`RM_XACT_ID -> xact_redo`、`RM_SMGR_ID -> smgr_redo`、`RM_HEAP2_ID -> heap2_redo`、`RM_HEAP_ID -> heap_redo`、`RM_BTREE_ID -> btree_redo`。
`rmgr.c` 用 `PG_RMGR` 宏展开成 `RmgrData RmgrTable[RM_MAX_ID + 1]`。恢复端最终执行：
```text
GetRmgr(record->xl_rmid).rm_redo(xlogreader)
```
这不是装饰性 callback，而是语义边界。`XLogReader` 只知道 record 怎么解码；heap、btree、xact、smgr 才知道 record 应该如何改变页面或控制状态。
heap 路径中，`heap_redo()` 按 `XLOG_HEAP_OPMASK` 分派到 `heap_xlog_insert()`、`heap_xlog_update()`、`heap_xlog_delete()` 等函数；`heap2_redo()` 处理 prune/freeze、multi-insert、lock-updated、logical rewrite 等 record。
btree 路径中，`btree_redo()` 分派到 `btree_xlog_insert()`、`btree_xlog_split()`、`btree_xlog_vacuum()`、`btree_xlog_unlink_page()` 等函数。btree 关心 downlink、high key、posting list、incomplete split flag、sibling link、deleted/half-dead page 等结构状态。
不同 rmgr 的业务语义完全不同，但页面边界一致：先用 `XLogReadBufferForRedo*()` 判断该 block 是否要动；只有 `BLK_NEEDS_REDO` 时才应用 delta；修改后 `PageSetLSN(page, record->EndRecPtr)`，再 `MarkBufferDirty(buffer)` 和 `UnlockReleaseBuffer(buffer)`。

## 10. full-page image 的插入端判断
FPI 不是 recovery 端凭空产生的。它来自 WAL 插入端。`xloginsert.c` 的 `XLogRecordAssemble()` 对每个 registered buffer 做类似判断：
```text
if REGBUF_FORCE_IMAGE:
    needs_backup = true
else if REGBUF_NO_IMAGE:
    needs_backup = false
else if !doPageWrites:
    needs_backup = false
else:
    needs_backup = (PageGetLSN(page) <= RedoRecPtr)
```
`doPageWrites` 来自 `fullPageWrites || runningBackups > 0`。`RedoRecPtr` 是 checkpoint redo pointer 的本地副本。判断含义是：如果页面的 page LSN 还不晚于 redo pointer，这个页面在当前 checkpoint 周期里尚未被安全地完整记录过；当前 record 必须带 full-page image。
为什么是 checkpoint 后第一次修改关键？假设 checkpoint redo pointer 是 R，页面 P 的 page LSN 仍然是 R 之前。backend 第一次修改 P 并生成 record W。如果 W 不带 FPI，crash 后磁盘上的 P 可能不是 checkpoint 完整写出的版本，也可能是 torn page；redo 从 R 开始时，W 的 delta 找不到可靠前态。W 带 FPI 后，恢复可以直接把 P 恢复成 WAL 里的整页镜像，然后把 page LSN 设成 `W.EndRecPtr`。
一旦页面被某条 record 修改并设置 page LSN 到更晚位置，后续同页 record 通常只需要 delta。FPI 解决第一步的页面完整性，record 顺序解决后续 delta 的前态依赖。
FPI 的成本和 checkpoint 策略直接相关。checkpoint 越频繁，页面越常在 checkpoint 后再次成为“第一次修改”；working set 越大，FPI 涉及页面越多；running backup 和 wal consistency checking 也会增加 FPI。`wal_compression` 可以减少字节量，但会把部分成本转成 CPU。

## 11. FPI 与 page LSN 的恢复端判断
`XLogReadBufferForRedoExtended()` 是页面 redo 的核心边界。它先解析 block tag，检查 `WILL_INIT` 与 read mode 是否一致，然后优先处理 FPI：
```text
if XLogRecBlockImageApply(record, block_id):
    read/init buffer
    RestoreBlockImage(record, block_id, page)
    if !PageIsNew(page):
        PageSetLSN(page, record->EndRecPtr)
    MarkBufferDirty(buffer)
    return BLK_RESTORED
else:
    read buffer
    lock buffer
    if record->EndRecPtr <= PageGetLSN(page):
        return BLK_DONE
    else:
        return BLK_NEEDS_REDO
```
FPI 优先于 page LSN delta 判断。只要 record 明确要求 apply image，恢复端先建立完整页面。这就是 torn-page 防护需要的强动作。
`RestoreBlockImage()` 在 `xlogreader.c`。它检查 `block_id` 是否有效、该 block 是否真的有 image、压缩方式是否受当前 build 支持、解压是否成功、hole offset/length 是否合理。没有 hole 时直接复制 BLCKSZ；有 hole 时复制前半段、把 hole 清零、再复制后半段。hole 是 page layout 优化，不改变“恢复结果是一整页”的语义。
`BLK_DONE` 是幂等边界。recovery 可能 replay 过一些 WAL，写出部分 dirty page，然后再次 crash。下一次 recovery 又从旧 checkpoint redo pointer 开始时，某些页面已经包含当前 record 的效果。如果 `PageGetLSN(page) >= record->EndRecPtr`，rmgr 必须跳过这个 block，否则会重复插入 tuple、重复删除 item、重复改 sibling link。
`BLK_NEEDS_REDO` 表示页面存在、已加锁、当前 record 效果尚未体现在 page LSN 中，调用者必须按 rmgr 语义修改页面。它不保证 heap offset 合法、btree opaque 状态符合预期、record data 与页面前态匹配，也不保证辅助结构已经同步。这些由 rmgr routine 自己检查。
`BKPBLOCK_WILL_INIT` 是另一个边界。插入端用 `REGBUF_WILL_INIT` 声明页面会被重新初始化，恢复端要求 `willinit` 与 zero mode 匹配。这样可以防止 record 声明会初始化但 redo routine 按旧页面做 delta，或 redo routine 想初始化页面但 WAL record 没声明该语义。

## 12. heap redo walkthrough
`heap_xlog_insert()` 展示了典型单页 redo 模式。若 record 带 `XLOG_HEAP_INIT_PAGE`，它调用 `XLogInitBufferForRedo(record, 0)`，`PageInit()` 后把 action 当成 `BLK_NEEDS_REDO`。否则调用 `XLogReadBufferForRedo(record, 0, &buffer)`。
只有 `action == BLK_NEEDS_REDO` 时，heap redo 才从 block data 重建 `HeapTupleHeader`，调用 `PageAddItem()`，设置 prunable 信息，执行 `PageSetLSN(page, record->EndRecPtr)` 和 `MarkBufferDirty(buffer)`。如果 block 已经 `BLK_RESTORED` 或 `BLK_DONE`，就不能重复插入 tuple。
`heap_xlog_update()` 更能说明多页 record。它先处理旧 tuple 页面：读 old block，如果 `BLK_NEEDS_REDO`，设置 old tuple 的 xmax/cmax/ctid、HOT 或非 HOT 标记、prunable 信息、page LSN 和 dirty flag。然后处理新 tuple 页面：如果 oldblk == newblk 复用同一个 buffer/action；否则按 `XLOG_HEAP_INIT_PAGE` 或 `XLogReadBufferForRedo()` 取得新页面。只有 new page 需要 redo 时，才从 WAL data 和 old tuple prefix/suffix 重建新 tuple。
一条 heap update record 可以触碰 old page、新 page、visibility map 和 FSM。每个 block 独立用 page LSN/FPI 判断，但 rmgr 必须保持 record 级语义，例如 old/new tuple chain。
heap redo 还提醒我们：`BLK_DONE` 是某个 block reference 的页面判断，不是整条 record 所有语义都完成。某些 record 中，即使 heap page 自己已经 done，仍可能需要处理 `visibilitymap_clear()` 或 `XLogRecordPageWithFreeSpace()`。

## 13. btree redo walkthrough
`btree_xlog_insert()` 典型结构是：如果是 internal insert，先 `_bt_clear_incomplete_split(record, child block)`；然后对主 block 调 `XLogReadBufferForRedo(record, 0, &buffer)`；只有返回 `BLK_NEEDS_REDO` 时，才用 `PageAddItem()` 插入 index tuple，设置 page LSN，标记 dirty。
btree record 的语义和 heap 不同。它关心 downlink、high key、posting list、incomplete split flag、sibling link、deleted page 和 half-dead page。但页面一致性边界仍然相同：`XLogReadBufferForRedo*()` 判断是否要动页面，btree redo routine 决定怎么动页面。
btree split、unlink page、mark half-dead page 都可能修改多个页面。正常执行时需要复杂锁耦合；redo 时没有并发 writer，但 Hot Standby reader 可能观察索引结构。因此 redo 不必完全复刻正常路径锁顺序，却仍要避免暴露不一致结构。
例如 `btree_xlog_unlink_page()` 会按 left sibling、target page、right sibling 的结构顺序修复链接。这里体现的是：LSN 顺序提供 record 间顺序；rmgr routine 仍要负责 record 内多页结构的语义顺序。
btree redo 内部如果看到页面状态不符合 record 预期，会 `elog(PANIC, ...)`。这类错误通常不是普通 SQL 错误，而是 WAL、数据页、checkpoint 起点或存储层之间的正确性假设被破坏。

## 14. WAL-before-data 与 page LSN
`bufmgr.c` 的 `FlushBuffer()` 在写数据页前取 `recptr = BufferGetLSN(buf)`，然后对 permanent buffer 执行 `XLogFlush(recptr)`。源码注释说明：描述数据页修改的 log updates 必须先于数据文件修改落盘。
没有 WAL-before-data，crash 后可能出现数据页已经包含某个修改，但对应 WAL record 没有 durable。recovery 看到 page LSN 后可能跳过 record，却无法解释页面状态来源。
redo 中的判断 `record->EndRecPtr <= PageGetLSN(page)` 隐含前提是：如果磁盘页带着 page LSN L，那么描述页面到 L 的 WAL 已经先于页面 durable。这个前提由 `FlushBuffer()` 的 `XLogFlush(page LSN)` 建立。
所以写出端和恢复端要成对理解：
```text
WAL-before-data:
  写数据页前建立 durable 顺序。

page LSN check:
  recovery 时消费这个 durable 顺序。
```
unlogged relation 是例外。`FlushBuffer()` 对非 permanent buffer 不按 page LSN flush WAL，因为 unlogged relation crash 后会丢失，某些 unlogged index page 的 LSN 还是 fake LSN，不能拿 fake LSN 去推动真实 WAL flush。

## 15. consistent state
`CheckRecoveryConsistency()` 判断的是实例级边界，不是单页是否已恢复。它检查是否达到 `minRecoveryPoint`，是否达到 `backupEndPoint` 或看到对应 backup end record，是否还有 unresolved invalid pages，tablespace directory 状态是否合理，Hot Standby snapshot 是否 ready。
达到一致时会设置 `reachedConsistency = true`，发送 `PMSIGNAL_RECOVERY_CONSISTENT`，日志出现：
```text
consistent recovery state reached at <LSN>
```
源码注释指出，crash recovery 中不会在中途宣布 reachedConsistency；archive recovery 中，越过 `minRecoveryPoint` 后可以一致。base backup 场景还需要处理 `backupStartPoint`、`backupEndPoint`、`backupEndRequired`。
`XLOG_BACKUP_END` 被 `xlogrecovery_redo()` 看到后，可能把 `backupEndPoint` 设成当前 record 的 `EndRecPtr`。下一次 `CheckRecoveryConsistency()` 再决定是否完成 backup recovery。
recovery 中还可能暂时记录 invalid pages。例如 WAL 引用的页面当下不可读，之后的 drop/truncate/create record 可能解释或消除它。到达 consistent state 前必须执行 `XLogCheckInvalidPages()`。这说明 PostgreSQL 允许恢复过程中存在短暂不完整信息，但不允许在对外宣布 consistent state 时留下 unresolved invalid page。

## 16. 生命周期与 cleanup
recovery 期间只有 startup process 执行 WAL redo。它持有 `xlogreader`、`xlogprefetcher`、WAL 来源状态、timeline 状态、`XLogRecoveryCtl` 中的 replay progress，以及 recovery target/pause/delay 状态。
Hot Standby backend 可以读，但不能成为永久数据的本地 writer。这样 startup process 对 redo 顺序有单一 owner，避免多个 backend 并发 replay 同一 WAL stream。
`XLogReadRecord()` 返回的 decoded data 生命周期只到下一次读 record 前。rmgr redo routine 必须立即消费，不能把 record 内部指针挂到长期结构。
`XLogReadBufferForRedoExtended()` 返回有效 buffer 时通常已经持有 buffer lock。rmgr routine 修改页面后必须设置 page LSN、标记 dirty，再 `UnlockReleaseBuffer()`。这涉及 buffer pin、buffer content lock、dirty flag、后续 checkpoint/bgwriter 写回、page checksum 和 WAL-before-data flush，不是普通内存 cleanup。
`RmgrStartup()` 和 `RmgrCleanup()` 给 rmgr replay 期间的临时状态一个生命周期，但不替代每条 record 的 buffer cleanup，也不替代 reader 内部 buffer 的短生命周期。

## 17. 正确性机制层次
| 层次 | 机制 | 保证 | 不保证 |
| --- | --- | --- | --- |
| Durable order | `FlushBuffer()` 中 `XLogFlush(page LSN)` | 数据页落盘前 WAL 先 durable。 | record 语义正确。 |
| Replay 起点 | `checkPoint.redo` | 从足够早的位置开始 replay。 | WAL 一定存在。 |
| Record 顺序 | `ReadRecord()` / `XLogReader` | 按 LSN 读完整 record。 | 页面前态一定符合 rmgr 预期。 |
| Record 完整性 | page header、length、CRC、timeline | 拒绝明显坏 WAL。 | access method invariants 一定成立。 |
| 页面完整性 | FPI / `RestoreBlockImage()` | 修复 checkpoint 后首次修改的 torn-page 前态。 | 后续 delta 自动正确。 |
| 幂等性 | page LSN 与 `EndRecPtr` 比较 | 已应用 record 不重复应用。 | 整条 record 所有副作用都可跳过。 |
| Rmgr 语义 | `heap_redo()`、`btree_redo()` 等 | 把 record 解释成具体状态变化。 | 其它 rmgr 状态自动同步。 |
| 实例一致 | `CheckRecoveryConsistency()` | 到达 minRecoveryPoint/base backup 边界后可一致。 | Hot Standby snapshot 一定 ready。 |
这张表的重点是：raw field 不是语义；field + WAL order + page state + rmgr semantics + lifecycle gate 才是 recovery correctness。

## 18. 错误路径与异常边界
checkpoint 起点错误通常表现为 `could not locate required checkpoint record`、`invalid checkpoint record`、`could not find redo location ... referenced by checkpoint record`、`unexpected record type found at redo point`。含义是 recovery horizon 不可信，不能继续解释 WAL 和页面。
WAL page 或 record 坏通常来自 reader 层，例如 `invalid magic number`、`unexpected pageaddr`、`out-of-sequence timeline ID`、`incorrect resource manager data checksum`、`record with invalid length`。standby mode 下可能换来源或继续等待；非 standby 或 required WAL 场景下会停止。坏 record 后面的字节没有可信边界。
FPI 恢复失败可能来自 block_id 无效、record 没有 image、压缩方法不被当前 build 支持、解压失败、hole 信息不合法。它说明 WAL 外层可能通过 CRC，但 block image 仍不可被当前 recovery 解释成一页。
rmgr redo 内部 PANIC 通常来自更深语义错误，例如 `invalid max offset number`、`offnum out of range`、`invalid lp`、`failed to add tuple`、`failed to add new item`、`unknown op code`。诊断入口是错误上下文中的 `WAL redo at <ReadRecPtr> for <description>`。
如果 PITR target 在 consistent state 前，会报 `requested recovery stop point is before consistent recovery point`。这说明你要求停在一个实例整体还不可解释的位置，即使某些页面已经 replay 到目标附近，也不能开放数据库。
`full_page_writes=off` 的备份风险也属于本节边界。`xlog.c` 对 standby backup 有错误提示 `WAL generated with "full_page_writes=off" was replayed since last restartpoint`，并要求启用 full_page_writes 后在 primary 上跑 CHECKPOINT。这是因为缺少必要 FPI 时，base backup 区间内 torn-page 风险无法被 delta redo 可靠修复。

## 19. 成本、资源与跨模块传播
WAL 读取成本随 WAL 量扩张。主要路径是 `XLogPageRead()`、`WaitForWALToBecomeAvailable()`、`pg_pread()`、`XLogReaderValidatePageHeader()`、`XLogDecodeNextRecord()`。成本来源包括顺序 WAL I/O、archive restore latency、streaming wait、timeline history lookup、跨页 record 组装、CRC 校验和 decode buffer 分配。
FPI 会放大 WAL，也会影响 standby redo。checkpoint 过频繁、working set 大、running backup、wal consistency checking 都可能增加 FPI。`wal_compression` 降低 WAL 字节量，但增加压缩/解压 CPU。
页面 redo 成本来自 buffer lookup、relation fork read I/O、buffer eviction、buffer lock、FPI 解压、`MarkBufferDirty()` 和后续写回压力。同样大小的 WAL，顺序 heap insert、随机 btree split、vacuum cleanup、checkpoint 后 FPI burst 的 redo 成本可能完全不同。
跨模块传播路径是：checkpoint 生成 redo pointer；WAL insertion 决定 FPI；buffer manager 保证 WAL-before-data；XLogReader 保证 record 顺序和完整性；rmgr redo routine 恢复具体语义；walreceiver/archive 提供 WAL；Hot Standby consistency gate 决定只读连接。
因此 standby lag 不是单点问题。它可能来自 WAL 还没收到、archive/streaming source 切换慢、reader 校验失败、redo I/O 慢、FPI 解压重、rmgr 处理复杂结构，或 consistent state 等待 backup/minRecoveryPoint/invalid pages。

## 20. 观测与诊断入口
server log 是第一入口。正常进度包括 `redo starts at <LSN>`、`redo in progress ... current LSN: <LSN>`、`redo done at <LSN> system usage: ...`、`consistent recovery state reached at <LSN>`、`completed backup recovery with redo LSN ... and end LSN ...`。
错误诊断先分层：checkpoint 起点错误看 checkpoint record、pg_control、redo pointer；reader 校验错误看 page header、CRC、timeline、archive/streaming 来源；rmgr redo 错误看 `WAL redo at <LSN> for ...` 并进入具体 rmgr routine。
`pg_waldump` 用来对齐 record：
```bash
pg_waldump -p "$PGDATA/pg_wal" -s 0/LSN -n 20
pg_waldump -p "$PGDATA/pg_wal" --rmgr=Heap -s 0/LSN -n 50
pg_waldump -p "$PGDATA/pg_wal" --rmgr=Btree -s 0/LSN -n 50
pg_waldump -p "$PGDATA/pg_wal" --rmgr=XLOG -s 0/LSN -n 50
```
重点看 record 起始 LSN、rmgr、description、block references、是否有 FPI、`prev` 链。`pg_waldump` 能解释 WAL record，但不能直接告诉你数据页当前 page LSN，也不能告诉你某个 block 返回 `BLK_DONE` 还是 `BLK_NEEDS_REDO`。
`pg_controldata` 辅助看 `Latest checkpoint location`、`Latest checkpoint's REDO location`、`Minimum recovery ending location`、`Backup start location`、`Backup end location`。这些对应 `CheckPointLoc`、`checkPoint.redo`、`minRecoveryPoint`、`backupStartPoint`、`backupEndPoint`。
standby 可连接后，SQL 入口包括：
```sql
SELECT pg_is_in_recovery();
SELECT pg_last_wal_replay_lsn();
SELECT pg_last_xact_replay_timestamp();
```
它们能看到最后成功 replay 的 LSN 和最后事务 replay 时间，但看不到当前正在处理的 record、某个 block 的 redo action、是否刚恢复 FPI、是否正在等待下一段 WAL。
gdb 断点建议：
```gdb
break PerformWalRecovery
break ReadRecord
break ApplyWalRecord
break xlogrecovery_redo
break XLogReadBufferForRedoExtended
break RestoreBlockImage
break heap_redo
break btree_redo
break xlog_redo
```
在 `ApplyWalRecord()` 看 `xlogreader->ReadRecPtr`、`xlogreader->EndRecPtr`、`record->xl_rmid`、`record->xl_info`。在 `XLogReadBufferForRedoExtended()` 看 `block_id`、`willinit`、`XLogRecBlockImageApply(record, block_id)` 和返回 action。`BLK_DONE` 追 page LSN；`BLK_RESTORED` 追 `RestoreBlockImage()`；`BLK_NEEDS_REDO` 进入具体 rmgr routine。

## 21. 常见误区
误区一：checkpoint record 之后才需要 replay。正确说法是 online checkpoint 的 redo pointer 可能早于 checkpoint record，恢复起点是 `checkPoint.redo`。
误区二：有 FPI 就不需要 rmgr redo routine。正确说法是 FPI 只恢复某个 block 的整页，record 可能还有其它 block、main data、控制状态和辅助结构副作用。
误区三：page LSN 到了某个位置，页面里的 tuple 就都可见。正确说法是 page LSN 是物理 redo 进度，tuple 可见性仍由 MVCC、pg_xact、snapshot、hint bit 等决定。
误区四：某个 block 返回 `BLK_DONE`，整条 record 就可以跳过。正确说法是 `BLK_DONE` 是单个 block reference 的判断，record 级语义仍可能要处理其它 block 或控制状态。
误区五：standby `pg_last_wal_replay_lsn()` 不动就是 redo CPU 慢。正确说法是可能缺 WAL、等 archive、等 streaming、recovery pause、apply delay、FPI 解压、random I/O、rmgr 结构复杂或 Hot Standby conflict。

## 22. 课堂实验
实验一：源码跟读一条 record 如何推进 LSN。断点设在 `PerformWalRecovery()`、`ReadRecord()`、`ApplyWalRecord()`、`XLogReadBufferForRedoExtended()`。观察 `xlogreader->ReadRecPtr`、`xlogreader->EndRecPtr`、`record->xl_rmid`、`record->xl_info`、`XLogRecBlockImageApply(record, block_id)` 和返回的 `XLogRedoAction`。回答：为什么页面设置的是 `EndRecPtr`？`lastReplayedEndRecPtr` 什么时候推进？同一条 record 的不同 block 是否可能返回不同 action？
实验二：用 `pg_waldump` 观察 checkpoint 后 FPI。执行 `CHECKPOINT`，对一张表做少量 UPDATE/INSERT，记录前后 `pg_current_wal_lsn()`，用 `pg_waldump -s 起点 -n 若干条` 查看 Heap/Btree/XLOG record，找 block references 中的 full-page image。回答：为什么 checkpoint 后首次修改更容易看到 FPI？为什么同页后续修改可能只记录 delta？
实验三：临时在 `src/backend/access/transam/xlogutils.c` 的 `XLogReadBufferForRedoExtended()` 打印 `record->ReadRecPtr`、`record->EndRecPtr`、`block_id`、`XLogRecBlockImageApply(record, block_id)`、`PageGetLSN(page)` 和 return action。重复 crash/recovery，观察 `BLK_DONE` 是否增加；在 FPI-heavy workload 下观察 `BLK_RESTORED` 是否集中出现在 checkpoint 后。实验日志不要保留到产品代码。

## 23. 源码深挖：从 checkpoint 到第一条 record
这一段训练的是“不要从 redo loop 中间开始读”。
很多 recovery 误判来自直接盯着 `ApplyWalRecord()`，却没有确认起点是否成立。
建议先在 `xlogrecovery.c` 中沿 checkpoint 选择路径画出四个 LSN：
`CheckPointLoc`。
`RedoStartLSN`。
`xlogreader->ReadRecPtr`。
`xlogreader->EndRecPtr`。
这四个值的相对关系决定了 recovery 的第一步。
如果 `RedoStartLSN < CheckPointLoc`，第一条被读的 record 应该是 `XLOG_CHECKPOINT_REDO`。
这说明 online checkpoint 的 redo horizon 早于最终 checkpoint record。
如果 `RedoStartLSN >= CheckPointLoc`，通常意味着 shutdown checkpoint 或等价边界。
此时 recovery 读的是 checkpoint 后的下一条 record。
这里最容易错的是把 `CheckPointLoc` 当成 replay 起点。
`CheckPointLoc` 是“checkpoint record 在哪里”。
`checkPoint.redo` 是“为了恢复这个 checkpoint，需要从哪里开始重做”。
二者可以相等、接近，也可以明显分离。
源码里会检查 `checkPoint.redo > CheckPointLoc` 的非法情况。
还会在 `checkPoint.redo < CheckPointLoc` 时验证 redo point 的 record 类型。
这两个检查都说明：
recovery 起点不是一个配置猜测。
它是 checkpoint record、WAL record 类型和 timeline 历史共同验证出来的结果。
读第一条 record 时，`ReadRecord()` 的 `emode` 也有语义。
需要的 checkpoint 或 redo marker 读不到时，错误级别更强。
普通后续 record 读不到时，则可能意味着 WAL 正常结束、等待 streaming 或 archive source 切换。
所以读源码时要问：
这次 `ReadRecord()` 是在 fetching checkpoint 吗？
这次 `ReadRecord()` 是随机读还是顺序读？
这次 replayTLI 是 checkpoint TLI 还是 redo start TLI？
是否已经进入 archive recovery？
是否处于 standby mode？
这些变量决定“读不到 WAL”是 fatal、retry、fallback 还是正常结束。
课堂上建议用一张表记录：
场景。
`RedoStartLSN` 与 `CheckPointLoc` 的关系。
第一条 record 的期望 rmgr/info。
失败时的错误。
是否允许 standby retry。
这样比背 `ReadRecord()` 的参数更有用。
这个起点检查也解释了为什么缺 WAL 的报错通常会带两个位置。
一个是 checkpoint record 位置。
一个是 checkpoint record 引用的 redo location。
前者告诉你“谁提出这个需求”。
后者告诉你“真正缺的是从哪里开始的 WAL”。
`pg_controldata` 可以看到最新 checkpoint 和 REDO location。
`pg_waldump` 可以验证该 REDO location 附近是否存在可解释 record。
但如果 WAL 文件已经回收、archive 缺失或 timeline history 不完整，`pg_waldump` 只能帮助定位，不能修复。
这一段的可迁移规律是：
恢复系统必须先证明 replay horizon。
在 horizon 不可信时继续 replay，只会把 corruption 推迟到更难解释的页面错误。

## 24. 源码深挖：reader validation 到 decoded record
`XLogReader` 是 recovery 的格式边界。
它不是 rmgr 的一部分，也不是简单的文件读取循环。
它负责把 WAL 字节流变成“可以交给 rmgr 的 decoded record”。
首先，reader 按 `NextRecPtr` 找下一条 record。
如果起点在 WAL page 开头，需要跳过 variable-size page header。
如果 record 跨页，需要识别 continuation page。
如果看到 `XLP_FIRST_IS_CONTRECORD` 或 `XLP_FIRST_IS_OVERWRITE_CONTRECORD`，还要处理 continuation 语义。
这些逻辑解释了为什么坏 WAL segment 可能表现为 contrecord 错误，而不只是 CRC 错误。
其次，reader 校验 WAL page header。
它会检查 magic number。
检查 info bits。
检查长页头中的 system identifier、segment size、`XLOG_BLCKSZ`。
检查 `xlp_pageaddr` 是否等于期望 LSN。
检查 timeline ID 是否倒退。
这些检查常用于识别 recycled WAL segment、错误 timeline、错误集群的 WAL。
然后，reader 校验 record header 和 CRC。
CRC 覆盖 record body 和 header 中除 crc 字段外的内容。
如果 CRC 不对，错误是 record 层，不是 heap/btree 语义层。
这时不要先去查 heap page。
应该先确认 WAL 来源是否正确、segment 是否完整、archive restore 是否把错误文件放到了正确名字下。
通过这些校验后，`DecodeXLogRecord()` 才解析 block headers。
它检查 block_id 是否递增。
检查 `BKPBLOCK_HAS_DATA` 与 data length 是否一致。
检查 full-page image 的 hole/compression header 是否自洽。
检查 `BKPBLOCK_SAME_REL` 是否有前一个 rel locator 可继承。
最后把 block image、block data 和 main data 拷贝到 decoded record 的连续区域。
rmgr routine 看到的是 decoded accessor，而不是原始 WAL 文件偏移。
因此 gdb 中调试 rmgr 时，不要试图手算 record body layout。
优先用 `XLogRecGetData()`、`XLogRecGetBlockData()`、`XLogRecGetBlockTag()` 这类 accessor。
reader 通过校验以后，record 仍可能在 rmgr 处失败。
这不是矛盾。
reader 只能证明“record 格式上是 WAL record”。
rmgr 还要证明“record 与当前页面前态组合后符合 access method 语义”。
例如 btree record 的 block data 长度正确，不代表目标 page 的 opaque flag 符合 split redo 预期。
heap record 的 tuple bytes 正确，不代表目标 offset 可插入。
这个分层对诊断很重要。
如果日志是 invalid magic、unexpected pageaddr、checksum，先查 WAL 来源。
如果日志是 WAL redo at 某 LSN 后接 rmgr PANIC，先用 `pg_waldump` 看 record，再进具体 rmgr routine。
如果错误只在 standby 出现，还要比较 primary 和 standby 的 WAL segment、timeline history、restore command 结果。
同一段 WAL 在 primary 上能解释，不代表 standby 一定读到了同一段 WAL。
文件名相同但内容来自旧 timeline 或 recycled segment，是 recovery 中非常危险的现场。

## 25. 源码深挖：`XLogReadBufferForRedoExtended()` action
这一节最值得单步的是 `XLogReadBufferForRedoExtended()`。
它把 WAL block reference 转成 buffer action。
第一步是 `XLogRecGetBlockTagExtended()`。
它从 decoded record 中取 relfilocator、fork number、block number 和 prefetch buffer。
如果 caller 给了不存在的 block_id，源码直接 `PANIC`。
这说明 rmgr routine 和 WAL record layout 必须严格匹配。
第二步是 `WILL_INIT` 检查。
如果 WAL block 标记 `BKPBLOCK_WILL_INIT`，redo routine 必须用 zero mode。
如果 redo routine 用 zero mode，WAL block 也必须声明 `WILL_INIT`。
这个双向检查避免“按旧页做 delta”和“偷偷初始化页面”之间的语义错位。
第三步是 FPI 分支。
只要 `XLogRecBlockImageApply()` 为真，恢复端就读或初始化目标 buffer，然后调用 `RestoreBlockImage()`。
恢复完成后，如果 page 不是 new page，就设置 page LSN 到 `record->EndRecPtr`。
然后标记 dirty，返回 `BLK_RESTORED`。
这条路径不会先比较旧 page LSN。
原因是 record 已经声明这个 image 应该 apply。
对于 torn page 风险，最稳妥动作就是用 WAL 中完整镜像覆盖。
第四步是普通页面读取。
如果没有 apply image，函数调用 `XLogReadBufferExtended()` 读目标 block。
如果 buffer 有效，会按需要获取普通 exclusive lock 或 cleanup lock。
然后比较：
`record->EndRecPtr <= PageGetLSN(page)`。
成立则返回 `BLK_DONE`。
不成立则返回 `BLK_NEEDS_REDO`。
如果 buffer 无效，返回 `BLK_NOTFOUND`。
`BLK_NOTFOUND` 的解释取决于 rmgr 和 record 类型。
有些场景下页面后来被 drop/truncate 可以解释。
有些场景下则会进入 invalid page 记录或直接报错。
这就是为什么 `XLogHaveInvalidPages()` 和 `XLogCheckInvalidPages()` 存在。
recovery 允许短时间“我还解释不了这个引用”。
但到 consistent state 时必须解释完。
调试 action 时要同时记录四个值：
`record->ReadRecPtr`。
`record->EndRecPtr`。
`XLogRecBlockImageApply(record, block_id)`。
`PageGetLSN(page)`。
只看 action 不够。
例如 `BLK_DONE` 可能是因为前一次 recovery 已写出更高 page LSN。
`BLK_RESTORED` 可能是正常 checkpoint 后首次修改。
`BLK_NEEDS_REDO` 可能是页面旧，也可能是 zero/init page 场景。
如果你在重复 crash/recovery 后看到大量 `BLK_DONE`，这通常说明幂等机制正在工作。
如果 checkpoint 后立刻看到大量 `BLK_RESTORED`，这通常说明 FPI 正在覆盖 checkpoint 后首次修改页面。
如果没有 FPI 且页面 LSN 很旧，rmgr delta 需要依赖之前 record 已经按顺序 replay。

## 26. 源码深挖：heap 与 btree 的差异
heap redo 和 btree redo 都调用 `XLogReadBufferForRedo*()`。
但它们对 record 的解释完全不同。
heap insert 的中心是 tuple version。
它从 block data 重建 tuple header 和 tuple payload。
它设置 xmin、cmin、ctid、infomask。
它把 tuple 放入指定 offset。
它可能设置 page prunable。
它可能清 all-visible。
它可能更新 FSM。
这些动作围绕 MVCC tuple 语义展开。
btree insert 的中心是 index page 结构。
它可能插入普通 index tuple。
也可能处理 posting list split。
internal insert 还可能清 child page 的 incomplete split flag。
btree split 可能重建 right page。
可能修改 original page。
可能更新 sibling link。
可能更新 metapage。
这些动作围绕搜索树结构和并发 reader 可观察状态展开。
因此不要把 rmgr redo routine 理解成“把 bytes 拷到 page”。
它是在 recovery 环境中重演 access method 需要的结构变化。
recovery 环境和正常执行环境又不同。
正常执行有并发 writer，需要锁耦合、死锁避免、buffer pin 协调。
recovery 中没有并发 writer，但可能有 Hot Standby reader。
因此 redo routine 可以省掉某些正常路径锁耦合，却不能暴露结构不一致。
这解释了 btree redo 中关于 lock coupling 的注释。
也解释了 heap update 中 old/new page 不能随便提前释放的注释。
如果 standby reader 看到 half-applied 状态，可能破坏只读查询。
record 间顺序由 WAL LSN 提供。
record 内多页顺序由 rmgr routine 提供。
page 是否已经应用由 page LSN 提供。
这三层不能互相替代。
如果一个 btree split record 涉及多个 block，其中一个 block `BLK_DONE`，另一个 block `BLK_NEEDS_REDO`，rmgr routine 必须按每个 block 的 action 做局部判断。
但它仍要维持整条 split record 的结构语义。
如果一个 heap update 的 old page done、新 page needs redo，也要避免重复修改 old tuple，同时正确构造 new tuple。
这类混合 action 在重复 recovery、partial page flush 后并不奇怪。
它正是 page-local 幂等和 record-global 语义共同工作的结果。

## 27. 诊断 case study：startup process 卡住
现场一：`pg_last_wal_replay_lsn()` 长时间不动，但没有错误。
第一步不要进 rmgr。
先看 server log 是否有 `redo in progress` 继续打印 current LSN。
如果没有推进，检查 walreceiver 是否还在接收。
检查本地 `pg_wal` 是否有后续 segment。
检查 restore_command 是否在反复失败。
检查是否有 recovery pause 或 recovery apply delay。
这类现场常卡在 `ReadRecord()` 之前。
现场二：日志出现 `invalid magic number` 或 `unexpected pageaddr`。
这通常是 WAL page header 层错误。
优先怀疑读取到了错误 segment、recycled segment、错误 timeline、错误 archive 文件。
用 `pg_waldump` 读同一文件同一 LSN。
对比文件名中的 timeline。
检查 `.history` 文件。
检查 restore_command 是否把旧文件覆盖到新 timeline 名字。
这类现场还没进入 rmgr redo。
现场三：日志有 `WAL redo at 0/XXXX for Heap ...`，随后 PANIC。
这说明 reader 已经把 record 解码到 rmgr 层。
下一步用 `pg_waldump -s 0/XXXX -n 1` 看 record 描述和 block references。
再在 `heap_redo()` 或对应子函数设断点。
打印 `record->EndRecPtr`。
打印 block action。
打印目标 page LSN。
如果 action 是 `BLK_NEEDS_REDO`，继续看 offset、item id、tuple header 是否符合 record 预期。
如果 action 是 `BLK_RESTORED` 却 rmgr 继续应用 delta，要检查 routine 对该 record 类型的分支是否符合预期。
现场四：recovery 到达目标 LSN 后仍不能开放。
看是否报 `requested recovery stop point is before consistent recovery point`。
查 `minRecoveryPoint`。
查 base backup 的 `backupStartPoint`、`backupEndPoint`。
查是否需要 `XLOG_BACKUP_END`。
查 invalid pages。
这类现场不是某个 page 没 redo，而是实例级 consistency gate 未满足。
现场五：standby 备份报 full_page_writes 相关错误。
不要把它当成普通 GUC 提示。
它说明 standby replay 的 WAL 区间里曾经出现 full_page_writes off 的 record。
base backup 需要依赖该区间内的 FPI 保证 torn-page 可恢复。
正确处理是按提示在 primary 启用 full_page_writes 并执行 CHECKPOINT，再重新开始安全备份。
这些 case 的共同点是：
先判断错误层级。
起点层。
reader 层。
rmgr 层。
consistent gate 层。
不要一上来把所有 recovery 失败解释成“WAL 损坏”或“redo 慢”。

## 28. 诊断工具边界
`pg_waldump` 能回答 WAL record 是什么。
它不能回答页面当前是什么。
它也不能判断 redo routine 是否会返回 `BLK_DONE`。
因为 action 取决于数据文件中的 page LSN 和 FPI apply flag。
`pg_controldata` 能回答控制文件里的 checkpoint 和 recovery 边界。
它不能证明 archive 中 WAL 完整。
也不能证明 timeline history 正确。
SQL 函数能回答 standby 已经成功 replay 到哪里。
`pg_last_wal_replay_lsn()` 对应的是已完成的 replay progress。
它看不到正在 replay 的 record。
它也看不到 startup process 是否卡在 WAL 读取、FPI 解压、buffer I/O 或 recovery pause。
server log 能给出 recovery 主线。
但 log granularity 取决于配置和代码路径。
`redo starts at`、`redo done at`、`consistent recovery state reached` 是阶段性 truth。
`WAL redo at` 是 rmgr 错误上下文。
`invalid magic`、`unexpected pageaddr` 是 reader 错误上下文。
gdb 能看到内部状态，但要避免破坏 timing。
在 standby streaming 场景中，断点会改变 walreceiver/startup 交互节奏。
调试时最好先复现到本地 WAL 文件，再在非生产环境单步。
如果必须在线调试，先只断在错误附近。
不要长期停住 startup process。
否则会制造新的 replication lag。
perf 或 flamegraph 适合分析 redo CPU 瓶颈。
它能显示时间花在 FPI 解压、CRC、heap redo、btree redo、buffer lookup 还是 I/O wait 周边。
但 perf 不能告诉你 recovery 起点是否正确。
工具之间要按层组合。
控制文件看 horizon。
`pg_waldump` 看 record。
日志看阶段和错误层。
SQL 看完成进度。
gdb 看内部分支。
perf 看热点。

## 29. 版本与实现边界
本课基于 PostgreSQL master commit `0e1f1ed157e90741e12a3715909e1b2d71ff9344`。
函数名和文件路径按该源码确认。
不同版本中，prefetcher、WAL reader buffer、compression method、checkpoint/recovery 周边细节可能演化。
但几条语义边界稳定：
checkpoint record 携带 redo pointer。
WAL record 有起始 LSN 和结束 LSN。
页面有 page LSN。
WAL-before-data 是 crash recovery 前提。
FPI 保护 checkpoint 后首次修改页面。
rmgr redo routine 解释 record 语义。
consistent state 是实例级 gate。
不要把当前版本的某个 helper 名字误当成系统本质。
例如 `XLogPrefetcherReadRecord()` 是当前读取优化路径的一部分。
如果将来 prefetcher 改名或重构，核心语义仍是 startup process 顺序取得可验证 WAL record。
同样，wal compression 支持的算法可能变化。
但 `RestoreBlockImage()` 必须把 WAL image 转成完整页面的语义不会变。
rmgrlist 的顺序也要谨慎理解。
`rmgrlist.h` 中条目的顺序定义内置 rmgr ID。
WAL record 持久化的是数值 ID。
随意改顺序会破坏 WAL 兼容。
这也是为什么新 rmgr 条目必须按规则追加，而不是为了源码美观重排。

## 30. 状态随时间推进：一页从 primary 到 recovery
把本节所有状态串起来，最好的方式是跟一页走。
设某个 heap page 的 page LSN 是 `P0`。
一次 online checkpoint 开始。
checkpoint 生成 redo pointer `R`。
如果 `P0 <= R`，说明这个页面还没有在当前 checkpoint 周期后被 WAL record 推进过。
此时 backend 修改该页。
修改前，调用者在 buffer lock 和 critical section 边界内注册 buffer。
`XLogRegisterBuffer()` 记录 block reference。
`XLogRecordAssemble()` 读取该页 page LSN。
它发现 `PageGetLSN(page) <= RedoRecPtr`。
如果 `doPageWrites` 为真，就把整页 image 加入 WAL record。
WAL record 插入完成后，前台路径会把页面 LSN 设置到该 record 的结束位置。
页面被标记 dirty。
之后它可能很快被写出。
也可能很久之后才由 checkpointer/bgwriter/backend 写出。
写出前，`FlushBuffer()` 读取 page LSN。
如果是 permanent buffer，它必须 `XLogFlush(page LSN)`。
这一步建立 WAL-before-data。
现在 crash 发生。
磁盘上的页面可能有三种状态。
第一种：页面还没写出，磁盘仍是旧版本。
第二种：页面完整写出，page LSN 已经是新 record 的 EndRecPtr。
第三种：页面 torn，部分内容来自新版本，部分内容来自旧版本。
recovery 从 `checkPoint.redo` 开始。
`ReadRecord()` 读到这条修改 record。
`XLogReader` 校验 page header、record header、CRC，并 decode 出 block image。
`ApplyWalRecord()` 更新 `replayEndRecPtr`。
然后分派到 `heap_redo()`。
`heap_redo()` 调 `XLogReadBufferForRedo()`。
如果 record 的 block image 要 apply，`XLogReadBufferForRedoExtended()` 直接 `RestoreBlockImage()`。
这覆盖了第一种和第三种状态。
旧版本或 torn page 都被 WAL 中的整页镜像替换。
恢复后设置 page LSN 为 record `EndRecPtr`。
如果页面已经完整写出，page LSN 可能已经大于等于 record `EndRecPtr`。
但如果 record 带 apply image，恢复端仍以 image 分支为准。
对后续没有 FPI 的 record，page LSN 判断会返回 `BLK_DONE`。
如果这是同页后续 delta record，恢复时不再有整页镜像。
它依赖前一条 FPI 或前一条 delta 已经把页面推进到正确前态。
这就是“FPI 只负责第一跳，LSN 顺序负责后续跳”的含义。
如果 recovery 过程中再次 crash，下次 recovery 仍从 checkpoint redo pointer 或更新后的 checkpoint redo pointer 开始。
已经写出的页面会因为 page LSN 较新而跳过旧 record。
未写出的页面会重新应用 record。
重复应用不会破坏页面，因为 page LSN 把每个 block 的完成进度编码在页面上。
这条状态故事说明：
WAL record 是全局历史。
page LSN 是页面本地历史。
checkpoint redo pointer 是恢复需要回看的历史 horizon。
FPI 是在 horizon 之后重新建立页面前态的保险。
rmgr redo routine 是把 record 语义变成具体页面变化的执行者。

## 31. 三个 LSN 不要混淆
本节至少有三类 LSN。
第一类是 checkpoint redo pointer。
它回答：
从哪里开始 replay 足够？
它来自 checkpoint record 中的 `checkPoint.redo`。
它被保存在 recovery 变量 `RedoStartLSN` 中。
它也是 FPI 判断中 `RedoRecPtr` 的语义来源。
第二类是 record LSN。
`ReadRecPtr` 是当前 record 起点。
`EndRecPtr` 是当前 record 结束后一字节。
日志定位和 `pg_waldump -s` 常用 `ReadRecPtr`。
页面应用完成后设置的是 `EndRecPtr`。
第三类是 page LSN。
它存在于数据页里。
它回答：
这个页面已经包含到哪个 WAL record 的效果？
它不回答事务可见性。
它不回答 recovery 是否 consistent。
它也不回答其它页面或辅助结构是否完成。
一个常见诊断错误是把 `pg_last_wal_replay_lsn()` 和某个 page LSN 混为一谈。
`pg_last_wal_replay_lsn()` 对应 startup process 已成功 replay 的全局进度。
page LSN 是某个页面自己的局部进度。
全局进度可以超过某页 page LSN。
因为某页未被相关 record 修改。
某页 page LSN 也可以在重复 recovery 时让旧 record 返回 `BLK_DONE`。
另一个常见错误是把 checkpoint redo pointer 和 latest checkpoint location 混为一谈。
`Latest checkpoint location` 是 checkpoint record 的位置。
`Latest checkpoint's REDO location` 才是 replay horizon。
`pg_controldata` 同时显示它们，正是因为二者不是同一语义。
第三个错误是把 record `ReadRecPtr` 当成 page LSN。
如果页面设置成 `ReadRecPtr`，下一次 recovery 可能认为同一 record 还没完成。
所以 redo routine 设置 `record->EndRecPtr`。
这种 off-by-one 式的 LSN 语义不是细节。
它是 redo 幂等性的边界。

## 32. 从一条错误日志回到源码
假设日志出现：
`WAL redo at 0/16B6D80 for Heap/INSERT ...`
第一步是确认这是 rmgr 错误上下文。
它来自 `rm_redo_error_callback()`。
说明 reader 已经把 record 交给 rmgr。
第二步用 `pg_waldump` 看该 LSN。
命令形态是：
`pg_waldump -p "$PGDATA/pg_wal" -s 0/16B6D80 -n 1`
关注 rmgr 是否真的是 Heap。
关注 desc 中的 operation。
关注 block references。
关注是否有 FPI。
关注 block number 和 fork。
第三步进源码。
如果是 Heap/INSERT，断在 `heap_redo()` 和 `heap_xlog_insert()`。
打印 `XLogRecGetInfo(record)`。
打印 `record->ReadRecPtr` 和 `record->EndRecPtr`。
进入 `XLogReadBufferForRedo()`。
看返回 action。
如果 action 是 `BLK_DONE`，理论上不应重复插入 tuple。
如果仍然报 offset 错误，要看错误是否来自其它 block 或辅助结构。
如果 action 是 `BLK_RESTORED`，说明 FPI 已经恢复了页面。
对某些 record 类型，这条 block 的 delta 不应再执行。
如果 action 是 `BLK_NEEDS_REDO`，继续检查 page 当前 max offset。
再检查 WAL record 里的 target offset。
如果 offset 超出页面语义，问题在 record/page 前态组合。
第四步回看 recovery 起点。
如果 page 前态不对，可能不是当前 record 自己坏。
可能是更早 record 没有 replay。
可能是 checkpoint redo pointer 错。
可能是 WAL 缺失后错误跳过。
也可能是数据页来自错误 base backup。
所以 rmgr PANIC 的诊断不能只看当前一条 record。
要验证从 redo pointer 到当前 LSN 的 WAL 是否连续。
还要验证 base backup 或数据目录是否与 WAL 属于同一 system identifier 和 timeline 历史。
如果日志是 reader 层错误，流程不同。
比如 `unexpected pageaddr`。
这时先不要进 heap/btree。
先检查 WAL 文件名、timeline、segment 内容、archive restore。
同样的 LSN，如果 `pg_waldump` 在 standby 上也报 pageaddr 错，说明 WAL 文件本身或来源有问题。
如果 primary 上同一 WAL 可读，standby 上不可读，优先怀疑传输、归档或文件替换。

## 33. FPI、checksum 与一致性检查
FPI 不是 checksum。
checksum 检查页面物理损坏。
FPI 提供恢复页面的完整镜像。
二者可以同时存在，但解决的问题不同。
如果页面 torn，checksum 可能帮助发现坏页。
但发现坏页不是恢复坏页。
恢复坏页需要 WAL 中有足够信息。
checkpoint 后首次修改页面时，FPI 就是这类足够信息。
`wal_consistency_checking` 又是另一层。
`xloginsert.c` 中如果某个 rmgr 开启 WAL consistency checking，会强制包含 full-page image 作为对比基础。
redo 后 `verifyBackupPageConsistency()` 会把 replay 后页面与 WAL 中的 backup image 做 masked comparison。
这个检查用于测试或开发诊断。
它不是正常 crash recovery 正确性的必要前提。
它也不是替代 rmgr redo routine。
因为一致性检查的 image 可能不带 `BKPIMAGE_APPLY` 语义。
record 中有 image，不代表 recovery 一定会用它覆盖页面。
要看 `BKPIMAGE_APPLY`，也就是 `XLogRecBlockImageApply()`。
这解释了一个细节：
`XLogRecHasBlockImage()` 和 `XLogRecBlockImageApply()` 不是同一件事。
前者说明 record 有 image。
后者说明这个 image 应该用于恢复页面。
普通 FPI 恢复依赖后者。
一致性检查可能依赖前者。
诊断时如果只看到 `has_image` 就断言页面会 `BLK_RESTORED`，会误判。
还要看 image header 中的 apply flag。

## 34. 为什么缺 WAL 不能靠数据页补回来
有时现场会问：
如果数据页已经写出了，为什么还需要旧 WAL？
答案是 recovery 不知道哪些页已经完整写出。
也不知道哪些页 torn。
更不知道哪些辅助结构和控制状态已经同步。
page LSN 只能说明单页局部进度。
checkpoint redo pointer 之后的 WAL 可能包含：
heap tuple 变化。
btree split。
visibility map 清理。
transaction commit/abort。
clog/multixact 变化。
relmap 变化。
smgr create/drop/truncate。
database/tablespace 操作。
parameter change。
backup end。
这些状态不都能从数据页反推出。
尤其是 transaction status 和 relation file lifecycle。
如果缺少从 redo pointer 到一致点之间的 WAL，系统无法证明数据目录处于某个合法历史。
这也是 replication slot、wal_keep_size、archive retention 对 standby 和 base backup 很重要的原因。
WAL 保留不是为了“日志完整好看”。
它保护的是 recovery 需要的历史证明。

## 35. custom rmgr 与扩展边界
本节主要讲内置 rmgr，但 `rmgr.c` 也支持 custom WAL resource manager。
`RegisterCustomRmgr()` 要求 custom rmgr 在 `shared_preload_libraries` 初始化阶段注册。
原因很直接：
recovery 看到 WAL record 时，必须已经知道 `xl_rmid` 对应的 redo routine。
如果 standby 或 crash recovery 环境缺少该 rmgr，`GetRmgr()` 无法把 record 分派到语义实现。
这不是普通 extension function 缺失。
这是 WAL 历史不可解释。
`rmgrlist.h` 中内置 rmgr 的顺序定义持久化 ID。
custom rmgr 的 ID 范围在 `RM_MIN_CUSTOM_ID` 到 `RM_MAX_CUSTOM_ID`。
开发阶段可用 `RM_EXPERIMENTAL_ID`，但生产系统必须避免 ID 冲突。
从 recovery 角度看，custom rmgr 必须遵守同样边界。
record 必须能被 `XLogReader` 解码。
block references 必须与 redo routine 读取方式匹配。
页面修改必须通过 `XLogReadBufferForRedo*()` 尊重 FPI/page LSN。
修改后必须设置 page LSN 到 `record->EndRecPtr`。
buffer 必须正确 dirty 和 release。
错误时应让 recovery 停在明确的 record 边界，而不是吞掉错误继续。
这也是为什么 custom rmgr 不是“随便把 bytes 放进 WAL”。
它接入的是 crash recovery 的正确性链路。
一旦写入持久 WAL，未来所有 replay 该 WAL 的节点都需要同样的语义实现。
这条边界对扩展作者尤其重要。
SQL API 可以升级。
extension catalog 可以迁移。
但已经写入 WAL 的 rmgr ID、record layout 和 redo 语义不能随意改变。
否则 standby、PITR 和 crash recovery 都会在历史 WAL 上失去解释能力。

## 36. 讨论题
1. 为什么 online checkpoint 的 replay 起点不能简单等于 checkpoint record 之后？
2. `ReadRecPtr` 和 `EndRecPtr` 在诊断和 page LSN 设置中分别承担什么语义？
3. 如果 `PageGetLSN(page) > record->EndRecPtr`，redo routine 为什么必须跳过该 block？
4. FPI 判断为什么使用 `PageGetLSN(page) <= RedoRecPtr`，而不是只看 buffer 是否 dirty？
5. 一条 heap update record 触碰 old page、新 page、VM 和 FSM 时，哪个状态能由单个 page LSN 完整表达？
6. 为什么 `XLogReader` 校验 CRC 后，rmgr redo routine 仍然可能 `PANIC`？
7. `replayEndRecPtr` 和 `lastReplayedEndRecPtr` 哪个表示已经成功 replay？
8. standby 卡住时，如何区分缺 WAL、坏 record、redo I/O 慢、FPI 解压重和 Hot Standby conflict？

## 37. 本节小结
本节核心链路是：`checkPoint.redo` 定义 recovery horizon；`ReadRecord()` 和 `XLogReader` 按 LSN 顺序读出完整 decoded record；`ApplyWalRecord()` 维护 `replayEndRecPtr` 与 `lastReplayedEndRecPtr` 的边界；`xlogrecovery_redo()` 处理恢复状态机特殊 record；`GetRmgr(...).rm_redo()` 分派到具体 rmgr；`XLogReadBufferForRedoExtended()` 用 FPI 和 page LSN 决定页面动作；rmgr 修改页面后把 page LSN 设到 `EndRecPtr`；`CheckRecoveryConsistency()` 决定实例是否到达一致状态。
核心不变量：WAL-before-data 让磁盘页上的 page LSN 有可解释来源；checkpoint redo pointer 让 recovery 从足够早的位置开始；FPI 修复 checkpoint 后首次修改页面的 torn-page 前态风险；page LSN 让 redo 幂等；rmgr redo routine 保留 access method 和控制状态语义；consistent state 是实例级恢复门槛。
错误诊断框架：起点坏时查 checkpoint record、pg_control、redo pointer；WAL 坏或缺时查 `XLogReader` 的 page header、CRC、timeline、archive/streaming 来源；页面语义坏时查 `WAL redo at <LSN>`，用 `pg_waldump` 和 gdb 进入具体 rmgr routine；一致点不到时查 `minRecoveryPoint`、`backupEndPoint`、invalid pages、Hot Standby snapshot。
可迁移规律：持久化恢复系统不是只靠“重放日志”成立；它必须同时定义日志顺序、数据页局部进度、checkpoint 起点、整页修复边界、模块语义分派和对外可用门槛。
本节判断仍有边界：FPI 数量依赖 checkpoint 频率、working set、`full_page_writes`、backup 和 compression；redo 速度依赖 WAL 形态、存储 I/O、buffer 命中、rmgr 类型和 standby conflict；SQL 视图只能看到 replay 进度，看不到每个 block 的 redo action；源码路径基于 PostgreSQL master commit `0e1f1ed157e90741e12a3715909e1b2d71ff9344`。
