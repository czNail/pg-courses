# PostgreSQL Dirty buffer、page LSN 与 WAL-before-data

## 课程定位

前置知识：已经理解 buffer pin、content lock、`BM_DIRTY`、page header 的存在，也知道 WAL-before-data 是 PostgreSQL crash safety 的核心边界。

本节唯一主问题：

```text
一个被修改的 page 如何从内存中的 dirty 状态，变成可以安全写回 relation 文件的状态？
```

核心矛盾：前台修改希望只留下 cheap dirty 标记并继续执行；但任何后续写出都必须证明 WAL 已经先于 data page 持久化。

一句话运行模型：

```text
访问方法修改 page 后调用 MarkBufferDirty() 标记 buffer；随后 XLogInsert() 返回 record end LSN，调用方把它写入 page header；后续 FlushBuffer() 写出永久数据页前读取 page LSN，并先 XLogFlush(page_lsn)，再写 data page。
```

学完后应能判断：`BM_DIRTY`、page LSN、WAL flush LSN 三者为什么分离；`MarkBufferDirty()` 为什么不生成 WAL；`MarkBufferDirtyHint()` 为什么有独立边界；当前基线为什么没有 `BM_JUST_DIRTIED`。

源码基线：本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`。

## 1. 本节在总主线中的位置

前面几节让 page 进入 shared buffer，并让访问方法可以持有 pin/content lock 修改它。本节开始回答“修改以后如何安全落盘”。它是后续 eviction、checkpoint、WAL record 和 FPI 课程的共同前提。

本节不完整展开 WAL record 格式，只先建立 dirty buffer、page LSN 和 WAL flush 的关系。后面讲 WAL record、WAL insertion、commit flush 时会反复回到这个不变量。

## 2. 核心矛盾与一句话运行模型

这里不能把“脏页”理解成一个布尔值。PostgreSQL 至少同时维护三类状态：buffer 是否 dirty、page header 记录的最近 WAL record end LSN、WAL 子系统已经 flush 到的位置。

核心顺序可以压成这条链：

```text
access method 修改 page
  -> MarkBufferDirty()
  -> XLogInsert() 返回 record end LSN
  -> PageSetLSN(page, recptr)
  -> FlushBuffer() 读取 page LSN
  -> XLogFlush(page_lsn)
  -> data page write
```

这使普通修改可以低成本返回，而真正写出数据页时仍守住 WAL-before-data。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节读什么 |
| --- | --- | --- |
| 1 | `src/backend/storage/buffer/bufmgr.c` | `MarkBufferDirty()`、`MarkBufferDirtyHint()`、`FlushBuffer()`、dirty flag 与 I/O 清理。 |
| 2 | `src/include/storage/buf_internals.h` | `BM_DIRTY`、`BM_VALID`、`BM_IO_IN_PROGRESS`、`BM_CHECKPOINT_NEEDED`、`BM_PERMANENT`，以及当前基线没有 `BM_JUST_DIRTIED` 的事实。 |
| 3 | `src/include/storage/bufpage.h` | `pd_lsn`、`PageGetLSN()`、`PageSetLSN()`。 |
| 4 | `src/backend/access/transam/xloginsert.c` | `XLogInsert()` / `XLogInsertRecord()` 返回 record end LSN 的语义。 |
| 5 | `src/backend/access/transam/xlog.c` | `XLogFlush()` 和 WAL flush position。 |
| 6 | `src/backend/access/transam/README` | WAL-before-data 的总体说明。 |
| 7 | `src/backend/access/heap/heapam.c`、`src/backend/access/nbtree/nbtinsert.c`、`src/backend/access/heap/heapam_visibility.c` | 典型调用点：heap、btree、hint bit 如何完成 dirty + WAL + page LSN 协议。 |

## 4. 三个容易混淆的“脏”

阅读这节课时，先把三个层次分开：
1. buffer 是否 dirty。
2. page 的 LSN 是多少。
3. WAL 已经 flush 到哪里。
buffer 是否 dirty 存在 `BufferDesc.state` 的 flag 位里。
这个状态服务 buffer manager。
它回答的问题是：这个 shared buffer 里的内容是否需要写回 relation 文件。
page LSN 存在 page header 的 `pd_lsn` 里。
这个状态服务 crash safety。
它回答的问题是：如果这个 page 现在落盘，至少需要 WAL 持久化到哪个位置，恢复时才有足够日志重做它。
WAL flush LSN 存在 WAL 子系统的写入状态里。
这个状态服务 durability。
它回答的问题是：WAL byte stream 中哪些字节已经真正被 fsync 到可恢复介质。
三者的关系可以写成一句话：
只有 dirty page 需要被写出；写出前必须让 WAL flush LSN >= page LSN。
注意这里不是“dirty 时就 flush WAL”。
大多数普通修改只是把 page 留在 shared buffers 里。
真正需要把 buffer 写到 relation 文件时，`FlushBuffer()` 才会检查 page LSN，并触发 `XLogFlush()`。
这使数据页写回可以延迟、合并、由 bgwriter/checkpointer/backend 任何一方执行。
同时又不会破坏 WAL-before-data。

## 5. BufferDesc flags：当前基线的事实

先读 `src/include/storage/buf_internals.h`。
本基线中 buffer state 的 flag 定义集中在 `buf_internals.h:95-127`。
关键位如下：
- `BM_LOCKED`：buffer header 自身正在被锁住。
- `BM_DIRTY`：data needs writing。
- `BM_VALID`：buffer 中的 page 内容有效。
- `BM_TAG_VALID`：buffer tag 已关联到 buffer mapping table。
- `BM_IO_IN_PROGRESS`：read 或 write I/O 正在进行。
- `BM_IO_ERROR`：上一次 I/O 失败。
- `BM_CHECKPOINT_NEEDED`：该 dirty buffer 属于当前 checkpoint 需要写出的集合。
- `BM_PERMANENT`：这个 buffer 属于 crash 后仍应存在的永久数据。
源码中对应定义：

```c
#define BM_DIRTY                 BUF_DEFINE_FLAG( 1)
#define BM_VALID                 BUF_DEFINE_FLAG( 2)
#define BM_IO_IN_PROGRESS        BUF_DEFINE_FLAG( 4)
#define BM_CHECKPOINT_NEEDED     BUF_DEFINE_FLAG( 8)
#define BM_PERMANENT             BUF_DEFINE_FLAG( 9)
```

本基线没有：

```c
BM_JUST_DIRTIED
```

`buf_internals.h` 在原来 bit 6 的位置写的是：

```c
/* flag bit 6 is not used anymore */
```

这不是遗漏。
这是本节需要特别强调的版本事实。
如果你读过旧资料，会看到 `BM_JUST_DIRTIED` 参与 dirty/write race 处理。
但在 `master @ bd4bd30...` 中，它已经被删除。
本地 git 历史里，删除提交是：

```text
b0f4ff3c92 bufmgr: Remove the, now obsolete, BM_JUST_DIRTIED
```

该提交说明删除原因：
- 现在 setting hint bits 和 flushing pages 都使用 share-exclusive 模式。
- buffer 在 flush 进行中不能再被 dirtied。
- 旧的 `BM_JUST_DIRTIED` 是为了处理 flush 过程中又被 dirtied 的情况。
- 这个情况在当前锁协议下不再可能。
历史上加入 `BM_JUST_DIRTIED` 的提交是：

```text
9ff69034b2 Fixing possible losing data changes
```

它当时解决的问题是：
- dirtier 同时设置 `BM_DIRTY` 和 `BM_JUST_DIRTIED`。
- flusher 在写出前清掉 `BM_JUST_DIRTIED`。
- flusher 写完后如果发现 `BM_JUST_DIRTIED` 又被置上，就不能清 `BM_DIRTY`。
- 这样避免把“写出期间产生的新修改”错误标成 clean。
所以本节对 `BM_JUST_DIRTIED` 的覆盖方式是：
- 当前基线：不存在，不参与任何当前逻辑。
- 历史意义：曾是重脏页 race 的保护位。
- 当前替代：content lock 等级提高，flush 和 hint dirtier 不能并行修改同一 page。

## 6. `MarkBufferDirty()` 的真实职责

`MarkBufferDirty()` 位于 `src/backend/storage/buffer/bufmgr.c:3147-3205`。
它的注释非常短：

```c
Marks buffer contents as dirty (actual write happens later).
```

这句话的重点是括号里的部分。
`MarkBufferDirty()` 不写 page。
`MarkBufferDirty()` 不 fsync。
`MarkBufferDirty()` 不插入 WAL。
`MarkBufferDirty()` 不设置 page LSN。
它只是让 buffer descriptor 上的 `BM_DIRTY` 变成 1。
入口检查也很关键：

```c
if (!BufferIsValid(buffer))
    elog(ERROR, "bad buffer ID: %d", buffer);
