# PostgreSQL Eviction、writeback 与 buffer cleanup
## 课程定位
前置知识：
你已经读过本模块前六节，知道 `BufferTag` 如何标识一个 shared buffer，知道 buffer mapping table 如何做 lookup，知道 clock sweep 如何挑 candidate，也知道 dirty buffer、page LSN 和 WAL-before-data 的基本关系。
本节唯一主问题：
`一个 miss 已经选中 victim buffer 之后，PostgreSQL 如何在不破坏 WAL-before-data、不丢 dirty page、不踩并发 pin、不把 writeback 误认为 fsync 的前提下，把旧页安全清理出去，并让 buffer slot 重新绑定到新页？`
核心矛盾：
```text
buffer manager 希望 miss 尽快复用一个 slot；
但 victim 可能是 dirty 的，dirty page 可能需要先 flush WAL，数据页写出后还要登记 checkpoint fsync，请求 writeback 还只是性能 hint，而且旧 tag 只有在没有并发 pin / dirty race 后才能从 mapping table 移除。
```
本节学完后应能独立判断：
```text
dirty victim 为什么不是 StrategyGetBuffer() 的职责；
FlushBuffer() 为什么先 XLogFlush(page LSN)，再 PageSetChecksum()，再 smgrwrite()；
RelationNeedsWAL() 和 BM_PERMANENT 为什么属于不同层次；
smgrwrite() 返回为什么不代表数据页 durable；
smgrregistersync() 解决的是跳过 fsync 后如何补登记，不是普通 FlushBuffer() 的直接调用点；
BM_TAG_VALID、BM_VALID、BM_DIRTY、BM_IO_IN_PROGRESS 在 eviction 和错误恢复中分别何时清理；
cleanup lock 等待 pin 的边界和 replacement pin 的边界为什么不同；
writeback context 为什么只是调度内核回写，不是 correctness 机制；
为什么一次 buffer miss 可能被 WAL flush、数据文件 write、fsync queue、writeback、retry loop 放大；
如何用源码断点、pg_stat_io、wait event 和小 shared_buffers 实验看到这些边界。
```
## 源码基线
本节基于本地源码：
```text
/home/nail/postgres-lab
branch: master
commit: bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8
```
行号来自：
`git -C /home/nail/postgres-lab show bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8:<path> | nl -ba`
重点源码：
```text
src/backend/storage/buffer/bufmgr.c
src/backend/storage/buffer/freelist.c
src/include/storage/buf_internals.h
src/include/storage/smgr.h
src/backend/storage/smgr/smgr.c
src/backend/storage/smgr/md.c
src/backend/storage/sync/sync.c
src/include/utils/rel.h
src/backend/access/transam/xlog.c
src/backend/access/transam/xloginsert.c
src/backend/access/transam/README
src/backend/catalog/storage.c
src/backend/storage/page/README
```
## 1. 本节在总主线中的位置
上一节讲到 dirty page 的 page LSN。
那一节的中心是：
```text
修改 page
  -> MarkBufferDirty()
  -> XLogInsert()
  -> PageSetLSN()
  -> 后续 FlushBuffer() 看到 page LSN 后先 XLogFlush()
```
本节从 buffer miss 的后半段开始。
也就是：
```text
PinBufferForBlock() miss
  -> BufferAlloc()
  -> GetVictimBuffer()
  -> StrategyGetBuffer() 选出一个已经 pin 住的 candidate
  -> 如果 candidate dirty，先 flush
  -> 如果 candidate 仍然可复用，删除 old tag
  -> BufferAlloc() 安装 new tag
  -> 后续 read path 填充 page 内容并设置 BM_VALID
```
这条路径最容易被误读成一句话：
`clock sweep 淘汰一个页。`
这句话太粗。
在当前源码中，至少有五个不同阶段：
```text
candidate selection
dirty write
writeback hint
mapping invalidation
new tag installation
```
它们不是一个原子动作。
其中任何一个阶段都可能因为并发 pin、dirty race、content lock、WAL flush、I/O error 或 fsync queue fallback 改变成本和结果。
本节只讲这一个问题：
`victim 选中以后，旧页如何安全离开 buffer slot。`
不再重复前几节已经讲清楚的 lookup hash、clock sweep 细节和普通 dirty 标记协议，只在需要解释 cleanup 边界时引用它们。
## 2. 一句话运行模型
一句话模型：
`PostgreSQL 把 victim 复用拆成“先 pin 住 candidate、必要时在 content lock 下写出 dirty page、通过 BM_IO_IN_PROGRESS 独占 I/O、把 fsync 责任交给 smgr/sync、再在 mapping lock 下清掉 old tag 和 flags”的多阶段协议。`
这个模型有四个关键词。
第一，pin。
replacement 不能复用被别人 pin 的 buffer。
`StrategyGetBuffer()` 的硬条件是 `refcount == 0`。
它找到 candidate 后会先把 refcount 加一。
这不是“我已经淘汰了旧页”，只是“我暂时防止这个 slot 被别人抢走”。
第二，content lock。
dirty page 写出需要至少 share-exclusive content lock。
原因不是为了改 descriptor，而是为了防止写出过程中 page 内容被并发 compact、hint bit 修改或结构性修改。
`FlushBuffer()` 在 `bufmgr.c:4505-4506` 明确要求调用者持有 pin，并且以 exclusive 或 share-exclusive 模式锁住 buffer contents。
第三，I/O ownership。
`FlushBuffer()` 会通过 `StartSharedBufferIO()` 设置 `BM_IO_IN_PROGRESS`。
这个 bit 的含义是：
`当前有一个 backend 或 AIO path 对这个 buffer 的读或写拥有 I/O 执行权。`
它不是 content lock，也不是 mapping lock。
它用来让同一个 buffer 上的读写 I/O 串行化，并让错误恢复路径知道需要清理什么状态。
第四，durability 分层。
`smgrwrite()` 把 page 交给内核。
`mdwritev()` 在成功写入后登记 dirty segment。
`sync.c` 的 pending fsync table 让 checkpointer 在 checkpoint 结束前处理 fsync。
`WritebackContext` 只是请求内核提前回写数据页，降低后续 fsync 压力；它不保证持久化。
把这四层合在一起看，eviction 不是“移除缓存项”。
它是一个跨 buffer manager、WAL、smgr、md、sync 和 OS page cache 的状态转移。
## 3. 核心文件分工与阅读顺序
建议按下面顺序读。
不要按文件名字母顺序读。
| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/storage/buf_internals.h` | `BufferDesc.state` 的 refcount、usage_count、flags、content lock 位，以及 `BufferTag`、`WritebackContext` |
| 2 | `src/backend/storage/buffer/freelist.c` | `StrategyGetBuffer()` 如何返回已经 pin 住的 candidate，ring 如何拒绝高成本 dirty victim |
| 3 | `src/backend/storage/buffer/bufmgr.c` | `BufferAlloc()`、`GetVictimBuffer()`、`InvalidateVictimBuffer()`、`FlushBuffer()`、I/O 状态清理 |
| 4 | `src/include/utils/rel.h` | `RelationNeedsWAL()` 的语义边界 |
| 5 | `src/backend/access/transam/xloginsert.c` | `XLogInsert()` 返回 end LSN，hint bit FPI 何时产生 |
| 6 | `src/backend/access/transam/xlog.c` | `XLogFlush()` 和 `XLogNeedsFlush()` 如何定义 WAL flush 边界 |
| 7 | `src/include/storage/smgr.h`、`src/backend/storage/smgr/smgr.c` | `smgrwrite()` 到 `smgrwritev()` 的抽象边界，`smgrregistersync()` 的使用场景 |
| 8 | `src/backend/storage/smgr/md.c` | `mdwritev()` 如何实际写文件并登记 dirty segment |
| 9 | `src/backend/storage/sync/sync.c` | checkpointer 如何吸收和处理 fsync request |
| 10 | `src/backend/catalog/storage.c` | `wal_level=minimal` 下 skip WAL 的 pending sync 路径 |
本节会多次强调一个事实：
`BufferTag 足以让 flusher 找到物理文件，但不足以知道“修改时是否需要 WAL”。`
`RelationNeedsWAL()` 是 relation/access-method 层在修改时做的决定。
`FlushBuffer()` 在 eviction 时没有 `Relation` 对象，也不应该依赖 relcache 可见性。
所以它用 `BM_PERMANENT` 判断是否需要对 page LSN 调用 `XLogFlush()`。
## 4. 关键数据结构与状态
先读 `buf_internals.h`。
`BufferDesc.state` 是一个 64-bit 组合状态。
当前基线在 `buf_internals.h:33-45` 写明：
```text
18 bits refcount
4 bits usage count
12 bits flags
18 bits share-lock count
1 bit share-exclusive locked
1 bit exclusive locked
```
这意味着 buffer descriptor 的许多状态变化可以通过 CAS 完成。
不要把 `BufferDesc.state` 当成一组独立 bool。
它的语义来自：
`refcount + usage_count + flags + content lock state + lifecycle context`
### 4.1 replacement 相关位
本节最重要的 flags 定义在 `buf_internals.h:105-127`：
| flag | 本节语义 |
| --- | --- |
| `BM_DIRTY` | buffer 内容需要写回 relation 文件 |
| `BM_VALID` | buffer 中的 page 内容有效 |
| `BM_TAG_VALID` | buffer tag 已关联到 buffer mapping table |
| `BM_IO_IN_PROGRESS` | read 或 write I/O 正在进行 |
| `BM_IO_ERROR` | 上一次 I/O 失败 |
| `BM_PIN_COUNT_WAITER` | 有 backend 正在等待该 buffer 只剩自己的 pin |
| `BM_PERMANENT` | buffer 属于 crash 后仍应存在的永久数据或 init fork |
注意 `BM_TAG_VALID` 的注释很关键。
`buf_internals.h:98-99` 的语义是：
`BM_TAG_VALID 基本表示 buffer hash table 中有一个与 tag 对应的 entry。`
所以清掉 `BM_TAG_VALID` 不是只清一个字段。
它意味着这个 slot 不再能通过 old tag 被 lookup。
### 4.2 `BufferTag` 是写回的最小身份
`BufferTag` 在 `buf_internals.h:161-168` 包含：
```text
spcOid
dbOid
relNumber
forkNum
blockNum
```
源码注释在 `buf_internals.h:150-156` 说明了为什么 tag 必须足以决定写到哪里：
后台写出 buffer 的 backend 可能根本看不到创建这个 relation 的事务。
它不能依赖 `pg_class` 或 relcache 的可见性。
它必须仅凭 `BufferTag` 找到物理 relation 文件。
这正是 `FlushBuffer()` 在 `bufmgr.c:4537-4539` 中如果没有 `SMgrRelation` 就用 `smgropen(BufTagGetRelFileLocator(&buf->tag), INVALID_PROC_NUMBER)` 的原因。
### 4.3 `WritebackContext` 不是 durability 状态
`WritebackContext` 定义在 `buf_internals.h:390-410`。
它只保存若干 `PendingWriteback`，每个 request 只有一个 `BufferTag`。
它没有 LSN。
它没有 dirty bit。
它没有 fsync 成功状态。
它的作用是：
`把刚写过的数据页 tag 暂存起来，排序、合并相邻 block，然后调用 smgrwriteback() 请求 OS 提前回写。`
因此它是性能机制。
它不是 correctness 机制。
如果 writeback request 丢了，正确性仍由 WAL-before-data 和 checkpoint fsync request 保证。
## 5. 主流程源码 walkthrough
本节主流程从 `BufferAlloc()` 开始。
`BufferAlloc()` 在 `bufmgr.c:2177-2180` 的注释已经把职责写得很准：
`如果没有已有 buffer，选择 replacement victim 并 evict old page，但不读入新 page。`
这句话有两个边界。
第一，evict old page 是 `BufferAlloc()` 的职责链。
第二，读入 new page 不是这里做。
`BufferAlloc()` 只负责让返回的 buffer slot 已经绑定到目标 tag，并处于 pinned 状态。
### 5.1 miss 后先释放 mapping lock
`BufferAlloc()` 先构造 `newTag`，算 hash 和 partition lock。
如果 `BufTableLookup()` miss，它在 `bufmgr.c:2257-2268` 释放 new tag 的 partition lock，然后调用：
`GetVictimBuffer(strategy, io_context)`
这里的释放非常重要。
挑 victim、写 dirty page、WAL flush、数据文件 write 都可能很慢。
如果一直持有 new tag 的 mapping partition lock，会把同一 partition 的 unrelated lookup / insert 全部拖住。
所以 PostgreSQL 先离开 mapping table 临界区，再做高成本 victim 处理。
### 5.2 StrategyGetBuffer() 只返回 candidate
`GetVictimBuffer()` 调用 `StrategyGetBuffer()`。
`freelist.c:168-181` 的注释说得很直接：
```text
caller 需要的是一个当前没有被 pin 的 buffer；
返回前这个 buffer 会被 pin 并记录到 ResourceOwner。
```
`StrategyGetBuffer()` 的全局 clock sweep 路径在 `freelist.c:239-317`。
关键条件是：
```text
refcount != 0       -> 不能用
usage_count != 0   -> 减一后继续扫
refcount == 0 && usage_count == 0 -> CAS 加 pin，返回
```
所以它并不检查 dirty。
dirty 不阻止 candidate 被选中。
dirty 只意味着后续 `GetVictimBuffer()` 需要付出写回成本。
这就是前一节和本节的交界：
`replacement candidate selection 不等于 eviction 完成。`
### 5.3 victim 被 pin 之后仍可能放弃
`GetVictimBuffer()` 在 `bufmgr.c:2569` 得到一个已经 pin 住的 buffer。
随后 `bufmgr.c:2572-2575` 检查当前 backend 对它只 pin 了一次。
这很重要。
如果同一 backend 已经持有这个 buffer 的别的 pin，复用它会破坏调用者对旧页的引用语义。
接下来进入 dirty victim 分支。
`bufmgr.c:2578-2583` 的注释指出一个 race：
```text
StrategyGetBuffer() 看到 buffer 未使用；
但在真正 invalidation 前，另一个 backend 可能 pin 它并 dirty 它。
```
源码的处理不是假装没有 race。
它在后面的 `InvalidateVictimBuffer()` 里重新检查：
```text
refcount 是否仍为 1；
BM_DIRTY 是否仍为 0。
```
只要不满足，就放弃这个 victim，unpin 后重新选。
### 5.4 dirty victim 必须先拿 share-exclusive content lock
如果 `buf_state & BM_DIRTY`，`GetVictimBuffer()` 在 `bufmgr.c:2584-2639` 处理 dirty victim。
第一步不是写。
第一步是拿 content lock：
`BufferLockConditional(buf, buf_hdr, BUFFER_LOCK_SHARE_EXCLUSIVE)`
当前基线用 share-exclusive，而不是只靠 pin。
原因在 `bufmgr.c:2590-2592`：
写出时如果有人正在 compact page 内容，可能写出不一致数据。
share-exclusive lock 足以阻止并发修改，同时允许它和历史上 hint-bit / flush race 的修复配合。
这里必须是 conditional acquisition。
`bufmgr.c:2593-2601` 解释了死锁风险：
另一个 backend 可能在 `StrategyGetBuffer()` 返回后 pin 并锁住了这个页。
如果 flusher 无条件等 content lock，而对方又等待 flusher 持有的资源，就可能死锁。
所以当前 backend 拿不到锁时：
```text
UnpinBuffer(buf_hdr)
goto again
```
这不是失败。
这是 replacement path 的正常退避。
### 5.5 ring strategy 可能拒绝高 WAL flush 成本 victim
在 dirty victim 上，如果使用非默认 strategy，并且 victim 来自 ring，源码还有一个特殊判断。
`bufmgr.c:2613-2628`：
```text
strategy && from_ring
BM_PERMANENT
XLogNeedsFlush(BufferGetLSN(buf_hdr))
StrategyRejectBuffer(...)
```
这段逻辑的核心是：
`如果复用 ring 中的 dirty permanent buffer 会触发 WAL flush，bulkread 可以拒绝这个 victim。`
`StrategyRejectBuffer()` 在 `freelist.c:741-770`。
当前基线只对 `BAS_BULKREAD` 返回 true。
拒绝时会把当前 ring slot 设为 `InvalidBuffer`，避免所有 ring 成员都 dirty 时无限撞同一个 candidate。
这解释了一个常见性能现象：
同样是 read miss，bulk scan 命中自己的 ring 时通常便宜；
但如果 ring 里积累了 dirty permanent buffer，并且 page LSN 还没 flush，miss 会突然变成 WAL flush pressure。
PostgreSQL 对 bulkread 的选择是：
`宁可放弃这个 ring candidate，去全局 clock sweep 再找，也不要让读路径承担不必要的 WAL flush。`
### 5.6 真正写 dirty page：FlushBuffer()
如果没有被拒绝，`GetVictimBuffer()` 调用：
`FlushBuffer(buf_hdr, NULL, IOOBJECT_RELATION, io_context)`
调用点在 `bufmgr.c:2633-2635`。
传 `reln = NULL` 很常见。
因为 victim old page 的 relation 可能和当前 miss 请求的 relation 无关。
`FlushBuffer()` 会从 `BufferTag` 重新 `smgropen()`。
写完后，`GetVictimBuffer()` 解锁 content lock，再调用：
`ScheduleBufferTagForWriteback(&BackendWritebackContext, io_context, &buf_hdr->tag)`
这是本节第一次出现 writeback context。
注意顺序：
```text
FlushBuffer() 完成 smgrwrite
  -> 释放 content lock
  -> 记录 writeback hint
