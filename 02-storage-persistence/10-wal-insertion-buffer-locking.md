# PostgreSQL WAL insertion、WAL buffers 与 insert lock


## 课程定位

本节主题：多个 backend 并发生成 WAL record 时，PostgreSQL 如何把 record 放进共享 WAL buffers，并在保留 LSN、拷贝字节、推进插入位置和减少锁竞争之间做折中。
前置知识：已理解 WAL record 的构造格式、record end LSN 的含义，以及 WAL flush 和 data page 写出的基本区别。
本节唯一主问题：一个已经组装好的 WAL record，怎样在并发 backend 中获得唯一 WAL 地址并安全进入共享 WAL buffers？
本节核心矛盾：WAL byte stream 必须有全局顺序；但如果把 reserve、copy、buffer 复用和 flush 全部串行化，提交和写入吞吐会被单点锁限制。
本节主流程：`XLogInsert()` 组装 record -> `XLogInsertRecord()` 取得 insertion lock -> reserve byte range -> copy 到 WAL buffers -> flush 侧等待 insertion 完成。
生命周期 / ownership / cleanup：WAL insertion lock 只保护短期 reservation 和 insertion state，record bytes 进入 WAL buffers 后由 flush/write 路径推进，backend 不长期拥有 WAL buffer page。
错误路径 / 异常路径包括 record assemble 重试、WAL buffer 复用前必须写出旧页、flush 侧等待未完成 insertion、WAL write/fsync 失败，以及错误持锁等待导致的死锁风险。
观测与诊断入口是 `pg_stat_wal`、WALWrite/Sync wait event、`wal_buffers` 压力、`pg_waldump` 的 record 顺序，以及 `XLogInsertRecord()`/`XLogWrite()` 断点。
这一节只看 WAL insertion 的“写入 shared WAL buffers”阶段。
它不展开每个 rmgr 的 redo 语义。
它不展开 commit record 的同步提交策略。
它也不展开 WAL segment 回收、归档和 checkpoint 调优。
这些主题在后续课程继续拆开。
本节要回答的是更底层的问题：
一个已经组装好的 WAL record，怎样变成 WAL 字节流中的一段连续内容。
为什么 `XLogInsert()` 返回的是 record end pointer。
为什么 `ReserveXLogInsertLocation()` 可以先把 insert position 往前推进，而 record 字节还没完全拷贝完。
为什么 WAL insertion lock 不是一把全局大锁，而是一组固定数量的锁。
为什么 WAL buffer page 被复用前可能要写出旧 WAL。
为什么 flush WAL 前必须等待相关 insertion 完成。
读完本节，你应该能回答：
- `XLogInsert()` 与 `XLogInsertRecord()` 的职责边界在哪里。
- `WALInsertLock` 保护的是“插入进行中状态”，而不是整个 WAL 字节流的唯一序列化点。
- `insertpos_lck` 为什么只保护 `CurrBytePos` 和 `PrevBytePos`。
- reserve LSN 时得到的 `StartPos`、`EndPos`、`xl_prev` 分别是什么意思。
- `CurrBytePos` 为什么用“usable byte position”，而不是直接用 `XLogRecPtr`。
- `CopyXLogRecordToWAL()` 如何处理 WAL page 边界。
- WAL page header 的 short / long header 对 record 起止位置有什么影响。
- `GetXLogBuffer()` 为什么在缺页时要更新 `insertingAt`。
- `AdvanceXLInsertBuffer()` 为什么会触发旧 WAL buffer 写出。
- `XLogFlush()` 为什么先调用 `WaitXLogInsertionsToFinish()`，再争抢 `WALWriteLock`。
- flush request 与 WAL insertion 的关系是什么，关系又不是什么。
- 哪些边界是普通 `ERROR`，哪些边界会进入 `PANIC` 或 critical section。

## 源码基线

源码仓库：`/home/nail/postgres-lab`
基线：`master` = `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`
本节重点阅读：
- `src/backend/access/transam/xloginsert.c`
- `src/backend/access/transam/xlog.c`
- `src/include/access/xlog_internal.h`
- `src/include/access/xlog.h`
辅助核对：
- `src/include/access/xlogdefs.h`
- `src/include/access/xlogrecord.h`
本节源码行号按上述基线记录。
如果你本地 checkout 不是这个提交，行号可能会漂移。

---


## 1. 先给结论

WAL insertion 的核心不是“拿一把锁，把 record 追加到文件”。
PostgreSQL 把这个过程拆成几段不同粒度的同步：
第一段是组装 record。
这发生在 backend 私有内存中。
`xloginsert.c` 中的 `XLogBeginInsert()`、`XLogRegisterBuffer()`、`XLogRegisterData()` 等调用把 rmgr 想写的内容登记起来。
`XLogInsert()` 再把这些登记内容组装成 `XLogRecData` 链。
第二段是 reserve WAL address range。
这发生在 `xlog.c` 的 `ReserveXLogInsertLocation()` 中。
它只用很短的 spinlock 临界区更新 `Insert->CurrBytePos` 和 `Insert->PrevBytePos`。
这一步决定 record 的 `StartPos`、`EndPos` 和 `xl_prev`。
第三段是把 record bytes 拷贝进共享 WAL buffers。
这发生在 `CopyXLogRecordToWAL()` 中。
不同 backend 拿到的 WAL address range 互不重叠。
所以大部分 memcpy 可以并发执行。
只有碰到尚未初始化的 WAL buffer page，或者需要复用旧 WAL buffer page 时，才进入 `WALBufMappingLock` 和可能的 WAL write 路径。
第四段是让外部写出者知道哪些 insertion 已完成。
每个插入者持有一个 `WALInsertLock`。
这个锁本身是 LWLock。
锁旁边有一个原子变量 `insertingAt`。
flush 路径用它判断：我要写到某个 LSN，是否还需要等待某个插入者把早于这个 LSN 的字节拷贝完。
第五段才是 write / fsync。
普通 `XLogInsert()` 只保证 record 已经拷贝到 WAL buffers。
它不等价于 WAL 已写到操作系统，也不等价于 WAL 已 fsync。
需要持久化时，调用方走 `XLogFlush(record_end_lsn)`。
`XLogFlush()` 会先等待相关 insertion 完成，再持有 `WALWriteLock` 调用 `XLogWrite()`。
所以你可以把这条路径记成：
```text
register -> assemble -> reserve -> copy to WAL buffers -> optionally write/flush
```
其中只有 reserve 是全局串行点。
copy 通常并行。
write/flush 由另一个锁域控制。

---


## 2. 关键名词

