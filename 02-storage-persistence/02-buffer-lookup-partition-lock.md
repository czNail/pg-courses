# PostgreSQL Buffer lookup、partition lock 与 mapping table

## 课程定位

前置知识：已经读过上一节，知道 `BufferTag` 是磁盘页身份，`BufferDesc` 是 buffer pool slot，pin 能让 slot identity 在使用期间不被替换。

本节唯一主问题：

```text
同一个磁盘 block 在 shared buffers 中最多只能有一个 buffer slot，PostgreSQL 如何在高并发 lookup、miss、victim reuse 和 I/O 等待之间维护这个唯一性？
```

核心矛盾：每次 page access 都要快速从 `BufferTag` 找到 `BufferDesc`；但所有 backend 又必须共同维护同一张 shared mapping table，不能让一个 block 被两个 slot 同时代表，也不能用全局大锁把 hot path 完全串行化。

一句话运行模型：

```text
ReadBuffer_common() 进入 PinBufferForBlock()，先在 partition lock 下查 BufferTag -> buf_id；命中时先 pin 再释放锁，miss 时由 BufferAlloc() 选择并安装唯一 new tag，遇到并发插入则放弃自己 candidate 并重试。
```

学完后应能判断：mapping table 只解决 identity uniqueness；pin 解决 slot lifetime；`BM_VALID` 解决 page bytes 是否可用；content lock 解决 page contents 并发访问。

源码基线：本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`。

## 1. 本节在总主线中的位置

上一节建立了 buffer identity 的分层，本节进入第一个并发共享点：多个 backend 如何从同一个 `BufferTag` 找到同一个 slot。它还不讲完整 replacement，也不讲 dirty writeback 和 content lock，只讲 shared mapping table 如何维护唯一性。

后续 clock sweep、read I/O 和 pin/content lock 都会回到这个结论：只要 mapping table 里有某个 `BufferTag`，系统就必须让后来者看见同一个 `buf_id`，而不是再创建第二个缓存副本。

## 2. 核心矛盾与一句话运行模型

`BufTableLookup()` 命中只是 mapping hit，不等于 page 已可读，也不等于调用者已经安全使用这个 slot。PostgreSQL 的做法是在 partition lock 保护下把 mapping entry 和 descriptor tag 作为一个 identity 边界处理，随后通过 pin、I/O 状态和 content lock 分别处理 lifetime、readiness 和 page bytes 并发。

本节要守住三个不变量：

```text
对任意有效 BufferTag，同一时刻 shared mapping table 中最多有一个 buf_id；
只要 mapping table 中存在该 tag，承载它的 BufferDesc 应该处于 BM_TAG_VALID；
但 BM_TAG_VALID 不承诺 page bytes 已经 BM_VALID。
```

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节读什么 |
| --- | --- | --- |
| 1 | `src/backend/storage/buffer/bufmgr.c` | `ReadBuffer_common()`、`PinBufferForBlock()`、`BufferAlloc()`、`InvalidateVictimBuffer()`、`StartSharedBufferIO()`。 |
| 2 | `src/backend/storage/buffer/buf_table.c` | `BufferLookupEnt`、`BufTableHashCode()`、`BufTableLookup()`、`BufTableInsert()`、`BufTableDelete()`。 |
| 3 | `src/include/storage/buf_internals.h` | `BufferTag`、`BufMappingPartitionLock()`、`BM_TAG_VALID`、`BM_VALID`、`BufferDesc.tag`。 |
| 4 | `src/backend/storage/buffer/freelist.c` | `StrategyGetBuffer()` 只作为 miss 后找 candidate 的边界，不展开 clock sweep。 |
| 5 | `src/backend/storage/buffer/localbuf.c` | local buffer 的无 shared partition lock 对照。 |

最短调用链：

```text
ReadBufferExtended()
  -> ReadBuffer_common()
  -> StartReadBuffer()
  -> StartReadBuffersImpl()
  -> PinBufferForBlock()
  -> BufferAlloc()
  -> BufTableLookup()/BufTableInsert()/BufTableDelete()
