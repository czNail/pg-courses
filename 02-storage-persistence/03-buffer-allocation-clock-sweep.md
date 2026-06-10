# PostgreSQL Buffer allocation、clock sweep 与 replacement strategy

## 课程定位

前置知识：已经知道 shared buffer lookup 通过 `BufferTag -> buf_id` mapping table 维护唯一 identity，也知道 lookup miss 后需要拿到一个可复用的 buffer slot。

本节唯一主问题：

```text
当目标 block 不在 shared buffers 中时，PostgreSQL 如何在 pin/refcount、usage_count、dirty 状态和 access strategy ring 之间选择一个可以复用的 victim buffer？
```

核心矛盾：

```text
buffer pool 必须尽快给 miss 找到 slot；
但它不能替换正在被 pin 的页，不能丢掉 dirty 页，不能让大顺序扫描冲掉整个 shared_buffers，也不能用一把大锁串行化所有 allocation。
```

本节只讲 miss 后的 allocation / replacement。

它不重新讲 mapping table 的完整 lookup 协议。

它不展开 WAL record 生成。

它不展开 page content lock 的完整 wait queue。

它只解释 victim 从“候选”到“可复用 slot”的状态推进。

版本边界也要先说明。

本课基于当前 master 基线。

旧版本中的 content lock 实现、AIO 回调和部分统计入口可能不同。

但 refcount 硬门槛、usage_count second chance、dirty victim 写出、old tag invalidation 这条 replacement 主线是稳定的。

学完后应能判断：

- 为什么 PostgreSQL 使用 clock sweep 而不是精确 LRU。
- 为什么 `refcount` / pin 是 replacement 的硬门槛。
- 为什么 `usage_count` 只是 second chance，不是 pin。
- 为什么 dirty victim 的处理不在 `StrategyGetBuffer()` 中完成。
- 为什么 bulkread / bulkwrite / vacuum 使用 backend-private ring。
- 为什么 ring 不是独立缓存池。
- 为什么 all buffers pinned 时选择 ERROR 而不是无限等待。
- 为什么 victim 被选中后仍可能被放弃。

源码基线：

```text
/home/nail/postgres-lab
branch: master
commit: bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8
```

核心源码锚点：

| 顺序 | 文件 | 本节读什么 |
| --- | --- | --- |
| 1 | `src/include/storage/buf_internals.h` | `BufferDesc.state` bit layout、`BUF_STATE_GET_REFCOUNT()`、`BUF_STATE_GET_USAGECOUNT()`、`BM_MAX_USAGE_COUNT`、`BM_LOCKED`、`BM_DIRTY`、`BM_VALID`、`BM_TAG_VALID`。 |
| 2 | `src/backend/storage/buffer/freelist.c` | `StrategyControl`、`ClockSweepTick()`、`StrategyGetBuffer()`、`GetBufferFromRing()`、`AddBufferToRing()`、`StrategyRejectBuffer()`。 |
| 3 | `src/backend/storage/buffer/bufmgr.c` | `BufferAlloc()`、`GetVictimBuffer()`、`InvalidateVictimBuffer()`、`FlushBuffer()`、`UnlockReleaseBuffer()`。 |
| 4 | `src/include/storage/bufmgr.h` | `BufferAccessStrategyType`、`BAS_NORMAL`、`BAS_BULKREAD`、`BAS_BULKWRITE`、`BAS_VACUUM`。 |
| 5 | `src/backend/storage/buffer/localbuf.c` | local buffer 的简化 victim selection，对照哪些复杂性来自 shared 并发。 |

本节主链路：

```text
PinBufferForBlock()
  -> BufferAlloc()
  -> GetVictimBuffer()
  -> StrategyGetBuffer()
  -> optional dirty FlushBuffer()
  -> InvalidateVictimBuffer()
  -> BufferAlloc() installs new BufferTag
  -> later read I/O sets BM_VALID
```