`XLogRecPtr` 是 64 位 WAL 位置。
定义在 `src/include/access/xlogdefs.h:17-29`。
0 是 `InvalidXLogRecPtr`。
WAL record 不会从 0 开始。
`LSN` 在许多语境里就是 `XLogRecPtr`。
但读源码时要注意上下文。
有时它表示 record start。
有时它表示 record end。
数据页的 page LSN 通常写 record end pointer。
`XLogRecord` 是 WAL record 的固定头。
定义在 `src/include/access/xlogrecord.h:41-55`。
关键字段是：
- `xl_tot_len`：整个 record 的总长度。
- `xl_xid`：相关 transaction id。
- `xl_prev`：前一条 record 的 start pointer。
- `xl_info`：rmgr info 和内部 flag。
- `xl_rmid`：resource manager id。
- `xl_crc`：record CRC。
`XLogRecData` 是用于插入前拼接 record 内容的链表节点。
定义在 `src/include/access/xlog_internal.h:302-309`。
它不是 WAL 文件格式的一部分。
它是内存中的 scatter/gather 描述。
`XLOG_BLCKSZ` 是 WAL page 大小。
源码中 WAL buffer 按 WAL page 管理。
一个 WAL segment 由许多 WAL page 组成。
每个 WAL page 有 page header。
WAL record 可以跨 WAL page。
WAL record 也可以跨 WAL segment，但 segment switch 这种特殊 record 有额外处理。
`XLogPageHeaderData` 定义在 `src/include/access/xlog_internal.h:32-55`。
它包含 `xlp_magic`、`xlp_info`、`xlp_tli`、`xlp_pageaddr` 和 `xlp_rem_len`。
`xlp_rem_len` 用于 record 跨 page 后的 continuation。
`SizeOfXLogShortPHD` 是普通 WAL page header 大小。
`SizeOfXLogLongPHD` 是 segment 第一页上的长 header 大小。
长 header 额外记录 system identifier、segment size 和 xlog block size。
`XLP_FIRST_IS_CONTRECORD` 表示当前 WAL page 的第一段内容是上一页 record 的延续。
定义在 `src/include/access/xlog_internal.h:74-81`。
`WAL buffer` 是共享内存中的 WAL page cache。
它不是 data buffer。
它由 `XLogCtl->pages` 和 `XLogCtl->xlblocks` 等共享状态管理。
`xlblocks[idx]` 记录某个 WAL buffer slot 当前装载的 WAL page 的 end pointer。
`XLogRecPtrToBufIdx(recptr)` 用 WAL page number 对 WAL buffer 数量取模。
定义在 `src/backend/access/transam/xlog.c:607-612`。
这意味着某个 WAL page 天然映射到固定 buffer slot。
如果 slot 里是旧 page，就必须先确保旧 page 已写出，才能复用。
`WALInsertLock` 是插入者持有的 LWLock 加状态槽。
定义在 `src/backend/access/transam/xlog.c:338-379`。
它包含：
- `lock`
- `insertingAt`
- `lastImportantAt`
`NUM_XLOGINSERT_LOCKS` 在本基线是 8。
定义在 `src/backend/access/transam/xlog.c:152-157`。
`insertpos_lck` 是 spinlock。
它只保护 `Insert->CurrBytePos` 和 `Insert->PrevBytePos`。
它不是 WAL insertion lock。
`CurrBytePos` 是已 reserve WAL 的末端。
`PrevBytePos` 是上一条已 reserve record 的起点。
二者存在 `XLogCtlInsert` 中，见 `src/backend/access/transam/xlog.c:400-452`。
`usable byte position` 是不计算 WAL page header 的逻辑字节位置。
它使 reserve 操作接近 `CurrBytePos += size`。
从 usable byte position 到真实 `XLogRecPtr` 的转换由 `XLogBytePosToRecPtr()` 和 `XLogBytePosToEndRecPtr()` 完成。
这两个函数在 `src/backend/access/transam/xlog.c:1893-1976`。

---


## 3. 从 `XLogInsert()` 进入

先读 `src/backend/access/transam/xloginsert.c:482-540`。
`XLogInsert(RmgrId rmid, uint8 info)` 是多数 rmgr 调用的入口。
它要求调用方之前已经调用 `XLogBeginInsert()`。
如果没有，会报：
```text
XLogBeginInsert was not called
```
这是编程错误。
`XLogInsert()` 还检查 `info` mask。
调用方只能使用 rmgr bits、`XLR_SPECIAL_REL_UPDATE` 和 `XLR_CHECK_CONSISTENCY`。
其他 bit 是 WAL insertion 内部保留位。
这里如果不合法，会 `PANIC`。
bootstrap mode 有特殊分支。
如果正在 bootstrap 且不是 `RM_XLOG_ID`，它不实际写 WAL，而是返回一个假的 record pointer。
普通路径是一个 `do ... while` 循环。
循环里先调用 `GetFullPageWriteInfo()`。
这一步读取当前用于判断 full page image 的信息。
此时还没有 WAL insertion lock。
所以这些信息可能随后被 checkpoint 或配置变化刷新。
然后 `XLogRecordAssemble()` 组装 `XLogRecData` 链。
再调用 `XLogInsertRecord()`。
如果 `XLogInsertRecord()` 返回 `InvalidXLogRecPtr`，说明它拿到 insertion lock 后发现 full page write 相关状态变化，当前组装的 record 不再安全。
这时 `XLogInsert()` 会重新 assemble。
这就是 `do ... while (!XLogRecPtrIsValid(EndPos))` 的意义。
`XLogInsert()` 最后调用 `XLogResetInsertion()` 清理 backend-local 登记状态。
返回值是 `EndPos`。
源码注释明确说它返回 record end pointer。
这个返回值可以用作被修改数据页的 page LSN。
原因是 WAL-before-data 需要的是“写出这个数据页前，WAL 至少 flush 到这条 record 的末尾”。
如果只写 record start pointer，无法证明整条 record 已持久化。

---


## 4. `XLogRecordAssemble()` 与重试边界

`XLogRecordAssemble()` 位于 `src/backend/access/transam/xloginsert.c:607` 之后。
本节不详细展开 WAL record 格式。
但这里有一个与 insertion locking 直接相关的边界。
组装 record 时，它可能根据 `RedoRecPtr` 和 `doPageWrites` 决定某个 block 是否需要 full-page image。
源码在 `xloginsert.c:678-700` 读取 registered buffer 的 page LSN。
如果 page LSN 小于等于 redo pointer，通常需要 backup image。
如果不需要 backup image，它会维护 `fpw_lsn`。
这个 `fpw_lsn` 的含义是：
当前 record 的正确性依赖于我们看到的 `RedoRecPtr` 和 full-page-write 状态没有变得更严格。
`XLogInsertRecord()` 拿到 WAL insertion lock 后会重新检查。
检查位置在 `src/backend/access/transam/xlog.c:878-895`。
如果拿锁后发现现在需要 page image，而组装时没有包含，它会释放 lock，退出 critical section，并返回 `InvalidXLogRecPtr`。
这不是错误。
这是正常重试协议。
这个设计让常见路径不用在 assemble 阶段拿 WAL insertion lock。
只有状态变化的少见情况才重新组装。
注意这个重试边界也说明一件事：
`XLogInsert()` 负责“准备可插入 record”。
`XLogInsertRecord()` 负责“在持有插入锁后确认 record 仍可插入，并执行插入”。

---


## 5. `XLogInsertRecord()` 的两步模型