```

无效 buffer ID 是编程错误。
local buffer 单独处理：

```c
if (BufferIsLocal(buffer))
{
    MarkLocalBufferDirty(buffer);
    return;
}
```

shared buffer 需要满足两个断言：

```c
Assert(BufferIsPinned(buffer));
Assert(BufferIsLockedByMeInMode(buffer, BUFFER_LOCK_EXCLUSIVE));
```

pin 防止 buffer slot 被替换。
exclusive content lock 防止别人在你修改 page 内容时读写或写出它。
`MarkBufferDirty()` 的核心 CAS loop 是：

```c
old_buf_state = pg_atomic_read_u64(&bufHdr->state);
for (;;)
{
    if (old_buf_state & BM_LOCKED)
        old_buf_state = WaitBufHdrUnlocked(bufHdr);
    buf_state = old_buf_state;
    Assert(BUF_STATE_GET_REFCOUNT(buf_state) > 0);
    buf_state |= BM_DIRTY;
    if (pg_atomic_compare_exchange_u64(&bufHdr->state, &old_buf_state,
                                       buf_state))
        break;
}
```

这里有几个细节：
- 它操作的是 `BufferDesc.state`，不是 page 内容。
- 它等待 `BM_LOCKED`，因为 `TerminateBufferIO()` 依赖 header lock 的一致性。
- 它要求 refcount 大于 0，也就是 buffer 至少被 pin。
- 它只 OR 上 `BM_DIRTY`。
- 如果 CAS 失败，`old_buf_state` 会被 compare-exchange 更新成最新值，然后重试。
成功后做 accounting：

```c
if (!(old_buf_state & BM_DIRTY))
{
    pgBufferUsage.shared_blks_dirtied++;
    if (VacuumCostActive)
        VacuumCostBalance += VacuumCostPageDirty;
}
```

只有从 clean 变 dirty 的第一次调用才计入 `shared_blks_dirtied`。
如果这个 buffer 已经 dirty，再次调用 `MarkBufferDirty()` 不重复计数。
这一点对理解统计视图很重要。
dirty 次数不是 page 内部每次修改次数。
dirty 次数更接近 clean-to-dirty transition 的次数。

## 7. local buffer 的 dirty 路径

`MarkBufferDirty()` 遇到 local buffer 会转到 `MarkLocalBufferDirty()`。
源码在 `src/backend/storage/buffer/localbuf.c:496-526`。
local buffer 只属于当前 backend。
所以它不需要 shared buffer 那套 content lock 和 CAS retry。
核心逻辑是：

```c
buf_state = pg_atomic_read_u64(&bufHdr->state);
if (!(buf_state & BM_DIRTY))
    pgBufferUsage.local_blks_dirtied++;
buf_state |= BM_DIRTY;
pg_atomic_unlocked_write_u64(&bufHdr->state, buf_state);
```

local buffer dirty 仍然表示需要写临时 relation 文件。
但它不参与 crash recovery 的 WAL-before-data。
临时关系 crash 后不需要恢复。
所以本节后面讨论 `FlushBuffer()` 的 WAL flush 规则时，重点是 shared buffer 和 permanent buffer。

## 8. 修改 page 的标准顺序

`src/backend/access/transam/README:440-469` 给出标准顺序。
这个顺序是理解本节的主线：
1. 在进入 critical section 前完成会抛错的准备工作。
2. `START_CRIT_SECTION()`。
3. 修改 shared buffer 中的 page 内容。
4. 调用 `MarkBufferDirty()`。
5. 如果关系需要 WAL，构造并插入 WAL record。
6. 用 `XLogInsert()` 返回的 LSN 设置 page LSN。
7. `END_CRIT_SECTION()`。
8. unlock/unpin buffer。
README 特别说：

```c
MarkBufferDirty() must happen before the WAL record is inserted
```

原因指向 `SyncOneBuffer()`。
这不是随便的风格要求。
这是 checkpoint 并发正确性要求。
标准代码形状是：

```c
START_CRIT_SECTION();
/* modify page */
MarkBufferDirty(buffer);
XLogBeginInsert();
XLogRegisterBuffer(...);
XLogRegisterData(...);
recptr = XLogInsert(rmgr_id, info);
PageSetLSN(page, recptr);
END_CRIT_SECTION();
```

注意 `MarkBufferDirty()` 在 `XLogInsert()` 之前。
注意 `PageSetLSN()` 在 `XLogInsert()` 之后。
这意味着存在一个短窗口：
- page 内容已经改了。
- buffer 已经 dirty。
- WAL record 还没插入。
- page LSN 还没更新。
这个窗口为什么安全？
因为 caller 仍持有 exclusive content lock。
`FlushBuffer()` 写出 page 需要至少 share-exclusive content lock。
所以 flusher 不能在 caller 完成 critical section 前把这个 page 写出去。
把 `BM_DIRTY` 提前设置的意义，不是让 page 立刻可以写。
而是让 checkpoint 扫描能看见“这个 buffer 已经进入需要考虑的 dirty 集合”。
如果反过来先插入 WAL、后 mark dirty，则可能出现危险窗口：
- WAL 已插入。
- checkpoint 已开始并确定 redo point。
- buffer 还没 dirty。
- checkpoint 扫描时漏掉这个 buffer。
- 后续 checkpoint record 可能声称 redo point 之后的内容已安全覆盖。
当前协议用“先 dirty，再 WAL”封住这个窗口。

## 9. `SyncOneBuffer()` 为什么依赖先 dirty 后 WAL

`SyncOneBuffer()` 位于 `src/backend/storage/buffer/bufmgr.c:4124-4198`。
它是 bgwriter/checkpointer 处理单个 buffer 的路径。
它先在不拿 content lock 的情况下检查 `BM_DIRTY`：

```c
buf_state = LockBufHdr(bufHdr);
if (!(buf_state & BM_VALID) || !(buf_state & BM_DIRTY))
{
    UnlockBufHdr(bufHdr);
    return result;
}
```

源码注释解释了为什么可以这么做：

```c
We can make this check without taking the buffer content lock so long
as we mark pages dirty in access methods before logging changes with
XLogInsert()
```

也就是说：
如果 `SyncOneBuffer()` 刚检查完发现 clean，随后别人把 page dirty 了，那也没关系。
因为这个 dirty 发生在对应 WAL record 插入之前。
当前 checkpoint 的 redo point 会在那条 WAL record 之前。
因此当前 checkpoint 不要求写出这个刚 dirty 的 buffer。
这就是 `MarkBufferDirty()` 顺序背后的并发证明。
很多人误以为 dirty bit 只是性能优化。
在 PostgreSQL 里，它同时参与 checkpoint 边界的正确性协议。

## 10. page LSN 的数据结构

page LSN 在 `src/include/storage/bufpage.h`。
`PageHeaderData` 中第一项是：

```c
PageXLogRecPtr pd_lsn;
```

注释是：

```c
LSN: next byte after last byte of xlog record for last change to this page
```

这句话很精确。
page LSN 不是 WAL record 的 start pointer。
它是“最后修改这个 page 的 WAL record 结束位置”。
`XLogInsert()` / `XLogInsertRecord()` 返回的正是 end pointer。
`PageGetLSN()` 和 `PageSetLSN()` 在 `bufpage.h:410-420`：

```c
static inline XLogRecPtr
PageGetLSN(const PageData *page)
{
    return PageXLogRecPtrGet(&((const PageHeaderData *) page)->pd_lsn);
}
static inline void
PageSetLSN(Page page, XLogRecPtr lsn)
{
    PageXLogRecPtrSet(&((PageHeader) page)->pd_lsn, lsn);
}
```

它们只是读写 page header。
它们不做 WAL flush。
它们不检查 `BM_DIRTY`。
它们不拿锁。
锁由调用方保证。
`src/backend/access/transam/README:620-626` 明确要求：
- 只有在 action 已序列化时才能用 `PageSetLSN()` / `PageGetLSN()`。
- startup 进程在 recovery 中是唯一修改 data block 的进程，所以可以安全使用。
- 其他进程需要持有 exclusive buffer lock。
- 或者持有 shared lock 加 buffer header lock。
- 或者直接写数据块且持有 relation 的 AccessExclusiveLock。
这也是 `BufferGetLSNAtomic()` 存在的原因。

## 11. WAL record LSN：为什么是 end pointer

`src/backend/access/transam/xloginsert.c:470-480` 注释说：
`XLogInsert()` 返回 XLOG pointer to end of record。
这个返回值可以作为 affected data pages 的 LSN。
原因是：
page 写出前，需要 WAL flushed through that point。
“through that point” 对应 record 的 end pointer。
如果只 flush 到 record start pointer，record 内容还没有完整持久化。
恢复时可能读不到完整 record。
所以 page LSN 必须是 end pointer。
`XLogInsertRecord()` 在 `src/backend/access/transam/xlog.c:754-781` 有同样说明。
它是低层插入函数。
高层 `xloginsert.c` 负责 assemble record。
低层 `xlog.c` 负责 reserve WAL space、copy record、维护 insert locks 和 WAL buffers。
标准路径是：

```c
XLogBeginInsert();
XLogRegisterBuffer(0, buffer, REGBUF_STANDARD);
XLogRegisterBufData(0, ...);
recptr = XLogInsert(RM_HEAP_ID, info);
PageSetLSN(page, recptr);
```

`recptr` 是 record end pointer。
这就是 page LSN。

## 12. 典型调用点：heap insert

看 `src/backend/access/heap/heapam.c:2140-2164`。
heap insert 的 WAL 部分大致是：

```c
XLogBeginInsert();
XLogRegisterData(&xlrec, SizeOfHeapInsert);
XLogRegisterBuffer(0, buffer, REGBUF_STANDARD | bufflags);
XLogRegisterBufData(0, &xlhdr, SizeOfHeapHeader);
XLogRegisterBufData(0, tuple_data, tuple_len);
XLogSetRecordFlags(XLOG_INCLUDE_ORIGIN);
recptr = XLogInsert(RM_HEAP_ID, info);
PageSetLSN(page, recptr);
```

前面的同一 critical section 中，page 已经被实际插入 tuple，并且 buffer 已被 `MarkBufferDirty()`。
这个调用点体现了三层状态的分工：
- page 内容由 heap AM 修改。
- buffer dirty bit 由 `MarkBufferDirty()` 设置。
- WAL record 由 xloginsert 构造并插入。
- page LSN 由 caller 用 `XLogInsert()` 返回值回填。
`MarkBufferDirty()` 不知道这是 heap insert。
`XLogInsert()` 不知道这个 buffer 以后什么时候写盘。
`FlushBuffer()` 不知道 heap tuple 的语义。
正确性来自它们之间的协议。

## 13. 典型调用点：多 page 修改

heap multi insert 在 `src/backend/access/heap/heapam.c:2574-2592`。
它可能同时修改 heap page 和 visibility map page。
代码形状是：

```c
XLogRegisterBuffer(0, buffer, REGBUF_STANDARD | bufflags);
if (all_frozen_set)
    XLogRegisterBuffer(1, vmbuffer, 0);