```
writeback hint 不参与 page 是否 clean 的判断。
clean 状态已经在 `FlushBuffer()` 内通过 `TerminateBufferIO(clear_dirty=true)` 完成。
### 5.7 dirty 写出后还要 invalidation
`FlushBuffer()` 写完 dirty page 后，old page 还没有从 buffer mapping table 删除。
`GetVictimBuffer()` 接着检查：
`if ((buf_state & BM_TAG_VALID) && !InvalidateVictimBuffer(buf_hdr))`
调用点在 `bufmgr.c:2664-2673`。
这里使用的 `buf_state` 是 `StrategyGetBuffer()` 返回时的旧状态。
它用于判断“这个 victim 原来是否有 tag”。
真正安全性检查在 `InvalidateVictimBuffer()` 里重新加锁完成。
`InvalidateVictimBuffer()` 的入口条件是：
```text
当前 backend 对 victim 有一个 pin；
buffer 有 valid tag；
不持有 buffer header lock。
```
它先拿 old tag 的 mapping partition lock，再锁 buffer header。
然后重新检查：
```text
refcount == 1
BM_DIRTY 未设置
tag 未变化
```
如果 refcount 不是 1，说明别的 backend 在我们 flush 或等待期间 pin 了它。
如果 `BM_DIRTY` 又变成 1，说明它又被修改了。
任一情况都返回 false，外层 unpin 并重新选 victim。
只有通过检查，才清理：
```text
ClearBufferTag()
unset BUF_FLAG_MASK
unset BUF_USAGECOUNT_MASK
BufTableDelete(old tag)
```
对应 `bufmgr.c:2520-2535`。
清理完成后的断言在 `bufmgr.c:2539-2541`：
```text
!(BM_DIRTY | BM_VALID | BM_TAG_VALID)
refcount > 0
```
这就是 victim 离场的关键状态：
```text
slot 仍被当前 backend pin 住；
但它不再代表任何 old BufferTag；
也不再包含有效 old page；
也不是 dirty。
```
### 5.8 BufferAlloc() 安装 new tag
`GetVictimBuffer()` 返回后，`BufferAlloc()` 重新拿 new tag 的 mapping partition lock。
它调用 `BufTableInsert()`。
如果另一个 backend 已经插入同一个 new tag，`BufferAlloc()` 放弃自己的 victim：
```text
UnpinBuffer(victim_buf_hdr)
PinBuffer(existing_buf_hdr)
return existing
```
这个冲突处理在 `bufmgr.c:2271-2317`。
如果插入成功，`BufferAlloc()` 锁 victim buffer header，在 `bufmgr.c:2324-2327` 断言：
```text
refcount == 1
!(BM_TAG_VALID | BM_VALID | BM_DIRTY | BM_IO_IN_PROGRESS)
```
然后设置：
```text
victim_buf_hdr->tag = newTag
BM_TAG_VALID
usage_count + 1
可能设置 BM_PERMANENT
```
注意此时不设置 `BM_VALID`。
`BM_TAG_VALID` 表示：
`这个 buffer slot 已经在 mapping table 中代表 new tag。`
`BM_VALID` 仍为 0 表示：
`new page 内容还没读入或初始化完成。`
这就是 miss path 中“tag 已安装但 page 还不可读”的中间状态。
后续 read I/O 完成后才会通过 `TerminateBufferIO(..., set_flag_bits=BM_VALID, ...)` 让它变成 valid。
## 6. FlushBuffer() 的 correctness 边界
`FlushBuffer()` 是本节最重要的函数。
它位于 `bufmgr.c:4495-4628`。
源码注释在 `bufmgr.c:4499-4503` 先划清边界：
```text
它实际只是把 buffer 内容传给内核；
真正写到磁盘要等内核调度；
checkpoint WAL 前还需要 fsync。
```
所以 `FlushBuffer()` 完成后只能说：
```text
shared buffer 中这个 page 不再 dirty；
数据页已经被 write() 交给 OS；
fsync responsibility 已经登记或由 fallback 处理；
它不是 durable-data guarantee 的终点。
```
### 6.1 StartSharedBufferIO() 独占 I/O
`FlushBuffer()` 的第一步是：
`StartSharedBufferIO(buf, false, true, NULL)`
调用点在 `bufmgr.c:4523-4529`。
`forInput=false` 表示写出。
`StartSharedBufferIO()` 在 `bufmgr.c:7241-7247` 明确了输入输出的判定：
```text
read I/O 只对 !BM_VALID 的 buffer；
write I/O 只对 BM_VALID && BM_DIRTY 的 buffer。
```
如果 I/O 已经被别人完成，返回 `BUFFER_IO_ALREADY_DONE`。
`FlushBuffer()` 直接 return。
如果可以开始 I/O，`StartSharedBufferIO()` 在 `bufmgr.c:7314-7319` 设置 `BM_IO_IN_PROGRESS`，并把这个 buffer I/O 记到 `CurrentResourceOwner`。
这解释错误恢复路径为什么能找到未完成 I/O。
只要 backend 在 I/O 中 ERROR，ResourceOwner cleanup 会调用 `AbortBufferIO()`。
### 6.2 page LSN 在 content lock 下读取
`FlushBuffer()` 在 `bufmgr.c:4547-4551` 读取：
`recptr = BufferGetLSN(buf)`
注释指出：
持有至少 share-exclusive content lock，所以 LSN 不会在 flush 期间变化，也不会 torn。
这里的 page LSN 来自 page header。
它不是 descriptor 上的 dirty bit。
它回答的是：
`这个 page 当前内容依赖 WAL 到哪里。`
`xloginsert.c:470-479` 对 `XLogInsert()` 返回值有同样语义：
返回 WAL record end pointer，可以作为受影响数据页的 LSN；
写数据页前 WAL 必须至少 flush 到这个 LSN。
### 6.3 XLogFlush(page LSN)
`FlushBuffer()` 的 WAL-before-data 核心在 `bufmgr.c:4553-4571`：
```text
if (BM_PERMANENT)
    XLogFlush(recptr)