核心注释在 `src/backend/access/transam/xlog.c:824-855`。
PostgreSQL 把插入 shared WAL buffer cache 分成两步：
第一步，reserve WAL 空间。
当前 reserve head 是 `Insert->CurrBytePos`。
它由 `insertpos_lck` 保护。
第二步，把 record 拷贝到刚刚 reserve 的 WAL 空间。
这需要找到对应 WAL buffer page，并把 `XLogRecData` 链里的字节写进去。
这一步可以由多个 backend 并行执行。
为了让 flush 路径知道哪些插入还没完成，每个插入者还要持有一个 insertion lock。
当插入跨 page，或者插入者可能睡眠时，它会更新 `insertingAt`。
这个值告诉别人：
我已经完成到哪里，或者我正在更靠后的哪里工作。
`XLogInsertRecord()` 的普通路径从 `src/backend/access/transam/xlog.c:858` 开始。
它先调用 `WALInsertLockAcquire()`。
然后检查 `RedoRecPtr` 和 `fullPageWrites`。
然后调用 `ReserveXLogInsertLocation()`。
然后计算 record header 的 CRC。
然后调用 `CopyXLogRecordToWAL()`。
最后释放 insertion lock。
特殊 record 有两类。
`XLOG_SWITCH` 需要 `WALInsertLockAcquireExclusive()`。
它要独占所有 insertion locks，因为它要计算当前 segment 剩余空间，并把剩余空间都 claim 掉。
`XLOG_CHECKPOINT_REDO` 也需要持有所有 insertion locks。
它要更新 shared `Insert->RedoRecPtr`。
普通 record 不需要持有所有锁。
它只需要一个 insertion lock。
这就是并发插入能扩展的关键。

---


## 6. WAL insertion state

`XLogCtlInsert` 是 shared state 中与 insertion 直接相关的部分。
定义在 `src/backend/access/transam/xlog.c:400-452`。
它的第一段是最热的字段：
```text
insertpos_lck
CurrBytePos
PrevBytePos
```
`CurrBytePos` 是 reserve head。
下一条 record 会从这里开始 reserve。
`PrevBytePos` 是上一条 record 的起点。
当前 record reserve 时，会把它转换成 `xl_prev`。
这让 WAL record 在逻辑上形成 backward chain。
`CurrBytePos` 和 `PrevBytePos` 被放在独立 cache line。
源码注释在 `xlog.c:417-424` 说明原因：
它们由高频 spinlock 保护。
`RedoRecPtr` 和 `fullPageWrites` 虽然每次插入都读，但更新很少。
如果放在同一 cache line，会增加不必要的 cache line 抖动。
第二段是 full page write 相关状态：
```text
RedoRecPtr
fullPageWrites
runningBackups
lastBackupStart
```
读取这些字段必须持有一个 insertion lock。
修改这些字段必须持有所有 insertion locks。
这正是 `XLogInsertRecord()` 拿锁后重新检查 full page write 状态的原因。
第三段是 insertion lock 数组。
`WALInsertLocks` 指向 shared memory 中的 `WALInsertLockPadded` 数组。
数组元素按 cache line padding。
定义在 `xlog.c:381-392`。
这避免不同 insertion lock slot 落在同一个 cache line 上。
本基线 `NUM_XLOGINSERT_LOCKS` 是 8。
它是固定数量，不是按 backend 动态扩展。
更多 insertion locks 可以提高并发插入机会。
但 flush 路径需要扫描所有 locks。
所以数量不是越大越好。

---


## 7. `WALInsertLock` 的真实职责

`WALInsertLock` 不是用来保护 `CurrBytePos` 的。
`CurrBytePos` 由 `insertpos_lck` 保护。
`WALInsertLock` 也不是用来保护 WAL buffers 中每个字节的。
每个 backend reserve 到的字节范围不重叠。
它的核心职责有三个。
第一，标识“有一个 WAL insertion 正在进行”。
flush 路径不能把某段 WAL 写出到磁盘，除非这段 WAL 对应的插入者已经把字节拷贝完。
第二，承载 `insertingAt`。
当插入者必须跨 page 或可能等待时，它用 `WALInsertLockUpdateInsertingAt()` 广播进度。
这让 flush 路径只等必要的插入。
第三，保护少量 insertion-wide 配置状态。
`RedoRecPtr`、`fullPageWrites`、`runningBackups` 读取时要求持有一个 insertion lock。
更新时要求持有所有 insertion locks。
`WALInsertLockAcquire()` 位于 `src/backend/access/transam/xlog.c:1409-1450`。
它优先尝试本 backend 上次用过的 lock。
第一次按 `MyProcNumber % NUM_XLOGINSERT_LOCKS` 选一个。
如果没能立即获得，下次会尝试下一个 lock。
这让短连接或高并发负载更均匀地分布到 8 个 lock 上。
`WALInsertLockAcquireExclusive()` 位于 `xlog.c:1452-1477`。
它依次拿所有 insertion locks。
拿所有锁时，除了最后一个锁，其余锁的 `insertingAt` 会被设成 `PG_UINT64_MAX`。
这样等待者不会错误地等待这些 slot。
真正位置用最后一个 lock 表示。
释放在 `WALInsertLockRelease()`。
它用 `LWLockReleaseClearVar()` 把 `insertingAt` 清回 0。
注释特别强调，清回 0 是为了让下一次 acquire 后 `LWLockWaitForVar()` 能正确阻塞。

---


## 8. reserve LSN：最短的全局串行点

`ReserveXLogInsertLocation()` 位于 `src/backend/access/transam/xlog.c:1130-1193`。
这段是性能关键路径。
注释明确说：
这是 `XLogInsert` 中必须跨 backend 序列化的部分。
它要尽量短。
函数先把 `size` 做 `MAXALIGN()`。
普通 record 的 size 必须大于 `SizeOfXLogRecord`。
真正持有 spinlock 的代码非常少：
```text
startbytepos = Insert->CurrBytePos
endbytepos = startbytepos + size
prevbytepos = Insert->PrevBytePos
Insert->CurrBytePos = endbytepos
Insert->PrevBytePos = startbytepos
```
然后立刻释放 `insertpos_lck`。
之后再把 byte position 转成 `XLogRecPtr`：
```text
StartPos = XLogBytePosToRecPtr(startbytepos)
EndPos   = XLogBytePosToEndRecPtr(endbytepos)
PrevPtr  = XLogBytePosToRecPtr(prevbytepos)
```
注意这里的顺序。
`CurrBytePos` 在 record 字节拷贝之前就推进了。
所以 `CurrBytePos` 表示“已 reserve 到哪里”，不是“已完整插入到哪里”。
这就是为什么还需要 insertion lock 和 `logInsertResult`。
reserve 后，别人可能看到 WAL insert pointer 已经前进。
但 flush 不能只看 reserve head。
flush 必须确认相关 insertion 已完成。
`StartPos` 是当前 record 的起点。
`EndPos` 是当前 record 的 end pointer。
`PrevPtr` 会写入 `rechdr->xl_prev`。
`xl_prev` 指向前一条 record 的起点，而不是前一条 record 的 end pointer。
`EndPos` 使用 `XLogBytePosToEndRecPtr()`。
这个函数在 byte position 落在 page 边界时，返回 page 开头，而不是跳过 header 后的位置。
这对“record end pointer”很重要。
record 可以刚好结束在 page 末尾。
下一条 record 的 start pointer 需要跳过新 page header。
而上一条 record 的 end pointer 可以表达为新 page 开头。
源码在 `xlog.c:1932-1976` 专门区分这两种转换。