一句话运行模型：

```text
PostgreSQL 用 backend-private ring 限制特殊访问模式污染，用全局 atomic clock hand 低成本扫描 BufferDesc，用 refcount 判断能否替换，用 usage_count 近似热度，用 dirty/writeback 路径守住 WAL-before-data 和 I/O 平滑边界。
```

本节最重要的区分是：

```text
candidate selection 不等于 eviction 完成；
pin victim 不等于 old tag 已删除；
dirty write 不等于 checkpoint/fsync；
ring reuse 不等于独立缓存池。
```

## 1. 问题：miss 后为什么不能直接拿一个 slot

lookup miss 只说明目标 `BufferTag` 当前不在 mapping table 中。

它没有告诉我们哪个 buffer slot 可以复用。

一个 slot 能否复用，至少要回答五个问题。

第一，有没有 backend 正在 pin 它？

如果 shared refcount 非零，slot 不能被替换。

持有 pin 的 backend 可能正在读 page contents、持有 page 内指针、等待 I/O，或者只是还没释放 buffer。

第二，它是否最近被使用？

如果 `usage_count > 0`，PostgreSQL 给它第二次机会。

clock sweep 递减 usage_count，而不是立即复用。

第三，它是否 dirty？

dirty page 不能被直接覆盖。

它要先按持久化规则写出。

如果是 permanent relation，还可能需要先 flush WAL 到 page LSN。

第四，它是否可以安全拿到 content lock 写出？

dirty write 需要至少 share-exclusive content lock。

如果无条件等待，可能与其他 backend 的 page operation 形成 deadlock。

所以当前基线在 victim writeback 前使用 conditional content lock。

第五，它的 old tag 是否能从 mapping table 删除？

如果 victim 被别的 backend 在短窗口内 pin 或再次 dirty，old tag invalidation 要放弃并 retry。

这五个问题分散在多个函数里。

`StrategyGetBuffer()` 只回答前两个问题的一部分。

它找一个当前未 pin、usage_count 可接受的 candidate，并先 pin 住。

`GetVictimBuffer()` 继续处理 dirty、content lock、writeback、old tag invalidation。

`BufferAlloc()` 最后安装 new tag。

把这些函数合成一句“clock sweep 找页”会漏掉关键正确性边界。

本节的状态故事是：

```text
lookup miss
  -> 找 candidate
  -> candidate 被当前 backend pin 住
  -> 如果 dirty，尝试 content lock 并写出
  -> old tag 从 mapping table 删除
  -> descriptor flags/tag 清成可复用状态
  -> BufferAlloc() 把 new tag 装上
  -> 后续 I/O 填充 content
```

这个过程里，任何一步都可能放弃当前 candidate。

放弃不是异常崩溃。

它是 shared buffer replacement 在并发下的正常 retry。

## 2. 状态：refcount、usage_count、clock hand、ring

`BufferDesc.state` 是 replacement 的核心状态载体。

本基线中它是 64-bit 原子变量。

本节关注四类位：

```text
18 bits refcount
4 bits usage_count
12 bits flags
content lock bits
```

refcount 是硬门槛。

`BUF_STATE_GET_REFCOUNT(state) != 0` 表示有 backend 或 I/O subsystem 正在 pin buffer。

这种 buffer 不能被 replacement 复用。

pin 不保护 page content。

但 pin 保护 slot identity 和生命周期。

replacement 首先尊重的是这个生命周期。

usage_count 是近似热度。

`BM_MAX_USAGE_COUNT` 当前为 5。

源码注释说明，较大的 usage_count 能更接近 LRU，但 clock sweep 可能需要更多完整循环才能找到 victim。

所以 PostgreSQL 接受一个小计数器。

它不是为了精确记录最近访问顺序。

它是为了在 hot path 上用很低成本给热页第二次机会。

flags 中，本节最重要的是：