```
这里不是 `RelationNeedsWAL()`。
原因有三层。
第一，`FlushBuffer()` 只有 `BufferTag`，没有可靠的 `Relation`。
第二，修改时是否需要 WAL 已经由 access method 根据 `RelationNeedsWAL()` 决定。
第三，flush 时必须保守保证永久数据页不能先于其 page LSN 对应 WAL 落盘。
`BM_PERMANENT` 是 buffer descriptor 上的写回属性。
`BufferAlloc()` 在 `bufmgr.c:2330-2338` 设置它：
`permanent relation 或 init fork -> BM_PERMANENT`
对于 unlogged relation 的普通 fork，`BM_PERMANENT` 不设置。
`FlushBuffer()` 注释在 `bufmgr.c:4558-4568` 解释了为什么跳过非 permanent buffer 的 WAL flush：
unlogged relation crash 后会丢失；
有些 unlogged index page 可能带 fake LSN；
fake LSN 可能超过 WAL insert point；
对 fake LSN 调用 `XLogFlush()` 会产生系统级灾难。
所以：
```text
RelationNeedsWAL() 决定修改时是否生成 WAL。
BM_PERMANENT 决定 flush 时是否必须按 page LSN flush WAL。
```
不要把二者互换。
### 6.4 RelationNeedsWAL() 的位置
`RelationNeedsWAL()` 定义在 `src/include/utils/rel.h:631-642`。
它为 true 的条件是：
```text
relation 是 permanent；
并且 XLogIsNeeded()
或者 relation 不是当前事务新建/首次 relfilenumber 的 WAL-skipping 对象。
```
注释明确说：
`wal_level=minimal` 下，如果 relation 在当前事务 created 或 truncated，可能返回 false。
`access/transam/README:745-770` 的 “Skipping WAL for New RelFileLocator” 解释了为什么这必须小心：
如果同一个 block 上 WAL-writing change 和 WAL-skipping change 混用，redo 对页面前态的假设会被破坏。
因此 WAL-skipping relfilenumber 在 commit 前必须靠同步数据文件来替代 WAL 保护。
相关代码在 `catalog/storage.c`：
```text
RelationCreateStorage() 对 permanent 且 !XLogIsNeeded() 的新 relation 调 AddPendingSync()
smgrDoPendingSyncs() 在 commit 时为 pending sync relation 选择 log_newpage_range() 或 smgrdosyncall()
```
这条路径服务的是：
`新 relfilenumber 在 wal_level=minimal 下跳过普通页面 WAL 后，如何在 commit 前让内容 crash-safe。`
它不是 `FlushBuffer()` 里判断是否 `XLogFlush(page_lsn)` 的替代品。
### 6.5 PageSetChecksum()
WAL flush 完成后，`FlushBuffer()` 在 `bufmgr.c:4579-4582` 做：
```text
bufBlock = BufHdrGetBlock(buf)
PageSetChecksum((Page) bufBlock, buf->tag.blockNum)
```
checksum 在这里设置，而不是每次修改 page 时设置。
`storage/page/README` 说明：
```text
page checksum 在 shared buffer 中并不总是有效；
它在 page 离开 shared buffer、被写出前设置；
WAL full-page image 不依赖 page checksum，而有自己的 WAL CRC。
```
这对诊断很重要。
如果你在 gdb 里看 shared buffer 内 page header 的 `pd_checksum`，它不一定代表当前内存 page 内容。
真正写出前，`FlushBuffer()` 会重新计算。
这也解释了 `MarkSharedBufferDirtyHint()` 中的注释：
hint bit 写了 FPI-for-hint 后不立刻重置 checksum。
checksum 会在本 checkpoint cycle 后续写出该 page 时更新。
### 6.6 smgrwrite()
`FlushBuffer()` 在 `bufmgr.c:4586-4590` 调：
`smgrwrite(reln, forkNum, blockNum, bufBlock, false)`
第五个参数 `skipFsync=false`。
`smgrwrite()` 是 `smgr.h:130-135` 的 inline wrapper，实际调用 `smgrwritev()` 写一个 block。
`smgrwritev()` 在 `smgr.c:790-797` 通过 storage manager switch 调 `mdwritev()`，并用 `HOLD_INTERRUPTS()` 包住。
`mdwritev()` 在 `md.c:1070-1166` 做实际文件写入。
关键点：
```text
FileWriteV()
处理短写循环
失败时 ereport(ERROR)
成功后如果 !skipFsync && !temp，register_dirty_segment()
```
`mdwritev()` 的返回仍然不是 durable。
它只是把数据页写给 OS。
durability 还要看 dirty segment 是否被登记，并最终由 checkpointer fsync。
### 6.7 register_dirty_segment() 和 fsync request
`register_dirty_segment()` 在 `md.c:1510-1557`。
它构造 `FileTag`，调用：
`RegisterSyncRequest(&tag, SYNC_REQUEST, false)`
如果请求成功，后续由 checkpointer 处理。
如果请求队列满且 `retryOnError=false`，`RegisterSyncRequest()` 返回 false。
`register_dirty_segment()` 的 fallback 是：
`本 backend 直接 FileSync()`
并用 `IOCONTEXT_NORMAL / IOOP_FSYNC` 计数。
这条 fallback 是 eviction miss 成本放大的重要来源。
一个 backend 本来只是想读一个 miss block。
它可能因为 dirty victim 写出，进一步承担：
```text
WAL fsync
data file write
sync request forward
如果队列满，则 data file fsync
```
### 6.8 smgrregistersync() 的边界
`smgrregistersync()` 在 `smgr.c:928-945`。
它的用途不是普通 `FlushBuffer()`。
注释明确说：
`可以在调用 smgrwrite() 或 smgrextend() 且 skipFsync=true 后，用它补登记下一次 checkpoint 需要 fsync 的 relation。`
还提醒：
如果 `smgrwrite()` 和 `smgrregistersync()` 之间已经发生 checkpoint，那么这个 checkpoint 已经错过该 relation。
那种情况下应该用 `smgrimmedsync()`。
所以本节需要把三类路径分清：
| 路径 | 典型调用 | fsync 责任 |
| --- | --- | --- |
| 普通 buffer flush | `FlushBuffer()` -> `smgrwrite(..., skipFsync=false)` | `mdwritev()` 自动 `register_dirty_segment()` |
| bulk / skip fsync 写 | `smgrwrite(..., skipFsync=true)` | 调用者必须另行 `smgrregistersync()` 或 `smgrimmedsync()` |
| WAL-skipping new relfilelocator commit | `smgrDoPendingSyncs()` | 小文件可 `log_newpage_range()`，大文件 `smgrdosyncall()` |
把 `smgrregistersync()` 误写进普通 dirty victim flush 主链路，是一个常见错误。
普通 `FlushBuffer()` 依赖的是 `skipFsync=false` 下 md 层自动登记 dirty segment。
### 6.9 TerminateBufferIO() 清 dirty 和 I/O bit
`FlushBuffer()` 在 `bufmgr.c:4615-4618` 最后调用：
`TerminateBufferIO(buf, true, 0, true, false)`
参数含义：
```text
clear_dirty = true
set_flag_bits = 0
forget_owner = true
release_aio = false
```
`TerminateBufferIO()` 在 `bufmgr.c:7366-7413`。
它必定清：
```text
BM_IO_IN_PROGRESS
BM_IO_ERROR
```
如果 `clear_dirty=true`，还清：
```text
BM_DIRTY
BM_CHECKPOINT_NEEDED
```
然后从 ResourceOwner 忘记这个 buffer I/O，并广播 buffer I/O condition variable。
所以 successful write 的状态结果是：
```text
BM_VALID 仍然保留；
BM_TAG_VALID 仍然保留；
BM_DIRTY 清掉；
BM_IO_IN_PROGRESS 清掉；
BM_IO_ERROR 清掉。
```
注意：`BM_VALID` 和 `BM_TAG_VALID` 不在 `FlushBuffer()` 中清。
它们在后续 eviction invalidation 中清。
这是两个阶段。
## 7. BM_TAG_VALID / BM_VALID / BM_DIRTY / BM_IO_IN_PROGRESS 生命周期
把四个状态位按时间顺序放在一起。
### 7.1 old clean page 被选为 victim
状态大致是：
```text
BM_TAG_VALID = 1
BM_VALID = 1
BM_DIRTY = 0
BM_IO_IN_PROGRESS = 0
refcount = 0
usage_count = 0
```
`StrategyGetBuffer()` CAS 加 pin 后：
```text
refcount = 1
flags 不变
```
`GetVictimBuffer()` 不需要 `FlushBuffer()`。
它直接进入 `InvalidateVictimBuffer()`。
`InvalidateVictimBuffer()` 清：
```text
BufferTag
BUF_FLAG_MASK
BUF_USAGECOUNT_MASK
```
结果：
```text
BM_TAG_VALID = 0
BM_VALID = 0
BM_DIRTY = 0
BM_IO_IN_PROGRESS = 0
refcount = 1
```
然后 `BufferAlloc()` 安装 new tag：
```text
BM_TAG_VALID = 1
BM_VALID = 0
BM_DIRTY = 0
BM_IO_IN_PROGRESS = 0
```
### 7.2 old dirty page 被选为 victim
初始：
```text
BM_TAG_VALID = 1
BM_VALID = 1
BM_DIRTY = 1
BM_IO_IN_PROGRESS = 0
```
`StrategyGetBuffer()` 加 pin。
`GetVictimBuffer()` 拿 share-exclusive content lock。
`FlushBuffer()` 调 `StartSharedBufferIO()`：
```text
BM_IO_IN_PROGRESS = 1
BM_DIRTY = 1
BM_VALID = 1
BM_TAG_VALID = 1
```
WAL flush、checksum、smgrwrite 成功后：
`TerminateBufferIO(clear_dirty=true)`：
```text
BM_IO_IN_PROGRESS = 0
BM_DIRTY = 0
BM_CHECKPOINT_NEEDED = 0
BM_VALID = 1
BM_TAG_VALID = 1
```
随后 `InvalidateVictimBuffer()`：
```text
BM_VALID = 0
BM_TAG_VALID = 0
usage_count = 0
tag cleared
```
最后 `BufferAlloc()` 安装 new tag：
```text
BM_TAG_VALID = 1
BM_VALID = 0
BM_DIRTY = 0
BM_IO_IN_PROGRESS = 0
```
### 7.3 read miss 绑定 new tag 后
`BufferAlloc()` 返回时，新页一般还没读入。
状态是：
```text
BM_TAG_VALID = 1
BM_VALID = 0
BM_DIRTY = 0
BM_IO_IN_PROGRESS = 0
```
后续 read path 调 `StartSharedBufferIO(forInput=true)`。
读取过程中：
```text
BM_IO_IN_PROGRESS = 1
BM_VALID = 0
```
读取成功后：
`TerminateBufferIO(..., set_flag_bits=BM_VALID, ...)`
结果：
```text
BM_IO_IN_PROGRESS = 0
BM_VALID = 1
```
这就是 hit path 为什么可能找到 tag 但 `BM_VALID=0`。
它代表：
```text
同一个 block 的 buffer slot 已经存在；
但内容还在 I/O 中，或者上次读失败，或者 StartReadBuffers() 尚未 WaitReadBuffers()。
```
`BufferAlloc()` 在 `bufmgr.c:2244-2252` 对这种情况把 `foundPtr` 改回 false，让调用者走读入/等待路径。
## 8. cleanup lock 与 pin 等待边界
eviction 的 pin 边界和 page cleanup 的 pin 边界不一样。
### 8.1 replacement 只要求 candidate 当前没有别人 pin
`StrategyGetBuffer()` 要求：
`refcount == 0`
然后它自己加一个 pin。
这个 pin 的目标是保护 slot 不被别人同时复用。
之后如果发现 dirty、content lock 拿不到、别人又 pin 了或又 dirty 了，可以放弃 candidate。
replacement 不会等待别人的 pin 释放。
如果所有 buffer 都 pin 住，`freelist.c:263-275` 会 ERROR：
`no unpinned buffers available`
这是一种保护。
无限等待可能把系统卡死，特别是 pin 泄漏或长时间持 pin 的错误路径。
### 8.2 cleanup lock 要求“只有自己的 pin”
`LockBufferForCleanup()` 在 `bufmgr.c:6663-6677` 定义了另一个边界。
删除 disk page 上的 item 需要：
```text
持有 exclusive content lock；
并观察到没有其他 backend pin 这个 buffer。
```
原因是其他 backend 只要 pin 住 buffer，就可能持有指向 page item 的指针或 offset 语义。
即使它没持 content lock，cleanup 也不能移动或删除它可能引用的 item。
所以 `LockBufferForCleanup()` 的目标不是 replacement。
它的目标是：
`在页面内部删除 item 前，确认没有旧读者仍靠 pin 保护页面内容位置。`
### 8.3 BM_PIN_COUNT_WAITER
`LockBufferForCleanup()` 的循环在 `bufmgr.c:6704-6818`。
它先拿 exclusive content lock，再锁 buffer header 检查 refcount。
如果 `refcount == 1`，说明只有自己这个 pin，成功返回。
如果还有别人 pin：
```text
设置 wait_backend_pgprocno = MyProcNumber
设置 BM_PIN_COUNT_WAITER
释放 exclusive content lock
等待信号
醒来后清理 waiter bit
重试
```
`BM_PIN_COUNT_WAITER` 当前只允许一个 waiter。
如果已经有 waiter，源码在 `bufmgr.c:6737-6743` 报错：
`multiple backends attempting to wait for pincount 1`
### 8.4 UnpinBuffer() 唤醒 cleanup waiter
`UnpinBufferNoOwner()` 在 `bufmgr.c:3474-3514`。
当本 backend 的私有 refcount 降到 0，它通过 atomic 减 shared refcount。
如果旧状态有 `BM_PIN_COUNT_WAITER`，调用：
`WakePinCountWaiter(buf)`
`TerminateBufferIO()` 也有类似处理。
`bufmgr.c:7403-7412` 说明：
如果 AIO subsystem 释放了最后一个 pin，也可能需要唤醒 cleanup waiter。
所以 cleanup lock 的等待者不是等 content lock。
它等的是：
`其他 pin 离开。`
### 8.5 和 eviction 的区别
把两者并排：
| 机制 | 等什么 | 为什么 |
| --- | --- | --- |
| eviction victim selection | 不等别人的 pin，找 refcount 为 0 的 candidate | 复用 slot，避免无限等待 |
| dirty victim flush | conditional content lock，拿不到就放弃 | 避免与持锁者死锁 |
| `LockBufferForCleanup()` | 等 refcount 降到 1 | 删除 item 前必须确认没有旧读者 pin |
| `WaitIO()` | 等 `BM_IO_IN_PROGRESS` 清掉 | I/O 状态必须串行化 |
这四种等待不要混为一个“buffer lock wait”。
诊断时要看 wait event 和源码位置。
## 9. Writeback context：性能 hint，不是 fsync
`WritebackContextInit()` 在 `bufmgr.c:7678-7693`。
它保存：
```text
max_pending
nr_pending
pending_writebacks[]
```
`ScheduleBufferTagForWriteback()` 在 `bufmgr.c:7696-7732`。
它在以下情况直接 return：
```text
direct I/O data path 启用；
fsync disabled。
```
如果启用 writeback control，它把 `BufferTag` 放进 pending array。
当 pending 数量达到上限，调用 `IssuePendingWritebacks()`。
`IssuePendingWritebacks()` 在 `bufmgr.c:7741-7827`。
它做三件事：
```text
按 BufferTag 排序；
合并同一 relation/fork 上连续或重复 block；
调用 smgrwriteback(reln, fork, block, nblocks)。
```
`smgrwriteback()` 在 `smgr.c:801-812` 转发到 `mdwriteback()`。
`mdwriteback()` 在 `md.c:1170-1224` 调 `FileWriteback()`。
源码注释说得很清楚：
```text
writeback 接收 block range，因为一次 flush 多页更高效；
如果 segment 不再打开，忽略即可。
```
这说明 writeback request 允许失败或丢失。
`IssuePendingWritebacks()` 的注释在 `bufmgr.c:7745-7746` 也说：
它只用于改善 OS I/O scheduling，尽量不 error out。
因此本节用一个公式区分：
```text
FlushBuffer -> smgrwrite -> register dirty segment -> checkpoint fsync 是 correctness path。
ScheduleBufferTagForWriteback -> smgrwriteback -> FileWriteback 是 performance path。
```
## 10. 错误路径与异常恢复
dirty victim 写出最危险的地方是错误路径。
它可能在 WAL flush、data file write、fsync request、writeback、checksum 或状态清理之间 ERROR。
当前源码靠 ResourceOwner、`BM_IO_IN_PROGRESS` 和 `BM_IO_ERROR` 保持 buffer descriptor 不悬空。
### 10.1 FlushBuffer() 的 error context
`FlushBuffer()` 在 `bufmgr.c:4531-4535` 安装 error context callback。
`shared_buffer_write_error_callback()` 在 `bufmgr.c:7465-7477`。
如果写出报错，用户能看到类似：
`writing block <blocknum> of relation "<path>"`
这不是装饰。
它让 data file write error 能回到具体 relfilenode、fork 和 block。
### 10.2 mdwritev() write error
`mdwritev()` 在 `md.c:1135-1145` 处理 `FileWriteV()` 失败。
如果 errno 是 `ENOSPC`，额外提示检查磁盘空间。
这里会 `ereport(ERROR)`。
因为 `FlushBuffer()` 已经设置了 `BM_IO_IN_PROGRESS` 并把 I/O 记入 ResourceOwner，ERROR unwinding 时会走 buffer I/O cleanup。
### 10.3 AbortBufferIO()
ResourceOwner callback 在 `bufmgr.c:7831-7837`：
```text
ResOwnerReleaseBufferIO()
  -> AbortBufferIO(buffer)