---


## 9. usable byte position 与真实 LSN

`CurrBytePos` 不是 `XLogRecPtr`。
它是 usable byte position。
源码在 `xlog.c:1162-1170` 解释了原因。
WAL page header 不属于 record payload 可用空间。
如果直接用 `XLogRecPtr` reserve，每次计算都会碰到 page header、segment first page long header 等边界。
用 usable byte position 后，reserve N 字节几乎就是加 N。
复杂的 page header 折算被推迟到 lock 外。
`XLogBytePosToRecPtr()` 用于 record start。
如果位置落在 segment 第一页，它跳过 `SizeOfXLogLongPHD`。
如果位置落在其他页，它跳过 `SizeOfXLogShortPHD`。
`XLogBytePosToEndRecPtr()` 用于 record end。
它的特殊点是：
如果 byte position 正好在 page 边界，返回 page 开头。
这样 end pointer 可以稳定表示“所有字节到这里为止”。
`XLogRecPtrToBytePos()` 是反向转换。
它从真实 pointer 中扣掉 long 或 short WAL page header。
这三者共同保证：
reserve 阶段可以快速推进 logical byte position。
copy 阶段可以拿真实 WAL address 找 buffer。
flush 阶段可以用真实 LSN 与 page/segment 边界交互。
不要把 `CurrBytePos` 直接理解成磁盘 offset。
也不要把 `StartPos + xl_tot_len == EndPos` 当成永远成立。
跨 WAL page 时，真实 address 空间中有 page headers。
`xl_tot_len` 只统计 record 自身。

---


## 10. copy record 到 WAL buffers

`CopyXLogRecordToWAL()` 位于 `src/backend/access/transam/xlog.c:1266-1400`。
它接收：
- `write_len`
- 是否是 xlog switch
- `XLogRecData` 链
- `StartPos`
- `EndPos`
- timeline id
它先把 `CurrPos` 设成 `StartPos`。
然后调用 `GetXLogBuffer(CurrPos, tli)`。
这返回 WAL buffer 中对应真实地址的内存指针。
接着用 `INSERT_FREESPACE(CurrPos)` 计算当前 WAL page 剩余空间。
`INSERT_FREESPACE` 定义在 `xlog.c:596-601`。
如果 `endptr % XLOG_BLCKSZ == 0`，剩余空间是 0。
否则就是当前 page 到 page 末尾的字节数。
copy 主循环遍历 `XLogRecData` 链。
每个节点内部又可能跨 WAL page。
如果 `rdata_len > freespace`，说明当前节点剩余内容放不进当前 WAL page。
函数会先把当前 page 能放的部分 `memcpy` 进去。
然后推进 `rdata_data`、`rdata_len`、`written` 和 `CurrPos`。
接着进入下一页。
下一页的 header 需要标记 continuation。
代码在 `xlog.c:1317-1320`：
它再次调用 `GetXLogBuffer(CurrPos, tli)`。
然后把返回位置当作 `XLogPageHeader`。
设置 `xlp_rem_len = write_len - written`。
设置 `XLP_FIRST_IS_CONTRECORD`。
随后跳过 page header。
如果 `CurrPos` 位于 segment 开头，就跳过 `SizeOfXLogLongPHD`。
否则跳过 `SizeOfXLogShortPHD`。
然后重新计算 freespace。
如果当前节点剩余内容能放下，就直接 `memcpy`。
循环结束后断言 `written == write_len`。
普通 record 最后把 `CurrPos` 做 `MAXALIGN64()`。
这对应 reserve 阶段的 `MAXALIGN(size)`。
所以 reserve 的空间和 copy 的推进必须匹配。
源码在 `xlog.c:1141-1142` 特别提醒：
reserve 的空间计算必须和 copy 的逻辑一致。

---


## 11. WAL buffer page 边界

每个 WAL page 都有 page header。
segment 第一页通常是 long header。
其他页是 short header。
`AdvanceXLInsertBuffer()` 初始化新 WAL page 时决定这一点。
代码在 `src/backend/access/transam/xlog.c:2140-2162`。
它先清零整个 WAL page。
然后填 `xlp_magic`、`xlp_tli`、`xlp_pageaddr`。
如果 page address 是 segment 开头，还填 long header 的 `xlp_sysid`、`xlp_seg_size` 和 `xlp_xlog_blcksz`。
同时设置 `XLP_LONG_HEADER`。
`CopyXLogRecordToWAL()` 只负责在跨 page 时设置 continuation 信息。
其他 header flag 在 page 初始化阶段已经设置。
源码注释在 `xlog.c:1312-1315` 明确说：
设置 `contrecord` flag 和 `xlp_rem_len` 不需要额外 page lock。
因为其他 flag 已经在 `AdvanceXLInsertBuffer()` 中初始化。
而当前 backend 是唯一需要设置 continuation flag 的 backend。
为什么唯一？
因为每条 record 的 WAL address range 已经由 reserve 阶段唯一分配。
跨到下一页的 continuation header 属于这条 record。
别的 backend reserve 的 record 不会写这条 record 的 continuation header。
这里还有一个容易误读的地方。
WAL page header 是 WAL byte stream 的物理结构。
它不属于 `xl_tot_len`。
所以 record 跨 page 时，`CopyXLogRecordToWAL()` 会跳过 page header，再继续拷贝 record 剩余字节。
reader 读取时根据 `XLP_FIRST_IS_CONTRECORD` 和 `xlp_rem_len` 重新拼出完整 record。

---


## 12. `GetXLogBuffer()`：从 LSN 找共享内存页

`GetXLogBuffer()` 位于 `src/backend/access/transam/xlog.c:1656-1771`。
它把一个 `XLogRecPtr` 映射到 shared WAL buffers 中的地址。
它有一个 backend-local fast path。
如果这次访问的 WAL page 和上次相同，就直接返回 cached pointer 加 page offset。
这优化了常见的小 record 或同一 record 在同一页内多次 copy 的情况。
否则它用 `XLogRecPtrToBufIdx(ptr)` 计算 buffer slot。
这个映射是确定的。
同一个 WAL page 永远映射到同一个 slot。
它读取 `XLogCtl->xlblocks[idx]`。
这个原子值表示 slot 当前装载 page 的 end pointer。
`expectedEndPtr` 是 `ptr` 所在 page 的 end pointer。
如果二者相等，说明目标 page 已经在这个 slot 里。
函数执行 memory barrier，确保 page 初始化可见，然后返回指针。
如果不相等，说明 page 尚未初始化，或者 slot 里还是旧 page。
这时 `GetXLogBuffer()` 不能直接覆盖。
它先计算一个 `initializedUpto`。
如果 `ptr` 正好指向 page header 后面，它会谨慎地把 advertised position 调回 page 开头。
原因在 `xlog.c:1720-1732` 的注释里：
如果第一个插入者把 `insertingAt` 广告成 header 后的位置，别人可能以为 page header 已可写出。
但 page 可能还没初始化。
所以第一位插入者在初始化 page 前，不能让别人认为 page header 之后已经安全。
然后它调用 `WALInsertLockUpdateInsertingAt(initializedUpto)`。
再调用 `AdvanceXLInsertBuffer(ptr, tli, false)`。
如果初始化后 `xlblocks[idx]` 仍不是 expected end pointer，就 `PANIC`。
这是一条内部一致性边界。