```

## 4. 入口现象：mapping hit 不是完整 buffer hit

普通读路径最终需要回答一个问题：

```text
目标 BufferTag 是否已经在 shared buffers 中？
```

如果在，就要找到对应 slot。

如果不在，就要分配一个 slot，并把 new tag 安装进去。

这里的“在”不是 SQL 层的 cache hit。

它只是 mapping table hit。

mapping table hit 表示：

```text
BufferTag -> buf_id 存在。
```

它不表示：

- page bytes 已经读完。
- caller 已经持有 pin。
- caller 已经持有 content lock。
- 该 buffer 没有 I/O error。
- 该 page 内容不会被其他 backend 修改。

源码把这些语义分到不同机制里。

mapping table 只解决 identity uniqueness。

pin 解决 slot lifetime。

`BM_VALID` 解决 content readiness。

content lock 解决 page bytes 并发访问。

`BufferAlloc()` 的注释直接给出它的职责：

```text
Handles lookup of a shared buffer.
If no buffer exists already, selects a replacement victim and evicts the old page,
but does NOT read in new page.
```

所以 `BufferAlloc()` 返回时，目标 buffer 已经 pinned，并且已经标记为 holding desired page。

但如果 `*foundPtr = false`，page contents 还没读入。

这就是本节的关键 runtime 现象：

一个 backend 可以查到一个 tag，pin 到同一个 slot，但还要等待另一个 backend 完成 I/O 才能读取 content。

如果把 mapping hit 误认为完整 buffer hit，就会误读 `BM_TAG_VALID`、`BM_VALID`、`StartSharedBufferIO()` 的关系。

本节沿一条 shared read 的时间线展开：

```text
构造 BufferTag
  -> 计算 hash code
  -> 找 partition lock
  -> shared lookup
  -> hit: pin 后释放 lock
  -> miss: 释放 lock，找 victim
  -> exclusive insert new tag
  -> collision: 放弃 victim，pin 已有 buffer
  -> success: 写 BufferDesc.tag，设置 BM_TAG_VALID
  -> 后续 I/O 设置 BM_VALID
```

这条线比“函数清单”更重要。

## 5. 状态：mapping table、partition lock 和 descriptor tag

`BufferTag` 是 hash key。

它包含 tablespace、database、relation file number、fork number、block number。

同一 relation 的 main fork 和 vm fork 是不同 tag。

同一 relation fork 的不同 block 是不同 tag。

`BufTableHashCode()` 对 `BufferTag` 算 hash。

调用者需要这个 hash code 有两个原因。

第一，hash lookup / insert / delete 要用它。

第二，调用者要用它计算应该拿哪把 mapping partition lock。

`BufMappingPartitionLock(hashcode)` 在 `buf_internals.h` 中把 hash code 映射到 `MainLWLockArray` 中的 buffer mapping lock。

这些 locks 按 hash partition 切分。

源码中的 `NUM_BUFFER_PARTITIONS` 必须是 2 的幂。

这不是按 relation 分锁。

也不是按 buffer id 分锁。

它按 hash partition 保护 shared lookup table 的一段 key space。

`buf_table.c` 的 entry 是：

```text
BufferLookupEnt
  key: BufferTag
  id: int buf_id