- `BM_LOCKED`：buffer header lock 正被持有，CAS path 要等待 header unlock。
- `BM_VALID`：page content 有效。
- `BM_DIRTY`：page 需要写出。
- `BM_TAG_VALID`：slot 当前有 mapping table identity。
- `BM_IO_IN_PROGRESS`：I/O 正在进行。

这些 flags 参与 replacement 的不同阶段。

`StrategyGetBuffer()` 不处理所有 flags。

它主要看 refcount、usage_count、`BM_LOCKED`。

dirty / valid / tag valid 在 `GetVictimBuffer()` 和 `InvalidateVictimBuffer()` 继续处理。

全局 clock hand 在 `freelist.c` 的 `StrategyControl` 中。

关键字段是：

- `nextVictimBuffer`：atomic clock hand。
- `completePasses`：clock sweep 完整循环次数。
- `numBufferAllocs`：近期 allocation 计数，供 bgwriter 估算消耗速度。
- `bgwprocno`：需要被唤醒的 bgwriter proc number。

`ClockSweepTick()` 用 atomic fetch-add 推进 `nextVictimBuffer`。

多个 backend 同时推进时，返回的 buffer id 可能看起来不是严格顺序。

这没破坏算法。

clock sweep 要的是整体推进，而不是每个 backend 的严格公平。

只有 wraparound 和统计一致性需要 `buffer_strategy_lock` spinlock。

backend-private ring 是另一个状态层。

`BufferAccessStrategyData` 包含：

- `btype`：策略类型。
- `nbuffers`：ring 元素数。
- `current`：最近使用的 ring slot。
- `buffers[]`：保存 `Buffer` 编号的数组。

ring 是 backend-private。

它不拥有 buffer。

它不隔离别的 backend。

它只是记录“这个 backend 的某类访问可以优先尝试复用哪些 shared buffer”。

ring 中的 buffer 仍然是 shared buffers 的普通成员。

它们仍可能被别人 pin、dirty、提高 usage_count、或被全局 clock sweep 扫到。

`BAS_NORMAL` 通常返回 NULL strategy。

普通访问不使用 ring。

`BAS_BULKREAD`、`BAS_BULKWRITE`、`BAS_VACUUM` 使用 ring，以限制大 scan 或 bulk operation 对整个 shared_buffers 的污染。

当前源码中 ring 大小由策略类型和 shared_buffers 上限共同决定，并 capped 到 `NBuffers / 8`。

这不是为 bulk operation 建一个独立 cache pool。

它只是把该 backend 的候选复用范围限制在一个小环里。

## 3. 主流程：从 lookup miss 到 new tag 安装

主流程从 `BufferAlloc()` 的第一次 lookup miss 之后开始。

`BufferAlloc()` 已经知道目标 tag 当前不在 mapping table 中。

它释放 shared mapping partition lock。

然后调用：

```text
victim_buffer = GetVictimBuffer(strategy, io_context)
```

### StrategyGetBuffer：选 candidate 并 pin

`GetVictimBuffer()` 内部先确保当前 backend 有空间记录 pin：

```text
ReservePrivateRefCountEntry()
ResourceOwnerEnlarge(CurrentResourceOwner)
```

然后调用 `StrategyGetBuffer()`。

如果传入了 nondefault strategy，`StrategyGetBuffer()` 先尝试 `GetBufferFromRing()`。

ring slot 为空时返回 NULL。

ring slot 指向的 buffer 如果 pinned，不能用。

usage_count 大于 1，说明可能被别人近期使用，也不能直接复用。

如果 ring candidate 可用，它会通过 CAS 增加 refcount，把 buffer pin 住，并 `TrackNewBufferPin()`。

如果 ring 不能提供 candidate，就进入全局 clock sweep。

clock sweep 的核心循环是：