---


## 13. `AdvanceXLInsertBuffer()`：初始化和复用 WAL buffer

`AdvanceXLInsertBuffer()` 位于 `src/backend/access/transam/xlog.c:2026-2185`。
它持有 `WALBufMappingLock`。
这个锁保护 WAL buffer slot 与 WAL page 的映射变化。
函数循环推进 `XLogCtl->InitializedUpTo`。
目标是确保 WAL insertion 需要的 page 已经有 buffer、已经清零、已经填好 page header。
每次要初始化下一页时，它先算出 `nextidx`。
然后读取旧 page 的 end pointer `OldPageRqstPtr`。
如果 `LogwrtResult.Write < OldPageRqstPtr`，说明这个 slot 里的旧 WAL page 还没写出。
不能覆盖它。
如果是 opportunistic 预初始化，就放弃。
否则它先更新 shared `LogwrtRqst.Write`。
然后刷新本地 `LogwrtResult`。
如果仍然需要写旧 page，它必须释放 `WALBufMappingLock`。
源码在 `xlog.c:2074-2078` 说明原因：
继续持有 mapping lock 等待 WAL write 可能死锁。
要写旧 page 前，还要调用 `WaitXLogInsertionsToFinish(OldPageRqstPtr)`。
这确保旧 page 上早于该位置的 insertion 已经完成。
然后拿 `WALWriteLock`。
如果别人还没写，就调用 `XLogWrite()`。
写完释放 `WALWriteLock`。
再重新拿 `WALBufMappingLock` 并重试。
只有旧 page 已写出后，slot 才能被复用。
初始化新 page 的顺序也很谨慎。
先把 `xlblocks[nextidx]` 写成 `InvalidXLogRecPtr`。
再写 barrier。
再清零 page。
再填 page header。
再写 barrier。
最后把 `xlblocks[nextidx]` 写成新 page end pointer。
并推进 `InitializedUpTo`。
这个顺序防止别人看到“看起来有效但实际只初始化了一半”的 WAL page。

---


## 14. insert position 推进与 inserted position

本节最容易混淆两个位置。
一个是 reserve head。
另一个是 known inserted position。
`Insert->CurrBytePos` 是 reserve head。
它在 `ReserveXLogInsertLocation()` 中推进。
这发生在 record bytes 拷贝之前。
所以它回答的问题是：
WAL address space 已经分配到哪里。
它不回答：
WAL buffers 已经安全写到哪里。
`XLogCtl->logInsertResult` 是“已知完成插入到哪里”的共享原子值。
`WaitXLogInsertionsToFinish()` 会推进它。
代码在 `src/backend/access/transam/xlog.c:1645-1651`。
这个函数会扫描 insertion locks。
它等到目标范围内的插入者释放 lock，或者看到它们的 `insertingAt` 已经移动到目标之后。
然后用 `pg_atomic_monotonic_advance_u64()` 推进 `logInsertResult`。
因此 `logInsertResult` 的推进并不一定由插入者自己完成。
它常常由需要写 WAL 的 backend 或 walwriter 在等待过程中推进。
`XLogInsertRecord()` 自己完成 copy 后，只释放 insertion lock。
释放动作让等待者能判断这条 insertion 已完成。
随后它更新 backend-local `ProcLastRecPtr` 和 `XactLastRecEnd`。
代码在 `xlog.c:1110-1127`。
`ProcLastRecPtr` 是本 backend 最近 record 的 start pointer。
`XactLastRecEnd` 是当前事务最近 record 的 end pointer。
对调用方来说，最重要的是返回的 `EndPos`。
数据页设置 page LSN 时通常用这个 end pointer。

---


## 15. concurrency：锁拆分的完整图

WAL insertion 路径至少有五个同步域。
第一，backend-local construction 没有 shared lock。
这包括 registered buffers、registered data、scratch header、`XLogRecData` 链。
它发生在 `xloginsert.c`。
第二，`WALInsertLock` 表示 insertion 正在进行。
普通插入只拿 8 个 slot 中的一个。
它还保护 `RedoRecPtr` 和 full page write 状态的读取。
第三，`insertpos_lck` 是极短的全局序列化点。
它只在 reserve address range 时持有。
它不做 memcpy。
它不做 I/O。
它不初始化 WAL buffer page。
第四，`WALBufMappingLock` 保护 WAL buffer slot 到 WAL page 的映射。
它只在初始化或替换 WAL buffer page 时需要。
如果目标 page 已经在 shared WAL buffers 中，copy record 不需要这个锁。
第五，`WALWriteLock` 保护 WAL buffers 到磁盘的 write/fsync。
`XLogWrite()` 和 `XLogFlush()` 使用它。
这个锁可能覆盖实际 I/O。
因此不能在持有它之前等待可能需要它的插入者。
这就是 `WaitXLogInsertionsToFinish()` 必须在拿 `WALWriteLock` 前调用的原因。
还有一个小锁 `info_lck`。
它保护 `XLogCtl->LogwrtRqst` 等共享请求状态。
它是 spinlock。
只用于很短的读写请求位置。
锁拆分带来的结果是：
WAL record 的全局顺序由 reserve 阶段决定。
record bytes 的写入可以并行。
WAL buffer page 初始化是按 page 串行。
磁盘 write/fsync 是另一个串行资源。
flush 等待用 insertion lock 的 `insertingAt` 精确缩小等待范围。

---


## 16. 为什么 `insertingAt` 不是普通进度条

`insertingAt` 的目标不是给监控显示百分比。
它是 deadlock avoidance 和 flush correctness 的一部分。
`WaitXLogInsertionsToFinish()` 位于 `src/backend/access/transam/xlog.c:1531-1654`。
它的输入是 `upto`。
它要保证早于 `upto` 的 WAL insertion 已经完成。
如果 `upto <= logInsertResult`，直接返回。
否则它读取当前 `CurrBytePos`，算出 `reservedUpto`。
如果请求 flush 的位置超过已生成 WAL，它会记录 LOG 并把 `upto` clamp 到 `reservedUpto`。
这通常说明请求来自损坏数据页上的 bogus LSN。
然后它扫描所有 insertion locks。
对每个 lock，它调用 `LWLockWaitForVar()` 等待锁释放或 `insertingAt` 变化。
如果 lock 是 free，说明没有 insertion 在这个 slot 上进行。
如果 lock 仍持有，但 `insertingAt >= upto`，说明这个插入者已经推进到目标之后。
早于 `upto` 的 WAL 对它而言已经安全。
如果 `insertingAt < upto`，就继续等。
小 record 如果不跨 page，通常不会更新 `insertingAt`。
它只是 copy 完后释放 lock。
等待者会通过 lock 释放知道它完成了。
大 record 或遇到未初始化 page 的 record，在可能阻塞前会更新 `insertingAt`。
这样 flush 路径不必等待它后续更靠后的部分。
这不是单纯优化。
源码 `xlog.c:353-358` 说明过死锁风险：
如果所有 WAL buffers 都 dirty，一个插入者持有 insertion lock 时可能需要 flush 旧 WAL buffer。
如果它不必要地等待另一个插入者，而另一个插入者也在等它，就可能死锁。
`insertingAt` 让等待者只等真正覆盖目标范围的 insertion。