recptr = XLogInsert(RM_HEAP2_ID, info);
PageSetLSN(page, recptr);
if (all_frozen_set)
    PageSetLSN(BufferGetPage(vmbuffer), recptr);
```

一条 WAL record 可以描述多个 page 的变化。
这些 page 的 LSN 都设置为同一个 record end pointer。
这表示任何一个 page 单独落盘前，都要求这条 WAL record 已经 flushed。
这并不要求多个 page 同时落盘。
page 可以各自独立写出。
但它们共享同一个 WAL-before-data 下界。

## 14. 典型调用点：btree insert 与 fake LSN

看 `src/backend/access/nbtree/nbtinsert.c:1409-1419`。
btree insert WAL 路径：

```c
recptr = XLogInsert(RM_BTREE_ID, xlinfo);
...
PageSetLSN(page, recptr);
```

如果 relation 不需要 WAL，代码可能走：

```c
recptr = XLogGetFakeLSN(rel);
```

fake LSN 的存在说明：
page LSN 不只用于 crash recovery。
一些 index AM 也用 LSN 检测并发 page 修改。
例如 btree scan hinting 会记录 page LSN，之后再读 `BufferGetLSNAtomic()`，如果 LSN 变了就放弃 hint。
`src/backend/access/nbtree/nbtutils.c:243-248` 就有这样的用法。
因此，非 WAL-logged relation 上也可能有 page LSN。
但这类 LSN 不能拿去要求 WAL flush。
这正是 `FlushBuffer()` 只在 `BM_PERMANENT` 时调用 `XLogFlush()` 的原因。

## 15. `FlushBuffer()` 的主线

`FlushBuffer()` 位于 `src/backend/storage/buffer/bufmgr.c:4496-4628`。
它负责 physically write out a shared buffer。
注释提醒：
- 它只是把 buffer 内容交给 kernel。
- 真正落盘取决于 kernel。
- checkpoint WAL 之前还需要 fsync relation files。
调用者必须：
- 持有 buffer pin。
- 持有 content lock，模式至少是 share-exclusive。
函数第一步是：

```c
if (StartSharedBufferIO(buf, false, true, NULL) == BUFFER_IO_ALREADY_DONE)
    return;
```

这里 `forInput = false`，表示要做 output I/O。
如果别的 backend 已经把 dirty buffer 写干净了，就直接返回。
如果可以开始 I/O，则设置 `BM_IO_IN_PROGRESS`。
然后建立 error context。
然后打开 smgr relation。
然后读 page LSN：

```c
recptr = BufferGetLSN(buf);
```

这里的 `BufferGetLSN` 是 `bufmgr.c` 内部宏：

```c
#define BufferGetLSN(bufHdr) (PageGetLSN(BufHdrGetBlock(bufHdr)))
```

源码注释解释：

```c
As we hold at least a share-exclusive lock on the buffer,
the LSN cannot change during the flush.
```

也就是说，不需要 `BufferGetLSNAtomic()`。
因为当前写出路径已经持有强 enough 的 content lock。
接下来是本节的核心：

```c
if (pg_atomic_read_u64(&buf->state) & BM_PERMANENT)
    XLogFlush(recptr);