```
`AbortBufferIO()` 在 `bufmgr.c:7416-7462`。
它的规则：
如果 buffer 还不是 valid，说明读入或初始化失败。
这时断言不能 dirty。
如果 buffer 是 valid，说明写 dirty page 失败。
这时断言必须 dirty。
最后调用：
`TerminateBufferIO(buf_hdr, false, BM_IO_ERROR, false, false)`
注意 `clear_dirty=false`。
写失败不能清 `BM_DIRTY`。
否则 dirty page 会被误认为已经安全写出，造成数据丢失。
错误后的状态是：
```text
BM_IO_IN_PROGRESS 清掉
BM_IO_ERROR 设置
BM_DIRTY 保留
BM_VALID 保留
BM_TAG_VALID 保留
```
后续再有人尝试写出时，`StartSharedBufferIO()` 可以重新开始 I/O。
如果再次失败，`AbortBufferIO()` 在 `bufmgr.c:7447-7457` 会 WARNING：
`Multiple failures --- write error might be permanent.`
### 10.4 XLogFlush() 的坏 LSN 路径
`XLogFlush()` 在 `xlog.c:2794-2977`。
如果 record 已经 flush，快速返回。
否则进入 critical section，等待 WAL insertions 完成，获取 `WALWriteLock`，调用 `XLogWrite()`。
写完后，如果 `LogwrtResult.Flush < record`，`xlog.c:2944-2969` 报 ERROR。
注释说明这个场景常见原因是数据页上的 LSN 损坏，flush request 超过 WAL end。
它还特别说明：
```text
从 xact.c 调用时因为在 critical section 内，ERROR 会升级为 PANIC；
从 bufmgr.c 调用时不在 critical section 内，所以坏数据页 LSN 不会强制重启整个系统。
```
这就是 `FlushBuffer()` 不在 critical section 内调用 `XLogFlush()` 的实际恢复收益。
一个坏 page LSN 不应该把整个 postmaster 打崩。
### 10.5 sync.c 的 fsync retry
`ProcessSyncRequests()` 在 `sync.c:284-476`。
checkpointer 在 checkpoint 期间处理 pending fsync entries。
关键 race 在 `sync.c:311-320`：
backend 可能在 checkpointer 扫到某个 buffer 前刚好写出并登记 fsync。
只要 checkpointer 在 `BufferSync()` 后再 `AbsorbSyncRequests()`，就能把这个 request 纳入 checkpoint。
处理 fsync 时，如果文件可能已被删除，`sync.c:400-459` 会吸收 cancel/forget request 后重试。
如果不是可能删除，或重试后仍失败，则 `data_sync_elevel(ERROR)`。
这说明 fsync request table 是 checkpoint correctness 的一部分。
它不是简单日志队列。
## 11. Eviction miss 的成本放大
现在把一次普通 miss 的成本拆开。
最便宜的 miss：
```text
mapping miss
StrategyGetBuffer 找到 clean invalid 或 clean valid candidate
InvalidateVictimBuffer 清 old tag
BufferAlloc 安装 new tag
read new block
```
高成本 miss：
```text
mapping miss
clock sweep 扫过大量 pinned 或 usage_count > 0 buffers
选中 dirty permanent victim
conditional content lock 失败，重新选
再次选中 dirty victim
page LSN 未 flush
XLogFlush(page LSN)
PageSetChecksum()
smgrwrite()
mdwritev()
RegisterSyncRequest()
ScheduleBufferTagForWriteback()
InvalidateVictimBuffer()
read new block
```
更糟糕的 miss：
```text
RegisterSyncRequest 队列满
register_dirty_segment fallback 到 FileSync()
```
这时一个读 miss 可能承担 data file fsync。
### 11.1 为什么 latency 会突然跳
你在生产环境里看到某些查询偶发变慢时，不能只看 shared_blks_read。
同样是 shared read miss，可能有以下差异：
| 额外成本 | 触发条件 | 可观察入口 |
| --- | --- | --- |
| clock sweep 多扫 | usage_count 高或大量 pin | `pg_stat_bgwriter`、perf、gdb |
| WAL flush | dirty permanent victim 的 page LSN 尚未 flush | `pg_stat_wal`、wait event WALWrite/WALSync |
| data file write | victim dirty | `pg_stat_io` relation write |
| backend fsync fallback | sync request queue 满或 standalone/checkpointer context | `pg_stat_io` fsync |
| cleanup wait | page-level cleanup 等 pin | `wait_event = BufferCleanup` |
| buffer I/O wait | 其他 backend 正在读/写该 buffer | `wait_event = BufferIO` |
### 11.2 为什么 bgwriter/checkpointer 不能完全消除 backend write
`SyncOneBuffer()` 在 `bufmgr.c:4124-4198` 是 bgwriter/checkpointer 写单个 dirty buffer 的核心。
它会：
```text
检查 BM_VALID && BM_DIRTY；
PinBuffer_Locked()；
FlushUnlockedBuffer()；
ScheduleBufferTagForWriteback()；
UnpinBuffer()。
```
但 bgwriter/checkpointer 只能提前清理一部分 dirty buffers。
当 backend miss 需要一个 victim，而当前 candidate 仍 dirty，backend 仍可能自己写。
这不是 bug。
这是 shared buffer pool 在压力下的正常 backpressure：
`产生 dirty page 的系统如果后台写回跟不上，前台 miss 会承担一部分写回成本。`
### 11.3 BAS_BULKREAD 的取舍
bulkread ring 试图避免大顺序扫描污染整个 shared buffers。
但 ring 中的 dirty victim 如果要 WAL flush，读路径会被放大。
当前源码通过 `StrategyRejectBuffer()` 对 bulkread 做局部退避。
它不是避免所有 dirty write。
它只避免：
`bulkread ring reuse 时因为 WAL flush 付出异常高成本。`
如果全局 pool 压力很高，最后仍可能写 dirty victim。
## 12. 常见误区
误区一：
`FlushBuffer() 完成就代表数据页已经落盘。`
不对。
它代表 page 已经 write 给内核，并登记了后续 fsync 责任。
durable 边界在 checkpoint fsync 或特殊路径的 immediate sync。
误区二：
`RelationNeedsWAL() 为 false，所以 FlushBuffer() 不需要看 page LSN。`
不对。
`RelationNeedsWAL()` 是修改时的 relation-level 决策。
`FlushBuffer()` 没有可靠 Relation，只用 `BM_PERMANENT` 判断是否对 page LSN 调 `XLogFlush()`。
误区三：
`BM_DIRTY 清掉以后就能马上复用 slot。`
不完整。
dirty 清掉只是写出完成。
复用前还要清 old tag 和 mapping entry。
这由 `InvalidateVictimBuffer()` 完成。
误区四：
`BM_TAG_VALID 和 BM_VALID 总是同时变化。`
不对。
miss path 安装 new tag 后，会出现：
```text
BM_TAG_VALID=1
BM_VALID=0
```
这表示 tag 已经可 lookup，但 page 内容还没 ready。
误区五：
`writeback context 是 fsync queue。`
不对。
writeback context 是 OS writeback hint。
fsync queue 在 sync.c 的 pendingOps / checkpointer request path。
误区六：
`cleanup lock 和 replacement 都是在等 pin，所以是一回事。`
不对。
replacement 不等 pin，只找当下未 pin 的 victim。
cleanup lock 必须等到只有自己的 pin，因为它要删除或移动 page 内 item。
误区七：
`checksum 每次 page 修改后立即有效。`
不对。
page checksum 在 shared buffer 中经常无效。
写出前 `FlushBuffer()` 才 `PageSetChecksum()`。
## 13. 观测与诊断入口
### 13.1 pg_stat_io
在支持 `pg_stat_io` 的当前基线中，重点看：
```sql
SELECT backend_type, object, context, reads, writes, writebacks, fsyncs,
       read_time, write_time, writeback_time, fsync_time