```text
buf = GetBufferDescriptor(ClockSweepTick())
read buf->state
if refcount != 0:
    trycounter--
    continue
if BM_LOCKED:
    WaitBufHdrUnlocked()
    retry same buffer state
if usage_count != 0:
    usage_count--
    trycounter = NBuffers
    continue
else:
    refcount++
    TrackNewBufferPin()
    return candidate
```

这里有几个重要细节。

refcount 非零时，clock sweep 不修改 usage_count。

因为 pinned buffer 正在使用。

它不是 replacement 的候选。

usage_count 非零且 refcount 为零时，clock sweep 递减 usage_count。

这次递减是一种全局老化。

成功递减后 `trycounter` 重置为 `NBuffers`。

这表示 sweep 做出了进展，不应把“所有 buffer 都 pinned”误判成错误。

只有扫描一整轮都没有任何 state change，才认为没有 unpinned buffers available。

找到 usage_count 为 0 且 refcount 为 0 的 buffer 后，`StrategyGetBuffer()` 不是返回裸 candidate。

它先用 CAS 增加 refcount。

也就是说，返回时 candidate 已经 pinned，并被当前 backend 的 ResourceOwner 记住。

这个 pin 很关键。

它防止别的 backend 在后续 dirty writeback / old tag invalidation 前把 candidate 换走。

### GetVictimBuffer：处理 dirty、writeback、old tag

`StrategyGetBuffer()` 返回后，`GetVictimBuffer()` 先做：

```text
CheckBufferIsPinnedOnce(buf)
```

当前 backend 不应该已经对这个 victim 有其他 pin。

如果 candidate state 中有 `BM_DIRTY`，进入 dirty victim 路径。

dirty victim 必须满足：

```text
BM_TAG_VALID
BM_VALID
```

一个 dirty page 没有 valid content 是不合理的。

写出 dirty page 前，`GetVictimBuffer()` 要拿 content lock。

它使用：

```text
BufferLockConditional(buf, buf_hdr, BUFFER_LOCK_SHARE_EXCLUSIVE)
```

如果拿不到，释放 pin，回到 `again` 重新找 victim。

它不阻塞等待。

原因是 dirty victim 写出发生在 replacement path 上。

无条件等待 content lock 可能与其他 page operation 形成死锁。

拿到 content lock 后，nondefault strategy 还可能拒绝这个 buffer。

`StrategyRejectBuffer()` 主要服务 `BAS_BULKREAD`。

如果该 dirty buffer 来自 ring，并且写出会要求 WAL flush，bulkread 可以把它从 ring 中清掉，要求选择另一个 victim。

这避免一个顺序读 workload 被迫为 ring 中的 dirty permanent page 承担 WAL flush latency。

如果没有拒绝，`GetVictimBuffer()` 调用 `FlushBuffer()`。

`FlushBuffer()` 完成 page write。

随后释放 content lock，并把 tag 交给 backend writeback context。

这里要区分：

dirty page write 不等于 checkpoint complete。

也不等于 relation file 已 fsync。

它只是 replacement 要覆盖 page 前必须完成的数据写出步骤。

dirty 处理后，`GetVictimBuffer()` 检查 `BM_VALID` 并记录 evict / reuse stats。

然后如果 `BM_TAG_VALID` 仍在，调用 `InvalidateVictimBuffer()` 删除 old tag。

`InvalidateVictimBuffer()` 的职责是把 victim 从“旧 identity 的 pinned page”变成“无 tag、无 valid、无 dirty、仍由当前 backend pin 住的 reusable slot”。

它读取 old tag。

因为当前 backend pinned victim，所以读取 tag 是安全的。

它按 old tag 计算 old hash 和 old partition lock。

然后拿 old partition exclusive lock，并锁 buffer header。

它重新检查：

```text
BM_TAG_VALID
refcount > 0
BufferDesc.tag 仍是 old tag
```

接着关键 recheck：

```text
if refcount != 1 or BM_DIRTY:
    release locks
    return false
```

如果别的 backend 在短窗口内 pin 了这个 buffer，refcount 就不再是 1。

