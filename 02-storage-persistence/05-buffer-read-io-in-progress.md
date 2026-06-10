# PostgreSQL Buffer read I/O 与 BM_IO_IN_PROGRESS
## 课程定位
本节基于 PostgreSQL 源码仓库 `/home/nail/postgres-lab`。
本节唯一主问题：同一个 relation fork block 被多个 backend 同时读取时，谁有权发起 I/O，其他 backend 如何等待同一个 buffer 变成 valid？
核心矛盾：buffer table 必须尽早暴露唯一 tag，防止同一磁盘页出现多个 shared buffer 副本；但 page bytes 在 I/O 完成前不能被任何普通访问者消费。
本节主流程：`ReadBuffer_common()` -> `PinBufferForBlock()` -> `BufferAlloc()` -> `StartSharedBufferIO()` / `StartBufferIO()` -> smgr/md read -> completion callback -> `TerminateBufferIO()` 或 `AbortBufferIO()`。
## 源码基线
基线为 `master`，提交号 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`。
## 阅读路径
`src/backend/storage/buffer/bufmgr.c`
`src/backend/storage/buffer/buf_init.c`
`src/backend/storage/buffer/localbuf.c`
`src/include/storage/buf_internals.h`
`src/backend/storage/smgr/smgr.c`
`src/backend/storage/smgr/md.c`
本节只讨论一条主线：一个 backend 要读一个 block，buffer manager 如何避免多个 backend 重复把同一页读进不同 buffer。
换句话说，我们要回答三个问题：
第一，buffer tag 什么时候进入 buffer table。
第二，谁有权真正发起磁盘读。
第三，其他 backend 发现读正在进行时如何等待、加入或重试。
这里的关键标志位是 `BM_IO_IN_PROGRESS`。
它不是“这个页已经在磁盘 I/O 队列里”的粗略提示。
它是 buffer 描述符上的并发协议位。
它表达的是：“这个 buffer 的内容还不能被普通访问者消费，当前有一个被授权的 I/O 拥有者正在把内容变成可用状态。”
读成功时，`BM_IO_IN_PROGRESS` 被清掉，并设置 `BM_VALID`。
读失败时，`BM_IO_IN_PROGRESS` 被清掉，并设置 `BM_IO_ERROR`，但不设置 `BM_VALID`。
后续 backend 再读同一个 block 时会看到 tag 已经存在，但 page 仍非 valid，于是可以重新尝试 I/O。
这一节的结论先放在前面：
同一个 relation fork block 在 shared buffers 中最多应该有一个 buffer tag 入口。
这个入口必须在 I/O 开始前可查。
真正发起 I/O 前必须把 `BM_IO_IN_PROGRESS` 置位。
完成 I/O 时必须用 `TerminateBufferIO()` 原子地清理状态并唤醒等待者。
错误退出时必须用 resource owner 触发 `AbortBufferIO()`，避免永久留下 `BM_IO_IN_PROGRESS`。
## 1. 源码坐标
先建立源码地图。
`buf_init.c` 的注释给出了 buffer manager 的两个不变量。
第一，buffer lookup 必须在 I/O 开始前可见。
如果不是这样，第二个 backend 读同一 block 时可能分配另一个 buffer。
那会让 buffer pool 对同一磁盘块拥有两个副本。
第二，buffer 在 I/O 期间不能被替换。
所以 buffer 在 I/O 期间会保持 pin。
这些注释位于 `buf_init.c` 约 45 到 70 行。
`buf_internals.h` 定义了状态位。
`BM_VALID` 表示 buffer 中的数据可用。
`BM_IO_IN_PROGRESS` 表示读或写正在进行。
`BM_IO_ERROR` 表示前一次 I/O 失败。
这些定义位于 `buf_internals.h` 约 105 到 116 行。
同一个头文件还定义了 `StartBufferIOResult`。
它有三个值：
`BUFFER_IO_ALREADY_DONE`
`BUFFER_IO_IN_PROGRESS`
`BUFFER_IO_READY_FOR_IO`
这三个返回值是读等待协议的核心语义。
它们定义在 `buf_internals.h` 约 558 到 569 行。
`BufferDesc` 的核心字段包括 `tag`、`state`、`wait_backend_pgprocno`、`io_wref` 和 content lock waiters。
`state` 是一个 `pg_atomic_uint64`，里面同时放 refcount、usagecount、content lock 状态和 `BM_*` 标志。
`io_wref` 是 AIO wait reference。
如果 buffer 上存在异步 I/O，其他 backend 可以拿这个引用去等待 AIO handle。
`BufferIOCVArray` 是每个 shared buffer 对应的 I/O condition variable 数组。
`BufferDescriptorGetIOCV()` 通过 `buf_id` 找到该 buffer 的 I/O condition variable。
这些内容位于 `buf_internals.h` 约 326 到 443 行。
`bufmgr.c` 是本节主角。
读入口在 `ReadBufferExtended()` 和 `ReadBuffer_common()`。
buffer 查找和分配在 `PinBufferForBlock()` 与 `BufferAlloc()`。
批量读准备在 `StartReadBuffersImpl()`。
实际发起读在 `AsyncReadBuffers()`。
等待读完成在 `WaitReadBuffers()`。
buffer I/O 状态协议在 `WaitIO()`、`StartSharedBufferIO()`、`StartBufferIO()`、`TerminateBufferIO()` 和 `AbortBufferIO()`。
`smgr.c` 把 buffer manager 的读请求转给 storage manager。
`md.c` 是磁盘文件实现。
同步读落到 `mdreadv()`。
异步读落到 `mdstartreadv()` 和 `md_readv_complete()`。
## 2. 读一页的总体调用链
普通调用常见入口是 `ReadBuffer()` 或 `ReadBufferExtended()`。
`ReadBuffer()` 只是读 `MAIN_FORKNUM` 的包装。
`ReadBufferExtended()` 允许指定 fork、block、read mode 和 access strategy。
`ReadBufferExtended()` 内部调用 `ReadBuffer_common()`。
`ReadBuffer_common()` 是所有 read buffer 变体的公共入口。
对普通读来说，主线是：
`ReadBufferExtended()`
`ReadBuffer_common()`
`StartReadBuffer()`
`StartReadBuffersImpl()`
`PinBufferForBlock()`
`BufferAlloc()`
`AsyncReadBuffers()`
`StartBufferIO()`
`smgrstartreadv()`
`mdstartreadv()`
`FileStartReadV()`
completion callbacks
`TerminateBufferIO()`
`WaitReadBuffers()`
这条链里最容易误读的是 `StartReadBuffer()`。
它不是“立刻读完一个 block”。
它是启动一个读操作，或者发现不需要读。
如果它返回 false，说明目标 buffer 已经有效，调用者不用等待。
如果它返回 true，调用者必须调用 `WaitReadBuffers()`，否则不能访问该 buffer 内容。
`ReadBuffer_common()` 对普通读总是设置 `READ_BUFFERS_SYNCHRONOUSLY`。
这个 flag 的意思不是完全绕过 AIO 代码。
在当前源码中，即使同步等待，也复用 AIO 框架。
它的意思是：调用者马上会等待，所以底层可以把这次 I/O 标记成同步型，减少异步调度开销。
如果 read mode 是 `RBM_ZERO_ON_ERROR`，还会加 `READ_BUFFERS_ZERO_ON_ERROR`。
`zero_damaged_pages` 打开时，`AsyncReadBuffers()` 也会在本 backend 发起 I/O 时把该行为编码到 read flags。
这是因为完成回调可能在其他进程或 I/O worker 中执行，不能依赖完成者本地的 GUC 值。
## 3. ReadBuffer_common 的分支
`ReadBuffer_common()` 位于 `bufmgr.c` 约 1271 到 1368 行。
这个函数先拒绝读取其他 session 的临时表。
如果 `rel` 是其他 backend 的 temp relation，会报错。
原因是 temp relation 使用 local buffers，当前 backend 看不到拥有者 session 的 local buffer 状态。
然后处理 `blockNum == P_NEW` 的兼容路径。
`P_NEW` 表示扩展 relation 并返回新 block。
新的推荐接口是 `ExtendBufferedRel()`。
但为了兼容旧调用，`ReadBuffer_common()` 遇到 `P_NEW` 会转到 `ExtendBufferedRel()`。
如果 mode 是 `RBM_ZERO_AND_LOCK` 或 `RBM_ZERO_AND_CLEANUP_LOCK`，会设置扩展时锁住新页的 flag。
普通 block 读继续向下。
函数根据 relation 或传入的 `smgr_persistence` 确定 persistence。
如果 mode 是 `RBM_ZERO_AND_LOCK` 或 `RBM_ZERO_AND_CLEANUP_LOCK`，走特殊路径。
特殊路径先 `PinBufferForBlock()`。
然后调用 `ZeroAndLockBuffer()`。
这个路径不从磁盘读 page。
如果 buffer 已经 valid，就只加内容锁。
如果 buffer 还不 valid，就把页面清零，拿内容锁，再把 buffer 标成 valid。
这样调用者拿到的是一个已锁住、可初始化的新页面。
除了 zero-and-lock 特殊路径，普通读会构造 `ReadBuffersOperation`。
字段包括 `smgr`、`rel`、`persistence`、`forknum` 和 `strategy`。
然后调用单块版本 `StartReadBuffer()`。
如果 `StartReadBuffer()` 返回 true，马上调用 `WaitReadBuffers()`。
最后返回 pinned buffer。
注意：`ReadBuffer_common()` 返回时，buffer 一定已经 pinned。
但是否持有 content lock 取决于 read mode。
普通 `RBM_NORMAL` 不返回 content lock。
`RBM_ZERO_AND_LOCK` 返回 exclusive content lock。
`RBM_ZERO_AND_CLEANUP_LOCK` 对已 valid buffer 会走 cleanup lock。
对于新清零的 invalid buffer，源码注释说明 exclusive lock 与 cleanup-strength lock 没有实际区别，因为还没有其他 backend 能看内容。
## 4. PinBufferForBlock 的职责
`PinBufferForBlock()` 位于 `bufmgr.c` 约 1217 到 1268 行。
它的职责很窄：
给定 relation fork block，返回一个 pinned buffer。
它还通过 `foundPtr` 告诉调用者这个 buffer 内容是否已经可用。
如果 persistence 是 `RELPERSISTENCE_TEMP`，调用 `LocalBufferAlloc()`。
否则调用 `BufferAlloc()`。
对于本节核心的 shared buffers，重点是 `BufferAlloc()`。
`PinBufferForBlock()` 还做统计。
如果 `foundPtr` 为 true，会调用 `TrackBufferHit()`。
如果传入了 `rel`，总是调用 `pgstat_count_buffer_read(rel)`。
这里容易混淆两个统计口径。
`pgBufferUsage` 的 read counter 只在真正发起 I/O 后增加。
per-relation 的 buffer read 统计则在进入这条读路径时就计数。
所以 cache hit、zero-and-lock、甚至后续发现不用读的情况，统计口径可能不同。
这不是并发协议的一部分，但跟实验观察有关。
## 5. BufferAlloc 的查找协议
`BufferAlloc()` 位于 `bufmgr.c` 约 2178 到 2351 行。
它是 `PinBufferForBlock()` 对 shared buffers 的核心子过程。
它做三件事：
第一，在 buffer mapping table 里查找目标 tag。
第二，如果找不到，选择并初始化一个 victim buffer。
第三，保证返回时 buffer 已经 pinned，且 tag 已经对应目标 block。
`BufferAlloc()` 一开始为当前 resource owner 和 private refcount 预留空间。
这是因为 pin buffer 可能需要记录资源。
随后用 relfilenode、forkNum、blockNum 构造 `BufferTag`。
再计算 hash 和 partition lock。
buffer mapping table 分区锁用于保护 tag 到 buffer id 的映射。
第一轮查找使用 `LW_SHARED`。
如果 `BufTableLookup()` 找到 existing buffer id，就拿到 `BufferDesc`。
然后调用 `PinBuffer(buf, strategy, false)`。
`PinBuffer()` 返回一个 bool，表示此刻是否看到 `BM_VALID`。
如果返回 true，`BufferAlloc()` 认为是 cache hit。
如果返回 false，说明 hash 里已有 tag，但内容还不能用。
源码注释列出三种情况：
有人正在读这个 page。
前一次读失败。
有人调用了 `StartReadBuffers()` 但尚未 `WaitReadBuffers()`。
在这些情况下，`BufferAlloc()` 仍然返回这个 pinned buffer。
但它把 `foundPtr` 置为 false。
这很关键。
`foundPtr=false` 不一定意味着 buffer table 里没有条目。
它只表示调用者还需要执行 I/O 协议，不能直接访问页面内容。
如果第一轮没找到，`BufferAlloc()` 释放 partition lock。
然后调用 `GetVictimBuffer()` 选择一个 victim。
选择 victim 不持有目标 tag 的 partition lock。
这样避免在 flush dirty victim 或等锁时长时间占住 mapping lock。
`GetVictimBuffer()` 返回 pinned victim。
victim 可能原来有 tag、valid 内容或 dirty 内容。
如果 dirty，先尝试拿 share-exclusive content lock，然后 `FlushBuffer()` 写出。
写出时也会用 `BM_IO_IN_PROGRESS` 保护 buffer。
本节主线是 read，但要知道 `BM_IO_IN_PROGRESS` 同时服务读和写。
dirty victim 写完后，`GetVictimBuffer()` 会调用 `InvalidateVictimBuffer()` 尝试从 mapping table 删除旧 tag。
删除前会确认 refcount 仍为 1 且没有被重新 dirty。
如果中途别人 pin 或 dirty 了这个 victim，就放弃这个 victim，重新找。
所以被返回给 `BufferAlloc()` 的 victim 满足：
当前 backend 持有唯一 pin。
没有 `BM_TAG_VALID`。
没有 `BM_VALID`。
没有 `BM_DIRTY`。
随后 `BufferAlloc()` 重新拿目标 tag 的 partition lock，并尝试 `BufTableInsert()`。
这里存在并发竞争。
另一个 backend 可能已经插入了同一个 tag。
如果插入发现 collision，当前 backend 放弃自己的 victim，转而 pin 已存在的 buffer。
这条路径的后半段必须与开头“查找到 existing buffer”的逻辑一致。
如果 `PinBuffer(existing_buf_hdr, strategy, false)` 看到 valid，则 found。
否则 found false，后续进入等待或重试。
如果插入成功，`BufferAlloc()` 锁住 victim buffer header。
它写入新的 `tag`。
然后设置 `BM_TAG_VALID` 和 usagecount。
如果 relation 是 permanent，或者 fork 是 init fork，还设置 `BM_PERMANENT`。
注意此时还不会设置 `BM_VALID`。
也还没有设置 `BM_IO_IN_PROGRESS`。
`BufferAlloc()` 的注释明确说：它不读入新页面。
它只负责让 buffer table 里有一个 pinned、带目标 tag、内容 invalid 的 buffer。
调用者后续通过 `StartBufferIO()` 取得 I/O 权限。
这一点正好呼应 `buf_init.c` 的不变量：
buffer 必须先能被 lookup 到，然后才开始 I/O。
否则多个 backend 可能给同一个 disk block 分配多个 buffer。
## 6. 正确性状态：BM_VALID、BM_IO_IN_PROGRESS、BM_IO_ERROR
这三个 flag 组成读 I/O 状态机。
`BM_VALID` 表示 buffer 内容已经可用。
对读来说，`BM_VALID` 通常在读完成并通过 page 验证后设置。
对 zero-and-lock 来说，页面被清零后也会设置 `BM_VALID`。
对 extend 来说，relation 文件扩展成功后，新 zero page 对应的 buffer 会设置 `BM_VALID`。
`BM_IO_IN_PROGRESS` 表示当前有读或写 I/O 正在进行。
它阻止其他 backend 同时对同一 buffer 发起另一个 I/O。
它也阻止其他 backend 在页面内容尚未准备好时把 invalid buffer 当成可用页。
`BM_IO_ERROR` 表示上一轮 I/O 失败。
它不是永久故障标记。
`TerminateBufferIO()` 每次结束 I/O 时都会先清掉旧的 `BM_IO_ERROR`。
如果本次 I/O 失败，再通过 `set_flag_bits` 把 `BM_IO_ERROR` 设置回来。
如果本次 I/O 成功，旧错误就被清除。
所以 `BM_IO_ERROR` 的用途是记录最近一次失败，并影响诊断和重试路径。
状态可以简化成以下几类。
初始新分配 buffer：
`BM_TAG_VALID=1`
`BM_VALID=0`
`BM_IO_IN_PROGRESS=0`
`BM_IO_ERROR=0`
读 I/O 已被某 backend 认领：
`BM_TAG_VALID=1`
`BM_VALID=0`
`BM_IO_IN_PROGRESS=1`
读成功：
`BM_TAG_VALID=1`
`BM_VALID=1`
`BM_IO_IN_PROGRESS=0`
`BM_IO_ERROR=0`
读失败：
`BM_TAG_VALID=1`
`BM_VALID=0`
`BM_IO_IN_PROGRESS=0`
`BM_IO_ERROR=1`
写 I/O 进行中则不同。
写通常要求 buffer 已 valid 且 dirty。
写成功会清 dirty，而不需要重新设置 valid。
写失败会保留 dirty，并设置 `BM_IO_ERROR`。
本节主线是读，但 `AbortBufferIO()` 中可以看到写失败会特别 warning 多次失败。
## 7. StartBufferIO 和 StartSharedBufferIO
`StartBufferIO()` 位于 `bufmgr.c` 约 7330 到 7345 行。
它只是 wrapper。
如果 buffer 是 local buffer，调用 `StartLocalBufferIO()`。
否则调用 `StartSharedBufferIO()`。
shared buffer 的真实协议在 `StartSharedBufferIO()`。
它位于 `bufmgr.c` 约 7211 到 7321 行。
调用前提是 buffer 已经 pinned。
`forInput=true` 表示读。
`forInput=false` 表示写。
`wait` 表示遇到已有 I/O 时是否可以同步等待。
`io_wref` 是可选输出参数，用于加入已存在的 AIO。
函数先 `ResourceOwnerEnlarge(CurrentResourceOwner)`。
如果它成功把 `BM_IO_IN_PROGRESS` 置位，就会把这次 buffer I/O 记入 resource owner。
然后进入循环。
循环中先锁 buffer header。
如果当前没有 `BM_IO_IN_PROGRESS`，跳出循环，继续检查是否需要 I/O。
如果已经有 `BM_IO_IN_PROGRESS`，说明别的 backend 或当前 backend 的另一个操作正在处理这个 buffer。
这时根据参数决定行为。
第一种情况：调用者传了非 NULL 的 `io_wref`，且 buffer 上的 `buf->io_wref` 有效。
函数把 buffer 的 wait reference 复制给调用者。
释放 header lock。
返回 `BUFFER_IO_IN_PROGRESS`。
这表示调用者可以异步等待已有 I/O。
注意这不受 `wait` 参数影响。
因为拿到 wait reference 比同步睡眠更适合 read stream 和并发扫描。
第二种情况：调用者没要 wait reference，且 `wait=false`。
函数释放 header lock。
返回 `BUFFER_IO_IN_PROGRESS`。
这常用于尝试合并相邻 block 的读。
如果第二个 block 已有人在读，就不要等它，停止本次合并即可。
第三种情况：`wait=true`，但无法通过 wait reference 加入。
例如调用者传了 `io_wref=NULL`。
或者另一个 backend 刚设置了 `BM_IO_IN_PROGRESS`，还没把 AIO wait reference 写入 buffer。
这时只能同步等待。
等待前会调用 `pgaio_submit_staged()`。
这是为了避免当前 backend 已经 staged 但未提交的 I/O 卡住别人。
然后调用 `WaitIO(buf)`。
`WaitIO()` 返回后回到循环，从头检查状态。
这点很重要。
等待结束不等于一定能自己发起 I/O。
可能别人已经读成功并设置 `BM_VALID`。
也可能别人读失败，留下 `BM_IO_ERROR`。
跳出循环时，保证当前 buffer 上没有 active I/O。
然后检查工作是否已经完成。
对读来说，如果 `BM_VALID` 已经是 1，就返回 `BUFFER_IO_ALREADY_DONE`。
对写来说，如果 buffer 已经不 dirty，就返回 `BUFFER_IO_ALREADY_DONE`。
如果工作还没完成，就通过 `UnlockBufHdrExt()` 设置 `BM_IO_IN_PROGRESS`。
随后调用 `ResourceOwnerRememberBufferIO()`。
最终返回 `BUFFER_IO_READY_FOR_IO`。
这个返回值表示：“你现在是这个 buffer I/O 的拥有者，必须最终调用 `TerminateBufferIO()` 或在错误处理中被 `AbortBufferIO()` 清理。”
## 8. WaitIO 的等待协议
`WaitIO()` 位于 `bufmgr.c` 约 7145 到 7208 行。
它只用于 shared buffers。
local buffers 不需要跨进程 condition variable。
`WaitIO()` 的目标很简单：
阻塞直到指定 buffer 的 `BM_IO_IN_PROGRESS` 被清掉。
但当前源码里它有两层等待机制。
第一层是 buffer 自己的 condition variable。
`BufferDescriptorGetIOCV(buf)` 从 `BufferIOCVArray` 找到对应 CV。
第二层是 AIO wait reference。
如果 buffer 的 `io_wref` 有效，就优先等待 AIO handle。
函数一开始断言没有 staged AIO。
这是为了避免进入 AIO 不感知的等待路径时留下未提交 I/O。
然后 `ConditionVariablePrepareToSleep(cv)`。
循环中先锁 buffer header。
在持有 header lock 时读取 `buf->io_wref`。
这样可以避免与另一个 backend 的 `TerminateBufferIO()` 并发清理 wait reference 产生竞态。
然后释放 header lock。
如果 `BM_IO_IN_PROGRESS` 已经清掉，循环结束。
如果 `io_wref` 有效，调用 `pgaio_wref_wait(&iow)`。
AIO 子系统内部也使用 condition variable。
等待 AIO 后，代码重新 `ConditionVariablePrepareToSleep(cv)`，避免因为 AIO 等待过程中被从 buffer CV 的 wait list 移除而多转一次。
如果 `io_wref` 无效，则调用 `ConditionVariableSleep(cv, WAIT_EVENT_BUFFER_IO)`。
所以在 `pg_stat_activity.wait_event` 中，等待 buffer I/O 可能看到 `BufferIO`。
如果等待的是 AIO handle，可能看到 `AioIoCompletion`。
真正的文件读等待在更低层，使用 `WAIT_EVENT_DATA_FILE_READ`。
扩展文件使用 `WAIT_EVENT_DATA_FILE_EXTEND`。
这些 wait event 名称来自 `src/backend/utils/activity/wait_event_names.txt`。
`WaitIO()` 结束前调用 `ConditionVariableCancelSleep()`。
等待者不直接修改 buffer 状态。
它只等发起 I/O 的拥有者清状态并广播。
## 9. TerminateBufferIO 的结束协议
`TerminateBufferIO()` 位于 `bufmgr.c` 约 7348 到 7413 行。
它是结束 shared buffer I/O 的唯一公共核心函数。
调用前提：
当前 I/O 拥有者正在执行 buffer I/O。
buffer 上设置了 `BM_IO_IN_PROGRESS`。
buffer 仍然 pinned。
参数 `clear_dirty` 用于写成功。
如果 true，会清掉 `BM_DIRTY` 和 `BM_CHECKPOINT_NEEDED`。
参数 `set_flag_bits` 用来设置结束状态。
读成功传 `BM_VALID`。
读失败传 `BM_IO_ERROR`。
写成功通常传 0。
写失败传 `BM_IO_ERROR`。
参数 `forget_owner` 控制是否从当前 resource owner 移除 buffer I/O 记录。
普通成功路径传 true。
resource owner 自己清理时传 false。
参数 `release_aio` 表示是否释放 AIO 子系统持有的 pin 和 wait reference。
AIO completion callback 中会传 true。
同步路径或非 AIO owner 路径通常传 false。
函数锁住 buffer header。
断言 `BM_IO_IN_PROGRESS` 必须存在。
它把 `BM_IO_IN_PROGRESS` 加入 unset bits。
它总是把 `BM_IO_ERROR` 加入 unset bits。
这意味着任何新一轮 I/O 完成都会清掉旧错误。
如果本轮失败，`set_flag_bits` 又会把 `BM_IO_ERROR` 设置回来。
如果 `clear_dirty` 为 true，再清 dirty 相关位。
如果 `release_aio` 为 true，断言 refcount 大于 0。
然后把 refcount change 设为 -1。
同时清掉 `buf->io_wref`。
接着用 `UnlockBufHdrExt()` 一次性应用 set bits、unset bits 和 refcount change。
如果 `forget_owner` 为 true，从 resource owner 里忘掉这次 buffer I/O。
然后 `ConditionVariableBroadcast(BufferDescriptorGetIOCV(buf))`。
这一步唤醒睡在 `WAIT_EVENT_BUFFER_IO` 上的 waiters。
如果 AIO pin 的释放可能让 cleanup lock waiter 满足条件，还会 `WakePinCountWaiter(buf)`。
这说明 buffer I/O 状态和 pin count 等待不是完全独立的。
AIO 子系统可能持有额外 pin。
释放这个 pin 后，等待 “sole pin” 的 backend 可能也要被唤醒。
## 10. AbortBufferIO 的错误清理
`AbortBufferIO()` 位于 `bufmgr.c` 约 7415 到 7462 行。
它是 resource owner 清理 buffer I/O 的回调路径。
`buffer_io_resowner_desc` 位于 `bufmgr.c` 约 285 到 292 行。
它的 release phase 是 `RESOURCE_RELEASE_BEFORE_LOCKS`。
ReleaseResource 是 `ResOwnerReleaseBufferIO()`。
`ResOwnerReleaseBufferIO()` 会调用 `AbortBufferIO(buffer)`。
也就是说，如果 backend 在发起 buffer I/O 后发生 ERROR，resource owner 会确保 buffer 不会永远停留在 `BM_IO_IN_PROGRESS`。
`AbortBufferIO()` 的注释明确说：
所有 LWLocks 和 content locks 可能已经释放。
但 buffer pins 还没释放。
所以 buffer 仍被当前 backend pin 住。
如果 I/O 正在进行，它总是设置 `BM_IO_ERROR`。
即使错误本身不一定来自 I/O。
这是保守选择。
因为一旦 error unwind 到这里，buffer 内容是否可靠已经不能证明。
函数先锁 header。
断言 buffer 至少有 `BM_IO_IN_PROGRESS` 或 `BM_TAG_VALID`。
如果 buffer 还不是 valid，说明大概率是读入过程中出错。
它断言 buffer 不 dirty，然后解锁。
如果 buffer 已经 valid，则对应写 I/O 场景。
写 I/O 期间 valid page 应该仍 dirty。
如果之前已经有 `BM_IO_ERROR`，会发 WARNING，提示同一 block 多次写失败，可能是永久写错误。
最后调用 `TerminateBufferIO(buf_hdr, false, BM_IO_ERROR, false, false)`。
这会清 `BM_IO_IN_PROGRESS`，清旧 `BM_IO_ERROR` 后再设置新 `BM_IO_ERROR`，广播等待者。
它不会从 resource owner 忘掉记录。
因为这正是在 resource owner release 过程中调用的。
## 11. AsyncReadBuffers 如何认领 I/O
`StartReadBuffersImpl()` 位于 `bufmgr.c` 约 1370 到 1592 行。
`AsyncReadBuffers()` 位于约 1938 到 2174 行。
普通单页读也会走这套 read buffers 机制。
`StartReadBuffersImpl()` 先为每个请求 block 调用 `PinBufferForBlock()`。
如果第一个 block 是 hit，直接返回 false。
这时不需要 `WaitReadBuffers()`。
如果后续 block 是 hit，而前面已经需要 I/O，会切分本次 I/O。
因为一个 readv 必须覆盖连续且都需要 I/O 的 blocks。
若允许 forwarding，已 pin 的 trailing buffer 可以转交给下一次调用继续处理。
对第一个 miss，`StartReadBuffersImpl()` 填充 operation。
然后如果 `io_method != IOMETHOD_SYNC`，会尝试直接调用 `AsyncReadBuffers()` 发起后台 I/O。
如果是 `IOMETHOD_SYNC`，它只标记 `READ_BUFFERS_SYNCHRONOUSLY`，并让 `WaitReadBuffers()` 中的 retry loop 发起 I/O。
`AsyncReadBuffers()` 里有一个非常关键的顺序。
它必须先取得 AIO handle，再调用 `StartBufferIO()`。
源码注释说明原因：
`pgaio_io_acquire()` 可能阻塞。
不能在设置 `BM_IO_IN_PROGRESS` 后再做可能阻塞的 handle 获取。
否则其他 backend 会看到 I/O in progress，却迟迟等不到真正提交。
如果非阻塞 acquire 失败，会先 `pgaio_submit_staged()`。
然后再阻塞 acquire。
拿到 handle 后，清空 operation 的 foreign_io 和 io_wref。
然后对第一个待读 buffer 调用：
`StartBufferIO(buffers[nblocks_done], true, true, &operation->io_wref)`
这里 `forInput=true`。
`wait=true`。
并传入 `io_wref`。
所以如果另一个 backend 已经开始 AIO 且 wait reference 可用，当前 backend 会加入那个 I/O，而不是另起 I/O。
如果返回 `BUFFER_IO_ALREADY_DONE`，说明别人已经完成读。
当前 backend 把 `nblocks_done` 加 1，并把这次当作 hit 统计。
如果返回 `BUFFER_IO_IN_PROGRESS`，说明有 foreign I/O。
此时 operation 标记 `foreign_io=true`。
`WaitReadBuffers()` 后续等待别人完成。
如果返回 `BUFFER_IO_READY_FOR_IO`，当前 backend 成为 I/O owner。
它把 buffer page 地址放进 `io_pages[0]`。
然后尝试合并后续相邻 buffers。
对后续 buffers 调用：
`StartBufferIO(buffers[i], true, false, NULL)`
这里 `wait=false` 且不要 wait reference。
因为这只是试探是否能把它并入同一次 readv。
如果某个后续 buffer 已经 valid 或已有 I/O，停止合并。
不会为合并而等待别人。
确定本次 readv 的 buffers 后，当前 backend 从 AIO handle 获取 wait reference。
把 buffer 列表写入 handle data。
注册 buffer read completion callbacks。
然后调用 `smgrstartreadv()`。
`smgrstartreadv()` 往下进入 `mdstartreadv()` 和 `FileStartReadV()`。
发起 I/O 后，`pgBufferUsage.shared_blks_read` 增加。
vacuum cost 也在发起 I/O 时计数，而不是等 I/O 完成时。
## 12. AIO staging 与 buffer pin
AIO read 的 completion callback 需要知道哪些 buffer 被这次 I/O 覆盖。
`buffer_stage_common()` 位于 `bufmgr.c` 约 8276 到 8391 行。
这个函数在 AIO handle staging 阶段执行。
它遍历 handle data 中的 buffer 列表。
对 shared buffer，它锁住 buffer header。
断言 read 场景下：
`BM_TAG_VALID` 必须存在。
`BM_VALID` 必须不存在。
`BM_DIRTY` 必须不存在。
`BM_IO_IN_PROGRESS` 必须存在。
然后把 `buf_hdr->io_wref` 设置为当前 AIO wait reference。
接着给 buffer 增加一个 refcount。
这个 pin 是 AIO 子系统持有的 pin。
原因是发起 I/O 的 backend 可能在 I/O 完成前发生 ERROR。
resource owner cleanup 可能释放该 backend 自己的 buffer pin。
如果没有 AIO pin，buffer 可能在底层 I/O 仍在写入时被替换。
这会破坏内存和 buffer pool 一致性。
因此 AIO 子系统持 pin 到 completion 阶段。
这个 pin 会在 `TerminateBufferIO(..., release_aio=true)` 中释放。
同时 staging 阶段会调用 `ResourceOwnerForgetBufferIO()`。
因为 I/O 的生命周期已转交给 AIO 子系统。
这也解释了为什么 `TerminateBufferIO()` 有 `release_aio` 参数。
非 AIO 同步路径不需要释放 AIO pin。
AIO completion 路径必须释放它并清 wait reference。
## 13. WaitReadBuffers 的完成与重试
`WaitReadBuffers()` 位于 `bufmgr.c` 约 1752 到 1917 行。
它等待由 `StartReadBuffers()` 或 `StartReadBuffer()` 启动的读完成。
返回值表示是否真的等待或做了额外 retry 工作。
它先根据 persistence 和 strategy 确定 I/O statistics 的 object/context。
如果 operation 没有 valid io_wref 且 `io_method != IOMETHOD_SYNC`，这是错误。
因为异步路径必须已经有可等待的 I/O。
然后进入 retry loop。
如果 operation 有 valid `io_wref`，先检查是否已经完成。
如果没完成，调用 `pgaio_wref_wait()`。
这个等待可能显示为 `AioIoCompletion`。
等待结束后，如果 operation 是 foreign I/O，处理逻辑与 own I/O 不同。
foreign I/O 表示这次读是别人发起的。
当前 backend 没有自己的 `PgAioReturn`。
因此它读取 buffer state。
如果 buffer 已经 `BM_VALID`，说明别人读成功。
它把 `nblocks_done` 加 1。
并把这次对当前 backend 统计为 hit。
如果 foreign I/O 失败，buffer 仍不是 valid。
这时 `nblocks_done` 不增加。
后面的 retry loop 会调用 `AsyncReadBuffers()`，让当前 backend 自己尝试读。
own I/O 则调用 `ProcessReadBuffersResult()`。
该函数位于约 1713 到 1750 行。
它检查 AIO result。
如果是 `PGAIO_RS_ERROR`，用 `pgaio_result_report(..., ERROR)` 报错。
如果是 `PGAIO_RS_WARNING`，用 WARNING 报告。
如果是 `PGAIO_RS_PARTIAL`，以 DEBUG 级别记录，并准备 retry。
如果成功，result 中的 block 数会加入 `nblocks_done`。
如果 `nblocks_done == nblocks`，等待完成。
否则说明有 partial read、某些 buffer 被别人读完、或本次 I/O 覆盖范围被限制。
循环会设置 `needed_wait=true`，然后再次调用 `AsyncReadBuffers()`。
这里不会缩短 operation 的总 `nblocks`。
因为调用者期望 `WaitReadBuffers()` 返回时整个 operation 完成或报错。
这套重试逻辑解释了一个重要行为：
如果 backend A 等待 backend B 的 foreign I/O，而 B 读失败留下 invalid buffer，A 不会直接把失败视为自己的最终失败。
A 会尝试自己重新发起读。
只有自己发起的 I/O 结果被 `ProcessReadBuffersResult()` 报成 ERROR，才会让当前 query 报错。
## 14. completion callback 如何设置 BM_VALID 或 BM_IO_ERROR
readv 的 buffer completion 主逻辑在 `buffer_readv_complete_one()`。
它位于 `bufmgr.c` 约 8533 到 8676 行。
进入该函数时，shared read buffer 应该满足：
`BM_TAG_VALID=1`
`BM_VALID=0`
`BM_IO_IN_PROGRESS=1`
`BM_DIRTY=0`
如果底层 I/O 没失败，会调用 `PageIsVerified()` 检查页面。
checksum 失败、页面头无效等都在这里处理。
如果 flags 里有 `READ_BUFFERS_ZERO_ON_ERROR`，验证失败时会把 buffer 内容清零。
然后把 `zeroed_buffer` 置 true。
这种情况下该 buffer 仍可以被标成 `BM_VALID`。
如果不能 zero on error，验证失败会把 `buffer_invalid` 置 true，并把 `failed` 置 true。
最后决定结束 flag：
`set_flag_bits = failed ? BM_IO_ERROR : BM_VALID`
shared buffer 调用：
`TerminateBufferIO(buf_hdr, false, set_flag_bits, false, true)`
这里 `forget_owner=false`，因为 AIO staging 已经把 I/O ownership 从当前 resource owner 转交出去。
`release_aio=true`，因为 completion 要释放 AIO pin 并清 wait reference。
读成功会设置 `BM_VALID`。
读失败会设置 `BM_IO_ERROR`，但不会设置 `BM_VALID`。
无论成功还是失败，`TerminateBufferIO()` 都会清 `BM_IO_IN_PROGRESS` 并广播 condition variable。
所以所有等待者都会被唤醒。
唤醒后它们根据 `BM_VALID` 判断是可以消费还是需要重试。
## 15. 多个 backend 同读一个 block
现在把协议串成一个并发故事。
假设 backend A 和 backend B 同时读 relation R 的 block 42。
一开始 buffer pool 中没有这个 tag。
A 进入 `BufferAlloc()`。
A 查 mapping table，未找到。
A 选择 victim。
A 在 mapping table 插入 tag `(R, MAIN_FORKNUM, 42)`。
A 设置 victim 的 `BM_TAG_VALID`。
A 返回 pinned buffer，`foundPtr=false`。
B 稍晚进入 `BufferAlloc()`。
B 查 mapping table，已经找到同一个 tag。
B pin 这个 buffer。
因为 A 还没完成 I/O，B 的 `PinBuffer()` 通常会看到 `BM_VALID=0`。
B 得到同一个 buffer，`foundPtr=false`。
此时两个 backend 都不会直接访问页面内容。
A 先进入 `AsyncReadBuffers()`。
A 调用 `StartBufferIO(buffer, true, true, &operation->io_wref)`。
此时没有 `BM_IO_IN_PROGRESS` 且没有 `BM_VALID`。
A 设置 `BM_IO_IN_PROGRESS`。
A 成为 I/O owner。
A 发起 `smgrstartreadv()`。
AIO staging 会把 `buf->io_wref` 填好，并增加 AIO pin。
B 也进入 `AsyncReadBuffers()`。
B 调用同样的 `StartBufferIO()`。
如果 A 已经设置 `BM_IO_IN_PROGRESS` 且 `io_wref` 有效，B 会复制 wait reference。
B 返回 `BUFFER_IO_IN_PROGRESS`。
B 标记 `foreign_io=true`。
B 不会发起另一个 read。
如果 B 来得更早，看到 `BM_IO_IN_PROGRESS` 但 `io_wref` 尚未设置，且它传了 `wait=true`，那它会走 `WaitIO()`。
这是一段很窄的窗口。
源码注释明确提到这种情况：
另一个 backend 已经 started IO，但还没 set wait reference。
此时没有办法异步加入，只能等到 I/O 完成或状态改变。
A I/O 成功后 completion 设置 `BM_VALID`，清 `BM_IO_IN_PROGRESS`，广播。
B 的 `WaitReadBuffers()` 发现 foreign buffer 已 valid。
B 把它作为 hit 统计并返回。
两个 backend 最终引用的是同一个 shared buffer。
没有重复读成两个副本。
如果 A I/O 失败，completion 设置 `BM_IO_ERROR`，清 `BM_IO_IN_PROGRESS`，不设 `BM_VALID`，广播。
B 被唤醒后看到 foreign buffer 仍 invalid。
B 不增加 `nblocks_done`。
B 的 retry loop 调 `AsyncReadBuffers()`。
这一次 `StartBufferIO()` 看到没有 I/O in progress，且 `BM_VALID=0`。
虽然 `BM_IO_ERROR=1`，但这不会阻止重试。
B 设置 `BM_IO_IN_PROGRESS`，自己发起读。
如果 B 成功，`TerminateBufferIO()` 清旧 `BM_IO_ERROR` 并设置 `BM_VALID`。
这就是 I/O 错误后的状态恢复路径。
## 16. 为什么不能只靠 content lock
`BM_IO_IN_PROGRESS` 不是 content lock 的替代品。
content lock 保护的是页面内容访问。
buffer header lock 保护的是 descriptor 元数据。
`BM_IO_IN_PROGRESS` 是一个状态协议位。
它告诉所有 backend：“这个 buffer 的页面内容还不能被解释为 relation block 的有效内容。”
`ZeroAndLockBuffer()` 的注释特别说明：
即使不做真实磁盘 I/O，也要获取 `BM_IO_IN_PROGRESS`。
原因是调用者要把页面清零，然后初始化。
如果只拿 exclusive content lock，不足以防止其他 backend 后续使用这个 pin 住的 page。
PostgreSQL 的 buffer access rules 允许读者在判定 tuple 可见后释放 content lock，但继续基于 pin 使用某些信息。
在 zero-and-lock 场景中，必须防止别人看到“清零但尚未由调用者初始化”的 page。
所以代码先 `StartSharedBufferIO()`。
这让其他 backend 按 I/O in progress 规则等待。
当前 backend 清零。
当前 backend 在设置 `BM_VALID` 前拿 content lock。
然后 `TerminateBufferIO(..., BM_VALID, ...)` 唤醒别人。
这样别人醒来后即使看到 `BM_VALID`，也会被 content lock 阻挡，直到调用者初始化并释放锁。
这就是 zero-and-lock 中 `BM_IO_IN_PROGRESS` 与 content lock 的组合意义。
## 17. ZeroAndLockBuffer 细读
`ZeroAndLockBuffer()` 位于 `bufmgr.c` 约 1131 到 1215 行。
调用前 buffer 已经 pinned。
mode 必须是 `RBM_ZERO_AND_LOCK` 或 `RBM_ZERO_AND_CLEANUP_LOCK`。
如果调用者已经知道 buffer valid，函数不再启动 I/O 协议。
它只需要按 mode 加锁。
如果不是 already valid，则区分 local 和 shared。
local buffer 调 `StartLocalBufferIO()`。
shared buffer 调 `StartSharedBufferIO(bufHdr, true, true, NULL)`。
这里 `forInput=true`。
虽然没有真实磁盘 input。
但从状态机角度，它是在把 invalid buffer 变成 valid page。
所以复用 input I/O 协议。
它传 `wait=true`。
如果别人已经在读或 zero 这个 buffer，就等对方结束。
它传 `io_wref=NULL`。
因为 zero-and-lock 需要立刻知道是否该清零，不做异步 join。
如果返回 `BUFFER_IO_READY_FOR_IO`，当前 backend 负责清零。
如果返回 `BUFFER_IO_ALREADY_DONE`，说明别人已经让 buffer valid。
断言不会返回 `BUFFER_IO_IN_PROGRESS`，因为 wait=true 且没有要求异步 wait ref。
需要清零时，函数 `memset(BufferGetPage(buffer), 0, BLCKSZ)`。
然后在设置 valid 前加 content lock。
shared buffer 用 `LockBuffer(buffer, BUFFER_LOCK_EXCLUSIVE)`。
不能用 `LockBufferForCleanup()`。
因为这些函数会断言 buffer 已 valid。
此时 buffer 还未 valid。
最后调用 `TerminateBufferIO(..., BM_VALID, ...)`。
如果 buffer 已 valid，shared buffer 根据 mode 加 exclusive lock 或 cleanup lock。
local buffer 不需要跨进程锁语义。
## 18. smgrreadv、mdreadv、mdstartreadv
buffer manager 不直接读文件。
它通过 storage manager 抽象层。
`smgrreadv()` 位于 `smgr.c` 约 710 到 728 行。
它接收 relation、fork、起始 block、buffer 数组和 block 数。
它调用当前 smgr implementation 的 `smgr_readv`。
md storage manager 下就是 `mdreadv()`。
`smgrstartreadv()` 位于 `smgr.c` 约 731 到 762 行。
它是异步版本。
注释强调 compared to `smgrreadv()`，调用者承担更多职责：
partial reads 需要上层重新发起 I/O。
smgr 可能只向 server log 记录一些问题。
高层负责把 warning/error 报给用户。
`mdreadv()` 位于 `md.c` 约 856 到 991 行。
它按 segment 边界处理读。
当前实现要求一次 readv 不跨 segment boundary。
如果 `nblocks_this_segment != nblocks`，会 `elog(ERROR, "read crosses segment boundary")`。
上层通过 `smgrmaxcombine()` 限制合并范围，避免跨 segment。
`mdreadv()` 用 `FileReadV(..., WAIT_EVENT_DATA_FILE_READ)` 发起文件读。
如果 `nbytes < 0`，报文件访问错误。
如果读到 0，说明 EOF 或 partial block at EOF。
正常情况下，读不存在的 block 是错误。
只有 `zero_damaged_pages` 或 recovery 下会把剩余 buffers 清零。
但当前注释说明这条逻辑在 recovery 中被认为不可达，并且在 `zero_damaged_pages` 下也不完整。
PG 18 代码中还放了 `Assert(false)`，未来倾向移除这段逻辑。
如果短读但不是 0，`mdreadv()` 会调整 offset 和 iovec 继续读。
直到读满本次请求或确认 EOF。
`mdstartreadv()` 位于 `md.c` 约 994 到 1061 行。
它准备异步读。
它设置 smgr target data。
注册 `PGAIO_HCB_MD_READV` callback。
然后调用 `FileStartReadV(..., WAIT_EVENT_DATA_FILE_READ)`。
注意 `mdstartreadv()` 没有复制 `mdreadv()` 里 `zero_damaged_pages` 对 EOF 清零的逻辑。
源码注释解释：这会受 definer 和 completor 的 GUC 差异影响，而且这段旧逻辑本身就有问题。
异步读完成后，`md_readv_complete()` 处理低层结果。
它位于 `md.c` 约 1987 到 2047 行。
如果低层 result 小于 0，转换为 `PGAIO_RS_ERROR`。
如果读到 0 个 block，也视为 error。
如果读到的 block 数小于请求数，则设置 `PGAIO_RS_PARTIAL`。
partial read 由 `WaitReadBuffers()` 的 retry loop 重新发起。
这解释了为什么 buffer manager 的 read wait 协议必须支持多次 `AsyncReadBuffers()`。
底层不保证一次异步 readv 就读完所有 blocks。
## 19. I/O 错误后的重试与状态恢复
读错误可能来自几类地方。
第一，md 层文件读错误。
例如 `FileStartReadV()` 启动失败或实际读返回 errno。
第二，读到了 EOF 或 partial read 且无法补齐。
第三，读成功但 page verification 失败。
第四，checksum 失败且不能忽略或修复。
在异步路径中，md 层错误先变成 `PGAIO_RS_ERROR` 或 `PGAIO_RS_PARTIAL`。
buffer read completion 再逐 buffer 做 page verification。
失败的 buffer 最终通过 `TerminateBufferIO(..., BM_IO_ERROR, ..., release_aio=true)` 结束。
这会留下：
`BM_TAG_VALID=1`
`BM_VALID=0`
`BM_IO_IN_PROGRESS=0`
`BM_IO_ERROR=1`
此时 buffer table 中仍有 tag。
这是有意的。
如果立刻删除 tag，等待者和已经 pin 住 buffer 的 backend 会更难协调。
保持 tag 让后续读者找到同一个 buffer，并通过状态机重新发起 I/O。
下一次读同一 block 时，`BufferAlloc()` 发现 mapping table 中有 tag。
它 pin buffer。
`PinBuffer()` 返回 false，因为没有 `BM_VALID`。
`BufferAlloc()` 把 `foundPtr=false` 返回。
`AsyncReadBuffers()` 调 `StartBufferIO()`。
`StartSharedBufferIO()` 看到没有 `BM_IO_IN_PROGRESS`。
对 read 来说，由于 `BM_VALID=0`，工作还没完成。
它不因为 `BM_IO_ERROR` 而拒绝。
它设置 `BM_IO_IN_PROGRESS`，记入 resource owner，并返回 `BUFFER_IO_READY_FOR_IO`。
如果重试成功，completion 调 `TerminateBufferIO(..., BM_VALID, ...)`。
`TerminateBufferIO()` 会先清旧 `BM_IO_ERROR`。
然后设置 `BM_VALID`。
状态恢复为可用 buffer。
如果重试仍失败，新的 `TerminateBufferIO()` 继续留下 `BM_IO_ERROR`。
如果失败发生在当前 backend 发起 I/O 后但 completion 前，且 ERROR unwinds，resource owner 会调用 `AbortBufferIO()`。
这条路径也会清 `BM_IO_IN_PROGRESS` 并设置 `BM_IO_ERROR`。
所以不会出现永久等待。
这是 buffer I/O 协议的关键安全网。
等待者永远不应该依赖“发起者正常返回”。
它们只依赖 buffer state 和 condition variable broadcast。
## 20. local buffer 的对照
本节重点是 shared buffer。
但 `StartBufferIO()` wrapper 也覆盖 local buffer。
local buffer 的实现位于 `localbuf.c`。
`LocalBufferAlloc()` 与 `BufferAlloc()` 类似，但不需要跨 backend 锁。
因为 local buffers 只属于当前 backend。
`StartLocalBufferIO()` 位于 `localbuf.c` 约 531 到 580 行。
它支持 AIO wait reference。
但注释明确说：
`BM_IO_IN_PROGRESS` 当前不用于 local buffers。
local buffers 也不通过 resource owner 跟踪 buffer I/O。
如果 local buffer 有有效 `io_wref`，说明当前 backend 内已经有 AIO 在进行。
可以返回 `BUFFER_IO_IN_PROGRESS` 或等待 `pgaio_wref_wait()`。
`TerminateLocalBufferIO()` 位于 `localbuf.c` 约 585 到 616 行。
它只调整 flags。
它清旧 `BM_IO_ERROR`。
需要时清 dirty。
如果 release_aio，释放 AIO pin 并清 `io_wref`。
local buffers 不使用 I/O condition variable。
因为没有其他进程能看到该 buffer。
local extend 路径 `ExtendBufferedRelLocal()` 也会清零 buffers 并调用 `smgrzeroextend()`。
但它不需要 relation extension lock 来协调其他 backend。
这组对照能帮助我们理解：
`BM_IO_IN_PROGRESS` 的主要价值是 shared buffer 的跨 backend 协议。
## 21. 成本、观测与诊断入口
这条读路径的成本不只来自磁盘读。
第一层成本是 buffer table lookup。
`BufferAlloc()` 需要按 `BufferTag` 计算 hash，进入对应 mapping partition lock。
命中时成本主要是 pin、usagecount 和统计更新。
miss 时要离开 mapping lock，进入 victim selection。
第二层成本是 I/O ownership。
同一个 block 的并发读会被收敛到一个 buffer tag 上。
只有拿到 `BUFFER_IO_READY_FOR_IO` 的 backend 发起底层读。
其他 backend 要么拿 AIO wait reference，要么睡 `WAIT_EVENT_BUFFER_IO`。
第三层成本是 partial/error retry。
`mdstartreadv()` 的异步结果可能是 partial。
`WaitReadBuffers()` 需要把没有完成的部分重新送回 `AsyncReadBuffers()`。
因此一次上层 read operation 不等于一次底层 readv。
第四层成本是 verification。
读完成后还要检查 page header、checksum、zero-on-error mode。
失败会留下 `BM_IO_ERROR`，后续读同一 tag 时重试。
能直接观察的入口：
- `pg_stat_io`：按 backend type、object、context 看 relation read 次数和时间。
- `pg_stat_activity.wait_event = BufferIO`：看到当前 backend 等 buffer I/O，但看不到它等待的是 read 还是 write。
- `EXPLAIN (ANALYZE, BUFFERS)`：看到单 query 的 shared read/hit，但看不到 `BM_IO_IN_PROGRESS`。
- gdb 断点：`StartSharedBufferIO()`、`WaitIO()`、`TerminateBufferIO()` 能直接看到状态位。
- tracepoints：`TRACE_POSTGRESQL_BUFFER_READ_START/DONE` 与 `SMGR_MD_READ_START/DONE` 能连接 buffer 层和 md 层。
只能推断的入口：
- “同一个 block 是否只分配一个 buffer”通常要结合 `BufferAlloc()` 断点或 instrumentation。
- AIO wait reference 是否被成功 join，SQL 层没有直接指标。
- page verification 失败后的 retry 是否发生，需要断点或日志增强。
这也是本节的 runtime truth：
看到 `BufferIO` 或 shared read 延迟时，不要只问磁盘慢不慢。
先问同一个 tag 是否已经存在、谁拥有 `BM_IO_IN_PROGRESS`、等待者醒来后看到的是 `BM_VALID` 还是 `BM_IO_ERROR`。
## 22. 实验 1：源码定位
在 `/home/nail/postgres-lab` 中确认基线：
```bash
git -C /home/nail/postgres-lab rev-parse HEAD
git -C /home/nail/postgres-lab branch --show-current
```
期望看到：
`bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`
`master`
定位 read buffer 入口：
```bash
rg -n "ReadBuffer_common|StartReadBuffer|WaitReadBuffers|AsyncReadBuffers" \
  src/backend/storage/buffer/bufmgr.c
