# PostgreSQL Relation extension 与 bulk read strategy
## 课程定位
本节主题：PostgreSQL 在 shared buffer 体系里如何扩展 relation 文件，如何把大规模读写限制在较小的 buffer ring 中，以及 heap insert、sequential scan、VACUUM/COPY 这些上层访问路径如何把策略传进 buffer manager。
本节唯一主问题：PostgreSQL 如何在保证 relation extension 不产生重复 block 的同时，让 bulk read/write 不把普通工作集挤出 shared buffers？
核心矛盾：扩展 relation 需要全局一致地推进物理 EOF；而大规模扫描和批量写入又必须限制自己的 buffer footprint，不能让一次大操作污染整个 shared buffer pool。
本节主流程：heap/scan 调用者选择 access strategy -> `ReadBuffer_common()` 或 `ExtendBufferedRelBy()` 进入 buffer manager -> relation extension lock 推进 EOF -> zero page 进入 buffer -> 上层初始化 page 或 bulk read 通过 ring 复用 buffer。
生命周期 / ownership / cleanup：extension lock 保护 EOF 分配，buffer pin 保护 slot identity，content lock 保护新页初始化，ResourceOwner 清理 pin/lock，bulk strategy ring 是 backend-local 对象，查询结束后随 backend-private state 释放。
观测与诊断入口：关注 relation extension lock 等待、`pg_stat_io` 的 bulk read/write IO context、`EXPLAIN (ANALYZE, BUFFERS)` 的大扫描 footprint，以及断点中的 `BufferAccessStrategyData` ring slot 变化。
这一节把三条线放在一起读：
- `P_NEW` / `ReadBufferExtended()` / `ReadBuffer_common()` 是“读一个 block 或兼容式扩展一个 block”的入口。
- `ExtendBufferedRel()` / `ExtendBufferedRelBy()` 是当前更明确的 relation extension API。
- `BufferAccessStrategy` 是给 bulk read、bulk write、vacuum 这类“不要污染整个 shared buffers”的访问模式用的 backend-private ring。
读完本节，你应该能回答：
- 为什么 `P_NEW` 不是一个天然 race-free 的抽象。
- 为什么现代 heap 插入不再直接依赖 `ReadBuffer(relation, P_NEW)`。
- relation extension lock 保护的是什么，不保护的又是什么。
- 扩展 relation 时为什么先找 victim buffer，再拿 extension lock。
- 新扩展出来的页面为什么先是 all-zero page，随后才由 heap 初始化成有效 heap page。
- `smgrzeroextend()` 与 `PageInit()` / `MarkBufferDirty()` / WAL 之间的边界在哪里。
- 大表顺序扫描为什么仍走 shared buffers，却不会把 shared buffers 大面积冲刷掉。
- `BAS_BULKREAD`、`BAS_BULKWRITE`、`BAS_VACUUM` 的 ring size、dirty page 处理、pin limit 有什么差异。
- `read_stream.c` 在 bulk read 中怎样结合 access strategy、AIO、IO combining 与预读。
## 源码基线
源码仓库：`/home/nail/postgres-lab`
基线：`master` = `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`
本节重点阅读：
- `src/backend/storage/buffer/bufmgr.c`
- `src/backend/storage/buffer/freelist.c`
- `src/backend/access/heap/hio.c`
- `src/backend/access/heap/heapam.c`
- `src/backend/storage/smgr/smgr.c`
- `src/backend/storage/smgr/md.c`
- `src/backend/storage/aio/read_stream.c`
辅助核对：
- `src/include/storage/bufmgr.h`
- `src/backend/storage/lmgr/lmgr.c`
- `src/backend/storage/buffer/README`
- `src/backend/commands/vacuum.c`
---
## 1. 先给结论
relation extension 是把一个 relation fork 的物理 EOF 往后推进。
buffer manager 不能只把一个 buffer tag 设置成“新 block”就算扩展完成。
它还必须确保底层文件真的拥有这个 block。
否则下一次 `P_NEW` 或 `smgrnblocks()` 仍会看到原来的 EOF，多个 backend 可能都认为自己拿到了同一个新 block。
这就是 relation extension lock 存在的核心原因。
`LockRelationForExtension()` 的注释说得很直接：`P_NEW` 的 bufmgr/smgr 定义不是 race-condition-proof。
本基线里，推荐的扩展路径是 `ExtendBufferedRel*()`。
`ReadBufferExtended(..., P_NEW, ...)` 仍然存在，但在 `ReadBuffer_common()` 中被标为 backward compatibility path。
它会调用 `ExtendBufferedRel()`，并传入 `EB_SKIP_EXTENSION_LOCK`。
这意味着它假设调用者已经用别的方式保证安全，例如已经持有 extension lock、持有足够强的 relation lock，或者处于 startup/recovery 这类特殊上下文。
heap 普通插入不会走这个旧路径。
heap 插入会先从 relcache cached target block、bulk insert current buffer、FSM、最后一个 block 中寻找可用页。
找不到时进入 `RelationAddBlocks()`。
`RelationAddBlocks()` 会调用 `ExtendBufferedRelBy()`，并且可以一次扩展多个 block。
扩展出来的第一个 buffer 会被返回并加 exclusive content lock。
heap 随后检查 `PageIsNew()`，执行 `PageInit()`，再 `MarkBufferDirty()`。
真正插入 tuple 时，`heap_insert()` / `heap_multi_insert()` 会生成 WAL，并把 WAL record 的 LSN 写入 page header。
所以要分清三层边界：
- `smgrzeroextend()` 让物理文件出现 zero-filled blocks。
- `TerminateBufferIO(..., BM_VALID, ...)` 让 shared buffer 中的 zero page 成为 valid buffer。
- `PageInit()`、tuple insert、`MarkBufferDirty()`、`XLogInsert()`、`PageSetLSN()` 才让它变成有 heap page layout、有修改、有 WAL 依赖的脏页。
access strategy ring 是另一个主题，但它会贯穿读和扩展。
`ExtendBufferedRelBy()` 接受 `BufferAccessStrategy strategy`。
这意味着 bulk insert 扩展 relation 时，选择 victim buffer 也能走 `BAS_BULKWRITE` ring。
大表顺序扫描通过 `BAS_BULKREAD` ring 读入页面。
VACUUM 通过 `BAS_VACUUM` ring 限制自己的 buffer footprint。
这些策略都不是绕开 shared buffers。
它们仍然把 page 放进 shared buffers。
区别是它们复用一个小 ring，避免一次大扫描或大写入把正常工作集从 shared buffers 中挤出去。
---
## 2. 关键名词
`BlockNumber` 是 relation fork 内的 block 编号。
`P_NEW` 在 `src/include/storage/bufmgr.h` 中定义为 `InvalidBlockNumber`。
它不是一个真实 block number。
它表达的是“请在 relation 末尾给我一个新 page”。
`ForkNumber` 区分 main fork、fsm fork、vm fork、init fork 等 relation fork。
本节 heap 插入和顺序扫描主要看 `MAIN_FORKNUM`。
`Relation` 是 relcache 层对象。
`SMgrRelation` 是 storage manager 层对象。
`BufferManagerRelation` 是 buffer manager 为了支持“有 Relation”或“只有 smgr + persistence”两类调用而使用的包装。
`BufferAccessStrategy` 是 backend-private 对象。
它不是共享内存结构。
它记录 ring 类型、ring 中 buffer 数量、当前 slot、每个 slot 里缓存的 buffer number。
`BM_TAG_VALID` 表示 buffer descriptor 已经有合法 tag，并进入 buffer mapping table。
`BM_VALID` 表示 buffer 内容可用。
`BM_IO_IN_PROGRESS` 表示某个 read/write/extend 相关的 IO 正在进行。
扩展 relation 时，这几个 flag 的顺序非常重要。
buffer manager 会先把新 block 的 tag 插入 buffer table，并启动 buffer IO。
然后才调用 `smgrzeroextend()` 推进文件 EOF。
这样一旦物理文件已经变大，其他 backend 读到这个新 block 时，会在 buffer table 中看到已有 buffer，并等待 IO 完成，而不是自己再读一份未初始化内容。
---
## 3. `ReadBufferExtended()`：普通读入口与旧式扩展入口
先读 `src/backend/storage/buffer/bufmgr.c:874-940`。
`ReadBuffer()` 只是 `ReadBufferExtended()` 的简写。
它固定读取 main fork，mode 是 `RBM_NORMAL`，strategy 是 `NULL`。
`ReadBufferExtended()` 接受五个核心参数：
- `Relation reln`
- `ForkNumber forkNum`
- `BlockNumber blockNum`
- `ReadBufferMode mode`
- `BufferAccessStrategy strategy`
普通调用中，`blockNum` 是一个真实 block number。
如果 `blockNum == P_NEW`，语义就从“读已有 block”变成“扩展 relation 并返回新 block”。
`ReadBufferExtended()` 本身不实现读取。
它调用：
```c
ReadBuffer_common(reln, RelationGetSmgr(reln), 0,
                  forkNum, blockNum, mode, strategy)
```
这里传 `RelationGetSmgr(reln)`，说明真正 I/O 会落到 smgr/md 层。
`strategy` 会原样传进 buffer allocation。
因此一个 seq scan、COPY、VACUUM 的调用端，只要把合适的 `BufferAccessStrategy` 传进来，就可以影响 buffer victim 选择和 IO context。
---
## 4. `ReadBuffer_common()` 的分支
`ReadBuffer_common()` 在 `src/backend/storage/buffer/bufmgr.c:1271-1368`。
它先做一个 canonical check：
如果传入的是其他 session 的 temp relation，就报错。
原因是 temp relation 的 buffer 是 session-local 的。
其他 backend 看不到拥有者的 local buffers，读出来很可能是错的。
接下来就是 `P_NEW` 分支。
本基线中的关键逻辑：
```c
if (unlikely(blockNum == P_NEW))
{
    uint32 flags = EB_SKIP_EXTENSION_LOCK;
    if (mode == RBM_ZERO_AND_LOCK || mode == RBM_ZERO_AND_CLEANUP_LOCK)
        flags |= EB_LOCK_FIRST;
    return ExtendBufferedRel(BMR_REL(rel), forkNum, strategy, flags);
}
```
这段代码有两个信息量很高的事实。
第一，`P_NEW` 已经被重定向到 `ExtendBufferedRel()`。
第二，它传了 `EB_SKIP_EXTENSION_LOCK`。
这不是说 relation extension 不需要锁。
这是说旧接口的调用者需要自己负责。
`src/include/storage/bufmgr.h:70-75` 对 `EB_SKIP_EXTENSION_LOCK` 的注释是：只有 relation 非共享、持有 access exclusive lock、或者 startup process 等场景才安全。
所以在新代码中，看到 `ReadBuffer(relation, P_NEW)` 应该警惕。
你需要继续往上找调用者是否已经 `LockRelationForExtension()`。
本基线里，BRIN 的 `brin_pageops.c` 仍有一个例子：先拿 extension lock，再 `ReadBuffer(irel, P_NEW)`。
heap 的主路径已经不用这种风格。
如果不是 `P_NEW`，`ReadBuffer_common()` 会按 mode 分两大类。
`RBM_ZERO_AND_LOCK` / `RBM_ZERO_AND_CLEANUP_LOCK` 会先 pin buffer，然后 `ZeroAndLockBuffer()`。
这类模式适用于调用者准备从头初始化一个已有 block 或安全的新 block。
其余模式构造 `ReadBuffersOperation`，设置 `READ_BUFFERS_SYNCHRONOUSLY`，必要时加 `READ_BUFFERS_ZERO_ON_ERROR`。
随后调用 `StartReadBuffer()`。
如果需要等待 IO，则调用 `WaitReadBuffers()`。
这说明单块 `ReadBufferExtended()` 在普通路径下仍复用多块 read infrastructure，只是要求同步等待。
---
## 5. 为什么 `P_NEW` 需要 extension lock
relation extension 的 race 可以用一个极简时间线说明。
假设 relation 当前有 100 个 block。
两个 backend 同时想执行 `P_NEW`。
如果没有额外同步：
1. backend A 调用 `smgrnblocks()`，看到 EOF 是 100。
2. backend B 调用 `smgrnblocks()`，也看到 EOF 是 100。
3. A 认为新 block 是 100。
4. B 也认为新 block 是 100。
5. 两个 backend 都可能把不同 tuple 写入同一个 logical block。
这不是 buffer mapping table 本身能完全解决的问题。
buffer tag 是 `(relfilenumber, fork, blockNum)`。
如果物理 EOF 没有推进，下一次 `P_NEW` 仍可能选回同一个 block number。
所以必须在“读取准确 EOF、决定 first new block、推进物理 EOF”之间建立互斥。
`src/backend/storage/lmgr/lmgr.c:414-432` 中的 `LockRelationForExtension()` 用 relation extension lock tag 做这件事。
它不是 relation-level ordinary lock。
它专门用于 interlock addition of pages to relations。
`RelationExtensionLockWaiterCount()` 还能数出正在等待这个 extension lock 的 backend 数量。
heap 的多块扩展会用这个数量决定是否“帮别人多扩几页”。
---
## 6. `ExtendBufferedRel()` 与 `ExtendBufferedRelBy()`
`ExtendBufferedRel()` 在 `src/backend/storage/buffer/bufmgr.c:966-982`。
它只是 `ExtendBufferedRelBy()` 的一页封装。
`extend_by = 1`。
返回第一个，也是唯一一个 buffer。
`ExtendBufferedRelBy()` 在 `src/backend/storage/buffer/bufmgr.c:984-1020`。
它的契约更丰富。
调用者传入想扩展的页数 `extend_by`。
函数会尽力扩展这么多页。
如果资源限制不允许，实际扩展页数可能更小。
但只要不报错，就至少扩展一页。
调用者传入的 `buffers` 数组必须至少能容纳 `extend_by` 个元素。
返回后，前 `*extended_by` 个元素都是 pinned buffer。
如果 flags 包含 `EB_LOCK_FIRST`，第一个返回 buffer 会被 exclusive locked。
heap 插入路径正是这样使用。
它需要确保第一个 page 是空的，并且在自己 `PageInit()` 前没有别人插入。
`ExtendBufferedRelBy()` 自己不直接区分本地 buffer 和 shared buffer。
它填好 relation persistence 后，调用 `ExtendBufferedRelCommon()`。
`ExtendBufferedRelCommon()` 在 `src/backend/storage/buffer/bufmgr.c:2747-2788`。
它负责 tracepoint 和根据 persistence 分派：
- temp relation 走 `ExtendBufferedRelLocal()`。
- permanent/unlogged relation 走 `ExtendBufferedRelShared()`。
本节重点看 shared buffer 路径。
---
## 7. shared relation extension 的真实流程
`ExtendBufferedRelShared()` 在 `src/backend/storage/buffer/bufmgr.c:2790-3060`。
这段函数是本节最重要的源码。
把它拆成九步理解。
第一步，限制一次扩展需要额外 pin 的 buffer 数。
代码先调用 `LimitAdditionalPins(&extend_by)`。
因为扩展 N 个 block 时，buffer manager 需要暂时同时 pin N 个 victim buffer。
如果 backend 已经持有很多 pin，不能无限追加。
第二步，在拿 extension lock 之前先找 victim buffer。
源码注释明确解释了原因。
找 victim buffer 可能触发写脏页。
写脏页可能需要 flush WAL。
清零 buffer 内存也有成本。
这些都不应该放在 extension lock 保护区里。
否则所有等待扩展同一 relation 的 backend 都会被迫排队等这些慢操作。
所以函数先循环 `extend_by` 次：
- `GetVictimBuffer(strategy, io_context)`
- `BufHdrGetBlock()`
- `MemSet(..., 0, BLCKSZ)`
这一步只得到“可用于新 tag 的 buffer frame”。
还没有决定新 block number。
第三步，按需拿 relation extension lock。
除非 flags 包含 `EB_SKIP_EXTENSION_LOCK`，否则调用 `LockRelationForExtension(bmr.rel, ExclusiveLock)`。
注释还指出：所有 fork 目前共用同一把 extension lock。
这比按 fork 加锁更保守。
原因是 fork 扩展不够频繁，细分锁的收益不大。
第四步，按需清空 smgr size cache，然后调用 `smgrnblocks()`。
这一步在 extension lock 内完成。
它得到的是当前准确 EOF。
注意此时可能已经有别的 backend 在我们等待锁期间完成了扩展。
因此 `first_block` 不一定等于我们进入函数前想象的 EOF。
第五步，处理 `extend_upto`。
`ExtendBufferedRelBy()` 传 `InvalidBlockNumber`，表示只是追加。
`ExtendBufferedRelTo()` 会传一个目标上界。
如果别的 backend 已经把 relation 扩到了目标位置之后，本 backend 会释放多余 victim buffer，实际扩展 0 页，然后返回当前 `first_block`。
这就是并发扩展 race 的正常处理。
第六步，检查最大 block number。
如果 `first_block + extend_by >= MaxBlockNumber`，报错。
PostgreSQL 不能创建编号等于 `InvalidBlockNumber` 的 block。
第七步，把新 block 的 tag 插入 buffer mapping table，并标记 IO in progress。
这一步必须发生在物理扩展之前。
源码注释说：一旦 relation 被扩展，其他 backend 就可以开始读这些 page。
因此在 `smgrzeroextend()` 之前，buffer table 中就要有这些 block 的 buffer tag。
常规路径中，victim buffer 原来没有 valid tag。
代码设置：
- `victim_buf_hdr->tag = tag`
- `BM_TAG_VALID`
- usage count 设为 1
- permanent relation 或 init fork 设置 `BM_PERMANENT`
- `StartSharedBufferIO(..., forInput=true, nowait=true, ...)`
这里的 `forInput=true` 容易让人困惑。
扩展时不是“读取磁盘”，而是在 buffer manager 的协议里使用 input-like IO state，避免别人看到未完成的 page。
第八步，调用 `smgrzeroextend()`。
`smgrzeroextend(BMR_GET_SMGR(bmr), fork, first_block, extend_by, false)` 会把底层文件从 `first_block` 开始增加 `extend_by` 个 zero-filled block。
这是真正推进物理 EOF 的步骤。
如果它失败，注释说明这些 buffer 会保持 allocated 但不标记 `BM_VALID`。
下次 relation extension 仍会选择同一个 block number，因为文件没有变长。
如果那些 buffer 还没被回收，下次会重新走到这里再尝试 `smgrzeroextend()`。
第九步，释放 extension lock，然后终止 buffer IO。
代码先 `UnlockRelationForExtension()`，再对每个扩展 buffer：
- 按 flags 决定是否 `LockBuffer(buf, BUFFER_LOCK_EXCLUSIVE)`
- `TerminateBufferIO(buf_hdr, false, BM_VALID, true, false)`
也就是说，relation 文件已经被扩展后，extension lock 可以释放。
唤醒等待 buffer IO 的 backend 可能有成本，所以放在释放 extension lock 之后。
最终这些 buffer 是 pinned、valid、zero-filled 的。
但它们还不是 heap initialized page。
它们也不是 dirty buffer。
---
## 8. existing buffer beyond EOF 这个角落
`ExtendBufferedRelShared()` 中最容易跳过的分支是 `BufTableInsert()` 返回 `existing_id >= 0`。
按常识，扩展 EOF 后的新 block 不应该已经在 buffer table 中。
源码注释列了几个例外。
一个例外是之前的 relation extension 失败了。
另一个例外是 `zero_damaged_pages` 相关的 beyond-EOF 读取曾留下 valid zero-filled buffer。
还可能是 relation 被外部进程覆盖这类异常场景。
代码处理方式很谨慎。
它 pin 住已有 buffer。
如果已有 buffer 是 valid 且不是 `PageIsNew()`，就报错。
如果看起来是合法 zero page，就清掉 `BM_VALID`，重新 `StartSharedBufferIO()`，确保后续一定会执行 `smgrzeroextend()`。
注释强调：必须在成功前做 `smgrzeroextend()`。
否则这个页面没有被 kernel 预留，下一次 `P_NEW` 又会返回同一个 page。
这个分支可以帮助你建立一个边界意识：
buffer table 中“有 block N 的 buffer”不等于底层 relation 文件真的有 block N。
relation extension 的正确完成条件必须包含物理 EOF 推进。
---
## 9. `smgrzeroextend()` 与 `mdzeroextend()`
`src/backend/storage/smgr/smgr.c:642-668` 是 storage manager 抽象层。
`smgrzeroextend()` 的语义是：新增多个 zeroed blocks。
它调用具体 storage manager 的 `smgr_zeroextend` 方法。
本基线中普通 relation 使用 md storage manager。
`smgr.c` 的 switch table 把 `smgr_zeroextend` 绑定到 `mdzeroextend()`。
调用完成后，`smgrzeroextend()` 更新 `smgr_cached_nblocks[forknum]`。
如果 cached value 正好等于本次扩展起点 `blocknum`，就设置为 `blocknum + nblocks`。
否则把缓存置为 `InvalidBlockNumber`。
这体现了 smgr size cache 的保守策略。
如果缓存与预期不一致，不猜测，下一次让 kernel 回答。
`src/backend/storage/smgr/md.c:547-663` 是 md 层的实现。
`mdzeroextend()` 会按 relation segment 分段扩展。
一个 relation 文件超过 `RELSEG_SIZE` 后会拆成多个 segment 文件。
所以一次多块扩展可能跨 segment。
每个 segment 内部，`mdzeroextend()` 先决定本 segment 要扩多少 block。
如果扩展数量大于 8 且配置允许，它优先用 `FileFallocate()`。
注释解释：fallocate 通常比写零更高效，因为它经常不会把新 page 放进 kernel page cache。
如果扩展很小，或者 `file_extend_method` 要求写零，就调用 `FileZero()`。
`FileZero()` 底层可以用 vectored write 避免每个 8KB block 单独写一次。
扩展后，如果不是 temp relation 且没有 `skipFsync`，md 层会 `register_dirty_segment()`。
这不是把 shared buffer 标成 dirty。
这是告诉 storage/sync 机制：这个 segment 需要在 checkpoint 体系下被 fsync。
这一点很关键。
物理 zero extend 的持久性责任由 smgr/md 的 dirty segment 机制处理。
后续 page 内容修改的持久性责任由 buffer dirty + WAL-before-data 协议处理。
---
## 10. `mdreadv()` 对 beyond EOF 的警示
`src/backend/storage/smgr/md.c:856-980` 是 md 层读取。
正常情况下，上层不应该读不存在的 block。
如果 `mdreadv()` 在 EOF 遇到 0 bytes，默认会报 `DATA_CORRUPTED`。
但在 `zero_damaged_pages` 或 recovery 情况下，它可能返回零页。
本基线注释对此非常谨慎。
它指出：把 beyond-EOF block 放进 buffer pool 很麻烦。
依赖 `smgrnblocks()` 的 scan 看不到这些 block。
relation extension 也不期望 beyond-EOF buffer 已经存在。
这正好对应上一节 existing buffer beyond EOF 分支。
课程里要记住一句话：
zero-filled buffer 不是“安全存在的 relation block”的充分条件。
它必须和物理文件大小一致。
---
## 11. zero page、initialized page、dirty page、WAL page
扩展路径里有四种状态很容易混在一起。
第一种是 all-zero page。
`ExtendBufferedRelShared()` 把 victim buffer 的内存 `MemSet(..., 0, BLCKSZ)`。
`smgrzeroextend()` 把底层文件扩展为 zero-filled blocks。
这时 buffer 内容是零。
`PageIsNew()` 会认为它是 new page。
第二种是 initialized page。
heap 看到 `PageIsNew()` 后会调用 `PageInit(page, BufferGetPageSize(buffer), 0)`。
这会写入 page header、line pointer 起始位置等 heap page layout 必需信息。
此时页面有结构了，但还不一定有 tuple。
第三种是 dirty page。
`PageInit()` 修改了 buffer 中的内容。
heap 在 `hio.c` 中紧接着 `MarkBufferDirty(buffer)`。
这表示 buffer 内容已经不同于 relation 文件中的 zero page，需要未来写回。
第四种是 WAL-covered page。
真正插入 tuple 后，`heap_insert()` 或 `heap_multi_insert()` 会构造 WAL record。
`heap_insert()` 在 `src/backend/access/heap/heapam.c:2087-2165` 中先 `MarkBufferDirty()`，再 `XLogInsert()`，最后 `PageSetLSN(page, recptr)`。
如果这是 page 上第一条 tuple，且 page 只有这条 tuple，`heap_insert()` 会设置 `XLOG_HEAP_INIT_PAGE` 并用 `REGBUF_WILL_INIT`。
这告诉 redo：恢复时可以重新初始化整个 page，而不是依赖旧 page 内容。
所以不要把“扩展时写了 zero page”理解成“已经完成了 heap page 的 WAL 保护”。
zero extend 只是让文件空间存在。
heap page 的逻辑内容仍由 heap WAL record 保护。
---
## 12. heap insert 如何选择 page
现在读 heap 调用端。
`heap_insert()` 在 `src/backend/access/heap/heapam.c:2003-2193`。
它准备 tuple 后调用：
```c
RelationGetBufferForTuple(relation, heaptup->t_len,
                          InvalidBuffer, options, bistate,
                          &vmbuffer, NULL, 0)
```
这个函数在 `src/backend/access/heap/hio.c:435-883`。
它返回一个 pinned 且 exclusive locked 的 buffer。
这个 buffer 至少有足够空间容纳目标 tuple。
函数入口先处理几个上层约束：
- tuple 长度按 `MAXALIGN()` 对齐。
- tuple 不能超过 `MaxHeapTupleSize`。
- 计算 fillfactor 需要预留的 free space。
- 如果 tuple 很大，使用 nearly-empty page 的特殊阈值，避免低 fillfactor 表中过度扩展。
然后它决定初始 `targetBlock`。
优先级是：
1. 如果 bulk insert state 有 `current_buf`，继续用这个 buffer 的 block。
2. 否则用 relcache 中的 `RelationGetTargetBlock(relation)`。
3. 如果没有 cached target 且允许 FSM，就调用 `GetPageWithFreeSpace()`。
4. 如果 FSM 没有记录，就尝试 relation 最后一个 block。
5. 如果仍然没有可用 page，就扩展 relation。
这说明 heap insert 并不是“每次插入都 append”。
它会尽量复用有空间的 page。
只有目标页/FSM/最后一页都失败时才扩展。
---
## 13. `ReadBufferBI()`：bulk insert 如何传入 `BAS_BULKWRITE`
`src/backend/access/heap/hio.c:82-125` 定义了 `ReadBufferBI()`。
BI 是 bulk insert 的意思。
如果没有 `bistate`，它等价于：
```c
ReadBufferExtended(relation, MAIN_FORKNUM, targetBlock, mode, NULL)
```
如果有 `bistate`，它会先检查 `bistate->current_buf` 是否已经 pin 住了目标 block。
如果是，就增加 refcount 后直接返回，省掉一次 buffer lookup/pin 循环。
如果不是，就释放旧 current buffer，然后：
```c
ReadBufferExtended(relation, MAIN_FORKNUM, targetBlock,
                   mode, bistate->strategy)
```
这里 `bistate->strategy` 来自 `GetBulkInsertState()`。
`GetBulkInsertState()` 在 `src/backend/access/heap/heapam.c:1933-1948`。
它调用 `GetAccessStrategy(BAS_BULKWRITE)`。
所以 bulk insert 不仅批量插入 tuple。
它还会把 heap page read/extend 过程中的 buffer victim 选择切到 bulk write ring。
---
## 14. existing page 成功路径
`RelationGetBufferForTuple()` 的主循环从 `targetBlock` 开始。
它先读 buffer，再处理 visibility map pin，然后给目标 buffer 加 exclusive lock。
如果 `otherBuffer` 有效，例如 heap update 需要同时锁旧 tuple page 和新 tuple page，它会按 block number 顺序锁两个 buffer。
这是为了避免两个 backend 以相反顺序锁两个 heap page 造成死锁。
锁住目标页后，如果 `PageIsNew(page)`，它会：
- `PageInit(page, BufferGetPageSize(buffer), 0)`
- `MarkBufferDirty(buffer)`
这可能发生在读取到的已有 block 上。
例如某些新扩展但尚未使用的 page 后来通过 FSM 被找到。
随后计算 `PageGetHeapFreeSpace(page)`。
如果 `targetFreeSpace <= pageFreeSpace`，就设置 relcache target block 并返回 buffer。
调用者拿到的是 pinned、exclusive locked 的 buffer，可以直接插入 tuple。
如果空间不够，它会释放 lock/pin，更新 FSM，并寻找下一个候选页。
如果 bulk insert state 里还有 `next_free` 到 `last_free` 的预扩展页面，就优先使用这些页面，而不是重新查 FSM。
---
## 15. 什么时候进入 relation extension
如果循环找不到合适 target block，就执行：
```c
buffer = RelationAddBlocks(relation, bistate, num_pages, use_fsm,
                           &unlockedTargetBuffer);
```
`RelationAddBlocks()` 在 `src/backend/access/heap/hio.c:211-432`。
它封装了 heap 对多块扩展的策略。
注意这个函数名是 heap 层命名。
真正 buffer manager 的扩展函数仍是 `ExtendBufferedRelBy()`。
`RelationAddBlocks()` 的参数 `num_pages` 表示调用者预计至少需要多少 page。
普通 `heap_insert()` 传 0，函数内部转成 1。
`heap_multi_insert()` 会估算本批 tuple 最坏需要多少 page，并传入 `npages - npages_used`。
因此 COPY/批量 insert 可以触发 multi-block extend。
---
## 16. heap 多块扩展的启发式
`RelationAddBlocks()` 先计算 `extend_by_pages`。
如果没有 bulk insert state 且不能使用 FSM，就只能扩展 1 页。
因为额外扩展的页面没有地方记录给后续使用。
如果有 bulk insert state 或者可以用 FSM，它会先至少扩展调用者需要的 `num_pages`。
然后读取 extension lock waiter 数：
```c
waitcount = RelationExtensionLockWaiterCount(relation)
```
如果有人在等同一个 relation 的 extension lock，就把扩展页数增加为：
```c
extend_by_pages += extend_by_pages * waitcount
```
这是一种“帮别人也扩一点”的策略。
它减少后续大家再次排队扩展的概率。
如果 `bistate` 之前已经扩展过，还会用 `already_extended_by` 做下限。
这避免扩展大小在文件系统层不断变化。
也能缓解间歇性 contention。
最后，扩展页数被限制到 `MAX_BUFFERS_TO_EXTEND_BY`。
本基线中这个值是 64。
限制原因很直接：扩展多少页，就要临时 pin 多少 victim buffer。
---
## 17. `RelationAddBlocks()` 如何处理 FSM 与 bulk state
多扩出来的页需要被后续找到。
如果没有 bulk insert state，但允许 FSM，那么除了马上返回的第一页外，额外页面会进入 FSM。
这样其他 backend 能找到这些空页。
如果有 bulk insert state，则当前 backend 更希望自己连续使用这些新页。
所以它会把 `next_free` 和 `last_free` 记录在 `bistate` 中。
后续 `RelationGetBufferForTuple()` 发现当前 page 不够时，会先拿 `bistate->next_free`。
这避免刚扩出来的页面立刻被别的 backend 抢走。
源码注释也说明：如果有 `bistate`，只把自己不需要的页面放进 FSM。
被返回的第一页永远不放进 FSM。
因为调用者马上要用它。
---
## 18. heap 调用 `ExtendBufferedRelBy()`
`RelationAddBlocks()` 的核心调用在 `src/backend/access/heap/hio.c:339-344`：
```c
first_block = ExtendBufferedRelBy(BMR_REL(relation), MAIN_FORKNUM,
                                  bistate ? bistate->strategy : NULL,
                                  EB_LOCK_FIRST,
                                  extend_by_pages,
                                  victim_buffers,
                                  &extend_by_pages);
```
有几个点要看清楚。
第一，strategy 来自 bulk insert state。
普通 insert 是 `NULL`。
bulk insert 是 `BAS_BULKWRITE`。
第二，flags 包含 `EB_LOCK_FIRST`。
buffer manager 返回的第一个 buffer 会被 exclusive locked。
第三，`extend_by_pages` 既是输入也是输出。
输入表示请求扩展多少页。
输出表示实际扩展多少页。
扩展后，heap 立即检查返回的第一页：
```c
if (!PageIsNew(page))
    elog(ERROR, ...)
PageInit(page, BufferGetPageSize(buffer), 0);
MarkBufferDirty(buffer);
```
这个检查很重要。
扩展出来的新页应该是 zero page。
如果不是，继续初始化会有覆盖有效数据的风险。
---
## 19. `heap_multi_insert()` 如何驱动多块扩展
`heap_multi_insert()` 在 `src/backend/access/heap/heapam.c:2320-2555` 附近展示了批量插入路径。
它先调用 `heap_multi_insert_pages()` 估算剩余 tuple 在最坏情况下需要多少 page。
然后调用：
```c
RelationGetBufferForTuple(relation, heaptuples[ndone]->t_len,
                          InvalidBuffer, options, bistate,
                          &vmbuffer, NULL,
                          npages - npages_used)
```
这就是 multi-block extend 的上层来源。
`RelationGetBufferForTuple()` 不能保证一次就拿到所有 page。
它保证“至少下一条 tuple 能放下”。
但它把 `num_pages` 传到 `RelationAddBlocks()`，让扩展可以更激进。
在拿到一个 page 后，`heap_multi_insert()` 会往同一个 page 放尽可能多的 tuple。
如果 page 满了，循环继续，可能使用 `bistate->next_free` 指向的预扩展页。
这条路径把三个优化连起来：
- bulk insert current buffer 避免重复 pin/unpin。
- bulk write access strategy 避免污染整个 shared buffers。
- multi-block extend 减少 relation extension lock contention 和底层文件扩展调用次数。
---
## 20. access strategy ring 的数据结构
`BufferAccessStrategyData` 在 `src/backend/storage/buffer/freelist.c:69-94`。
它包含：
- `btype`
- `nbuffers`
- `current`
- `buffers[]`
`buffers[]` 是 buffer number 数组。
`InvalidBuffer` 表示该 slot 还没绑定 buffer。
这个对象在 backend 私有内存中。
多个 backend 各自有自己的 ring 状态。
ring 里保存的是 shared buffer 的编号。
也就是说，它复用的是 shared buffers 中的一小批 buffer frame。
不是另开一块私有 page cache。
`GetBufferFromRing()` 每次把 `current` 前进一个 slot。
如果 slot 是空的，返回 `NULL`，调用者会走 normal clock-sweep 找 victim，并把新 victim 放入 ring。
如果 slot 有 buffer，它会检查能不能复用。
可复用条件大致是：
- refcount 为 0。
- usage_count 不大于 1。
- buffer header 没有处在无法 CAS 的状态。
如果 buffer 被 pin，不能复用。
如果 usage_count 大于 1，说明 ring 之外有人比较频繁地触碰了这个 buffer，也不复用。
这样 ring 不会强行夺走其他 backend 的热页面。
---
## 21. `StrategyGetBuffer()` 如何结合 ring 与 clock sweep
`StrategyGetBuffer()` 在 `src/backend/storage/buffer/freelist.c:169-317`。
它是 `GetVictimBuffer()` 用来选择 victim 的入口。
流程是：
1. 如果传了 strategy，先尝试 `GetBufferFromRing()`。
2. 如果 ring 返回 buffer，标记 `from_ring = true`，直接返回。
3. 如果 ring 没有可用 buffer，走 clock-sweep。
4. clock-sweep 找到 refcount 为 0 且 usage_count 为 0 的 buffer 后 pin 住。
5. 如果有 strategy，把这个新 victim 放入当前 ring slot。
这说明 ring 的初始填充仍然需要正常 replacement 算法。
ring 不是凭空获得 buffer。
它只是之后尽量复用这些 buffer。
`StrategyGetBuffer()` 中还有一个统计细节。
普通 clock-sweep allocation 会增加 `numBufferAllocs`，供 bgwriter 估算 buffer 消耗率。
注释说明：strategy object 回收的 buffer 不计入这里。
这正是 ring 的目的。
它让一次大扫描的 buffer 消耗不被看成普通工作集持续增长。
---
## 22. 三类内置策略
`BufferAccessStrategyType` 在 `src/include/storage/bufmgr.h:34-41`。
本基线有四个枚举值：
- `BAS_NORMAL`
- `BAS_BULKREAD`
- `BAS_BULKWRITE`
- `BAS_VACUUM`
`BAS_NORMAL` 返回 `NULL`。
也就是说“默认策略”不是一个 ring 对象。
`GetAccessStrategy()` 在 `src/backend/storage/buffer/freelist.c:421-500` 选择 ring size。
`BAS_BULKREAD` 的基础大小是 256KB。
本基线还会根据 `io_combine_limit * effective_io_concurrency` 增加空间。
原因是 read stream 可能同时 pin 多个正在读入的 buffer。
但这个大小又会被 pin limit 和最小大小约束。
`BAS_BULKWRITE` 的 ring size 是 16MB。
`BAS_VACUUM` 的默认 ring size 是 2048KB，也就是 2MB。
不过 VACUUM 通常通过 `GetAccessStrategyWithSize(BAS_VACUUM, ring_size)` 创建。
`vacuum_buffer_usage_limit` 或 `VACUUM/ANALYZE BUFFER_USAGE_LIMIT` 可以覆盖这个大小。
无论哪种策略，`GetAccessStrategyWithSize()` 最后还会把 ring buffer 数限制在 `NBuffers / 8` 以内。
所以 ring 不会超过 shared buffers 的八分之一。
---
## 23. BulkRead ring 的 dirty page 处理
Bulk read 的目标是大规模读一次。
典型场景是 large sequential scan。
这种 scan 通常不修改 page。
最多可能设置 hint bits。
如果 ring 中的 buffer 被 dirty，并且重用它需要先 flush WAL，bulk read 会尽量拒绝这个 buffer。
逻辑在两个地方配合完成。
`GetVictimBuffer()` 在 `src/backend/storage/buffer/bufmgr.c:2577-2639` 中发现 victim 是 dirty 时，需要写出。
如果 victim 来自 strategy ring，且是 permanent buffer，且 `XLogNeedsFlush(BufferGetLSN(...))`，会调用 `StrategyRejectBuffer()`。
`StrategyRejectBuffer()` 在 `src/backend/storage/buffer/freelist.c:741-770`。
它只对 `BAS_BULKREAD` 返回 true。
并且只有 victim 确实来自当前 ring slot 才拒绝。
拒绝时，它把当前 slot 设置成 `InvalidBuffer`。
这样避免所有 ring 成员都 dirty 时出现无限循环。
这体现了 bulk read 的取舍：
如果一个 ring buffer 因 hint bit 等原因变 dirty，并且现在复用它会逼迫当前 backend 先 flush WAL，那就把它踢出 ring，重新从正常 clock-sweep 里找别的 victim。
对于 read-only scan，这通常很少发生。
对于大量修改页面的 scan，README 也说明 ring 会退化得更像正常策略。
---
## 24. BulkWrite 与 Vacuum ring 的 dirty page 处理
`BAS_BULKWRITE` 和 `BAS_VACUUM` 不使用 `StrategyRejectBuffer()` 的拒绝逻辑。
`StrategyRejectBuffer()` 明确写着只有 bulkread 模式才做。
这意味着 bulk write 或 vacuum 如果复用 ring buffer 时遇到 dirty page，需要按 buffer manager 的普通规则写出。
如果 page LSN 需要 WAL flush，就会发生 WAL flush。
为什么 vacuum 不像 bulk read 一样拒绝？
`src/backend/storage/buffer/README:233-238` 解释了设计动机。
VACUUM 使用 ring。
但 dirty pages 不从 ring 中移除。
如果复用需要 WAL，就 flush WAL。
历史上 VACUUM buffer 被送到 freelist，效果像 ring size 为 1，导致过度 WAL flushing。
现在用较大的 ring 减少这种抖动。
Bulk write 类似 VACUUM。
README 中说 bulk write 使用 16MB ring。
更小的 ring 会让 COPY 经常被 WAL flush 阻塞。
所以三类策略不是只有 ring size 不同。
它们对 dirty ring member 的态度也不同。
---
## 25. access strategy 与 IO context
`IOContextForStrategy()` 在 `src/backend/storage/buffer/freelist.c:708-738`。
它把 strategy 映射成 IO statistics context：
- `NULL` -> `IOCONTEXT_NORMAL`
- `BAS_BULKREAD` -> `IOCONTEXT_BULKREAD`
- `BAS_BULKWRITE` -> `IOCONTEXT_BULKWRITE`
- `BAS_VACUUM` -> `IOCONTEXT_VACUUM`
这影响 `pg_stat_io` 里的 context。
也影响 `GetVictimBuffer()` 写出 dirty victim 时统计为哪个 context 的 write、evict、reuse。
所以 access strategy 不只是 replacement hint。
它也是 IO 归因的一部分。
当你做实验时，可以通过 `pg_stat_io` 区分 normal、bulkread、bulkwrite、vacuum。
---
## 26. access strategy 的 pin limit
`GetAccessStrategyPinLimit()` 在 `src/backend/storage/buffer/freelist.c:559-599`。
read stream 和其他 look-ahead 代码会问 strategy：我最多应该同时 pin 多少 ring buffer。
如果 strategy 是 `NULL`，返回 `NBuffers`。
如果是 `BAS_BULKREAD`，返回整个 ring 大小。
注释说：bulk read 使用 `StrategyRejectBuffer()`，dirty buffers 不应该成为问题，所以调用者可以 pin 整个 ring。
其他 strategy 返回 ring 的一半。
这是在 look-ahead 距离和 WAL/writeback 压力之间做折中。
如果 bulk write 或 vacuum 一次 pin 太多 ring buffer，就可能迫使自己频繁写出 dirty data 并 flush WAL。
read stream 在初始化时会把这个 pin limit 纳入 `max_pinned_buffers`。
---
## 27. 大表顺序扫描何时使用 `BAS_BULKREAD`
`heapam.c` 的 `initscan()` 片段在 `src/backend/access/heap/heapam.c:360-416`。
它先确定扫描要看的 block 数 `rs_nblocks`。
然后判断表是否“大”。
条件是：
```c
!RelationUsesLocalBuffers(scan->rs_base.rs_rd) &&
scan->rs_nblocks > NBuffers / 4
```
如果表大小超过 shared buffers 的四分之一，并且 scan flags 允许 strategy，就创建：
```c
scan->rs_strategy = GetAccessStrategy(BAS_BULKREAD)
```
同一个阈值也用于 synchronized scan 的启用判断。
这不是一个精确的成本模型。
它是一个工程阈值：小表值得让 normal clock-sweep 缓存；大表全扫通常只读一次，不能让它把 shared buffers 中的工作集冲掉。
本地 temp buffers 不用这个 shared buffer strategy。
因为 temp relation 使用 local buffers。
---
## 28. sequential scan 与 read stream
本基线里，heap scan 不再只是循环 `ReadBuffer()`。
`heap_beginscan()` 附近在 `src/backend/access/heap/heapam.c:1272-1304` 设置 `scan->rs_read_stream`。
对于 sequential scan 和 TID range scan，它调用：
```c
read_stream_begin_relation(READ_STREAM_SEQUENTIAL |
                           READ_STREAM_USE_BATCHING,
                           scan->rs_strategy,
                           scan->rs_base.rs_rd,
                           MAIN_FORKNUM,
                           cb,
                           scan,
                           0)
```
`scan->rs_strategy` 可能是 `BAS_BULKREAD`，也可能是 `NULL`。
也就是说 read stream 负责预读和 IO 合并。
access strategy 负责 buffer replacement footprint。
二者是正交但协同的。
`read_stream_begin_relation()` 最终进入 `read_stream_begin_impl()`。
在 `src/backend/storage/aio/read_stream.c:758-969` 中，它会：
- 根据 tablespace IO concurrency 决定 `max_ios`。
- 根据 `io_combine_limit` 决定一次 IO 可合并多少 block。
- 计算 `max_pinned_buffers = (max_ios + 1) * io_combine_limit`。
- 用 `GetAccessStrategyPinLimit(strategy)` 再限制它。
- 设置 circular queue 和 in-progress IO queue。
- 把 `strategy` 写入每个 `ReadBuffersOperation`。
这就是 bulk read ring 和 read stream 连接的地方。
---
## 29. read stream 的状态机
`src/backend/storage/aio/read_stream.c:1-64` 顶部注释概括了算法。
用户提供 callback 生成 block number。
read stream 尝试把连续 block 合并成较大的 read。
如果不能合并，就提交已有 pending read，再开始新的 pending read。
它维护两条队列：
- pinned buffers circular queue。
- in-progress I/O circular queue。
`read_stream_look_ahead()` 在 `src/backend/storage/aio/read_stream.c:657-747`。
它循环做三件事：
1. 判断是否还能继续 look ahead。
2. 从 callback 拿下一个 block number。
3. 尝试合并进 pending read，或者提交 pending read。
`read_stream_start_pending_read()` 在 `src/backend/storage/aio/read_stream.c:318-547`。
它调用 `StartReadBuffers()`，传入 pending read 的起点和 block 数。
如果 `StartReadBuffers()` 表示需要等待，read stream 把这次 IO 记入 in-progress queue。
如果是 cache hit，不需要等待，就只增加 pinned buffer 计数。
`read_stream_next_buffer()` 在 `src/backend/storage/aio/read_stream.c:1030-1367`。
它返回最老的 pinned buffer。
如果该 buffer 对应一个还没 wait 的 IO，就先 `WaitReadBuffers()`。
如果 wait 真的阻塞了，它会增大 readahead distance。
如果一段时间都是 cache hit，它会逐渐减小 look-ahead。
这就是自适应预读。
---
## 30. read stream 的 fast path
`read_stream_next_buffer()` 顶部还有一个 fast path。
如果扫描基本全是 cache hit，且没有 per-buffer data，它可以保持一个 buffer slot。
每次返回上一次 pin 好的 buffer，同时同步 pin 下一块。
这避免完整 queue 管理的成本。
如果遇到 miss，fast path 退回普通路径。
这个细节提醒我们：
`BAS_BULKREAD` 不等于“一定疯狂预读”。
read stream 会根据 hit/miss 和 wait 情况动态调整。
全缓存扫描时，预读距离可以保持很小。
真正需要 IO 时，才逐步扩大合并和预读。
---
## 31. 顺序扫描如何避免冲刷 shared buffers
大表顺序扫描仍然把页面读入 shared buffers。
它没有绕过 buffer manager。
它避免冲刷 shared buffers 的手段是使用小 ring。
假设 shared buffers 很大，普通 OLTP 工作集中有很多热页。
一个大表 seq scan 如果走 normal strategy，就会不断通过 clock-sweep 获取 victim。
随着扫描推进，它可能把大量热页 usage_count 减到 0 并替换掉。
使用 `BAS_BULKREAD` 后，seq scan 通常只在一个小 ring 里循环。
ring 初始填充需要拿一批 victim。
之后扫描推进时反复复用这批 buffer。
这样被替换的 shared buffer 数量受 ring size 限制。
README 中对 256KB ring 的解释也很实际。
这个大小小到可以适配 CPU cache 传输效率。
又足够容纳扫描中并发 pin 的页面，并给 synchronized seq scan 留一点 cache trail。
所以“避免冲刷”不是不发生 evict。
它是把 evict/reuse 的范围限制在 ring 的 footprint 内。
如果扫描过程中大量页面被修改，ring 会承担写出 dirty page 的成本。
对于 pure read 或只偶尔 hint bit 的扫描，ring 的效果最好。
---
## 32. BulkWrite：COPY/CTAS 与扩展路径
Bulk write 的典型来源是 COPY IN、CREATE TABLE AS SELECT 这类批量写入。
在 heap AM 层，它通过 `BulkInsertState` 表达。
`GetBulkInsertState()` 创建 `BAS_BULKWRITE` strategy。
`FreeBulkInsertState()` 释放 current buffer 和 strategy。
`ReleaseBulkInsertStatePin()` 不只释放当前 buffer。
它还重置 `next_free` 和 `last_free`。
源码注释说明，这是为了避免 partition 切换时，把一个 partition 的预扩展页 offset 用到另一个 partition 上。
Bulk write ring 的 16MB 默认大小比 bulk read 大得多。
原因是写路径更容易产生 dirty buffers 和 WAL flush。
ring 太小会让同一小批 buffer 被快速复用，进而频繁写出尚未合适落盘的页面。
在 relation extension 中，bulk write strategy 会传给 `ExtendBufferedRelBy()`。
所以新页扩展时需要 victim buffer，也优先从 bulk write ring 里复用。
这让批量导入的读/写/扩展 footprint 都围绕同一个策略收敛。
---
## 33. Vacuum：可配置 ring
VACUUM 的 strategy 创建点在 `src/backend/commands/vacuum.c:430-465`。
普通 VACUUM 或 ANALYZE 会在 cross-transaction memory context 中创建 buffer strategy。
如果命令指定了 `BUFFER_USAGE_LIMIT`，使用命令参数。
否则使用 `VacuumBufferUsageLimit`，也就是 `vacuum_buffer_usage_limit` GUC。
如果 ring size 是 0，`GetAccessStrategyWithSize()` 返回 `NULL`。
这表示 VACUUM 允许完整使用 shared buffers，不使用 ring。
默认 VACUUM ring 在 `GetAccessStrategy(BAS_VACUUM)` 中是 2MB。
实际 VACUUM 通常走 with-size API，因此以 GUC/命令参数为准。
VACUUM 和 bulk read 的重要差异是 dirty page 不会被踢出 ring。
这符合 VACUUM 的工作性质。
VACUUM 本来就可能修改 visibility map、freeze 信息、hint bits 或 heap page。
它需要一个受控但不极端小的 ring，避免 WAL flush 过度频繁。
---
## 34. `GetVictimBuffer()` 是策略落地的位置
`GetVictimBuffer()` 在 `src/backend/storage/buffer/bufmgr.c:2547-2675`。
它先通过 `StrategyGetBuffer(strategy, ...)` 得到一个 pinned victim。
然后检查 victim 是否 dirty。
如果 dirty，需要拿 share-exclusive content lock。
这里用 conditional lock。
因为从 `StrategyGetBuffer()` 返回到尝试写出之间，别人可能重新 pin 并 lock 了这个 buffer。
如果无条件等待，可能和对方形成死锁。
拿不到锁就释放 victim，重新选择。
如果 dirty 且需要 WAL flush，bulk read 可以拒绝 ring buffer。
如果最终要写出，就调用 `FlushBuffer()`。
写出后，若 buffer 仍 valid，会统计为 evict 或 reuse。
然后 `InvalidateVictimBuffer()` 从 buffer mapping table 删除旧 tag。
如果删除失败，说明期间又有人 pin 或 dirty 了它，就释放并重试。
这说明 access strategy 只是 victim 候选来源。
最终能不能复用，还要通过 buffer manager 的并发协议验证。
---
## 35. 并发扩展 race：现代路径如何处理
现代 `ExtendBufferedRelShared()` 的并发控制有几个层次。
第一层是 relation extension lock。
它保护“决定 EOF 并推进 EOF”的临界区。
第二层是 buffer mapping table partition lock。
它保护新 block tag 插入 buffer table。
第三层是 buffer IO state。
它保护新 block 在 zero extend 完成前不被其他 backend 当作 valid page 使用。
第四层是 buffer content lock。
它保护返回给 heap 初始化的第一页不被别人同时修改。
最常见 race 是：
1. backend A 先拿到 victim buffers。
2. backend B 也拿到 victim buffers。
3. B 先拿到 extension lock，并把 relation 从 100 扩到 108。
4. A 后拿到 extension lock。
5. A 调用 `smgrnblocks()` 时看到 EOF 已是 108。
6. A 的 `first_block` 就是 108，而不是 100。
这不是错误。
A 只是在 B 后面继续扩。
更复杂的是 `ExtendBufferedRelTo()`。
它的语义是“至少扩到 N 个 block，并返回 N-1 block 的 buffer”。
如果别的 backend 已经扩到了目标，当前 backend 会释放多余 victim buffers，实际扩展 0 页。
随后 `ExtendBufferedRelTo()` 会 fallback 到 `ReadBuffer_common()` 读取目标 block。
这就是 `src/backend/storage/buffer/bufmgr.c:1115-1126` 的逻辑。
并发下，“我准备扩展”不保证“我实际扩展”。
调用者必须按返回值和 `extended_by` 处理。
---
## 36. `P_NEW` 旧路径与新路径的对照
旧风格：
```c
LockRelationForExtension(rel, ExclusiveLock);
buf = ReadBuffer(rel, P_NEW);
LockBuffer(buf, BUFFER_LOCK_EXCLUSIVE);
... initialize page ...
UnlockRelationForExtension(rel, ExclusiveLock);
```
这种风格的问题是扩展锁可能覆盖过多工作。
比如找 victim buffer、写出 dirty victim、清零 buffer，都可能发生在锁内或围绕锁的复杂区域。
新风格：
```c
buf = ExtendBufferedRel(BMR_REL(rel), fork, strategy, EB_LOCK_FIRST);
... initialize page ...
```
或者批量：
```c
first = ExtendBufferedRelBy(..., EB_LOCK_FIRST,
                            extend_by, buffers, &extended_by);
```
`ExtendBufferedRelShared()` 会先找 victim 和清零，再拿 extension lock。
锁内只做必须串行化的 EOF 决策、buffer tag 安装、physical zero extend。
这就是 `ReadBuffer_common()` 注释里说“在 `ExtendBufferedRel()` 内部获取 extension lock scales a lot better”的原因。
如果你在新代码中需要扩展 relation，优先找 `ExtendBufferedRel*()`。
只有维护旧 AM 或特殊调用点时，才应该继续走 `P_NEW`，而且必须审核锁。
---
## 37. zero page 与 WAL 边界的常见误区
误解一：`smgrzeroextend()` 已经把 page 写到磁盘，所以后续 `PageInit()` 不需要 dirty。
错误。
`smgrzeroextend()` 写的是全零页。
`PageInit()` 把 page header 改成合法 page layout。
这已经改变 shared buffer 内容，必须 `MarkBufferDirty()`。
误解二：新扩展 page 是空页，所以不需要 WAL。
不完整。
对 permanent logged relation，插入 tuple 仍需要 WAL。
如果是 page 第一条 tuple，可以使用 `XLOG_HEAP_INIT_PAGE` 优化 redo。
但这仍然是 WAL record。
误解三：`MarkBufferDirty()` 会写 WAL。
错误。
`MarkBufferDirty()` 只设置 buffer dirty 状态。
WAL 由 heap、btree、gin 等访问方法自己生成。
page LSN 也由调用者在拿到 `XLogInsert()` 返回值后设置。
误解四：BulkRead 绕过 shared buffers。
错误。
它仍读入 shared buffers。
只是用 ring 限制 replacement footprint。
误解五：Vacuum ring 会避免所有 WAL flush。
错误。
VACUUM dirty ring buffer 复用时仍可能 flush WAL。
ring 的作用是把缓冲区占用和 flush 频率控制在合理范围，而不是取消 WAL-before-data。
---
## 38. 可执行实验一：观察大表顺扫的 bulkread IO context
前提：使用 `/home/nail/postgres-lab` 构建出的 PostgreSQL，能连接到一个测试库。
如果你的实例没有开启 `track_io_timing`，也能看计数；开启后能看时间。
准备：
```sql
DROP TABLE IF EXISTS course_bulkread;
CREATE TABLE course_bulkread AS
SELECT g AS id, repeat(md5(g::text), 8) AS payload
FROM generate_series(1, 2000000) AS g;
VACUUM ANALYZE course_bulkread;
CHECKPOINT;
SELECT pg_stat_reset_shared('io');
```
执行顺序扫描：
```sql
SET enable_indexscan = off;
SET enable_bitmapscan = off;
SET enable_seqscan = on;
EXPLAIN (ANALYZE, BUFFERS)
SELECT count(*) FROM course_bulkread;
```
观察 IO context：
```sql
SELECT backend_type, object, context,
       reads, writes, extends, hits, evictions, reuses
FROM pg_stat_io
WHERE context IN ('bulkread', 'normal')
ORDER BY backend_type, object, context;
```
预期现象：
- 如果表足够大，heap scan 会使用 `BAS_BULKREAD`。
- `pg_stat_io` 中应能看到 `bulkread` context 的读或 hit/reuse/evict 变化。
- `EXPLAIN (BUFFERS)` 会显示 shared read/hit，但不会告诉你 access strategy，需要结合 `pg_stat_io`。
可选观察 shared buffers 中保留了多少该表页面：
```sql
CREATE EXTENSION IF NOT EXISTS pg_buffercache;
SELECT count(*) AS cached_pages,
       count(*) FILTER (WHERE isdirty) AS dirty_pages,
       min(usagecount) AS min_usage,
       max(usagecount) AS max_usage
FROM pg_buffercache b
WHERE b.relfilenode = pg_relation_filenode('course_bulkread'::regclass)
  AND b.reldatabase = (SELECT oid FROM pg_database WHERE datname = current_database());
```
注意：
- `pg_buffercache` 显示的是某个瞬间。
- bulkread 不是不缓存，而是 ring footprint 小。
- 如果其他 session 同时访问该表，缓存页面数量会被干扰。
---
## 39. 可执行实验二：观察 bulk write 与 relation extend
准备：
```sql
DROP TABLE IF EXISTS course_bulkwrite;
SELECT pg_stat_reset_shared('io');
CHECKPOINT;
```
执行 CTAS：
```sql
CREATE TABLE course_bulkwrite AS
SELECT g AS id, repeat(md5(g::text), 8) AS payload
FROM generate_series(1, 1000000) AS g;
```
观察：
```sql
SELECT backend_type, object, context,
       reads, writes, extends, evictions, reuses
FROM pg_stat_io
WHERE context IN ('bulkwrite', 'normal')
ORDER BY backend_type, object, context;
```
再看 relation 大小：
```sql
SELECT pg_relation_size('course_bulkwrite') AS bytes,
       pg_relation_size('course_bulkwrite') / current_setting('block_size')::int AS blocks;
```
预期现象：
- CTAS/COPY 类路径通常会使用 bulk insert state。
- bulk insert state 使用 `BAS_BULKWRITE`。
- relation 文件增长会反映在 `extends` 或 write 相关统计中，具体计数受版本、IO method、缓存命中状态影响。
如果要更贴近 COPY：
```sql
DROP TABLE IF EXISTS course_copy;
CREATE TABLE course_copy(id int, payload text);
SELECT pg_stat_reset_shared('io');
COPY course_copy
FROM PROGRAM
'seq 1 1000000 | awk ''{ printf "%s\\t%s\\n", $1, "payload_"$1 }'''
WITH (FORMAT text);
SELECT backend_type, object, context,
       reads, writes, extends, evictions, reuses
FROM pg_stat_io
WHERE context = 'bulkwrite'
ORDER BY backend_type, object;
```
如果系统禁用了 `COPY FROM PROGRAM`，可以在 shell 里生成文件后使用 `COPY FROM '/path'`。
---
## 40. 可执行实验三：观察 extension lock 等待
这个实验需要两个或多个 shell。
它依赖并发插入制造 relation extension contention。
准备表：
```sql
DROP TABLE IF EXISTS course_ext_lock;
CREATE TABLE course_ext_lock(id bigint, payload text) WITH (fillfactor = 100);
```
创建一个 pgbench 脚本，例如 `/tmp/course_ext_insert.sql`：
```sql
INSERT INTO course_ext_lock
SELECT :client_id * 1000000000 + g,
       repeat(md5((:client_id * 1000000000 + g)::text), 8)
FROM generate_series(1, 50) AS g;
```
在 shell 1 运行：
```bash
pgbench -n -c 16 -j 16 -t 2000 -f /tmp/course_ext_insert.sql your_database
```
在 shell 2 反复观察 locks：
```sql
SELECT locktype, relation::regclass, mode, granted, count(*)
FROM pg_locks
WHERE locktype = 'extend'
GROUP BY locktype, relation, mode, granted
ORDER BY relation::regclass::text, granted;
```
也可以观察等待事件：
```sql
SELECT pid, wait_event_type, wait_event, query
FROM pg_stat_activity
WHERE wait_event_type = 'Lock'
  AND query LIKE 'INSERT INTO course_ext_lock%';
```
预期现象：
- 是否能观察到等待取决于机器速度、表大小、tuple 宽度和并发度。
- 如果没有看到，增加 `-c`、增加每次 insert 的行数、或者把 payload 做宽一些。
- 观察到的 `extend` lock 等待，对应 `LockRelationForExtension()`。
源码对应：
- heap 的 `RelationAddBlocks()` 会用 `RelationExtensionLockWaiterCount()` 读取等待者数量。
- 有等待者时，它会多扩展一些页，减少后续竞争。
---
## 41. 讨论题
1. 为什么 `P_NEW` 是兼容入口，而不是新代码应优先使用的 relation extension API？
2. `ExtendBufferedRelShared()` 为什么先拿 victim buffer，再进入 relation extension lock？
3. 为什么必须在 `smgrzeroextend()` 前把新 block 的 buffer tag 和 I/O in progress 状态放进 buffer table？
4. `smgrzeroextend()`、`PageInit()`、`MarkBufferDirty()`、`XLogInsert()`、`PageSetLSN()` 分别推进了哪个状态？
5. heap insert 为什么先找 bulk current buffer、FSM、last block，而不是总是 append？
6. `BAS_BULKREAD` 为什么可以拒绝需要 WAL flush 的 dirty ring buffer，而 `BAS_BULKWRITE` / `BAS_VACUUM` 的取舍不同？
7. 大表顺序扫描仍经过 shared buffers，为什么通常不会把正常工作集全部冲掉？
8. `pg_stat_io` 能看到 bulkread/bulkwrite context，但看不到哪些 relation extension 内部状态？
9. 如果写一个新 access method，需要如何证明新增 page 的 extension lock、dirty、WAL、page LSN 和 free-space 可发现性都成立？
## 42. 本节小结
`P_NEW` 是一个历史兼容入口，不是推荐的新扩展 API。
它在 `ReadBuffer_common()` 中会转到 `ExtendBufferedRel()`，并跳过内部 extension lock。
所以使用 `P_NEW` 的调用者必须自己证明并发安全。
`ExtendBufferedRelBy()` 是当前理解 relation extension 的核心入口。
它能一次扩展多个 block，返回 pinned buffers，并按 flags 锁定第一个或目标 buffer。
shared relation extension 会先找 victim buffer 并清零，再拿 relation extension lock。
锁内读取准确 EOF、插入 buffer tag、启动 buffer IO、执行 `smgrzeroextend()`。
物理扩展完成后释放 extension lock，再把 buffer 标为 `BM_VALID` 并唤醒等待者。
heap insert 并不是直接 append。
它先尝试 bulk current buffer、relcache target block、FSM 和最后一页。
只有找不到空间时才通过 `RelationAddBlocks()` 扩展。
批量插入会估算需要 page 数，结合 waiter count、历史扩展大小和 64 页上限做 multi-block extend。
zero page、initialized page、dirty page、WAL-covered page 是四个不同状态。
`smgrzeroextend()` 只保证物理 zero blocks 存在。
`PageInit()` 初始化 page layout。
`MarkBufferDirty()` 标记 buffer 内容需要写回。
`XLogInsert()` 与 `PageSetLSN()` 建立 WAL-before-data 所需的 page LSN 边界。
access strategy ring 是 shared buffers 内的小范围复用机制。
`BAS_BULKREAD` 默认 256KB 起步，适合大表一次性读，并能拒绝需要 WAL flush 的 dirty ring buffer。
`BAS_BULKWRITE` 默认 16MB，适合 COPY/CTAS 等批量写入，减少写路径过小 ring 导致的 WAL flush 抖动。
`BAS_VACUUM` 默认 2MB，但通常由 `vacuum_buffer_usage_limit` 或命令的 `BUFFER_USAGE_LIMIT` 控制。
大表顺序扫描仍走 shared buffers。
它通过 bulk read ring 限制 footprint，通过 read stream 做自适应 look-ahead 和 IO combining。
这就是“顺序扫描避免冲刷 shared buffers”的真实含义。
并发扩展 race 的正确处理不是“大家都不 race”，而是把 race 收束到明确协议里。
extension lock 串行化 EOF 决策和物理扩展。
buffer table 和 IO state 防止其他 backend 看到半初始化 page。
`ExtendBufferedRelTo()` 能处理别人已经扩到目标的情况。
existing beyond-EOF buffer 分支则处理失败扩展或异常读留下的边界状态。
读这部分源码时，最重要的习惯是分层。
buffer tag、buffer valid、物理文件 EOF、page layout、dirty bit、page LSN、WAL flush，它们不是同一个状态。
PostgreSQL 的正确性正来自这些状态之间清晰且严格的推进顺序。