如果别的 backend 又 dirtied 了它，不能删除 old tag。

`GetVictimBuffer()` 收到 false 后释放 pin 并 retry。

如果 recheck 通过，`InvalidateVictimBuffer()` 清 tag、清 flags / usage_count，从 mapping table 删除 old tag。

返回 true 时，victim 已可复用。

### BufferAlloc：安装 new tag

`GetVictimBuffer()` 返回后，`BufferAlloc()` 重新进入 mapping table 逻辑。

它拿 new tag partition exclusive lock。

调用 `BufTableInsert(newTag, newHash, victim_buf_id)`。

如果返回已有 `buf_id`，说明另一个 backend 已经插入同一目标 tag。

当前 backend 释放 victim pin，并转而 pin existing buffer。

如果 insert 成功，`BufferAlloc()` 锁 victim header，写：

```text
victim_buf_hdr->tag = newTag
set BM_TAG_VALID
set BUF_USAGECOUNT_ONE
set BM_PERMANENT if permanent or init fork
```

然后释放 locks，返回 pinned victim。

`foundPtr = false`。

原因是 new tag 已安装，但 content bytes 还没读入。

后续 read I/O 设置 `BM_VALID`。

这条链路把本节和前后两节连接起来：

```text
lookup miss 属于上一节；
victim selection 属于本节；
new tag 安装回到上一节的 mapping table 协议；
content lock 和 pin 的细边界进入下一节。
```

## 4. 边界：哪些机制各管一段

第一条边界：refcount 不是 usage_count。

refcount 是硬约束。

只要非零，replacement 不能复用该 slot。

usage_count 是软热度。

非零时 clock sweep 可以递减。

递减后将来仍可能复用。

把 usage_count 当 pin，会误以为热页永远不能被替换。

把 pin 当 usage_count，会误以为被访问过的页只是在 eviction 中“扣分”。

第二条边界：StrategyGetBuffer 不处理 dirty writeback。

它只返回一个 pinned candidate。

如果把 dirty handling 放进去，就要把 content lock、WAL flush 判断、writeback stats、old tag invalidation 都塞入 candidate selection。

这会让 clock sweep hot path 变重。

当前设计让 StrategyGetBuffer 保持窄职责。

第三条边界：dirty page 写出需要 content lock。

写出 dirty page 时，别人可能正在 compact page、设置 hint bits、修改 page header。

没有 content lock 写出可能得到不一致 bytes。

但 replacement path 不能无条件等待 content lock。

所以 `GetVictimBuffer()` 使用 conditional acquire，失败就重新选 victim。

第四条边界：old tag invalidation 必须按 old tag 分区。

victim slot 的 old tag 与 new tag 可能落在不同 mapping partition。

删除 old mapping entry 要拿 old partition exclusive lock。

插入 new mapping entry 要拿 new partition exclusive lock。

这解释了为什么 victim reuse 不是单个锁下的一步更新。

第五条边界：ring 不是 bypass。

ring candidate 仍是 shared buffer。

它仍受 shared refcount、usage_count、dirty、content lock、mapping table 的所有规则约束。

ring 只改变“先尝试哪些 candidate”。

它不改变 buffer ownership。

第六条边界：bgwriter 只接收压力信号。

`StrategyGetBuffer()` 会唤醒 `bgwprocno` 指向的 bgwriter。

它还增加 `numBufferAllocs`，让 bgwriter 估算前台 allocation rate。

但 bgwriter 不能替前台 guarantee 立即有 clean victim。

前台 miss 仍可能自己写 dirty victim。

第七条边界：checkpoint / fsync 不在本节完成。

`FlushBuffer()` 可以把 dirty page 写到 OS。

checkpoint / fsync / WAL-before-data 是更大的持久化协议。

本节只关心 replacement 要覆盖 page 前，不能丢掉 dirty bytes。

第八条边界：local buffer 的 replacement 没有 shared 并发。