```

只有 permanent buffer 才 flush WAL。
原因：
- permanent relation crash 后要恢复，所以必须遵守 WAL-before-data。
- unlogged relation crash 后会丢弃或重建，不靠 WAL 恢复。
- unlogged index page 可能有 fake LSN。
- fake LSN 可能超过真实 WAL insert position。
- 对 fake LSN 调用 `XLogFlush()` 可能造成系统级失败。
WAL flushed 后，才进入 data write：

```c
bufBlock = BufHdrGetBlock(buf);
PageSetChecksum((Page) bufBlock, buf->tag.blockNum);
smgrwrite(..., bufBlock, false);
```

写完后：

```c
TerminateBufferIO(buf, true, 0, true, false);
```

`clear_dirty = true`。
这会清掉：
- `BM_IO_IN_PROGRESS`
- `BM_IO_ERROR`
- `BM_DIRTY`
- `BM_CHECKPOINT_NEEDED`
所以 `FlushBuffer()` 的正确性顺序是：
1. 拿到 I/O ownership。
2. 在 content lock 下读稳定 page LSN。
3. permanent buffer 先 `XLogFlush(page_lsn)`。
4. 临写前更新 checksum。
5. `smgrwrite()` relation block。
6. 成功后清 dirty 和 checkpoint-needed。

## 16. `StartSharedBufferIO()`：谁有权写这个 buffer

`StartSharedBufferIO()` 位于 `bufmgr.c:7210-7322`。
它处理同一个 buffer 的 I/O ownership。
输出 I/O 只对 `BM_VALID && BM_DIRTY` 的 buffer 有意义。
源码注释写得很直接：

```c
output operations only on buffers that are BM_VALID and BM_DIRTY
```

函数循环中先检查 `BM_IO_IN_PROGRESS`。
如果别的 backend 已经在做 I/O：
- 调用方可选择异步等待。
- 调用方可选择不等待并返回 `BUFFER_IO_IN_PROGRESS`。
- 调用方可选择同步等待。
`FlushBuffer()` 传的是 `wait = true`。
所以如果有 I/O 正在进行，它会等。
等到没有 I/O 后，它检查是否已经不需要 I/O：

```c
if (forInput ? (buf_state & BM_VALID) : !(buf_state & BM_DIRTY))
{
    UnlockBufHdr(buf);
    return BUFFER_IO_ALREADY_DONE;
}
```

对输出路径来说，只要 `BM_DIRTY` 已经没了，就说明别人写好了。
否则它设置：

```c
BM_IO_IN_PROGRESS
```

并把当前 resource owner 记为 buffer I/O owner。
这个状态保证同一时刻只有一个执行者真正写这个 buffer。

## 17. `TerminateBufferIO()`：成功写出后清什么

`TerminateBufferIO()` 位于 `bufmgr.c:7348-7413`。
核心逻辑：

```c
unset_flag_bits |= BM_IO_IN_PROGRESS;
unset_flag_bits |= BM_IO_ERROR;
if (clear_dirty)
    unset_flag_bits |= BM_DIRTY | BM_CHECKPOINT_NEEDED;
```

当前基线没有 `BM_JUST_DIRTIED` 判断。
只要 successful write 走 `clear_dirty = true`，就清 dirty。
这在旧版本会让人紧张：
如果写出过程中 page 又被修改了怎么办？
当前答案是：
不会在同一个 page 上并发发生。
写出路径持有 share-exclusive content lock。
普通修改需要 exclusive content lock。
hint bit 修改也必须升级到 share-exclusive content lock。
因此，flush 持锁期间没人能把 page 内容重新 dirtied。
这就是 `BM_JUST_DIRTIED` 被删除的关键前提。
如果 I/O 出错，走 `AbortBufferIO()`。
`AbortBufferIO()` 在 `bufmgr.c:7415-7462`。
它会调用：

```c
TerminateBufferIO(buf_hdr, false, BM_IO_ERROR, false, false);
```

这里 `clear_dirty = false`。
所以失败后 dirty 不会被清掉。
buffer 仍然需要后续重试写出。

## 18. page checksum 的位置

checksum 不是 page 每次内存修改时更新。
`src/backend/storage/page/README:17-24` 明确说：
- checksum 在 page 离开 shared buffer pool 时才有效。
- 之后 page 重新通过 I/O 进入 shared buffer pool 时会被检查。
- 修改 page 内容会隐式让当前 `pd_checksum` 失效。
`PageSetChecksum()` 在 `src/backend/storage/page/bufpage.c:1504-1529`。
它的调用点之一就是 `FlushBuffer()`。
`FlushBuffer()` 在 `smgrwrite()` 前调用：

```c
PageSetChecksum((Page) bufBlock, buf->tag.blockNum);
```

所以 checksum 的语义是：
- 写出时，按即将写出的 page 内容计算 checksum。
- shared buffer 里的 `pd_checksum` 可能在两次写出之间过时。
- 不要把内存中 checksum 字段当成 page 当前内容总是自洽的证据。
local buffer 写出也是类似。
`FlushLocalBuffer()` 在 `localbuf.c:203` 调用 `PageSetChecksum()`。
checksum 与 page LSN 的关系：
- page LSN 决定 WAL-before-data 的 flush 边界。
- checksum 检测 data page 从 storage 读回时是否损坏。
- 二者是不同机制。
checksum 与 WAL FPI 的关系：
- WAL record 自己有 WAL CRC。
- full page image 在 WAL 中受 WAL CRC 保护。
- data page checksum 保护的是 relation 文件中的 block。
- WAL replay 不应该用 data page checksum 检查 WAL 中的 FPI。

## 19. `XLogFlush()` 的触发边界

`XLogFlush()` 位于 `src/backend/access/transam/xlog.c:2794-2977`。
本节关注由 `FlushBuffer()` 触发的场景。
触发条件是：

```c
buffer is dirty
buffer is being flushed
buffer has BM_PERMANENT
page_lsn = PageGetLSN(page)
XLogFlush(page_lsn)
```

也就是只有 data page 真要写出时，才需要强制 WAL flush。
普通事务提交也可能调用 `XLogFlush(commit_lsn)`。
但这属于 commit durability。
本节的边界是 data page write safety。
这两个场景经常重合，但不是同一个机制。
一个未提交事务修改过的 dirty page 也可能被 bgwriter 写出。
即使事务未提交，只要这个 page 要写出，对应 WAL record 也必须先 flush。
因为 crash recovery 需要知道如何重做或撤销/判断这些物理状态。
WAL-before-data 不等同于 commit-before-data。
WAL-before-data 只要求描述这个 page 物理修改的 WAL 在 data page 前持久化。

## 20. `XLogFlush()` 的执行边界

`XLogFlush(record)` 第一个分支是 recovery 场景：

```c
if (!XLogInsertAllowed())
{
    UpdateMinRecoveryPoint(record, false);
    return;
}
```

redo 时不是写新的 WAL。
此时所谓 flush 边界转为 minRecoveryPoint 边界。
正常运行时，先 quick exit：

```c
if (record <= LogwrtResult.Flush)
    return;
```

这说明 `XLogFlush()` 不一定真的写盘。
如果本 backend 本地知道 WAL 已经 flush 到足够位置，函数直接返回。
如果本地状态过旧，后面会调用：

```c
RefreshXLogWriteResult(LogwrtResult);
```

再次检查是否已经由别人 flush。
如果还不够，`XLogFlush()` 会：
- 进入 critical section。
- 等待相关 WAL insertions 完成。
- 争用 `WALWriteLock`。
- 获取锁后再次检查别人是否已经完成。
- 可选执行 commit delay，让更多 backend 加入 group commit。
- 把写/刷请求推进到 `insertpos`。
- 调用 `XLogWrite(WriteRqst, insertTLI, false)`。
- 释放 `WALWriteLock`。
- 唤醒 walsenders 和等待 primary flush LSN 的进程。
一个重要细节是：
`XLogFlush(record)` 可能 flush 到超过 `record` 的位置。
源码注释说 fsync 很贵，所以尽量 piggyback 更多 WAL。
这对性能很重要。
请求边界是“至少到 record”。
实际结果可能是“到当前已完成插入的更远位置”。
最后有错误检查：

```c
if (LogwrtResult.Flush < record)
    elog(ERROR, "xlog flush request ... is not satisfied")
```

注释指出，来自 `bufmgr.c` 的调用不在 critical section 内。
所以数据页上腐坏的 LSN 导致 flush request 超过 WAL end 时，不会直接把全系统 PANIC。
来自 `xact.c` 的 commit critical section 则可能被提升为 PANIC。
这也是错误边界之一。

## 21. `XLogNeedsFlush()` 的轻量判断

`XLogNeedsFlush()` 在 `xlog.c:3147-3227`。
它回答的问题是：
给定 LSN 是否仍需要 flush 或更新 minRecoveryPoint。
正常运行时：

```c
if (record <= LogwrtResult.Flush)
    return false;