```

它只存 `buf_id`。

不存 `BufferDesc *`。

不存 `Buffer` handle。

不存 page pointer。

这让 hash table entry 可以小而稳定。

`buf_id` 再通过 `GetBufferDescriptor(buf_id)` 找 descriptor。

`BufferDesc.tag` 必须与 hash key 保持一致。

但这种一致性不是 hash table 自动维护的。

它由调用者在持锁协议中维护。

`buf_table.c` 顶部注释说明：

```text
The routines in this file do no locking of their own.
The caller must hold a suitable lock on the appropriate BufMappingLock.
```

原因是修改 mapping table 时，调用者通常还要调整 buffer header。

例如插入新 tag 后，必须写 `victim_buf_hdr->tag` 并设置 `BM_TAG_VALID`。

删除旧 tag 时，必须清 `BufferDesc.tag`、清 flags / usage_count，再从 hash table 删除。

如果把锁封装进 `BufTableInsert()` 内部，caller 无法把 hash entry 和 descriptor state 放在同一个协议里提交。

本节涉及三类锁，但它们保护的对象不同。

mapping partition lock 保护 hash table partition。

buffer header lock 保护 `BufferDesc` header 状态，例如 tag、flags、waiter。

content lock 保护 page contents。

本节主要关心前两者。

content lock 只在 dirty victim / I/O 边界短暂出现。

`BM_TAG_VALID` 是 mapping table 与 descriptor 的连接点。

`buf_internals.h` 注释说：

```text
BM_TAG_VALID essentially means that there is a buffer hashtable entry associated with the buffer's tag.
```

注意这个 “essentially”。

它不是 public API contract。

它是在当前 buffer manager 内部协议下形成的状态约定。

因此诊断时要同时看：

```text
BufferDesc.tag
BM_TAG_VALID
mapping table entry
持有的 mapping partition lock
是否有 pin 保护 tag 不变
```

`BM_VALID` 则是另一个维度。

`BM_VALID` 表示 page data valid。

`BufferAlloc()` 成功安装 new tag 后，只设置 `BM_TAG_VALID`。

读 I/O 完成后才由 `TerminateBufferIO()` 设置 `BM_VALID`。

这说明 lookup 层与 I/O 层之间有一个刻意保留的中间状态。

它允许多个 backend 围绕一个 identity 汇合。

它也避免未读完的 content 被当成有效 page。

## 6. 主流程：hit、miss、collision

shared read 的入口不在 `buf_table.c`。

`buf_table.c` 只是 hash table helper。

入口在 `bufmgr.c`。

普通同步读大致走：

```text
ReadBufferExtended()
  -> ReadBuffer_common()
  -> StartReadBuffer()
  -> StartReadBuffersImpl()
  -> PinBufferForBlock()
  -> BufferAlloc()
```

`ReadBuffer_common()` 先排除 other-session temp relation。

这个错误路径和本节有关。

other-session temp relation 的 buffer identity 在另一个 backend 的 local buffer manager 中，不在 shared mapping table 中。

如果当前 backend 继续走 shared lookup，结果不是“找不到”，而是语义上不可访问。

然后 `ReadBuffer_common()` 排除 `P_NEW` relation extension。

本节关注已有 block 的 ordinary lookup。

`PinBufferForBlock()` 根据 persistence 决定 shared/local。

shared path 进入 `BufferAlloc()`。

### hit path

`BufferAlloc()` 先 `ResourceOwnerEnlarge()`，再 `ReservePrivateRefCountEntry()`。

这一步必须在拿锁前做。

否则 pin 成功后没有空间记入 ResourceOwner，会让 cleanup 语义变复杂。

然后它创建 `newTag`。

然后计算 `newHash` 和 `newPartitionLock`。

hit path 的顺序是：

```text
LWLockAcquire(newPartitionLock, LW_SHARED)
  -> existing_buf_id = BufTableLookup(newTag, newHash)
  -> buf = GetBufferDescriptor(existing_buf_id)
  -> valid = PinBuffer(buf, strategy, false)
  -> LWLockRelease(newPartitionLock)
  -> foundPtr = valid ? true : false
  -> return pinned buf
```

关键是 pin 在释放 partition lock 之前发生。

mapping lock 下 lookup 只说明 entry 当前存在。

只要还没 pin，slot 仍可能被别人 invalidation 后复用。

pin 成功后，`BufferDesc.tag` 不会在 caller 脚下改变。

这时才能释放 mapping lock。

`PinBuffer()` 返回 `BM_VALID`。

如果 valid 为 false，`BufferAlloc()` 把 `*foundPtr = false`。

这意味着：

```text
lookup hit
  -> identity hit
  -> content miss 或 content not ready