FROM pg_stat_io
WHERE object = 'relation'
ORDER BY backend_type, context;
```
解释方向：
```text
client backend 的 writes 增长，说明前台 backend 正在写 dirty buffers。
client backend 的 fsyncs 增长，说明出现了 backend fsync fallback 或 immediate sync 类路径。
background writer / checkpointer 的 writes 增长，说明后台正在承担 dirty buffer write。
writebacks 增长，说明 writeback hint 被发出，不代表 fsync 完成。
```
### 13.2 wait events
可关注：
```sql
SELECT pid, wait_event_type, wait_event, query
FROM pg_stat_activity
WHERE wait_event IS NOT NULL
ORDER BY pid;
```
常见解释：
```text
BufferIO        等某个 buffer 的 BM_IO_IN_PROGRESS 清掉
BufferCleanup   LockBufferForCleanup() 等其他 pin 释放
WALWrite/WALSync 或相关 WAL wait 可能来自 XLogFlush(page LSN)
DataFileWrite   smgr/md 写数据页
DataFileFlush   writeback 或 data file flush
```
wait event 名称会随版本和平台略有差异。
诊断时要回到源码调用点确认。
### 13.3 gdb 断点
调试版构建可以在这些点下断：
```gdb
break GetVictimBuffer
break FlushBuffer
break StartSharedBufferIO
break TerminateBufferIO
break InvalidateVictimBuffer
break XLogFlush
break mdwritev
break register_dirty_segment
break IssuePendingWritebacks
break LockBufferForCleanup
```
进入 `FlushBuffer()` 后可以看：
```gdb
p buf->tag
p/x pg_atomic_read_u64(&buf->state)
p BufferGetLSN(buf)
```
注意 `BufferGetLSN(buf)` 需要在合适锁语境下读。
不要在生产上随意 attach 并长时间停住 backend。
### 13.4 log_checkpoints
打开：
```conf
log_checkpoints = on
track_io_timing = on
```
可以看到 checkpoint write/sync 时间。
如果 eviction miss 经常被 backend write 放大，可能同时看到：
```text
client backend relation writes 增长；
checkpoint sync 时间增长；
bgwriter 无法充分提前清理 dirty buffers。
```
## 14. 可执行实验
下面实验都可以在临时测试库执行。
不要在生产库运行。
路径假设你已经能从 `/home/nail/postgres-lab` 构建 PostgreSQL。
如果已经有构建产物，可以跳过 configure/make。
### 实验 1：制造前台 backend dirty victim write
目标：
观察小 `shared_buffers` 下读 miss 被 dirty victim 写回放大。
准备：
```bash
cd /home/nail/postgres-lab
git rev-parse HEAD
./configure --enable-debug --enable-cassert --prefix=/tmp/pg-evict-lab
make -s -j"$(nproc)"
make -s install
rm -rf /tmp/pg-evict-data
/tmp/pg-evict-lab/bin/initdb -D /tmp/pg-evict-data
cat >> /tmp/pg-evict-data/postgresql.conf <<'EOF'
shared_buffers = '16MB'
checkpoint_timeout = '30min'
max_wal_size = '4GB'
track_io_timing = on
log_checkpoints = on
bgwriter_lru_maxpages = 0
EOF
/tmp/pg-evict-lab/bin/pg_ctl -D /tmp/pg-evict-data -l /tmp/pg-evict.log start
/tmp/pg-evict-lab/bin/createdb evictlab
```
执行：
```sql
SELECT pg_stat_reset_shared('io');
CREATE TABLE dirty_eviction AS
SELECT g AS id, repeat(md5(g::text), 20) AS payload
FROM generate_series(1, 300000) AS g;
CHECKPOINT;
UPDATE dirty_eviction
SET payload = payload || 'x'
WHERE id % 2 = 0;
CREATE TABLE scan_target AS
SELECT g AS id, repeat(md5((g * 17)::text), 20) AS payload
FROM generate_series(1, 300000) AS g;
SELECT count(*) FROM scan_target;
SELECT backend_type, object, context, reads, writes, writebacks, fsyncs,
       round(read_time::numeric, 3) AS read_ms,
       round(write_time::numeric, 3) AS write_ms,
       round(writeback_time::numeric, 3) AS wb_ms,
       round(fsync_time::numeric, 3) AS fsync_ms