RefreshXLogWriteResult(LogwrtResult);
if (record <= LogwrtResult.Flush)
    return false;
return true;
```

这个函数不执行 flush。
它只是判断。
`bufmgr.c` 中一个典型用法在 buffer replacement 路径。
`StrategyGetBuffer()` 找到 dirty victim 后，如果使用非默认 strategy ring，代码会检查：

```c
buf_state & BM_PERMANENT &&
XLogNeedsFlush(BufferGetLSN(buf_hdr)) &&
StrategyRejectBuffer(...)
```

如果复用这个 ring buffer 会触发 WAL flush，strategy 可以拒绝它，重新找 victim。
这是性能策略，不是正确性最终防线。
最终只要进入 `FlushBuffer()`，仍会调用 `XLogFlush()`。

## 22. checkpoint 与 `BM_CHECKPOINT_NEEDED`

`BM_CHECKPOINT_NEEDED` 在 `BufferSync()` 中设置。
代码位于 `bufmgr.c:3560-3795`。
checkpoint 开始时，`BufferSync()` 扫描所有 buffers。
默认 online checkpoint 只考虑：

```c
BM_DIRTY | BM_PERMANENT
```

shutdown checkpoint、end-of-recovery 或 flush unlogged 时规则更宽。
扫描时如果 buffer 匹配 mask，就设置：

```c
BM_CHECKPOINT_NEEDED
```

源码注释解释目的：
- 只写 checkpoint 开始时已经 dirty 的 pages。
- 不写 checkpoint 过程中后来才被 dirtied 的 pages。
- 后来 dirty 的 pages 不属于当前 checkpoint 的必须完成集合。
`BM_CHECKPOINT_NEEDED` 是一个 checkpoint snapshot 标记。
它不是 dirty 的同义词。
一个 buffer 可以 `BM_DIRTY = 1` 且 `BM_CHECKPOINT_NEEDED = 0`。
这表示它是 checkpoint 开始后才 dirty，或者还没被当前 checkpoint 标入集合。
一个 buffer 可以 `BM_DIRTY = 1` 且 `BM_CHECKPOINT_NEEDED = 1`。
这表示当前 checkpoint 需要把它写出去。
写出成功后，`TerminateBufferIO(clear_dirty=true)` 同时清：

```c
BM_DIRTY | BM_CHECKPOINT_NEEDED
```

如果写出失败，源码注释说留下 `BM_CHECKPOINT_NEEDED` 也没关系。
因为下次 checkpoint 仍然会需要写这个 buffer。

## 23. checkpoint 与 RedoRecPtr/FPI

checkpoint 推进 redo point。
`xlog.c:7538-7560` 中，checkpoint 过程中会设置新的 `RedoRecPtr`。
注释说明：
- 必须在持有所有 WAL insertion locks 时更新 shared `RedoRecPtr`。
- 即使 checkpoint 后来失败，RedoRecPtr 指得太靠后也可接受。
- 后果只是 XLogInsert 可能多写 full page images。
- 不能推迟推进，因为 checkpoint dump buffers 期间发生的新修改必须假设没有包含进这个 checkpoint。
这和 FPI 判定直接相关。
`xloginsert.c:678-700` 中，注册的 buffer 是否需要 full page image 取决于：

```c
needs_backup = (page_lsn <= RedoRecPtr);
```

直观含义：
如果这个 page 自上一个 redo point 以来还没有被 WAL 保护过，那么第一次修改需要写 FPI。
这样就算 data page 发生 torn write，recovery 也能从 WAL 中的整页镜像恢复。
`XLogInsertRecord()` 会在拿到 insert lock 后重新检查 `RedoRecPtr` 和 `fullPageWrites`。
如果发现调用方 assemble record 时的判断过期，它会返回 `InvalidXLogRecPtr`，让 `XLogInsert()` 重新 assemble。
这处理了 checkpoint 刚推进时的并发边界。

## 24. FPI 不是每次修改都有

`full_page_writes` 打开时，PostgreSQL 不是每次 page 修改都写整页。
FPI 的关键条件是：

```c
PageGetLSN(page) <= RedoRecPtr
```

也就是 page 自当前 redo point 之后第一次被修改。
如果 page 已经有一个大于 redo point 的 LSN，说明本 checkpoint round 中已经有 WAL record 覆盖过它。
后续修改只需要记录 delta。
这就是 page LSN 同时参与两个判断：
- 写出时，作为 `XLogFlush()` 的请求点。
- 插入 WAL 时，作为是否需要 FPI 的判断依据。
两个方向不要混淆。
写出路径问：这个 page 落盘前 WAL 至少要 flush 到哪里？
插入路径问：这次 WAL record 是否需要携带整页镜像？

## 25. `MarkBufferDirtyHint()` 的定位

`MarkBufferDirtyHint()` 位于 `bufmgr.c:5815-5848`。
注释说它与 `MarkBufferDirty()` 类似，但有三个不同点：
1. caller 不写普通 WAL，所以 checksums 开启时可能需要 `XLOG_FPI_FOR_HINT`。
2. caller 可能只有 share-exclusive lock，而不是 exclusive lock。
3. 它不保证总能把 buffer 标 dirty，例如 hot standby 上可能不能。
因此：
`MarkBufferDirty()` 用于关键修改。
`MarkBufferDirtyHint()` 用于非关键 hint。
hint 的定义不是“不重要到可以随便乱写”。
hint 的定义是“丢了也能从其他真实状态重新推导”。
例如 heap tuple 的 visibility hint bit。
例如 btree `LP_DEAD` 标记。
例如 FSM page 的 next-target pointer 某些更新。
这些 hint 可以提升性能。
但 crash 后丢失不会破坏逻辑数据。

## 26. hint bit 与 checksum/torn page 的冲突

如果没有 checksums，也没有 `wal_log_hints`，hint bit 可以不写 WAL。
即使 crash 丢掉 hint，也只是以后重新判断可见性。
问题出在 torn page 与 checksum。
`src/backend/storage/page/README:38-48` 解释：
- 写 data block 可能发生 torn page。
- full page writes 可以保护 torn page。
- 已经 dirty 的 page 设置 hint bit 通常没问题，因为本 checkpoint round 已经有 FPI 或必要 WAL 保护。
- clean page 上设置 hint bit 可能让磁盘出现“部分 bit 写入”的 torn page。
- hint 本身丢不丢不重要，但 checksum 会不匹配。
- 所以 checksums 开启且 `full_page_writes=on` 时，需要为 hint bit 写 FPI WAL。
`XLogHintBitIsNeeded()` 在 `src/include/access/xlog.h:114-123`：

```c
#define XLogHintBitIsNeeded() (wal_log_hints || DataChecksumsNeedWrite())
```

也就是说，只要 `wal_log_hints=on` 或 data checksums 需要写，就进入 hint WAL 保护路径。
`DataChecksumsNeedWrite()` 在 `xlog.c:4660-4679`。
它不仅在 checksums enabled 时返回 true。
checksums 正在 enable/disable 的 in-progress 状态也需要写 checksum。

## 27. `MarkSharedBufferDirtyHint()` 的顺序

本节不逐行展开内部 helper，只保留诊断必须知道的边界。
`MarkSharedBufferDirtyHint()` 仍遵守“先 dirty，再让 checkpoint 能看到这个 buffer”的思想。
如果需要 FPI-for-hint，`XLogSaveBufferForHint()` 可能写一条 `XLOG_FPI_FOR_HINT`。
返回 valid LSN 时，helper 会把这个 LSN 写入 page header。
如果正在 recovery 或 relation 正在跳过 WAL，hint 可能已经改了内存 page，但不会被 marked dirty。
这是允许的，因为 hint 可以丢。
当前基线还要求 hint setting 与 flush 在 content lock 层互斥。
只持 share lock 的 caller 需要通过 `SharedBufferBeginSetHintBits()` / `BufferBeginSetHintBits()` 尝试取得 share-exclusive 能力。
这正是 `BM_JUST_DIRTIED` 在本基线中不再需要的重要原因。
`XLogSaveBufferForHint()`、commit LSN 检查和 `BufferGetLSNAtomic()` 都只是服务这条边界。
你不需要把它们背成第二条主流程。
需要定位 hint 问题时，再回到 `bufmgr.c:5697-5848`、`xloginsert.c:1120-1176` 和 `heapam_visibility.c:152-164`。
最重要的判断是：
hint 可丢，但 checksum、torn page 和未持久化 commit 不能被 hint 路径绕过。

## 28. WAL-before-data 的 crash 正确性

现在把协议串起来。
假设 heap page P 被修改。
修改方做：
1. 持有 P 的 exclusive content lock。
2. 修改 P 的内存内容。
3. `MarkBufferDirty(P)`。
4. 插入 WAL record R。
5. `PageSetLSN(P, end(R))`。
6. 释放 lock。
之后某个 backend/checkpointer/bgwriter 写 P：
1. 持有 P 的 pin 和 share-exclusive content lock。
2. `StartSharedBufferIO()` 设置 `BM_IO_IN_PROGRESS`。
3. 读取 `page_lsn = PageGetLSN(P)`。
4. 如果 P 是 permanent，调用 `XLogFlush(page_lsn)`。
5. `PageSetChecksum(P)`。
6. `smgrwrite(P)`。
7. `TerminateBufferIO(clear_dirty=true)`。
如果系统在步骤 4 之前 crash，P 尚未写出或写出不会合法完成。
如果系统在 WAL flush 后、data write 前 crash，recovery 有 WAL，可以重做。
如果系统在 data write 后 crash，recovery 也有 WAL。
如果 data write torn，FPI/delta + page LSN/redo 规则提供恢复路径。
坏情况是：
- data page 写到磁盘。
- 但对应 WAL record 没有持久化。
恢复时无法知道这个 page change 的来源。
如果 page 中含有未提交事务的物理修改、index split、FSM/VM 状态等，系统可能进入不可解释状态。
`FlushBuffer()` 的 `XLogFlush(page_lsn)` 正是防止这个坏情况。

## 29. page LSN 与 redo skip

redo 过程中通常会检查目标 page 的 LSN。
如果 page LSN 已经大于等于 WAL record 的 LSN，说明这个 page 上的修改已经在磁盘镜像中体现。
redo 可以跳过或做幂等处理。
相关逻辑分散在各 rmgr 的 redo 代码中。
例如 heap、btree、gin、hash 等 redo 函数会在应用修改后 `PageSetLSN(page, lsn)`。
这体现了 page LSN 的第二个作用：
- 正常运行时作为写出前的 WAL flush 下界。
- recovery 时作为判断 page 是否已经包含某条 WAL 修改的版本戳。
这两个作用依赖同一个字段。
所以 `PageSetLSN()` 必须严格跟随 WAL record。
不能随意把 page LSN 设置成“当前时间”或“当前 insert pointer”。
它必须表达“这个 page 内容中最后一次 WAL-logged 修改的 record end LSN”。

## 30. permanent、unlogged、temp 的区别

`BM_PERMANENT` 在 buffer tag 赋值时设置。
`bufmgr.c:2331-2338`：

```c
if (relpersistence == RELPERSISTENCE_PERMANENT || forkNum == INIT_FORKNUM)
    set_bits |= BM_PERMANENT;