```
定位 I/O 状态协议：
```bash
rg -n "WaitIO|StartSharedBufferIO|StartBufferIO|TerminateBufferIO|AbortBufferIO" \
  src/backend/storage/buffer/bufmgr.c
```
定位状态位：
```bash
rg -n "BM_VALID|BM_IO_IN_PROGRESS|BM_IO_ERROR|StartBufferIOResult" \
  src/include/storage/buf_internals.h
```
定位 smgr/md 读：
```bash
rg -n "smgrreadv|smgrstartreadv|mdreadv|mdstartreadv|md_readv_complete" \
  src/backend/storage/smgr src/backend/storage/buffer
```
第一遍跟读建议按这个顺序：
`buf_init.c` 的 buffer lookup 注释。
`buf_internals.h` 的 flag 定义。
`ReadBuffer_common()`。
`PinBufferForBlock()`。
`BufferAlloc()`。
`StartReadBuffersImpl()`。
`AsyncReadBuffers()`。
`StartSharedBufferIO()`。
`WaitIO()`。
`TerminateBufferIO()`。
`buffer_readv_complete_one()`。
`WaitReadBuffers()`。
`smgrstartreadv()`。
`mdstartreadv()`。
`md_readv_complete()`。
第二遍跟读时，不要按函数出现顺序读。
按状态变化读。
先找谁设置 `BM_TAG_VALID`。
再找谁设置 `BM_IO_IN_PROGRESS`。
再找谁设置 `BM_VALID`。
再找谁设置 `BM_IO_ERROR`。
最后找谁清掉这些位。
## 23. 实验 2：画状态转移表
准备一张表，列为：
场景。
函数。
进入状态。
动作。
退出状态。
等待者如何继续。
第一行写新 block miss。
函数是 `BufferAlloc()`。
进入状态是 mapping table 无 tag。
动作是选择 victim、插入 tag、设置 `BM_TAG_VALID`。
退出状态是 tag valid 但 page invalid。
等待者如果此时进入，可以找到同一个 tag。
第二行写 read owner 认领。
函数是 `StartSharedBufferIO()`。
进入状态是 `BM_VALID=0` 且 `BM_IO_IN_PROGRESS=0`。
动作是设置 `BM_IO_IN_PROGRESS`。
退出状态是 I/O in progress。
等待者会进入 join 或 sleep。
第三行写 read success。
函数是 `buffer_readv_complete_one()` 加 `TerminateBufferIO()`。
进入状态是 I/O in progress。
动作是验证 page，设置 `BM_VALID`，清 `BM_IO_IN_PROGRESS`。
退出状态是 valid。
等待者把它作为 hit。
第四行写 read failure。
动作是设置 `BM_IO_ERROR`，不设置 `BM_VALID`。
等待者 retry。
第五行写 retry success。
动作是新 backend 再次 `StartSharedBufferIO()`，然后 `TerminateBufferIO(..., BM_VALID, ...)`。
退出状态是 `BM_IO_ERROR=0` 且 `BM_VALID=1`。
这张表要自己手写。
手写的价值在于确认 `BM_IO_ERROR` 不阻止 `StartSharedBufferIO()`。
它只是前一次失败的痕迹。
## 24. 实验 3：两个 backend 同读
目标是观察“两个 session 同读同一冷 block”时，为什么只会有一个 buffer tag。
可选方法一是用 gdb 或 rr。
在一个 debug build 上给以下函数打断点：
```gdb
break BufferAlloc
break StartSharedBufferIO
break WaitIO
break TerminateBufferIO
break buffer_readv_complete_one
```
让 session A 执行一个会读冷数据页的查询。
在 A 命中 `StartSharedBufferIO()` 并设置 `BM_IO_IN_PROGRESS` 后停住。
让 session B 执行相同查询。
观察 B 是否在 `BufferAlloc()` 中找到 existing tag。
继续观察 B 的 `StartSharedBufferIO()` 返回值。
如果 AIO wait reference 已经就绪，B 会拿到 `BUFFER_IO_IN_PROGRESS`。
如果窗口较早，B 可能进入 `WaitIO()`。
这个实验的重点不是稳定复现某个等待事件。
重点是看到 B 不会分配另一个 buffer。
可选方法二是加临时 instrumentation。
只在本地实验分支加日志，不要提交到课程文件。
在 `StartSharedBufferIO()` 的三个返回点打 `elog(LOG, ...)`。
分别记录 buffer id、blockNum、返回值、flags。
用很小的 `shared_buffers` 和两个并发顺序扫描制造 miss。
观察同一 block 的返回值变化。
如果不想改源码，可以使用 DTrace/SystemTap/eBPF tracepoints。
源码中有 `TRACE_POSTGRESQL_BUFFER_READ_START` 和 `TRACE_POSTGRESQL_BUFFER_READ_DONE`。
也有 `TRACE_POSTGRESQL_SMGR_MD_READ_START` 和 DONE。
跟踪同一 relfilenode block 是否出现重复底层 read。
注意并发场景和 read-ahead 会让输出顺序不完全线性。
## 25. 常见误区
误区一：`BM_TAG_VALID` 等同于 `BM_VALID`。
不是。
`BM_TAG_VALID` 表示 mapping table 中这个 buffer 代表某个 block。
`BM_VALID` 表示 buffer 内容可读。
read miss 后，tag 会先 valid，但内容还 invalid。
误区二：`foundPtr=false` 只表示没有找到 buffer。
不是。
`BufferAlloc()` 找到 existing tag 但 `BM_VALID=0` 时，也返回 `foundPtr=false`。
调用者必须进入 I/O 协议。
误区三：`BM_IO_ERROR` 会阻止后续读。
不是。
它记录前一次失败。
后续 `StartSharedBufferIO()` 只关心是否 already done。
对读来说 already done 是 `BM_VALID=1`。
只要 `BM_VALID=0` 且没有 active I/O，就可以重试。
误区四：等待者被唤醒后一定能访问 page。
不是。
等待者醒来后必须重新检查 `BM_VALID`。
如果 foreign I/O 失败，page 仍 invalid。
等待者会 retry，而不是访问内容。
误区五：zero-and-lock 不做磁盘 I/O，所以不需要 `BM_IO_IN_PROGRESS`。
不是。
它也在把 invalid buffer 转成 valid buffer。
这段期间其他 backend 必须按 I/O 协议等待。
误区六：extend 只要 `smgrzeroextend()` 成功就行。
不是。
必须先把即将新增的 blocks 放进 buffer table，并设置 I/O in progress。
否则文件变大到 buffer table 可见之间有竞态窗口。
误区七：condition variable 是唯一等待机制。
不是。
当前源码里，如果有 AIO wait reference，会优先 `pgaio_wref_wait()`。
只有没有 wait reference 时才睡 buffer 的 I/O condition variable。
## 26. 一页读入的最小伪代码
下面伪代码不是源码翻译，而是抽取协议骨架。
```c
buf = BufferAlloc(tag, &found);
if (found)
    return pinned_valid_buffer;