```

这种状态正常存在。

典型原因是另一个 backend 正在读入页面。

### miss path

如果第一次 lookup 返回 -1，`BufferAlloc()` 释放 shared mapping lock。

然后调用 `GetVictimBuffer()`。

这一步可能很贵。

它可能推进 clock sweep。

它可能遇到 dirty victim 并写回。

它可能因为 content lock conditional acquire 失败而放弃候选。

它可能在 invalidation old tag 时发现 concurrent pin / dirty 而 retry。

这些都不是 mapping table lookup 本身该承担的工作。

找 victim 成功后，当前 backend 持有 victim pin。

这个 pin 防止 victim slot 被别人换掉。

随后 `BufferAlloc()` 重新拿 new tag 对应的 partition lock，这次是 exclusive：

```text
LWLockAcquire(newPartitionLock, LW_EXCLUSIVE)
  -> existing_buf_id = BufTableInsert(newTag, newHash, victim_buf_id)
```

`BufTableInsert()` 有两个返回形态。

返回 `-1` 表示当前 backend 插入成功。

返回已有 `buf_id` 表示别人已经插入同一个 tag。

第二种不是错误。

它正是 miss path 的并发 race。

### collision path

collision 的状态是：

```text
backend A lookup miss
backend A 释放 shared mapping lock
backend B 插入同一 newTag
backend A 找到 victim
backend A exclusive insert 时发现已有 entry
```

源码处理是：

```text
UnpinBuffer(victim_buf_hdr)
  -> existing_buf_hdr = GetBufferDescriptor(existing_buf_id)
  -> valid = PinBuffer(existing_buf_hdr, strategy, false)
  -> release newPartitionLock
  -> return existing buffer