```

permanent relation 的普通 fork 需要 WAL-before-data。
unlogged relation 的普通 fork crash 后不靠 WAL 恢复。
unlogged relation 的 init fork 需要像 permanent 一样处理。
temp relation 使用 local buffers。
所以 `FlushBuffer()` 中：

```c
if (pg_atomic_read_u64(&buf->state) & BM_PERMANENT)
    XLogFlush(recptr);
```

这个判断不是性能捷径。
它防止把 fake LSN 当真实 WAL LSN flush。
也避免给无需 crash recovery 的数据做无意义 WAL flush。

## 31. 错误路径：坏 page LSN

`XLogFlush()` 注释中特别讨论了 corrupted heap page 的 LSN。
如果 page LSN 损坏到超过可用 WAL 末尾，`XLogFlush(record)` 最终会发现：

```c
LogwrtResult.Flush < record
```

于是报 ERROR。
注释说，早期把这视为 PANIC 会降低系统鲁棒性。
如果只是一页数据上的坏 LSN，不应该直接让整个数据库实例崩溃。
从 `bufmgr.c` 调用 `XLogFlush()` 不在 critical section 内。
因此这种 ERROR 不会自动升级为 PANIC。
但 commit 路径可能在 critical section 内调用 flush。
这种 ERROR 就可能升级。
课程重点：
page LSN 是安全边界。
如果它损坏，系统宁愿拒绝写出这个 page，也不能跳过 WAL-before-data。

## 32. 错误路径：I/O 失败

写 shared buffer 失败时，error cleanup 会走 `AbortBufferIO()`。
它不会清 dirty。
核心调用：

```c
TerminateBufferIO(buf_hdr, false, BM_IO_ERROR, false, false);
```

`clear_dirty = false`。
所以 buffer 仍是 dirty。
`BM_IO_ERROR` 被设置。
等待者通过 condition variable 被唤醒。
后续写出可以重试。
如果多次失败，`AbortBufferIO()` 会发 WARNING，提示 write error might be permanent。
这个边界说明：
成功写出才可以清 `BM_DIRTY`。
失败写出必须保留 dirty，否则会丢失内存中尚未安全落盘的数据。

## 33. 并发边界：替换 victim

当 buffer allocation 选择 victim 时，如果 victim 是 dirty，需要先写出。
`bufmgr.c:2584-2634` 中的逻辑：

```c
if (buf_state & BM_DIRTY)
{
    if (!BufferLockConditional(buf, buf_hdr, BUFFER_LOCK_SHARE_EXCLUSIVE))
    {
        UnpinBuffer(buf_hdr);
        goto again;
    }
    ...
    FlushBuffer(buf_hdr, NULL, IOOBJECT_RELATION, io_context);
}
```

为什么是 conditional lock？
源码注释说，虽然 `StrategyGetBuffer()` 返回 victim 时它未 pinned，但等当前 backend 到这里时，别人可能已经 pin 并 lock 了它。
如果无条件等待，可能和对方形成 deadlock。
例如两个 backend 同时 split btree page。
所以拿不到 content lock 就放弃这个 victim，重新找。
这里的正确性边界是：
- 不能在没有 content lock 的情况下写 dirty page。
- 也不能为了写 victim 无条件等待，制造 deadlock。

## 34. 并发边界：Flush 与 page 修改

普通 page 修改需要 exclusive content lock。
Flush 需要 share-exclusive content lock。
hint bit 设置也要求 share-exclusive 或 exclusive。
因此同一 buffer 上这些动作互斥：
- flush 不能和普通修改并行。
- flush 不能和 hint bit 设置并行。
- 普通修改不能和 hint bit 设置并行。
这让以下事情成立：
- `FlushBuffer()` 读到的 page LSN 在写出期间不会变化。
- `PageSetChecksum()` 计算期间 page 内容不会变化。
- `TerminateBufferIO(clear_dirty=true)` 不会误清 flush 期间新产生的 dirty。
这就是当前基线删掉 `BM_JUST_DIRTIED` 的核心逻辑。

## 35. 并发边界：header state 与 content lock 分工

buffer header state 包含：
- refcount
- usage_count
- flags
- content lock state
`MarkBufferDirty()` 用 CAS 修改 header state。
这不是替代 content lock。
header state 原子更新只保证 `BM_DIRTY` 这个 metadata transition 的一致性。
page 内容修改的一致性仍靠 content lock。
所以：
- pin 防止 buffer slot 被替换。
- content lock 防止 page 内容并发读写/写出。
- header state lock/CAS 防止 descriptor 状态并发丢失。
三个层次都必要。
不要把 pin 误解成 page 内容锁。
不要把 `BM_DIRTY` 误解成 page 内容锁。
不要把 content lock 误解成 replacement pin。

## 36. 并发边界：checkpoint 漏掉后来的 dirty 为什么安全

`BufferSync()` 在 checkpoint 开始时标记 `BM_CHECKPOINT_NEEDED`。
后续新 dirty 的 pages 不会带这个 bit。
这看起来像漏写。
实际上安全。
因为当前 checkpoint 的 redo point 在这些新修改的 WAL record 之前。
crash recovery 从 checkpoint redo point 开始重放，会看到这些修改的 WAL。
因此当前 checkpoint 不需要保证这些后来 dirty 的 page 已经写出。
这也是为什么 access method 必须先 `MarkBufferDirty()` 再 `XLogInsert()`。
这个顺序保证“被 checkpoint 漏掉”的 dirty 一定对应更晚的 WAL。

## 37. 并发边界：FPI 判定过期

`XLogInsert()` assemble record 时会取 cached `RedoRecPtr` 和 `doPageWrites`。
但拿 WAL insert lock 前，这些值可能变化。
`XLogInsertRecord()` 拿 lock 后重新检查。
如果发现：
- `full_page_writes` 变成需要。
- 或 `RedoRecPtr` 前移导致某个未带 FPI 的 page 现在需要 FPI。
它返回 `InvalidXLogRecPtr`。
`XLogInsert()` 外层 loop 会重新 assemble record。
这避免 checkpoint 推进和 WAL record 构造之间的 race。
如果不这样做，可能漏掉 checkpoint 后第一次修改 page 的 FPI。

## 38. 并发边界：`PageGetLSN()` 的撕裂读取

page LSN 是 8 字节。
不是所有平台都保证 8 字节读写原子。
`BufferGetLSNAtomic()` 用条件编译处理这个差异。
当前平台如果有 `PG_HAVE_8BYTE_SINGLE_COPY_ATOMICITY`，直接读。
否则在需要 hint bit WAL 的场景中用 buffer header lock 保护。
普通 `FlushBuffer()` 不需要 `BufferGetLSNAtomic()`。
因为它已持有 share-exclusive content lock，page LSN 不会被并发修改。
只持有 share lock 的 reader 应该使用 `BufferGetLSNAtomic()`。

## 39. 错误边界：不能把 hint 当关键修改

`MarkBufferDirtyHint()` 的注释明确说：

```c
This function does not guarantee that the buffer is always marked dirty
```

它可能在 recovery 或 skip-WAL 场景直接 return。
因此不能用它标记关键数据结构变化。
如果一个修改必须 crash 后存在或必须由 WAL redo 重建，就必须走普通 WAL + `MarkBufferDirty()` 协议。
错误用法示例：
- 修改 heap tuple 物理字段后只调用 `MarkBufferDirtyHint()`。
- 修改 btree structural link 后只当 hint。
- 修改 visibility map 真值后不写必要 WAL。
这些都会破坏恢复语义。
hint 的判断标准是：
丢失后是否可以从持久化真相重新推导。
如果不能，就不是 hint。

## 40. 跟读：从 dirty 到 writeback 的代码路径

建议按下面顺序跟读源码。
第一段：`buf_internals.h`。
- 看 `BufferDesc.state` 的 flags。
- 确认 `BM_DIRTY`、`BM_CHECKPOINT_NEEDED`、`BM_PERMANENT` 的定义。
- 确认 bit 6 当前没有 `BM_JUST_DIRTIED`。
第二段：`bufpage.h`。
- 看 `PageHeaderData.pd_lsn`。
- 看 `PageGetLSN()`。
- 看 `PageSetLSN()`。
第三段：`bufmgr.c:3147-3205`。
- 读 `MarkBufferDirty()`。
- 标出 local buffer 分支。
- 标出 pinned/exclusive lock 断言。
- 标出 CAS loop。
- 标出 accounting 只在 clean-to-dirty 时发生。
第四段：`bufmgr.c:4124-4198`。
- 读 `SyncOneBuffer()`。
- 重点看“不拿 content lock 检查 dirty 为什么安全”的注释。
- 把它和“先 dirty 后 WAL”连起来。
第五段：`bufmgr.c:4496-4628`。
- 读 `FlushBuffer()`。
- 标出 `StartSharedBufferIO()`。
- 标出 `recptr = BufferGetLSN(buf)`。
- 标出 `XLogFlush(recptr)` 的 `BM_PERMANENT` 条件。
- 标出 `PageSetChecksum()`。
- 标出 `TerminateBufferIO(clear_dirty=true)`。
第六段：`bufmgr.c:7210-7413`。
- 读 `StartSharedBufferIO()`。
- 读 `TerminateBufferIO()`。
- 理解 `BM_IO_IN_PROGRESS` 的 ownership。
- 理解成功 write 清 dirty，失败 write 不清 dirty。
第七段：`xloginsert.c:470-540`。
- 读 `XLogInsert()`。
- 确认返回 record end pointer。
- 看外层 loop 如何处理 `InvalidXLogRecPtr`。
第八段：`xloginsert.c:657-700`。
- 读 FPI 判定。
- 重点看 `PageGetLSN(page) <= RedoRecPtr`。
第九段：`xlog.c:754-1022`。
- 读 `XLogInsertRecord()`。
- 标出 reserve WAL space。
- 标出 WAL insert locks。
- 标出 recheck `RedoRecPtr` / `fullPageWrites`。
第十段：`xlog.c:2794-2977`。
- 读 `XLogFlush()`。
- 区分 quick exit、wait insertion、WALWriteLock、group commit、最终错误检查。
第十一段：`bufmgr.c:5697-5848`。
- 读 `MarkSharedBufferDirtyHint()` 和 `MarkBufferDirtyHint()`。
- 标出 `XLogHintBitIsNeeded()`。
- 标出 recovery/skip-WAL return。
- 标出先 dirty 后 `XLogSaveBufferForHint()`。
- 标出 valid LSN 时 `PageSetLSN()`。
第十二段：`heapam_visibility.c:152-164`。
- 读 commit LSN 与 hint bit 的边界。
- 理解 `BufferGetLSNAtomic(buffer) < commitLSN` 时为什么不能设置 hint。

## 41. 成本、资源与观测入口

这条路径的成本在修改时和写出时分离。
修改时，`MarkBufferDirty()` 只是 descriptor 状态变化，热路径成本很小。
真正可能变慢的是后续任意 backend、bgwriter 或 checkpointer 写 dirty page 时承担的 WAL flush、checksum、data write 和 fsync request。
成本随几个变量扩张：
- dirty page 数越多，checkpoint 和 replacement 要处理的候选越多。
- page LSN 越靠近未 flush WAL 尾部，`FlushBuffer()` 越可能把普通 buffer write 放大成 WAL flush stall。
- checksums 或 `wal_log_hints` 打开后，hint bit 可能引入 FPI-for-hint WAL。
- `BM_CHECKPOINT_NEEDED` 集合越大，checkpoint 越容易形成写出和 fsync burst。
- unlogged/temp 路径不能按 permanent page 的 WAL flush 模型解释。
能直接观察的入口：
- `EXPLAIN (ANALYZE, BUFFERS, WAL)`：看单条语句产生的 buffer 和 WAL 量。
- `pg_stat_io`：看 relation write、fsync、writeback 的 backend type 和 IO context。
- `pg_stat_wal`：看 WAL bytes、write、sync 的实例级变化。
- `log_checkpoints`：看 checkpoint 写出量、sync 时间和距离 redo point 的关系。
- gdb：`MarkBufferDirty()`、`FlushBuffer()`、`XLogFlush()`、`TerminateBufferIO()` 能直接串起 dirty -> LSN -> WAL-before-data。
不能直接看到的事实：
- SQL 视图不会告诉你某个 page 的 `pd_lsn` 是否就是导致 `XLogFlush()` 的边界。
- `BM_DIRTY` 和 `BM_CHECKPOINT_NEEDED` 是 shared buffer 内部状态，通常需要断点或临时 instrumentation。
- hint bit 是否因为 lock 升级失败被放弃，普通统计不会显示。
诊断时先把现象压回本节模型：
看到 write latency 或 checkpoint spike，不要只问“谁写了这张表”。
要问 dirty page 的 page LSN 指向哪段 WAL、谁在写出它、它是不是 permanent、是否落入当前 checkpoint 集合。

## 42. 实验 1：静态 grep 跟踪标准顺序

在 `/home/highgo/postgres` 中执行：

```bash
rg -n "MarkBufferDirty\\(|XLogInsert\\(|PageSetLSN\\(" src/backend/access/heap/heapam.c
```

找 heap insert/update/delete 的调用点。
对每个调用点检查：
- 是否已经持有 buffer lock。
- 是否在 critical section 内。
- `MarkBufferDirty()` 是否在 `XLogInsert()` 前。
- `PageSetLSN()` 是否使用 `XLogInsert()` 返回值。
再执行：

```bash
rg -n "MarkBufferDirty\\(|XLogInsert\\(|PageSetLSN\\(" src/backend/access/nbtree
```

观察 btree split/insert 等复杂路径。
重点不是背代码。
重点是验证协议的普遍形状。

## 43. 实验 2：GDB 观察 insert 的 dirty/LSN 顺序

需要 debug build。
启动一个测试集群后，在 backend 上设置断点：

```gdb
break MarkBufferDirty
break XLogInsert
break heap_insert
```

SQL：

```sql
CREATE TABLE t_dirty_lsn(id int primary key, v text);
INSERT INTO t_dirty_lsn VALUES (1, 'a');
```

在 `heap_insert` 中跟到 page 修改后。
观察：
- buffer 是否 pinned。
- content lock 是否 exclusive。
- `MarkBufferDirty()` 是否先执行。
- `XLogInsert()` 返回的 `recptr`。
- `PageSetLSN(page, recptr)` 是否在返回后执行。
如果 `PageSetLSN()` 是 inline，不一定能直接断在函数上。
可以在 `heapam.c` 对应行号设置断点。
目标是观察“dirty bit 与 page LSN 不是同一时间设置”的事实。

## 44. 实验 3：checkpoint 的 `BM_CHECKPOINT_NEEDED`

阅读 `BufferSync()`。
在 GDB 中断：

```gdb
break BufferSync
break SyncOneBuffer
break TerminateBufferIO
```

执行：

```sql
CHECKPOINT;
```

在 `BufferSync()` 第一轮扫描中观察：
- 哪些 buffer 同时满足 `BM_DIRTY` 和 `BM_PERMANENT`。
- 这些 buffer 被设置 `BM_CHECKPOINT_NEEDED`。
在 `SyncOneBuffer()` 中观察：
- 如果 buffer 已经 clean，它直接返回。
- 如果 buffer dirty，它 pin + share-exclusive lock + flush。
在 `TerminateBufferIO()` 中观察：
- 成功写出清 `BM_DIRTY | BM_CHECKPOINT_NEEDED`。
再思考：
checkpoint 开始后新 dirty 的 page 为什么没有 `BM_CHECKPOINT_NEEDED`。
答案回到 redo point 和“先 dirty 后 WAL”。

## 45. 常见误区与误解

误解 1：`MarkBufferDirty()` 会写 WAL。
纠正：不会。它只设置 `BM_DIRTY`。
误解 2：`MarkBufferDirty()` 会设置 page LSN。
纠正：不会。page LSN 由调用方在 `XLogInsert()` 后设置。
误解 3：page LSN 是 WAL record start pointer。
纠正：是 record end pointer，也就是下一条 record 的开始位置。
误解 4：事务提交 flush WAL 后，dirty page 写出就不需要检查 WAL。
纠正：`FlushBuffer()` 仍按 page LSN 检查。dirty page 可能来自未提交事务，也可能 commit flush 与 page 所需 WAL 不完全同一边界。
误解 5：checkpoint 必须写出 checkpoint 期间产生的所有 dirty pages。
纠正：只需要写 checkpoint 开始时属于该 checkpoint 集合的 dirty permanent pages；后来 dirty 的 pages 由 redo point 之后的 WAL 覆盖。
误解 6：checksum 字段在 shared buffer 中总是有效。
纠正：page 被修改后 checksum 通常失效，写出前才重新计算。
误解 7：hint bit 不重要，所以可以随便写。
纠正：hint 可丢，但写它仍可能影响 checksum/torn page，因此需要 `MarkBufferDirtyHint()` 协议。
误解 8：当前 PostgreSQL 仍用 `BM_JUST_DIRTIED` 防止 flush race。
纠正：本基线没有该 flag；当前靠 share-exclusive content lock 防止 flush 中被 dirtied。

## 46. 讨论题

1. 为什么 `MarkBufferDirty()` 必须在 `XLogInsert()` 前，而 `PageSetLSN()` 必须在 `XLogInsert()` 后？
2. `BM_DIRTY`、page LSN、WAL flushed LSN 分别回答哪个问题？为什么任意一个都不能替代另外两个？
3. `FlushBuffer()` 为什么用 `BM_PERMANENT` 判断是否调用 `XLogFlush()`，而不是调用 `RelationNeedsWAL()`？
4. checkpoint 开始后新 dirtied 的 page 为什么可以不带 `BM_CHECKPOINT_NEEDED`？
5. 写 shared buffer 失败时，为什么 `AbortBufferIO()` 不能清 `BM_DIRTY`？
6. hint bit 可丢，为什么仍可能需要 FPI-for-hint WAL？
7. 当前基线没有 `BM_JUST_DIRTIED`，这个事实依赖哪个 content lock 约束？
8. 如果 `pg_stat_io` 看到 client backend relation writes 增长，你如何判断它来自用户写 SQL、replacement 写 dirty victim，还是 checkpoint/bgwriter 行为？

## 47. 本节小结

`BM_DIRTY` 是 buffer manager 的写回标记。
page LSN 是 page header 中的 WAL 依赖边界。
WAL flushed LSN 是 WAL 子系统的持久化边界。
`MarkBufferDirty()` 只设置 `BM_DIRTY`。
它必须在 `XLogInsert()` 前执行，因为 checkpoint 可以不拿 content lock 检查 dirty bit。
`PageSetLSN()` 必须在 `XLogInsert()` 后执行，因为 page LSN 要使用 WAL record end pointer。
`FlushBuffer()` 是 WAL-before-data 的最终执行点。
它在写 permanent dirty page 前调用 `XLogFlush(PageGetLSN(page))`。
`XLogFlush()` 的请求边界是“至少 flush 到 record”，实际可能借 group commit 推进更远。
`BM_CHECKPOINT_NEEDED` 是 checkpoint 对 dirty set 的快照标记，不是 dirty 的同义词。
`MarkBufferDirtyHint()` 只能用于可丢失的 hint 修改。
当 checksums 或 `wal_log_hints` 要求保护 hint bit 时，它会通过 `XLogSaveBufferForHint()` 写 FPI-for-hint。
checksum 在 page 写出前计算；shared buffer 中的 checksum 字段不保证随时有效。
本基线没有 `BM_JUST_DIRTIED`。
旧版本用它防止 flush 过程中被重新 dirtied 后误清 dirty。
当前基线通过 share-exclusive content lock 让 flush 与 hint/普通修改互斥，所以这个 flag 已经 obsolete。
理解本节后，读 buffer eviction、checkpoint、FPI、redo 时，应始终问三个问题：
1. 这个 page 是否 dirty？
2. 这个 page 的 LSN 指向哪条 WAL record 的 end？
3. 这个 page 写出前 WAL 是否已经 flush 到那个 LSN？
只要这三个问题能回答清楚，PostgreSQL 的 WAL-before-data 路径就不会神秘。