`localbuf.c` 的 `GetLocalVictimBuffer()` 不需要 mapping partition lock。

local refcount 只在当前 backend 中变化。

dirty local temp page 写出到 temp storage，不走 WAL 和 checkpoint 的同一套规则。

但 local buffer 仍然有 usage_count、refcount、tag valid、content valid。

这说明算法简化来自可见性范围变小，而不是状态语义消失。

第九条边界：statistics 晚于 decision。

`pg_stat_io`、`EXPLAIN (BUFFERS)`、`pg_buffercache` 能看到结果或快照。

它们看不到每一次 clock tick、每一次 CAS 失败、每一次 candidate 被拒绝。

因此诊断 replacement 时，需要把 SQL 指标和 gdb/perf/源码路径结合。

## 5. 异常：压力下如何继续保持 correctness

异常路径一：所有 buffer 都 pinned。

`StrategyGetBuffer()` 用 `trycounter = NBuffers`。

遇到 pinned buffer 且没有做任何 usage_count 递减时，trycounter 下降。

如果下降到 0，源码报：

```text
no unpinned buffers available
```

它不无限等。

因为 replacement path 无法知道哪个 pin 会释放，也不应在全局 allocation hot path 中无界等待。

这个错误通常说明 backend 长期持有 pin、buffer pool 太小、或扩展 / access method 泄漏 pin。

异常路径二：`BM_LOCKED`。

如果 candidate 的 buffer header lock 正被持有，clock sweep 不能直接改 state。

它调用 `WaitBufHdrUnlocked()`。

这不是 content lock wait。

它等的是 descriptor header state 可被 CAS 操作。

header lock 等待通常很短。

如果这里持续成为热点，说明大量 backend 在同一组 descriptors 上竞争 flags、tag、refcount 或 waiter state。

异常路径三：usage_count 很高。

usage_count 最高为 `BM_MAX_USAGE_COUNT`。

clock sweep 每轮只递减一次。

高 usage_count 会让 buffer 多撑几轮。

这避免热页被一次冷 scan 迅速冲掉。

但它也意味着在 shared_buffers 很大、热点页很多时，miss 可能扫描更多 descriptors。

PostgreSQL 接受这种近似，而不是维护精确 LRU 链表。

原因是精确 LRU 需要每次 access 改共享结构，hot path contention 会更差。

异常路径四：dirty victim conditional lock 失败。

`BufferLockConditional()` 失败时，`GetVictimBuffer()` 释放 pin 并重来。

这可能增加 miss latency。

但它避免 replacement path 卡在可能死锁的 content lock wait 上。

如果 workload 频繁 page split、vacuum cleanup 或大量 dirty pages，candidate rejection 会变多。

异常路径五：bulkread ring 遇到 dirty permanent page。

如果 dirty page 来自 `BAS_BULKREAD` ring，且写出需要 WAL flush，`StrategyRejectBuffer()` 可以把该 ring slot 设为 `InvalidBuffer` 并返回 true。

调用者释放 buffer，重新找 victim。

这避免顺序读 workload 为自己 ring 中的 dirty page 承担不合适的 WAL flush。

它也说明 ring 是策略提示，不是强制 ownership。

异常路径六：old tag invalidation 失败。

`InvalidateVictimBuffer()` 发现 refcount 不为 1 或 `BM_DIRTY` 重新出现，会返回 false。

`GetVictimBuffer()` 释放 pin 并 retry。

这条 recheck 是 correctness 关键。

`StrategyGetBuffer()` 的观察和 `InvalidateVictimBuffer()` 的承诺之间存在窗口。

PostgreSQL 不假装窗口不存在。

它通过重新检查修正过期观察。

异常路径七：I/O failure。

dirty victim writeback 失败会沿 `FlushBuffer()` / `TerminateBufferIO()` / ERROR 处理。

`ResourceOwner` 负责释放当前 backend 持有的 pin 和 I/O owner。