```

这条路径保持了唯一 identity。

当前 backend 为 miss 做的 victim 工作被丢弃。

它不是浪费到需要报错的程度。

这是避免持 mapping lock 做慢操作的代价。

PostgreSQL 接受这个 race，是为了缩短 partition lock 的持有时间。

### successful insert path

如果 `BufTableInsert()` 返回 -1，当前 backend 抢到了 new tag。

然后还不能直接返回。

它需要写 descriptor：

```text
victim_buf_state = LockBufHdr(victim_buf_hdr)
assert refcount == 1
assert no BM_TAG_VALID / BM_VALID / BM_DIRTY / BM_IO_IN_PROGRESS
victim_buf_hdr->tag = newTag
set BM_TAG_VALID
set BUF_USAGECOUNT_ONE
set BM_PERMANENT if needed
UnlockBufHdrExt(...)
LWLockRelease(newPartitionLock)
foundPtr = false
```

注意这个顺序里，mapping table insert 发生在 `BufferDesc.tag` 写入之前。

但两者都在 new partition exclusive lock 持有期间完成。

其他 backend 不能在这段中间状态上做同一 partition 的 lookup。

释放 partition lock 时，hash entry、descriptor tag、`BM_TAG_VALID` 已经形成一致状态。

`foundPtr = false` 表示 content 未读入。

后续 read path 会调用 `StartSharedBufferIO()`。

如果另一个 backend 在 `BM_TAG_VALID` 后、`BM_VALID` 前进来，它会在 hit path pin 同一个 descriptor。

然后看到 valid false，并进入等待 / I/O 协调。

这就是 mapping table 的核心功能：

让并发 miss 在 identity 阶段汇合，而不是在 I/O 完成后才发现重复。

## 7. 边界：锁粒度和模块职责

第一条边界：partition lock 不是 relation lock。

buffer mapping locks 按 `BufferTag` hash 分区。

同一 relation 的不同 block 可能落在不同 partition。

不同 relation 的 block 也可能落在同一 partition。

这是一种 hash concurrency design。

它追求平均分散，不提供 relation 级隔离。

因此不能说“某个 relation 的 buffer lookup 被一把锁保护”。

第二条边界：partition lock 不是 buffer id lock。

victim invalidation 要按 old tag 计算 old partition。

new tag insert 要按 new tag 计算 new partition。

一个 buffer slot 被复用时，old tag 和 new tag 可能落在不同 partition。

这也是 `GetVictimBuffer()` 和 `BufferAlloc()` 分工复杂的原因之一。

第三条边界：`buf_table.c` 不做锁。

`BufTableLookup()` 要求 caller 至少持 shared lock。

`BufTableInsert()` 和 `BufTableDelete()` 要求 caller 持 exclusive lock。

这不是遗漏。

它让 caller 能把 mapping table 与 descriptor state 的更新放到正确顺序里。

第四条边界：hit path 不重新验证 tag。

在 `BufferAlloc()` hit path 中，lookup 在 shared partition lock 下返回 `existing_buf_id`。

随后在释放 partition lock 前 pin。

这个协议足以保证 slot 不会被替换。

所以 hit path 不需要像 `ReadRecentBuffer()` 那样 pin 后再比较 tag。

`ReadRecentBuffer()` 之所以要比较，是因为它拿到的是 hint，不是 mapping lock 下 lookup 的结果。

第五条边界：mapping lock 不保护 page contents。

mapping lock 保护 hash table key space。

它不保护 page header、tuple line pointer、LSN、visibility bits。

page contents 由 content lock 保护。

pin 保护 slot 不被替换。

三者不能互相替代。

第六条边界：`GetVictimBuffer()` 返回的 candidate 不等于 eviction 完成。

`StrategyGetBuffer()` 只能找一个当前看起来 unpinned 的 buffer，并把它 pin 住。

dirty writeback、old tag invalidation、mapping table delete 都在后续完成。

这意味着 victim selection 和 mapping table mutation 是两个阶段。

第七条边界：local buffer 没有 shared partition lock。

`LocalBufferAlloc()` 用 backend-local `LocalBufHash`。

没有其他 backend 并发插入同一 local tag。

所以 shared path 中正常的 `BufTableInsert()` collision，在 local path 中变成 corruption。

local 对照能帮助判断哪些复杂性来自 shared state。

第八条边界：lookup 成本不是单个函数成本。

一次 shared buffer lookup 的 hot path 包含：

```text
构造 BufferTag
  -> hash_any 计算 hash
  -> mapping partition LWLock acquire
  -> dynahash lookup
  -> descriptor pin CAS
  -> ResourceOwner / private refcount 记录