---


## 17. flush request 与 insertion 的关系

普通 `XLogInsert()` 不负责 fsync。
但 insertion 路径会更新 write request。
在 `XLogInsertRecord()` 中，如果 record 跨 WAL page，就会更新 `XLogCtl->LogwrtRqst.Write`。
代码在 `src/backend/access/transam/xlog.c:1000-1011`。
这只是请求“这些 WAL page 需要被写出”。
它不是 fsync。
也不是当前 backend 一定会马上写。
`AdvanceXLInsertBuffer()` 复用 WAL buffer slot 时，也会更新 `LogwrtRqst.Write`。
如果旧 page 还没写出，它会自己触发 `XLogWrite()`。
这里的 `WriteRqst.Flush` 是 `InvalidXLogRecPtr`。
也就是说，它只需要 write，不要求 fsync。
这是为了腾出 WAL buffer slot。
commit 或数据页写出需要的是 flush。
`XLogFlush(record)` 位于 `src/backend/access/transam/xlog.c:2801-2975`。
它要求 WAL 最终 flush 到至少 `record`。
它先快速检查本地 `LogwrtResult.Flush`。
如果已经满足，直接返回。
否则进入 critical section。
它把 `WriteRqstPtr` 初始化成目标 record。
循环中先刷新本地 write/flush result。
然后读取 shared `LogwrtRqst.Write`。
如果共享 write request 更靠后，就把 `WriteRqstPtr` 提高。
这让一个 flush 顺带写更多已经请求写出的 WAL。
然后关键步骤是：
```text
insertpos = WaitXLogInsertionsToFinish(WriteRqstPtr)
```
只有确保目标范围内 insertion 完成，才能写 WAL buffers。
之后才尝试拿 `WALWriteLock`。
拿不到时，它先等待 lock 可用，再回头检查是否别人已经完成 flush。
这支持 group commit。
拿到 `WALWriteLock` 后，它可能执行 `CommitDelay`。
延迟后会再次尝试推进 `insertpos`。
但源码注释强调：
一般不能在持有 `WALWriteLock` 时调用可能等待的 `WaitXLogInsertionsToFinish()`。
这里之所以安全，是因为第二次只允许把已经完成的位置往前推进。
最后设置：
```text
WriteRqst.Write = insertpos
WriteRqst.Flush = insertpos
```
然后调用 `XLogWrite()`。
这会把 WAL buffers 写到文件，并在需要时 fsync。

---


## 18. `XLogWrite()` 与 `LogwrtResult`

`XLogWrite()` 位于 `src/backend/access/transam/xlog.c:2320` 之后。
它必须在持有 `WALWriteLock` 时调用。
调用前必须已经调用 `WaitXLogInsertionsToFinish(WriteRqst)`。
这是函数注释直接写出的前置条件。
`XLogWrite()` 使用本地 `LogwrtResult` 作为进度。
`LogwrtResult.Write` 表示已经 write 到哪里。
`LogwrtResult.Flush` 表示已经 fsync 到哪里。
共享版本是 `XLogCtl->logWriteResult` 和 `XLogCtl->logFlushResult`。
它们用 atomic 存储。
每个 backend 有本地 cache。
需要时调用 `RefreshXLogWriteResult()` 刷新。
`XLogWrite()` 循环从当前未写 page 开始。
它读取 `xlblocks[curridx]` 得到当前 WAL buffer page 的 end pointer。
如果写请求超过已初始化 page，会 `PANIC`。
这是内部一致性错误。
它尽量把连续 WAL buffer pages 合并成一次 write。
到 segment 末尾时会立即 fsync segment。
这是为了避免后续 fsync request 还要重新打开旧 segment。
如果 `WriteRqst.Flush` 要求 fsync，且 `LogwrtResult.Flush < WriteRqst.Flush`，它会调用 `issue_xlog_fsync()`。
最后它更新 shared request/result。
写 shared result 时先写 `logWriteResult`。
然后 memory barrier。
再写 `logFlushResult`。
读者要用相反顺序和匹配 barrier，避免看到 flush 超过 write 的不一致视图。
这说明 `LogwrtResult` 是持久化边界。
它和 `CurrBytePos` 不是同一类状态。
`CurrBytePos` 说“地址分配到哪里”。
`logInsertResult` 说“字节拷贝完成到哪里”。
`logWriteResult` 说“文件 write 到哪里”。
`logFlushResult` 说“fsync 到哪里”。

---


## 19. WAL buffers 的大小与复用压力

`XLOGbuffers` 是 WAL buffers 的页数。
本基线中自动选择逻辑在 `src/backend/access/transam/xlog.c:5012-5033`。
默认按 `shared_buffers / 32` 估算。
上限是一个 WAL segment 的页数。
下限是 8 blocks。
这不是说 WAL buffers 越大越安全。
WAL correctness 不依赖“大到不复用”。
正确性来自：
- 复用 slot 前旧 page 必须写出。
- 写出前相关 insertion 必须完成。
- page 初始化和 `xlblocks` 更新有 memory barrier。
- flush 前必须等待 insertion 并持有 `WALWriteLock`。
WAL buffers 太小会增加复用压力。
插入者更容易在 `GetXLogBuffer()` / `AdvanceXLInsertBuffer()` 中发现旧 page 还没写出。
这会增加前台 backend 自己写 WAL buffer 的概率。
统计上会表现为 `wal_buffers_full` 增加。
但这属于性能问题，不是正确性漏洞。

---


## 20. 错误边界

WAL insertion 路径处在很多 critical section 旁边。
理解错误边界很重要。
`XLogInsert()` 没有先调用 `XLogBeginInsert()` 会 `ERROR`。
位置在 `xloginsert.c:486-488`。
这通常是调用方 bug。
`XLogInsert()` 的 `info` mask 不合法会 `PANIC`。
位置在 `xloginsert.c:490-497`。
这是内部不变量损坏。
`XLogInsertRecord()` 在 recovery 中被调用会 `ERROR`。
位置在 `xlog.c:814-816`。
正常 recovery 不能生成新的 WAL entry。
full page write 状态变化导致 `XLogInsertRecord()` 返回 `InvalidXLogRecPtr`。
这不是错误。
它是让 `XLogInsert()` 重新 assemble record 的协议。
`ReserveXLogInsertLocation()` 中多数是 `Assert`。
例如普通 record size 必须大于 `SizeOfXLogRecord`。
这些是开发期不变量。
`GetXLogBuffer()` 初始化后仍找不到目标 WAL buffer，会 `PANIC`。
位置在 `xlog.c:1748-1750`。
这意味着 WAL buffer mapping 严重不一致。
`WaitXLogInsertionsToFinish()` 发现 flush 请求超过已 reserve WAL，会记录 LOG 并 clamp。
位置在 `xlog.c:1571-1585`。
注释说明这可能来自磁盘数据页上的 bogus LSN。
`XLogFlush()` 最后如果仍没 flush 到请求点，会 `ERROR`。
位置在 `xlog.c:2944-2969`。
这在 `bufmgr.c` 从脏页 page LSN 触发 flush 时，不一定导致全库 PANIC。
但如果调用方已经在 critical section，比如事务提交路径，`ERROR` 可能升级为 `PANIC`。
`XLogWrite()` 写 WAL 文件失败会 `PANIC`。
WAL write/fsync 失败是持久化系统无法继续安全运行的边界。
`AdvanceXLInsertBuffer()` 为避免死锁，会在需要写旧 WAL buffer 前释放 `WALBufMappingLock`。
如果这里错误地持锁等待，就可能让插入者和写出者互相阻塞。
这是锁顺序边界，不是错误处理代码。