失败不能让 buffer 永远停留在 `BM_IO_IN_PROGRESS`。

后续访问会看到 `BM_IO_ERROR` 或重新发起 I/O。

本节不展开所有 read/write error 分支。

但要记住 replacement 不只选择 candidate，也必须在 ERROR 下释放 candidate pin。

异常路径八：bgwriter 追不上前台 allocation。

`numBufferAllocs` 增长会被 bgwriter 看到。

bgwriter 可以提前写 dirty buffers，减少前台 miss 遇到 dirty victim 的概率。

但如果写入压力超过 bgwriter / storage 能力，前台仍会在 `GetVictimBuffer()` 中承担 writeback。

诊断上表现为 query latency 中夹杂 relation write / WAL flush / buffer content lock wait。

## 6. 诊断与实验

本节可观测的 runtime truth 是：

```text
miss latency 不只是磁盘读；它可能先花在 victim selection、usage_count 老化、dirty victim writeback、old tag invalidation 和 mapping table retry 上。
```

能直接观察：

- `pg_buffercache` 可以看 buffer id、usagecount、isdirty、pinning_backends 的快照。
- `EXPLAIN (ANALYZE, BUFFERS)` 可以看 shared hit/read/dirtied/written。
- `pg_stat_io` 可以按 context / object / operation 看 read、write、evict、reuse 等累计统计。
- gdb 可以在 `StrategyGetBuffer()`、`GetVictimBuffer()`、`InvalidateVictimBuffer()`、`FlushBuffer()` 断点观察 candidate 状态。
- perf / flamegraph 可以看 buffer allocation hot path 是否集中在 atomic CAS、LWLock、writeback 或 smgr I/O。

只能推断：

- 每次 clock hand 走过哪些 buffer。
- 每个 candidate 被拒绝的完整历史。
- ring 中 buffer 被其他 backend 触摸的瞬时过程。
- dirty victim 是否因为 WAL flush 被策略拒绝。

几乎不可见：

- backend-private ring 的完整数组内容。
- 单次 CAS retry 的细粒度原因。
- `trycounter` 的每次变化。

实验 1：观察 usage_count 老化。

目标：把 `usage_count` 与 pin 区分开。

准备 extension：

```sql
CREATE EXTENSION IF NOT EXISTS pg_buffercache;
CREATE TABLE bm_sweep_demo AS SELECT generate_series(1, 200000) AS id;
```

操作：

```sql
SELECT count(*) FROM bm_sweep_demo;

SELECT usagecount, count(*)
FROM pg_buffercache
WHERE relfilenode = pg_relation_filenode('bm_sweep_demo')
GROUP BY usagecount
ORDER BY usagecount;
```

继续制造大 scan 或重启实例后重复观察。

解释：

usage_count 会被访问提升，也会被 clock sweep 递减。

它不是“正在使用”的证据。

pinning_backends 才更接近“当前被 pin”的快照，但也只是观察时刻。

实验 2：用 gdb 跟踪 `StrategyGetBuffer()`。

目标：确认 candidate 返回时已经 pinned。

断点：

```text
b StrategyGetBuffer
b ClockSweepTick
b GetVictimBuffer
```

观察点：

- `ClockSweepTick()` 返回的 victim id。
- candidate state 中 refcount / usage_count。
- CAS 成功后 refcount 增加。
- `TrackNewBufferPin()` 被调用。

解释：

`StrategyGetBuffer()` 不是“给一个建议”。

它返回的是当前 backend 已经 pin 住的 candidate。

但 candidate 还没有完成 dirty writeback 和 old tag invalidation。

实验 3：制造 dirty victim。

目标：观察 miss path 先处理 dirty victim，再安装 new tag。

SQL：

```sql
CREATE TABLE bm_dirty_demo AS SELECT generate_series(1, 200000) AS id;
UPDATE bm_dirty_demo SET id = id + 1 WHERE id % 10 = 0;
```