```

其中 hash table lookup 只是中间一步。

在低并发 workload 中，主要成本通常来自 hash、cache miss 和少量 atomic 操作。

在高并发 workload 中，成本会迁移到 mapping partition lock contention、buffer header CAS retry、I/O wait 或 content lock wait。

因此诊断 lookup 相关问题时，不能只问“hash table 快不快”。

要同时问：

```text
tag 分布是否集中到少数 partition？
backend 数是否放大 LWLock 排队？
miss 比例是否让大量 backend 进入 GetVictimBuffer()？
victim 是否频繁 dirty，导致 lookup miss 延伸成 writeback latency？
foundPtr=false 是否让多个 backend 等同一个 I/O？
```

这些成本变量跨过了本节的 mapping table 边界。

但它们都从 lookup 的状态分流开始。

`BufTableLookup()` hit 后，后续成本主要是 pin 和可能的 I/O wait。

`BufTableLookup()` miss 后，成本会传播到 replacement、writeback、WAL flush、mapping table insert 和 read I/O。

所以本节只讲 lookup，但诊断时要知道它会把压力转给谁。

与相邻模块的连接也有明确边界。

storage manager 提供 `SMgrRelation` 和 locator，帮助构造 `BufferTag`。

relation manager / relcache 提供 persistence 和 fork 信息，但 `BufferTag` 一旦形成，写回位置不能再依赖 relcache 可见性。

replacement strategy 在 miss 后提供 victim，但不拥有 new tag 的 mapping table entry。

I/O 层在 `BM_TAG_VALID` 后推进 `BM_IO_IN_PROGRESS` 和 `BM_VALID`。

statistics 层在 `TrackBufferHit()`、`pgstat_count_buffer_read()`、`pgstat_count_io_op()` 等位置记录结果，但它记录的是较晚阶段的现象，不是每一次 mapping table 状态转换。

这也是为什么 `EXPLAIN (BUFFERS)` 能告诉你 hit/read，却不能告诉你 hit 是 mapping hit、content valid hit，还是先 hit identity 再等待 I/O 完成。

第九条边界：后台进程看到的是同一组 shared state。

bgwriter 和 checkpointer 不通过 SQL 层访问 page。

它们会扫描 shared buffers，并依据 `BufferDesc.tag`、flags、dirty 状态和 checkpoint sort item 决定写回顺序。

因此 invalidation 时清 `BufferDesc.tag` 和 flags 的意义不只服务前台 lookup。

它也避免后台线性扫描把已经失效的 slot 误当成仍有合法 identity 的 buffer。

本节不展开 checkpoint。

但要记住 mapping table 与 descriptor tag 的一致性会被前台 backend、bgwriter、checkpointer、I/O completion path 共同依赖。

## 8. 异常：race、retry 和 failure 如何收尾

异常路径一：并发插入同一 tag。

这已经在 collision path 中出现。

它的处理原则是：

```text
谁已经插入 mapping table，谁拥有该 tag identity；
其他 backend 放弃自己的 victim，pin 已有 slot。
```

这里没有 ERROR。

因为 race 来自刻意释放 mapping lock 去做慢工作。

异常路径二：lookup hit 但 `BM_VALID` 为 false。

`PinBuffer()` 返回 false。

`BufferAlloc()` 把 `foundPtr` 改成 false。

调用者通过 `StartSharedBufferIO()` 协调 I/O。

如果 I/O 已经在进行，`StartSharedBufferIO()` 可以返回 `BUFFER_IO_IN_PROGRESS`。

如果 I/O 已经被别人完成，返回 `BUFFER_IO_ALREADY_DONE`。

如果当前 backend 成功设置 `BM_IO_IN_PROGRESS`，返回 `BUFFER_IO_READY_FOR_IO`。

这条路径说明：

mapping table 的职责到 identity 为止。

content readiness 由 I/O state 继续推进。

异常路径三：old tag delete 找不到 entry。

`BufTableDelete()` 删除不存在的 entry 会 `elog(ERROR, "shared buffer hash table corrupted")`。

这是严重内部不一致。

因为删除旧 tag 的 caller 应该在 exclusive old partition lock 下，且 buffer state 表示 old tag valid。

如果 hash table 中没有 entry，mapping table 与 descriptor 已经脱节。

异常路径四：victim 被选中后又被 pin 或 dirty。

`InvalidateVictimBuffer()` 在删除 old tag 前重新拿 old partition exclusive lock 和 buffer header lock。

它检查 refcount 是否仍为 1。

它检查 `BM_DIRTY` 是否没有重新出现。

如果有人在 `StrategyGetBuffer()` 后 pin 了这个 buffer，或者有人又 dirtied 它，函数返回 false。

`GetVictimBuffer()` 释放 pin 并重新选 victim。

这条 retry 保证：

```text
candidate selection 是观察；
invalidation 才是承诺。
```

异常路径五：dirty victim 需要 content lock。

dirty victim 在真正复用前必须写出。

`GetVictimBuffer()` 会对 dirty buffer 尝试 `BufferLockConditional(..., BUFFER_LOCK_SHARE_EXCLUSIVE)`。

它不能无条件等待。

因为另一个 backend 可能持有或等待相关 page lock，两个 backend 都在 split / cleanup 之类路径中相互等待，可能死锁。

如果 conditional lock 失败，当前 backend 放弃 victim 并重新选择。

这条异常路径属于下一节 replacement 的主体，但本节要知道它会推迟 old tag 删除。

异常路径六：no unpinned buffers。

`StrategyGetBuffer()` 扫描一轮没有任何 state change，说明所有 buffer 在观察时都 pinned。

源码选择 `elog(ERROR, "no unpinned buffers available")`。

它不无限等待。

因为无法知道哪个 pin 会释放，也不想让 allocation hot path 卡成无界等待。

这条错误会向上中断当前操作。

ResourceOwner 再释放本 backend 已经持有的 pin。

异常路径七：other-session temp relation。

`ReadBuffer_common()` 对 `RELATION_IS_OTHER_TEMP(rel)` 报错。

这不是 lookup miss。

这是访问边界错误。

因为 other session 的 local buffers 对当前 backend 不存在可验证的 shared identity。

## 9. 诊断与实验

本节可观测的 runtime truth 是：

```text
mapping table 先把并发 backend 汇合到同一个 BufferDesc，之后 BM_VALID / I/O path 再决定 page contents 是否已经可用。
```

能直接观察：

- gdb 断点能看到 `newTag`、`newHash`、`newPartitionLock`、`existing_buf_id`。
- `pg_buffercache` 能在较粗粒度上看到 relation / fork / block 与 buffer id 的当前对应。
- `pg_stat_activity` wait event 可以看到 buffer I/O、buffer content lock、LWLock 等等待。
- `EXPLAIN (ANALYZE, BUFFERS)` 能看到 query 粒度 hit/read，但看不到 mapping table hit 和 content valid 的区别。

只能推断：

- `BufTableInsert()` collision 很短，通常需要 gdb 条件断点或动态追踪。
- `BM_TAG_VALID=true, BM_VALID=false` 的窗口常常很短，需要断点放大。
- partition lock 的 hash 分布不能从 SQL 视图直接看到。

不要从一个 `shared hit` 指标直接推出“没有 I/O 协调”。

SQL 层的 hit/read 粒度晚于 mapping table 层。

实验 1：跟踪一次 lookup hit。

目标：确认 hit 后先 pin 再 release partition lock。

断点：

```text
b BufferAlloc
b BufTableLookup
b PinBuffer
```

SQL：

```sql
CREATE TABLE bm_lookup_demo(id int);
INSERT INTO bm_lookup_demo SELECT generate_series(1, 10000);
SELECT count(*) FROM bm_lookup_demo;
SELECT count(*) FROM bm_lookup_demo;
```

观察点：

- 第二次 scan 更容易进入 mapping hit。
- 在 `BufferAlloc()` 中记录 `existing_buf_id`。
- 确认 `PinBuffer()` 发生在 `LWLockRelease(newPartitionLock)` 之前。

解释：

这个顺序让 `Buffer` 从 hash entry 返回的 slot id 变成稳定 pinned slot。

实验 2：放大 miss 后 insert collision。

目标：理解第一次 miss 不是最终事实。

方法：

在两个 backend 中同时读取同一个尚未缓存的大表 block。

在 gdb 中对 `BufferAlloc()` 中 `BufTableInsert()` 前设置断点。

让 backend A 在第一次 miss 后暂停。

让 backend B 继续插入同一 tag。

再恢复 backend A。

观察点：

- backend A 的第一次 `BufTableLookup()` 返回 -1。
- backend A 后续 `BufTableInsert()` 返回已有 `buf_id`。
- backend A 调用 `UnpinBuffer(victim_buf_hdr)`。
- backend A 转而 `PinBuffer(existing_buf_hdr, ...)`。

解释：

这条路径证明 mapping table 的 owner 是 exclusive insert 的结果，不是第一次 lookup miss 的观察。

实验 3：观察 tag valid 与 content valid 分离。

目标：捕捉 `BM_TAG_VALID` 已设置但 `BM_VALID` 未设置的窗口。

断点：

```text
b BufferAlloc
b StartSharedBufferIO
b TerminateBufferIO
```

观察点：

- `BufferAlloc()` successful insert path 设置 `BM_TAG_VALID` 后返回 `foundPtr=false`。
- `StartSharedBufferIO()` 设置 `BM_IO_IN_PROGRESS`。
- `TerminateBufferIO()` 设置 `BM_VALID`。

解释：

这个实验把本节主线闭合：

mapping table 先保证唯一 identity，I/O 完成后才保证 content readiness。

常见误区：

- 把 `BufTableLookup()` hit 当成完整 page hit。
- 认为 `BufTableInsert()` collision 是错误。
- 认为 `BM_TAG_VALID` 等于 `BM_VALID`。
- 认为 mapping partition lock 保护 page contents。
- 认为 partition lock 是按 relation 或 buffer id 切分。
- 认为 victim 一旦被 `StrategyGetBuffer()` 返回就已经 eviction 完成。

诊断时最有用的问题顺序是：

```text
目标 tag 是什么？
hash code 落在哪个 mapping partition？
lookup 是 hit 还是 miss？
hit 后是否已经 pin？
foundPtr 是 true 还是 false？
如果 miss，exclusive insert 是否 collision？
如果要复用 old slot，old tag 是否已从 mapping table 删除？
```

这组问题按状态推进排序。

它比背诵函数参数更接近真实排障过程。

## 10. 讨论题

1. 为什么 mapping table hit 不能直接等价为 page 内容可读？
2. miss 后释放 partition lock 再找 victim，会引入什么 race？`BufTableInsert()` 怎样把它收回来？
3. 为什么 pin 必须在释放 mapping partition lock 前完成？
4. old tag 删除时为什么要重新检查 `refcount`、`BM_DIRTY` 和 tag？
5. 如果把 partition lock 粒度改成 relation 级或全局级，成本和正确性边界会怎样变化？

## 11. 本节小结

本节核心链路是：

```text
PinBufferForBlock()
  -> BufferAlloc()
  -> BufTableLookup()
  -> hit: PinBuffer() then release partition lock
  -> miss: GetVictimBuffer()
  -> BufTableInsert()
  -> collision or install new tag
  -> later StartSharedBufferIO()/TerminateBufferIO()