status = StartBufferIO(buf, true, true, &wref);
if (status == BUFFER_IO_ALREADY_DONE)
    return pinned_valid_buffer;
if (status == BUFFER_IO_IN_PROGRESS)
{
    wait(wref or BufferIOCV);
    if (buf is valid)
        return pinned_valid_buffer;
    else
        retry;
}
if (status == BUFFER_IO_READY_FOR_IO)
{
    smgrstartreadv(...);
    wait own io;
    completion sets BM_VALID or BM_IO_ERROR;
    if success
        return pinned_valid_buffer;
    else
        throw or retry depending on caller/result path;
}
```
真实源码更复杂。
它要处理 readv 合并。
它要处理 AIO handle 生命周期。
它要处理 partial reads。
它要处理 checksum 和 page verification。
它要处理 local buffers。
它要处理 resource owner cleanup。
但核心状态机就是这几步。
## 27. 讨论题
1. 为什么 `BufferAlloc()` 可以在 page 内容还没有读入时先把 tag 放进 buffer table？
2. `BM_TAG_VALID=1`、`BM_VALID=0`、`BM_IO_IN_PROGRESS=0`、`BM_IO_ERROR=1` 这组状态对下一次读意味着什么？
3. 为什么等待者被唤醒后不能假设 page 已经可读，而必须重新检查 `BM_VALID`？
4. 如果读 owner 在 `smgrstartreadv()` 之后 ERROR，哪个 cleanup 路径负责清 `BM_IO_IN_PROGRESS`？
5. `WaitIO()` 等 condition variable 与等 AIO wait reference 的语义差异是什么？
6. 为什么 zero-and-lock 没有真实磁盘读，仍然复用 `StartSharedBufferIO(..., forInput=true, ...)`？
7. SQL 层看到 `wait_event = BufferIO` 时，哪些事实可以直接判断，哪些只能通过断点或 tracepoint 推断？
8. 如果要改 `mdstartreadv()` 的 partial read 行为，你需要回到 `WaitReadBuffers()` 核对哪两个 retry 假设？
## 28. 本节小结
PostgreSQL 读 shared buffer 的核心不是“读磁盘”本身。
核心是先建立唯一 buffer 身份，再用状态位协调谁读、谁等、谁重试。
`BM_TAG_VALID` 让其他 backend 能找到同一个 buffer。
`BM_IO_IN_PROGRESS` 给一个 backend 独占 I/O 权限，并让其他 backend 等待或加入。
`BM_VALID` 是读者可以访问页面内容的门槛。
`BM_IO_ERROR` 是失败痕迹，不是永久禁止重试的锁。
`StartSharedBufferIO()` 是状态机入口。
它要么发现已经完成，要么发现正在进行，要么把当前 backend 变成 I/O owner。
`WaitIO()` 和 `WaitReadBuffers()` 是等待层。
前者等待单个 buffer 的 in-progress 位清除。
后者等待 read operation 完成，并处理 foreign I/O、partial read 和 retry。
`TerminateBufferIO()` 是正常完成和失败完成的统一出口。
它必须广播 condition variable。
`AbortBufferIO()` 是异常路径保险丝。
它保证 ERROR 发生后等待者不会永久睡眠。
zero-and-lock 不读磁盘，但仍然需要 I/O 协议。
因为它也把 invalid buffer 变成 valid page。
extend 场景也需要 I/O 协议。
因为文件变大前必须先让即将新增的 blocks 在 buffer table 中可见，并阻止别人抢读。
`smgrreadv/mdreadv` 是真正文件读的下层。
当前异步读的 partial/error 需要 buffer manager 的 retry 和 reporting 配合。
理解这一节后，再看 checkpoint 写回、buffer replacement、read stream readahead，会更容易判断哪些代码是在做 I/O，哪些代码是在维护 buffer pool 的并发一致性。