---


## 21. 一条普通 heap insert 的位置感

本节不展开 heap。
但为了把 WAL insertion 放回上下文，可以记住典型调用顺序。
heap 修改 page。
调用方 `MarkBufferDirty()`。
调用方注册 WAL 数据和 buffer。
调用 `XLogInsert()`。
`XLogInsert()` 返回 record end pointer。
调用方把这个 end pointer 写入 page LSN。
之后数据页可以留在 shared buffers 中。
真正写出数据页时，buffer manager 会读 page LSN，并调用 `XLogFlush(page_lsn)`。
这就是 WAL-before-data。
本节研究的是 `XLogInsert()` 里面 record 如何进入 WAL buffers。
下一节会更系统地看 page LSN 与 data page write 的持久化顺序。

---


## 22. 源码跟读路线

第一步，读 `src/backend/access/transam/xloginsert.c:482-540`。
确认 `XLogInsert()` 的入口检查、assemble 循环和返回值。
重点看 `do ... while (!XLogRecPtrIsValid(EndPos))`。
第二步，读 `xloginsert.c:607-700`。
确认 `fpw_lsn` 为什么会让 record assemble 依赖 `RedoRecPtr`。
重点看 page LSN 与 full page image 的判断。
第三步，读 `src/backend/access/transam/xlog.c:338-452`。
确认 `WALInsertLock` 和 `XLogCtlInsert` 的字段。
重点看 `insertingAt` 注释和 `CurrBytePos` 注释。
第四步，读 `xlog.c:784-1011`。
确认 `XLogInsertRecord()` 的普通路径和特殊路径。
重点看拿一个 insertion lock 与拿所有 insertion locks 的差异。
第五步，读 `xlog.c:1130-1193`。
确认 reserve 阶段只在 spinlock 内更新 byte positions。
重点看 `StartPos`、`EndPos`、`PrevPtr` 的转换。
第六步，读 `xlog.c:1266-1400`。
确认 `CopyXLogRecordToWAL()` 如何跨 WAL page。
重点看 `xlp_rem_len`、`XLP_FIRST_IS_CONTRECORD` 和 short/long header skip。
第七步，读 `xlog.c:1656-1771`。
确认 `GetXLogBuffer()` 如何判断目标 page 是否已经在 WAL buffers 中。
重点看为什么 page header 后的位置要把 `insertingAt` 调回 page 开头。
第八步，读 `xlog.c:2026-2185`。
确认 `AdvanceXLInsertBuffer()` 何时写旧 WAL buffer、何时初始化新 page。
重点看释放 `WALBufMappingLock` 再拿 `WALWriteLock` 的锁顺序。
第九步，读 `xlog.c:1531-1654`。
确认 `WaitXLogInsertionsToFinish()` 如何扫描 insertion locks。
重点看 `LWLockWaitForVar()` 的等待条件。
第十步，读 `xlog.c:2801-2975`。
确认 `XLogFlush()` 为什么先等 insertion，再拿 `WALWriteLock`。
重点看 group commit 和最终 unsatisfied flush 的错误边界。

---


## 23. 跟读练习一：画出 reserve 与 copy

选择 `ReserveXLogInsertLocation()`。
在纸上画三条 record：
```text
R1 size = 100
R2 size = 200
R3 size = 300
```
假设初始 `CurrBytePos = C0`，`PrevBytePos = P0`。
依次写出每次 reserve 后：
- `startbytepos`
- `endbytepos`
- `prevbytepos`
- 新 `CurrBytePos`
- 新 `PrevBytePos`
然后回答：
`R2.xl_prev` 指向哪里。
`R3.xl_prev` 指向哪里。
为什么 `xl_prev` 是上一条 record 的 start，而不是 end。
再把题目改成两个 backend 并发。
Backend A 拿到 insertion lock 后 reserve R1。
Backend B 拿到另一个 insertion lock 后 reserve R2。
Backend B 先 copy 完。
这是否破坏 WAL record 顺序。
答案是不破坏。
顺序由 reserve 阶段写入的 address range 和 `xl_prev` 决定。
但 flush R2 end 之前，仍然要等待 R1 涉及的早期 insertion 完成。

---


## 24. 跟读练习二：追踪跨 page record

选择 `CopyXLogRecordToWAL()`。
假设 `CurrPos` 当前 page 只剩 40 bytes。
当前 `XLogRecData` 节点还有 100 bytes。
跟着源码回答：
前 40 bytes 写到哪里。
`written` 如何变化。
`CurrPos` 如何变化。
下一页 header 的 `xlp_rem_len` 设置成多少。
`XLP_FIRST_IS_CONTRECORD` 什么时候设置。
为什么设置 header 后要跳过 `SizeOfXLogShortPHD` 或 `SizeOfXLogLongPHD`。
最后确认：
`written` 统计的是 record bytes。
它不统计 WAL page header bytes。
`CurrPos` 是真实 WAL address。
它会跳过 WAL page header。
这就是 `written` 和 `CurrPos - StartPos` 不总是相等的原因。

---


## 25. 跟读练习三：解释一个等待

选择 `WaitXLogInsertionsToFinish(upto)`。
假设系统有 8 个 insertion locks。
其中 lock 2 正被 backend A 持有，`insertingAt = 0`。
lock 5 正被 backend B 持有，`insertingAt = 0/5000`。
其他 locks 空闲。
如果 `upto = 0/4000`，flush 路径要等谁。
它必须等 A。
因为 A 的 `insertingAt = 0`，表示不知道 A 正在插入哪里。
它不需要等 B。
因为 B 已经广告自己推进到 `0/5000`。
早于 `0/4000` 的 WAL 对 B 来说已经完成。
如果 backend A 是小 record，它可能永远不更新 `insertingAt`。
它 copy 完后释放 lock。
等待者会通过 lock release 继续。
这个练习的重点是：
`insertingAt = 0` 不是“插入在 LSN 0”。
它表示“持锁但尚未公布可安全跳过的位置”。

---


## 26. 实验一：只用源码验证锁拆分