在较小 `shared_buffers` 测试实例上，继续扫描超过 buffer pool 的数据。

断点：

```text
b GetVictimBuffer
b BufferLockConditional
b FlushBuffer
b InvalidateVictimBuffer
```

观察点：

- dirty candidate 是否进入 `FlushBuffer()`。
- conditional content lock 是否失败。
- writeback 后是否调用 `InvalidateVictimBuffer()`。
- old tag 删除后 `BufferAlloc()` 是否安装 new tag。

解释：

miss latency 可能包含 writeback。

如果只看 shared read，很容易漏掉 replacement 前置成本。

常见误区：

- 把 clock sweep 说成精确 LRU。
- 认为 usage_count 越高越“不能替换”。
- 认为 ring 是单独 cache。
- 认为 dirty victim 会在 `StrategyGetBuffer()` 中写出。
- 认为 victim 被 pin 后就已经从 mapping table 删除。
- 认为 bgwriter 能保证前台永远不写 dirty victim。
- 认为 no unpinned buffers 是磁盘慢，而不是 pin/lifetime 问题。

诊断时可以按状态推进问：

```text
miss 比例是否高？
candidate 是否被 refcount 挡住？
usage_count 是否导致多轮老化？
dirty victim 是否把 allocation 变成 writeback？
ring 是否因 pinned / dirty candidate 失效？
old tag invalidation 是否被 concurrent pin / dirty 打断？
new tag insert 是否 collision？
```

这些问题共同解释的是“miss 到可复用 slot”的时间。

## 7. 讨论题

1. 为什么 `StrategyGetBuffer()` 只负责 candidate selection，而不负责完整 eviction？
2. `usage_count`、pin/refcount 和 access strategy ring 分别保护什么运行时目标？
3. dirty victim 为什么会把一个 buffer miss 放大成 WAL flush、data write 和 fsync queue 压力？
4. bulk read 为什么宁可放弃某些 ring victim，也不把读路径拖进高成本 writeback？
5. 哪些现象能通过 `pg_stat_io` 或 wait event 观察，哪些必须靠断点看 `BufferDesc.state`？

## 8. 本节小结

本节核心链路是：

```text
BufferAlloc() miss
  -> GetVictimBuffer()
  -> StrategyGetBuffer()
  -> candidate pinned
  -> optional dirty writeback
  -> InvalidateVictimBuffer()
  -> reusable slot
  -> BufferAlloc() installs new tag
```

核心状态和边界是：

- refcount / pin 是 replacement 的硬门槛。
- usage_count 是 clock sweep 的近似热度。
- `StrategyControl.nextVictimBuffer` 是全局推进点。
- `BufferAccessStrategyData` 是 backend-private ring。
- dirty / valid / tag valid 在 `GetVictimBuffer()` 和 `InvalidateVictimBuffer()` 中处理。
- mapping table insert / delete 仍受上一节的 partition lock 协议约束。

ownership 是：

```text
StrategyGetBuffer() 返回前先 pin candidate；
ResourceOwner 记录这个 pin；
后续任何 retry 都必须释放 candidate pin；
成功返回给 BufferAlloc() 时，slot 仍由当前 backend pin 住。
```

异常路径的共同模式是：

```text
先用低成本观察找 candidate；
再在更强边界下 recheck；
如果观察过期，就释放 pin 并 retry。
```

能观测的是 shared buffer 快照、I/O 统计、query buffer 统计、wait event、gdb/perf。

看不到的是每一次 clock tick 和 candidate 被拒绝的完整历史。

可迁移规律：

```text
共享缓存 replacement 的核心不是找到“最旧”的对象；
而是在高并发和有限 metadata 成本下，快速证明某个对象现在可以安全失去旧身份。
```

下一节继续拆一个经常被误读的边界：

为什么 victim selection 依赖 pin/refcount，而 page 内容读写又必须依赖 content lock？