FROM pg_stat_io
WHERE object = 'relation'
ORDER BY backend_type, context;
```
观察点：
```text
client backend 的 relation writes 是否增加；
writebacks 是否增加；
如果环境触发 sync request fallback，client backend fsyncs 也可能增加；
日志中 checkpoint 尚未发生时，前台也可能已经写 dirty buffers。
```
解释：
`SELECT count(*) FROM scan_target` 看似读另一个表。
但它产生 shared buffer miss，需要 victim。
如果 victim 来自 `dirty_eviction` 的 dirty pages，就会走 `GetVictimBuffer() -> FlushBuffer()`。
### 实验 2：观察 page LSN 触发 XLogFlush()
目标：
用 gdb 看到 dirty victim flush 前读取 page LSN，并进入 `XLogFlush()`。
步骤：
```bash
psql -d evictlab -c "SELECT pg_backend_pid();"
gdb -p <pid>
```
gdb：
```gdb
set pagination off
break FlushBuffer
commands
  silent
  printf "FlushBuffer buf_id=%d block=%u state=%lx\n", buf->buf_id, buf->tag.blockNum, pg_atomic_read_u64(&buf->state)
  printf "LSN: "
  p BufferGetLSN(buf)
  continue
end
break XLogFlush
commands
  silent
  printf "XLogFlush record="
  p record
  continue