在 `/home/nail/postgres-lab` 中运行：
```bash
rg -n "WALInsertLockAcquire|ReserveXLogInsertLocation|CopyXLogRecordToWAL|WALBufMappingLock|WALWriteLock" src/backend/access/transam/xlog.c
```
把输出按调用路径排序。
你应该能看到：
- `XLogInsertRecord()` 调用 `WALInsertLockAcquire()`。
- `XLogInsertRecord()` 调用 `ReserveXLogInsertLocation()`。
- `XLogInsertRecord()` 调用 `CopyXLogRecordToWAL()`。
- `GetXLogBuffer()` 在缺页时调用 `AdvanceXLInsertBuffer()`。
- `AdvanceXLInsertBuffer()` 使用 `WALBufMappingLock`。
- `AdvanceXLInsertBuffer()` 必要时使用 `WALWriteLock`。
- `XLogFlush()` 使用 `WALWriteLock`。
然后回答：
如果所有 WAL 插入都被一把全局锁包住，哪些锁可以消失。
PostgreSQL 为什么没有这么做。
预期答案：
全局锁可以让 `insertingAt` 和部分等待逻辑变简单。
但它会让所有 backend 的 record copy 串行化。
当前实现只让 reserve 串行，把 copy 和 write/flush 分离。

---


## 27. 实验二：观察 WAL page header 边界

这个实验不需要编译。
在源码中打开：
```bash
nl -ba src/include/access/xlog_internal.h | sed -n '32,84p'
nl -ba src/backend/access/transam/xlog.c | sed -n '1893,1976p'
```
对照写出：
- short header 包含哪些字段。
- long header 多了哪些字段。
- 为什么 segment 第一页需要 long header。
- `XLogBytePosToRecPtr()` 如何跳过 header。
- `XLogBytePosToEndRecPtr()` 在 page boundary 上为什么不跳过 header。
然后回到 `CopyXLogRecordToWAL()`。
检查跨 page 时使用的是哪一个 header size。
最后确认：
record continuation page 的第一段不是一条新 record。
它是上一条 record 的剩余字节。
因此需要 `XLP_FIRST_IS_CONTRECORD` 和 `xlp_rem_len`。

---


## 28. 实验三：构建后用 gdb 看一次插入

如果你已经在另一个 build 目录编译了 PostgreSQL，可以做这个实验。
本课程文件不要求你在源码树里修改任何文件。
启动一个临时实例。
在一个 backend 上用 gdb 设置断点：
```gdb
break XLogInsert
break XLogInsertRecord
break ReserveXLogInsertLocation
break CopyXLogRecordToWAL
break XLogFlush
```
执行一条简单写入：
```sql
create table wal_insert_demo(id int primary key, v text);
insert into wal_insert_demo values (1, 'a');
```
观察：
- `XLogInsert()` 可能被调用多次。
- 不同 rmgr 会生成不同 record。
- `ReserveXLogInsertLocation()` 中的 `StartPos` 和 `EndPos` 是 record 的地址范围。
- `CopyXLogRecordToWAL()` 中的 `CurrPos` 可能和 `StartPos` 同页。
- 提交时或数据页写出时才会进入需要 flush 的路径。
如果断点太多，可以只保留：
```gdb
break XLogInsertRecord
break ReserveXLogInsertLocation
commands
  silent
  continue
end
```
然后逐步减少噪音。
这个实验的目标不是记住某条 SQL 产生几条 WAL。
目标是确认：
reserve 和 copy 是分开的。
`XLogInsert()` 返回 end pointer。
flush 是另一条路径。

---


## 29. 讨论题

1. 为什么 WAL insertion 只把 reserve 做成全局短临界区，而不是让所有 backend 串行完成 reserve、copy 和 flush？
2. `CurrBytePos` 已经推进时，为什么 flush 仍必须通过 insertion locks 确认目标范围内的 copy 已完成？
3. `insertingAt = 0` 表示什么？为什么它不是一个真实 LSN，也不能被当作“无需等待”？
4. WAL record 跨 WAL page 时，为什么 continuation page 必须跳过 page header，并记录 `xlp_rem_len`？
5. `wal_buffers_full` 上升通常说明什么资源压力？它为什么是性能信号，而不是 WAL correctness 信号？
6. 如果 `wal_buffers` 很小，`AdvanceXLInsertBuffer()` 为什么可能把前台插入路径拖进 WAL write？
7. `WALInsertLock`、`WALBufMappingLock` 和 `WALWriteLock` 分别保护哪一层状态？哪一个不能用来解释事务 durability？
8. 本节的可迁移规律是什么：怎样把全局有序 byte stream 拆成短串行点和可并行 copy 阶段？

---


## 30. 常见误区

误区一：
`XLogInsert()` 会把 WAL fsync 到磁盘。
不对。
它把 record 插入 WAL buffers，并返回 record end pointer。
是否 flush 取决于调用方和事务/数据页写出路径。
误区二：
`CurrBytePos` 表示所有 WAL 都已经可以写出。
不对。
它表示 WAL address space 已 reserve 到哪里。
可写出边界要看 insertion locks 和 `logInsertResult`。
误区三：
WAL insertion lock 保护 `CurrBytePos`。
不对。
`CurrBytePos` 由 `insertpos_lck` 保护。
WAL insertion lock 保护 insertion-in-progress 状态和 full page write 相关状态读取。
误区四：
record 跨 WAL page 时，下一页从 offset 0 继续写 record。
不对。
下一页 offset 0 是 WAL page header。
record continuation 要跳过 short 或 long page header。
误区五：
flush 只要看 `LogwrtRqst.Write`。
不对。
write request 只是请求。
真正写之前必须等 insertion 完成。
真正 fsync 完成要看 `LogwrtResult.Flush`。
误区六：
多个 insertion locks 会改变 WAL record 的全局顺序。
不对。
全局顺序由 reserve 出来的 address range 和 `xl_prev` 决定。
多个 insertion locks 只是允许 copy 阶段并发。

---


## 31. 本节小结

WAL insertion 是一个拆分过的并发协议。
`XLogInsert()` 负责把 caller 登记的数据组装成 WAL record，并调用 `XLogInsertRecord()`。
`XLogInsertRecord()` 在 insertion lock 保护下确认 full page write 状态没有破坏 record 正确性。
真正的全局序列化点是 `ReserveXLogInsertLocation()`。
它只更新 `CurrBytePos` 和 `PrevBytePos`。
它把 record 分配到唯一的 WAL address range。
record 字节的 copy 发生在 `CopyXLogRecordToWAL()`。
不同 backend 的 copy 可以并行，因为它们的 address range 不重叠。
WAL page 边界由 short/long page header、`XLP_FIRST_IS_CONTRECORD` 和 `xlp_rem_len` 处理。
WAL buffers 是共享内存中的 WAL page cache。
`GetXLogBuffer()` 根据 LSN 找 slot。
目标 page 不在 buffer 中时，`AdvanceXLInsertBuffer()` 初始化新 page。
如果要复用的旧 page 还没写出，它会先等待相关 insertion 完成并写旧 page。
`WALInsertLock` 的关键价值是告诉 flush 路径哪些 insertion 仍在进行。
`insertingAt` 让等待者只等目标范围内的插入。
这既减少等待，也避免某些 WAL buffer 满时的死锁。
flush 是另一条边界。
`XLogFlush()` 必须先调用 `WaitXLogInsertionsToFinish()`，再拿 `WALWriteLock` 写出并 fsync。
因此 WAL 插入、WAL 写出、WAL flush 是三个不同层次。
把这三个层次分清，是理解 WAL-before-data、commit latency、walwriter 和 checkpoint 的前提。