```

核心状态是：

- `BufferTag`：hash key，磁盘页身份。
- mapping table：`BufferTag -> buf_id`。
- partition lock：保护 hash table partition。
- `BufferDesc.tag`：slot 当前承载的 identity。
- `BM_TAG_VALID`：tag 与 mapping table entry 已建立。
- `BM_VALID`：page bytes 已经可用。
- pin：lookup 后到 caller 使用期间保护 slot identity。

ownership 和 cleanup 是：

```text
lookup 不持有长期 owner；
pin 由 PrivateRefCountEntry 和 ResourceOwner 管；
old tag 删除由持有 old partition exclusive lock 的 invalidation 路径完成；
I/O owner 由 ResourceOwner 在 ERROR 时清理。
```

异常路径的共同模式是：

```text
短锁保护 shared mapping；
慢操作放到锁外；
回来用 insert / recheck / retry 把过期观察修正成唯一事实。
```

能观测的是 SQL 层 hit/read、shared buffer 快照、wait event 和 gdb 中的短时状态。

看不到的是完整 mapping table 历史和每一次 partition lock 下的瞬时 race。

可迁移规律：

```text
高并发 shared cache 通常不会让 lookup lock 覆盖所有慢路径；
它会把“观察”和“承诺”拆开，再用 recheck、collision handling 和 retry 把唯一性补回来。
```

下一节从本节的 miss path 继续：

当 mapping table 没有目标 tag 时，PostgreSQL 到底如何选择一个可以安全复用的 victim buffer？