end
```
在另一个 session 重跑实验 1 的 update 和 scan。
观察：
```text
FlushBuffer 先打印 page LSN；
随后可能进入 XLogFlush；
如果 WAL 已经 flush 到该 LSN，XLogFlush 快速返回；
如果未 flush，可能看到 WAL 写/同步等待。
```
### 实验 3：区分 writeback 与 fsync
目标：
证明 writeback 不是 fsync。
SQL：
```sql
SELECT pg_stat_reset_shared('io');
UPDATE dirty_eviction
SET payload = payload || 'y'
WHERE id % 3 = 0;
SELECT count(*) FROM scan_target;
SELECT backend_type, object, context, writes, writebacks, fsyncs,
       write_time, writeback_time, fsync_time
FROM pg_stat_io
WHERE object = 'relation'
ORDER BY backend_type, context;
```
再执行：
```sql
CHECKPOINT;
SELECT backend_type, object, context, writes, writebacks, fsyncs,
       write_time, writeback_time, fsync_time
FROM pg_stat_io
WHERE object = 'relation'
ORDER BY backend_type, context;
```
观察：
```text
scan 期间可能出现 writes/writebacks；
CHECKPOINT 后 fsyncs 通常由 checkpointer 增长；
writebacks 不是 durable boundary。
```
## 15. 讨论题
1. `StrategyGetBuffer()` 返回一个 pinned candidate 后，为什么还不能说旧页已经被淘汰？
2. dirty victim 为什么要用 conditional content lock，而不是无条件等待？
3. `FlushBuffer()` 成功后为什么只清 `BM_DIRTY / BM_IO_IN_PROGRESS`，不清 `BM_TAG_VALID / BM_VALID`？
4. `smgrwrite()` 返回成功、`register_dirty_segment()` 成功、checkpoint fsync 完成分别保证什么？
5. 为什么 `RelationNeedsWAL()` 属于修改路径，而 `BM_PERMANENT` 属于 flush 路径？
6. 如果 `pg_stat_io` 看到 client backend writes 增长，你如何判断是不是 read miss 被 dirty victim 放大？
7. cleanup lock 等 pin 与 replacement 找未 pinned victim 的边界有什么不同？
8. writeback context 为什么不能作为 correctness 证据？
## 16. 本节小结
本节的核心结论：eviction 不是一个删除动作，而是一串有顺序的安全边界。
第一，`StrategyGetBuffer()` 只选 candidate 并 pin 住。
它不处理 dirty，也不删除 old tag。
第二，dirty victim 必须在 share-exclusive content lock 下写出。
拿不到锁时，replacement path 放弃该 victim，而不是无条件等待。
第三，`FlushBuffer()` 的 correctness 顺序是 `StartSharedBufferIO()` -> 读取 page LSN -> permanent buffer 调 `XLogFlush(page LSN)` -> `PageSetChecksum()` -> `smgrwrite(..., skipFsync=false)` -> `TerminateBufferIO(clear_dirty=true)`。
第四，`RelationNeedsWAL()` 和 `BM_PERMANENT` 不在同一层。
`RelationNeedsWAL()` 是修改时是否生成 WAL 的 relation/access-method 决策。
`BM_PERMANENT` 是 flush 时避免永久数据页越过 WAL-before-data 的 buffer descriptor 决策。
第五，`smgrwrite()` 不是 fsync。
普通 `FlushBuffer()` 通过 `skipFsync=false` 让 `mdwritev()` 登记 dirty segment。
checkpointer 后续通过 `sync.c` 的 pending fsync table 处理 durable 边界。
`smgrregistersync()` 用于 skipFsync 写之后补登记，不是普通 dirty victim flush 的主调用点。
第六，`WritebackContext` 只是性能 hint。
它排序、合并并发出 OS writeback 请求，降低后续 fsync 压力，但不保证持久化。
第七，状态清理分两段：`FlushBuffer()` 清 `BM_DIRTY / BM_IO_IN_PROGRESS`；`InvalidateVictimBuffer()` 清 `BM_TAG_VALID / BM_VALID / tag / usage_count`；`BufferAlloc()` 再安装 new tag，但 `BM_VALID` 等待读入完成。
第八，cleanup lock 与 eviction pin 边界不同。
replacement 找当前未 pin 的 victim，找不到会 ERROR。
`LockBufferForCleanup()` 为了删除 page 内 item，必须等到只有自己的 pin。
可迁移的系统规律：缓存淘汰中的“释放空间”通常不是单一资源回收，而是把 identity、dirty state、I/O ownership、durability obligation、concurrent readers 和后台 flush pipeline 逐层解耦。
在 PostgreSQL 中，`BufferTag / BM_TAG_VALID` 解决 identity，`BM_VALID` 解决内容可读性，`BM_DIRTY` 解决写回需求，page LSN + `XLogFlush()` 解决 WAL-before-data，`BM_IO_IN_PROGRESS` 解决 I/O ownership，`RegisterSyncRequest` 解决 checkpoint fsync obligation，`WritebackContext` 解决 OS writeback scheduling，pin / cleanup lock 解决页面内部指针和物理删除边界。
能把这些层次分开，才能解释为什么同一次 shared buffer miss 有时只是一次读，有时会变成 WAL flush、数据文件 write、fsync request、writeback 和 retry loop 的组合成本。
