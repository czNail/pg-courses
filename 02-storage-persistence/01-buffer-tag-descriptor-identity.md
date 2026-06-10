# PostgreSQL Buffer tag、descriptor 与 buffer identity

## 课程定位

前置知识：已经知道 PostgreSQL relation 会被拆成 relation fork 中的 block，也知道 shared buffers 是所有 backend 共同访问的 shared memory。

本节唯一主问题：

```text
一个 relation fork 的 block 如何在 shared/local buffer 中获得稳定身份，并在 slot 复用、I/O、pin、content lock 同时发生时不被认错？
```

核心矛盾：

```text
buffer pool slot 必须被不断复用，才能用有限内存缓存无限磁盘页；
但同一时刻，一个磁盘 block 又必须只有一个可被并发 backend 查到、pin 住、等待 I/O、访问内容的 buffer identity。
```

本节只回答 identity 的问题。

它不展开 partition lock 的竞争细节。

它不展开 clock sweep 的 victim 选择。

它不展开 content lock 的完整协议。

这些会在后续三节分别处理。

本节结束后，你应该能判断：

- `Buffer` 为什么只是 handle，不是磁盘页身份。
- `BufferTag` 为什么必须能脱离 relcache / catalog 定位磁盘页。
- `BufferDesc.buf_id`、`BufferDesc.tag`、`BufferBlocks` 为什么不能合成一个概念。
- `BM_TAG_VALID` 和 `BM_VALID` 为什么是两个不同阶段。
- shared pin、private refcount、`ResourceOwner` 各自兜住哪一段生命周期。
- temporary relation 为什么走 local buffer，为什么其他 backend 不能直接访问。

源码基线：

```text
/home/nail/postgres-lab
branch: master
commit: bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8
```

核心源码锚点：

| 顺序 | 文件 | 本节读什么 |
| --- | --- | --- |
| 1 | `src/include/storage/buf.h` | `typedef int Buffer`、`InvalidBuffer`、正数 shared / 负数 local 的 handle 编码。 |
| 2 | `src/include/storage/bufmgr.h` | `BufferBlocks`、`LocalBufferBlockPointers`、`BufferGetBlock()`、`BufferGetPage()`。 |
| 3 | `src/include/storage/buf_internals.h` | `BufferTag`、`BufferDesc`、`BufferDescPadded`、`BM_TAG_VALID`、`BM_VALID`、refcount / usage_count bit layout。 |
| 4 | `src/backend/storage/buffer/buf_init.c` | `BufferManagerShmemRequest()`、`BufferManagerShmemInit()` 如何分配 descriptor、page memory、I/O condition variable。 |
| 5 | `src/backend/storage/buffer/buf_table.c` | `BufferTag -> buf_id` mapping table 的 entry、lookup、insert、delete。 |
| 6 | `src/backend/storage/buffer/bufmgr.c` | `ReadBuffer_common()`、`PinBufferForBlock()`、`BufferAlloc()`、`PinBuffer()`、`UnpinBuffer()`、`StartSharedBufferIO()`、`TerminateBufferIO()`。 |
| 7 | `src/backend/storage/buffer/localbuf.c` | `LocalBufferAlloc()`、`PinLocalBuffer()`、`UnpinLocalBufferNoOwner()` 的 backend-local 对照。 |

短阅读路径：

```text
Buffer handle 编码
  -> BufferTag 字段
  -> BufferDesc 字段
  -> BufferManagerShmemInit()
  -> BufTableLookup()/BufTableInsert()
  -> ReadBuffer_common()
  -> BufferAlloc()
  -> PinBuffer()/UnpinBuffer()
  -> LocalBufferAlloc()
```

本节要带走的模型很短：

```text
磁盘身份: BufferTag
cache slot: BufferDesc.buf_id / Buffer handle
页内容地址: BufferBlocks / LocalBufferBlockPointers
共享并发状态: BufferDesc.state
本 backend 使用状态: PrivateRefCountEntry / LocalRefCount / ResourceOwner
```

只要把这五层分清，后续 lookup、replacement、pin/content lock 的复杂路径就不会混成一团。

## 1. 问题：一个 buffer 到底是谁

从调用者看，访问页面经常只有两行：

```c
Buffer buf = ReadBuffer(reln, blockNum);
Page page = BufferGetPage(buf);
```

这两行容易诱导一个错误直觉：

`Buffer` 好像就是页面身份。

源码不是这样设计的。

`src/include/storage/buf.h` 中 `Buffer` 只是 `typedef int Buffer`。

`InvalidBuffer` 是 0。

正数表示 shared buffer handle，外部编码为 `1..NBuffers`。

负数表示 local buffer handle，外部编码为 `-1..-NLocBuffer`。

所以 `Buffer` 更接近“当前 backend 可用的 slot handle”。

它不是 `(relation, fork, block)`。

它也不是 `BufferDesc *`。

它更不是 page memory 的地址。

shared buffer descriptor 数组下标从 0 开始。

调用者看到的 shared `Buffer` 是 `buf_id + 1`。

`BufferDescriptorGetBuffer()` 对 shared descriptor 返回 `bdesc->buf_id + 1`。

因此 `Buffer = 1` 表示 descriptor 0。

local buffer 反过来用负数编码。

`localbuf.c` 初始化 local descriptor 时让第一个 local handle 变成 `-1`。

这个编码有两个直接后果。

第一，`Buffer` 的数值不等于磁盘 block number。

同一个 block 本次可能在 buffer 42，下一次可能在 buffer 108。

第二，`Buffer` 的数值只在当前时刻有意义。

如果没有 pin，它指向的 shared slot 可能被 replacement 换成别的 `BufferTag`。

本节的主问题就是：

PostgreSQL 如何让一个可复用 slot 在需要时临时获得一个稳定身份？

答案不是一个字段。

答案是一组互相约束的状态。

`BufferTag` 表示磁盘身份。

`BufferDesc` 表示 slot 和并发状态。

mapping table 把磁盘身份映射到 slot。

pin 让 slot 在使用期间不能换身份。

`BM_TAG_VALID` 表示 tag 已经进入 mapping。

`BM_VALID` 表示 page bytes 已经可读。

`ResourceOwner` 保证 ERROR / abort 时 pin 不泄漏。

local buffer 则把“跨进程可见”这个维度拿掉，只保留本 backend 的同构状态。

这个分层不是为了抽象好看。

它服务一个具体 runtime 现象：

一个 backend miss 后刚插入 mapping table，还没把 page 从磁盘读完。

另一个 backend 此时查同一个 block，必须找到同一个 slot，而不是再分配一个副本。

所以 identity 必须先于 I/O 完成可见。

这也是 `BM_TAG_VALID` 与 `BM_VALID` 必须分开的根本原因。

如果 identity 等到 I/O 后才可见，buffer pool 会出现同一磁盘 block 的两个候选。

如果 content bytes 未读完就标成 valid，调用者会读到未初始化页面。

因此本节的核心矛盾可以落到一句话：

```text
tag 要早可见，content 要晚可见，pin 要覆盖这两个阶段之间的 slot 身份。
```

## 2. 状态：identity 被拆在五层

第一层是 `BufferTag`。

`BufferTag` 在 `src/include/storage/buf_internals.h` 中定义。

它包含：

- `spcOid`：tablespace OID。
- `dbOid`：database OID。
- `relNumber`：relation file number。
- `forkNum`：relation fork。
- `blockNum`：fork 内 block number。

这五个字段合在一起回答：

```text
如果这个 buffer 要写回磁盘，应该写到哪里？
```

源码注释强调，`BufferTag` 必须足够定位磁盘位置，不能依赖 `pg_class` 或 `pg_tablespace` 当前是否对本 backend 可见。

这是 crash safety 和并发可见性的基础。

后台进程可能在一个 relation 对自己的事务 snapshot 还不可见时写出它的 buffer。

storage manager 不能靠 relcache 重新推导写回目标。

所以 `BufferTag` 不是 relcache 的快捷索引。

它是 buffer identity 的最小持久化坐标。

第二层是 `BufferDesc`。

`BufferDesc` 在 `buf_internals.h` 中包含：

- `tag`：当前 slot 代表哪个磁盘 block。
- `buf_id`：descriptor 数组中的固定 id。
- `state`：64-bit 原子状态，包含 flags、refcount、usage_count、content lock bits。
- `wait_backend_pgprocno`：cleanup lock 等待 pin count 归 1 时使用。
- `io_wref`：AIO wait reference。
- `lock_waiters`：content lock waiter queue。

`BufferDesc.buf_id` 初始化后不变。

`BufferDesc.tag` 会随着 eviction/reuse 改变。

这正是 identity 课程最容易混淆的地方：

`buf_id` 是 slot 身份。

`tag` 是 slot 当前承载的磁盘身份。

第三层是 page memory。

shared buffer 的 page bytes 放在 `BufferBlocks`。

`BufferBlocks` 在 `buf_init.c` 中按 `NBuffers * BLCKSZ` 申请 shared memory。

`BufferGetBlock()` / `BufferGetPage()` 根据 handle 找到对应 block address。

page memory 本身不携带磁盘身份。

同一段 BLCKSZ 内存今天可以是 relation A 的 block 7，明天可以是 relation B 的 block 30。

它的身份来自 `BufferDesc.tag` 和 mapping table。

第四层是 mapping table。

`buf_table.c` 的 `BufferLookupEnt` 只有两个字段：

- `key`：`BufferTag`。
- `id`：buffer descriptor id。

这个 hash table 是 `BufferTag -> buf_id`。

它不存 page 内容。

它也不存 `BufferDesc *`。

它的职责非常窄：

```text
如果某个 tag 已经在 shared buffers 中，找到承载它的 slot id。
```

这种窄职责让 shared buffer identity 有一个全局唯一入口。

同一 `BufferTag` 在同一时刻最多对应一个 `buf_id`。

第五层是 backend-local pin 状态。

shared `BufferDesc.state` 中有 shared refcount。

但同一个 backend 重复 pin 同一个 buffer 时，不会每次都增加 shared refcount。

`bufmgr.c` 的 `PrivateRefCountEntry` 记录本 backend 对某个 buffer 的重复 pin 次数。

这降低了 hot path 上 shared atomic 的写入频率。

`ResourceOwnerRememberBuffer()` 再把 pin 交给事务 / portal / executor 的资源清理体系。

这意味着：

```text
shared refcount 说明有 backend 或 I/O 正在使用这个 slot；
private refcount 说明本 backend 自己使用了几次；
ResourceOwner 说明 ERROR / abort 时由谁兜底释放。
```

local buffer 有同构但更简单的状态。

`localbuf.c` 中 `LocalBufferDescriptors`、`LocalBufferBlockPointers`、`LocalRefCount` 都是 backend-local。

`LocalBufHash` 是本 backend 的 local hash table。

local temp relation 不需要跨 backend 的 mapping lock。

但它仍然需要 `BufferTag`、`BufferDesc.state`、`BM_TAG_VALID`、`BM_VALID`。

原因是本 backend 内部也需要区分“slot 已绑定 tag”和“content 已读完”。

`BufferDesc.state` 是几层状态的压缩承载点。

本基线中它是一个 64-bit 原子变量。

源码注释列出布局：

```text
18 bits refcount
4 bits usage_count
12 bits flags
18 bits share-lock count
1 bit share-exclusive locked
1 bit exclusive locked
```

对本节最重要的 flags 是：

- `BM_TAG_VALID`：tag 已分配，基本意味着 buffer hashtable 中有对应条目。
- `BM_VALID`：page data 已经有效。
- `BM_IO_IN_PROGRESS`：读或写正在进行。
- `BM_IO_ERROR`：上一次 I/O 失败。
- `BM_DIRTY`：page bytes 需要写回。

这里不能把 raw field 当语义。

`BM_VALID` 单独看不出“这个 page 是谁”。

`BufferDesc.tag` 单独看不出“content 是否已经读完”。

`Buffer` handle 单独看不出“slot 是否还稳定”。

必须把字段放回生命周期：

```text
mapping table 中存在 tag
  -> BufferDesc.tag 与 mapping 一致
  -> BM_TAG_VALID 已经设置
  -> pin 防止 slot 被换走
  -> BM_VALID 决定 page bytes 是否可读
  -> content lock 决定 page bytes 能否并发读写
```

这是后续三节共同复用的 mental model。

## 3. 主流程：从 ReadBuffer 到稳定身份

主流程从一个普通 read 开始。

调用链是：

```text
ReadBuffer()
  -> ReadBufferExtended()
  -> ReadBuffer_common()
  -> StartReadBuffer()
  -> StartReadBuffersImpl()
  -> PinBufferForBlock()
  -> BufferAlloc() 或 LocalBufferAlloc()
```

`ReadBufferExtended()` 的语义是返回一个 pinned buffer。

它不保证调用者已经持有 content lock。

普通 `RBM_NORMAL` 返回时 page 应该已经可用。

`RBM_ZERO_AND_LOCK` / `RBM_ZERO_AND_CLEANUP_LOCK` 会在返回时额外持有 content lock 或 cleanup-strength lock。

`ReadBuffer_common()` 先处理几个不属于本节主线的分支。

它拒绝访问其他 session 的 temporary relation。

这不是权限细节，而是 identity 边界。

其他 session 的 temp buffers 只在 owning backend 的 local buffer manager 中可见。

当前 backend 无法用 shared mapping table 找到它们。

然后 `ReadBuffer_common()` 处理 `P_NEW`。

`P_NEW` 是 relation extension 语义，不是普通 lookup。

本节先看已有 block 的普通读取。

接下来 `PinBufferForBlock()` 根据 persistence 分流：

```text
RELPERSISTENCE_TEMP
  -> LocalBufferAlloc()

permanent / unlogged
  -> BufferAlloc()
```

shared buffer 命中路径在 `BufferAlloc()` 中先构造 `newTag`。

`InitBufferTag()` 用 smgr locator、fork number、block number 填出 `BufferTag`。

然后 `BufTableHashCode()` 计算 hash code。

然后 `BufMappingPartitionLock(newHash)` 找到对应 mapping partition lock。

命中路径的核心顺序是：

```text
拿 new partition shared lock
  -> BufTableLookup(newTag)
  -> 找到 existing_buf_id
  -> GetBufferDescriptor(existing_buf_id)
  -> PinBuffer()
  -> 释放 partition lock
  -> 根据 BM_VALID 决定 foundPtr
```

这里有一个必须记住的顺序：

先 pin，再释放 mapping lock。

原因是 mapping table 命中只说明“刚才这个 tag 指向这个 slot”。

如果不 pin 就释放 mapping lock，其他 backend 可以把这个 slot invalidation 并复用。

pin 之后，`buf_internals.h` 的注释给出关键保证：

持有 pin 时，`BufferDesc.tag` 不会在你脚下变化。

这就是 `Buffer` handle 从“不稳定 slot 编号”变成“当前可安全使用的 buffer”的瞬间。

`PinBuffer()` 返回一个 bool。

返回值表示 `BM_VALID` 是否已经成立。

因此 mapping hit 不一定等于 content hit。

它可能命中了一个已经分配 tag、但 read I/O 尚未完成的 buffer。

`BufferAlloc()` 对这种情况会设置 `*foundPtr = false`。

调用者随后通过 read I/O 路径等待或完成 content。

miss 路径更能说明 identity 的时间顺序。

`BufferAlloc()` 第一次 lookup miss 后释放 shared mapping lock。

它不能一直拿着 mapping lock 去找 victim。

找 victim 可能触发 clock sweep、dirty page writeback、content lock conditional acquire、mapping table 删除旧 tag。

这些工作都不应阻塞同一 partition 上其他 unrelated lookup。

于是 miss 的状态变成：

```text
第一次 lookup miss
  -> 释放 shared mapping lock
  -> GetVictimBuffer() 找到一个 pinned victim
  -> 回来拿 new partition exclusive lock
  -> BufTableInsert(newTag, victim_buf_id)
```

`BufTableInsert()` 才是真正抢占 new tag 的动作。

它返回 `-1` 表示插入成功。

它返回已有 `buf_id` 表示另一个 backend 已经抢先插入了同一个 tag。

如果插入冲突，当前 backend 释放刚拿到的 victim pin。

然后像 hit 路径一样 pin 住已有 buffer。

这条路径解释了一个重要边界：

第一次 lookup miss 不是事实，只是一次过期观察。

真正的事实以 exclusive mapping lock 下的 insert 为准。

插入成功后，`BufferAlloc()` 还没有读 page。

它先拿 victim buffer header lock。

它断言该 victim 只有当前 backend 的一个 pin，且没有 `BM_TAG_VALID`、`BM_VALID`、`BM_DIRTY`、`BM_IO_IN_PROGRESS`。

然后写入 `victim_buf_hdr->tag = newTag`。

然后设置：

```text
BM_TAG_VALID
BUF_USAGECOUNT_ONE
BM_PERMANENT 视 persistence 和 fork 而定
```

最后释放 buffer header lock 和 mapping partition lock。

此时 returned buffer 已经 pinned。

但 `*foundPtr = false`。

也就是说：

```text
identity 已经全局可见；
content bytes 还不是有效页面。
```

后续 `StartSharedBufferIO()` 会对这个 buffer 设置 `BM_IO_IN_PROGRESS`。

读完成后 `TerminateBufferIO(..., BM_VALID, ...)` 设置 `BM_VALID` 并广播 I/O condition variable。

这条时间线是本节最重要的状态故事：

```text
slot 空闲或旧 identity 被清掉
  -> new tag 插入 mapping table
  -> BufferDesc.tag 写成 newTag
  -> BM_TAG_VALID 设置
  -> 调用者持有 pin
  -> I/O 开始，BM_IO_IN_PROGRESS 设置
  -> I/O 完成，BM_VALID 设置
  -> 调用者才能把 page bytes 当成目标 block 内容
```

shared buffer 初始化提供了这条故事的初始状态。

`BufferManagerShmemRequest()` 申请四块 shared memory：

- `BufferDescriptors`。
- `BufferBlocks`。
- `BufferIOCVArray`。
- `CkptBufferIds`。

`BufferManagerShmemInit()` 对每个 descriptor 做：

- `ClearBufferTag(&buf->tag)`。
- `pg_atomic_init_u64(&buf->state, 0)`。
- `buf->wait_backend_pgprocno = INVALID_PROC_NUMBER`。
- `buf->buf_id = i`。
- `pgaio_wref_clear(&buf->io_wref)`。
- 初始化 content lock waiters 和 I/O condition variable。

这说明一个 freshly initialized shared buffer slot 的语义是：

```text
slot id 固定存在；
page memory 已经分配；
但没有 tag，没有 content validity，没有 pin，没有 usage_count。
```

local buffer 的主流程同构但少了跨进程锁。

`LocalBufferAlloc()` 也构造 `newTag`。

它在 `LocalBufHash` 中查找。

命中时调用 `PinLocalBuffer()`。

miss 时调用 `GetLocalVictimBuffer()`，然后把 `bufHdr->tag` 写成 `newTag`，设置 `BM_TAG_VALID | BUF_USAGECOUNT_ONE`。

local buffer 使用 `pg_atomic_unlocked_write_u64()` 更新 state。

原因是 local descriptor 只有当前 backend 操作。

这不表示 local buffer 没有状态机。

它只表示该状态机不需要跨 backend 原子同步。

local temp table 仍然需要 `BM_VALID`。

`LocalBufferAlloc()` 设置 tag 后，普通 read 还要通过 local I/O 设置 content validity。

`PinLocalBuffer()` 返回 `buf_state & BM_VALID`，与 shared `PinBuffer()` 的语义对应。

## 4. 边界：哪些机制不能互相替代

第一条边界是 `Buffer` 与 `BufferTag`。

`Buffer` 是 handle。

`BufferTag` 是磁盘身份。

把 `Buffer` 缓存到一个长期结构里，必须同时证明持有 pin 或后续会重新验证 tag。

否则这个 handle 可能已经指向另一个 block。

`ReadRecentBuffer()` 就体现了这种边界。

它接收一个 recent buffer hint。

但它会构造期望的 `BufferTag`。

shared 路径先做一次 unlocked tag compare，再 `PinBuffer()`，pin 成功后再比较 tag。

这说明 recent buffer 只是 hint。

它不能替代 mapping lookup 的正确性。

第二条边界是 mapping table 与 `BufferDesc.tag`。

mapping table 是 `BufferTag -> buf_id`。

`BufferDesc.tag` 是 slot 当前承载的 identity。

两者必须在 mapping partition lock 和 buffer header lock 的协议下保持一致。

`buf_table.c` 明确说明内部不加锁。

调用者必须持有合适的 `BufMappingLock`。

这是因为 caller 往往要在释放 mapping lock 前同步调整 buffer header。

如果 hash table 内部自带锁，反而无法表达“hash entry 与 descriptor tag 一起提交”的临界区。

第三条边界是 `BM_TAG_VALID` 与 `BM_VALID`。

`BM_TAG_VALID` 说明 tag 已分配，基本意味着 buffer lookup table 中存在条目。

`BM_VALID` 说明 page bytes 可用。

一个 buffer 可以 `BM_TAG_VALID = true` 且 `BM_VALID = false`。

典型场景是 miss 后刚安装 new tag，但 read I/O 尚未完成。

另一个 backend 在这段时间查到同一 tag，必须等待同一 buffer 的 I/O，而不是创建第二份 page。

所以 `BM_TAG_VALID` 的早可见是防重复 identity。

`BM_VALID` 的晚可见是防读脏内容。

第四条边界是 pin 与 content lock。

pin 防止 slot identity 被 replacement 改掉。

content lock 保护 page bytes 的并发读写。

持有 pin 不表示 page 内容不会变。

持有 content lock 之前必须持有 pin。

释放 pin 前不能还持有该 buffer 的 content lock。

本节只把这条边界放在 identity 视角：

pin 让 `BufferDesc.tag` 稳定。

content lock 让 page header、line pointer、LSN、tuple data 的读写有并发语义。

第五条边界是 header lock 与 mapping lock。

buffer header lock 由 `BM_LOCKED` 表达。

它保护 `BufferDesc` 的 header 状态。

mapping partition lock 保护 hash table 分区。

修改 tag 和 mapping table 时常常需要两者配合。

但 header lock 不保护 page content。

mapping lock 也不保护 page content。

第六条边界是 shared buffer 与 local buffer。

shared buffer identity 对所有 backend 可见。

local buffer identity 只对当前 backend 可见。

temporary relation 的 buffer 不进入 shared mapping table。

`ReadBuffer_common()` 拒绝访问 other-session temp relation，正是为了避免当前 backend 误把不可见 local state 当成 shared identity。

parallel worker 也不能随意访问 leader 的 local buffer 指针。

local buffer 的指针和 refcount 都是 backend-local。

第七条边界是 `BM_PERMANENT` 与 identity。

`BM_PERMANENT` 不是身份字段。

但它在 `BufferAlloc()` 安装 new tag 时根据 persistence / fork 设置。

它影响 checkpoint / writeback 语义。

这说明 identity 安装时不只决定“是谁”，还决定后续持久化路径上的一些 flag。

本节不展开 WAL-before-data。

但需要知道 `BufferTag` 和 persistence flag 在 writeback 路径上共同决定“写到哪里、是否按 permanent buffer 处理”。

## 5. 异常：identity 如何在非 happy path 收尾

异常路径一：lookup hit 但 content invalid。

`BufferAlloc()` 命中 mapping table 后会 `PinBuffer()`。

`PinBuffer()` 的返回值来自 `BM_VALID`。

如果返回 false，`BufferAlloc()` 把 `*foundPtr` 改回 false。

调用者随后需要通过 read I/O 路径等待或完成页面读取。

这条路径处理了三类情况：

- 另一个 backend 正在读入页面。
- 前一次 read attempt 失败，留下 `BM_IO_ERROR`。
- 有人调用了 `StartReadBuffers()` 但还没有 `WaitReadBuffers()`。

正确性来自：

```text
同一个 tag 已经映射到同一个 slot；
所有 backend 都围绕同一个 slot 等待 content validity。
```

异常路径二：miss 后插入冲突。

第一次 `BufTableLookup()` miss 后，当前 backend 释放了 shared mapping lock。

在它找 victim 的窗口里，另一个 backend 可能已经完成同一 new tag 的 insert。

所以 `BufTableInsert()` 可能返回已有 `buf_id`。

这不是 hash table corruption。

这是正常 race。

当前 backend 释放自己拿到的 victim pin，转而 pin 已有 buffer。

这条路径避免了同一磁盘 block 出现两个 shared buffer identities。

异常路径三：I/O 失败。

`StartSharedBufferIO()` 设置 `BM_IO_IN_PROGRESS` 后，I/O owner 交给 `ResourceOwnerRememberBufferIO()`。

正常完成时 `TerminateBufferIO()` 清 `BM_IO_IN_PROGRESS`，按情况设置 `BM_VALID` 或清 `BM_DIRTY`，并广播 `BufferDescriptorGetIOCV(buf)`。

ERROR 时 `AbortBufferIO()` 由 `ResOwnerReleaseBufferIO()` 调用。

失败会留下 `BM_IO_ERROR`。

但 pin 还在资源清理阶段释放。

这保证了异常不会把 buffer 留在“别人永远以为 I/O 正在进行”的状态。

异常路径四：relation drop / truncate invalidation。

`InvalidateBuffer()` 保存 old tag。

然后计算 old hash 和 partition lock。

它拿 exclusive mapping lock，重新锁 buffer header，并检查 tag 是否仍是 old tag。

如果等待期间 tag 已变，直接返回。

如果 refcount 非零，说明仍有人 pin 或 I/O 正在使用，它释放锁、等待 I/O，再 retry。

如果能 invalidation，就清 tag、清 flags / usage_count，并从 mapping table 删除 old tag。

这里有两个关键点。

第一，按 old tag 找 partition，而不是按 buffer id。

因为 mapping table 的 key 是 `BufferTag`。

第二，清 `BufferDesc.tag` 虽然不总是理论必要，但能避免线性扫描 shared buffers 时廉价 pre-check 误判。

异常路径五：local buffer 的哈希冲突。

`LocalBufferAlloc()` 在 miss 后 `HASH_ENTER`。

如果 found 为 true，源码直接 `elog(ERROR, "local buffer hash table corrupted")`。

因为 local buffer 没有其他 backend 并发插入同一 tag。

shared path 的 insert collision 是正常 race。

local path 的同类 collision 是内部状态错误。

这对比能帮助判断：

并发 race 是否正常，取决于状态是否跨 backend 可见。

异常路径六：ResourceOwner 清理 pin。

`PinBuffer()` 第一次 pin shared buffer 时调用 `TrackNewBufferPin()`。

`TrackNewBufferPin()` 创建 private refcount entry，并调用 `ResourceOwnerRememberBuffer()`。

重复 pin 同一 buffer 时，`PinBuffer()` 增加 private refcount，并再次记入 ResourceOwner。

ERROR / abort 时 `ResOwnerReleaseBuffer()` 会释放可能残留的 content lock，再 `UnpinBufferNoOwner()`。

这条路径解释了为什么“事务结束时会释放”这种说法太粗。

真正的 owner 是 `ResourceOwner`。

真正的释放顺序要能同时处理 content lock 和 pin。

## 6. 诊断与实验

本节可观测的 runtime truth 是：

```text
同一个磁盘 block 在 shared buffers 中先获得 tag identity，再获得 valid content；pin 是跨过这两个阶段的稳定引用。
```

能直接看到的状态：

- `pg_buffercache` 可以看到 shared buffers 中的 relation / fork / block、usagecount、pinning_backends、isdirty 等快照。
- `EXPLAIN (ANALYZE, BUFFERS)` 可以看到 shared hit/read 的 query 粒度统计。
- `pg_stat_io` 可以看到 relation / temp relation I/O 的累计统计。
- gdb 可以在 `BufferAlloc()`、`PinBuffer()`、`StartSharedBufferIO()`、`TerminateBufferIO()` 断点上观察 tag、state flags、refcount。

只能推断的状态：

- 某一次 SQL 的 exact `BM_TAG_VALID -> BM_VALID` 时间窗通常太短，SQL 层只能从等待或断点推断。
- `BufferDesc.state` 的 private refcount 部分不在 SQL 视图里。
- mapping table 的 partition lock 竞争需要 wait event、perf、gdb 或动态追踪配合。

几乎不可见的状态：

- 本 backend 的 `PrivateRefCountEntry` 数组 / hash 表。
- local buffer 的内部 `LocalBufHash`。
- AIO wait reference 的短暂变化。

实验 1：观察 shared hit / miss 的 identity 安装。

目标：确认 `BM_TAG_VALID` 可以先于 `BM_VALID`。

准备：

```text
cd /home/nail/postgres-lab
```

断点：

```text
b BufferAlloc
b PinBuffer
b StartSharedBufferIO
b TerminateBufferIO
```

SQL：

```sql
CREATE TABLE bm_identity_demo(id int);
INSERT INTO bm_identity_demo SELECT generate_series(1, 10000);
CHECKPOINT;
SELECT count(*) FROM bm_identity_demo;
```

观察点：

- 第一次进入 `BufferAlloc()` 时记录 `newTag`。
- miss 后观察 victim `BufferDesc.tag` 写成 `newTag`。
- 在 `StartSharedBufferIO()` 前后观察 `BM_IO_IN_PROGRESS`。
- 在 `TerminateBufferIO()` 后观察 `BM_VALID`。

解释：

如果你在 `BufferAlloc()` 返回后、I/O 完成前看到 `BM_TAG_VALID` 已经设置，这是正确行为。

它保证其他 backend 查同一个 block 时会围绕同一个 buffer 等待。

实验 2：观察 local buffer 的负数 handle。

目标：确认 temporary relation 走 `LocalBufferAlloc()`，并且不会进入 shared mapping table。

断点：

```text
b ReadBuffer_common
b LocalBufferAlloc
b BufferAlloc
b PinLocalBuffer
```

SQL：

```sql
CREATE TEMP TABLE bm_local_demo(id int);
INSERT INTO bm_local_demo SELECT generate_series(1, 1000);
SELECT count(*) FROM bm_local_demo;
```

观察点：

- `ReadBuffer_common()` 中 persistence 为 `RELPERSISTENCE_TEMP`。
- `PinBufferForBlock()` 分流到 `LocalBufferAlloc()`。
- 返回的 `Buffer` 为负数。
- `BufferAlloc()` 不应为 temp table 普通访问触发。

解释：

local buffer 仍然有 `BufferTag` 和 `BM_VALID`。

它只是没有跨 backend 的 shared mapping lock。

实验 3：观察重复 pin 的 shared/private 分层。

目标：确认同一 backend 重复 pin 同一 buffer 时，主要增长的是 private refcount。

断点：

```text
b PinBuffer
b TrackNewBufferPin
b UnpinBufferNoOwner
```

操作：

```sql
BEGIN;
SELECT * FROM bm_identity_demo WHERE id = 1;
SELECT * FROM bm_identity_demo WHERE id = 1;
COMMIT;
```

观察点：

- 第一次 pin 需要创建 private refcount entry。
- 重复 pin 命中 existing private entry。
- 释放时 private refcount 归零后才减少 shared refcount。

解释：

shared refcount 是跨 backend 的“有人在用”。

private refcount 是本 backend 内的“我用了几次”。

诊断时不要把 `pinning_backends` 误读成每个 backend 内部 pin 次数。

常见误区：

- 把 `Buffer` 数字当 block number。
- 把 `BM_TAG_VALID` 当 page bytes 已经可读。
- 认为 mapping table hit 一定是 buffer hit。
- 认为 pin 会保护 page 内容不变。
- 认为 local buffer 因为本地化就不需要 tag / valid 状态。
- 认为 `ResourceOwner` 管的是内存生命周期；它在这里管的是 pin 和外部资源释放。

排查 buffer identity 相关问题时，可以按这个顺序问：

```text
这个 handle 是 shared 还是 local？
它现在是否 pinned？
它的 BufferTag 是否仍是期望 tag？
mapping table 是否已经有该 tag？
BM_TAG_VALID 和 BM_VALID 分别是什么？
如果 ERROR 发生，ResourceOwner 是否能释放 pin / I/O owner？
```

这不是检查清单，而是按 identity 生命周期排序的最小诊断路径。

## 7. 讨论题

1. 为什么 `Buffer` handle、`BufferTag`、`BufferDesc.buf_id` 不能合并成一个概念？
2. 如果 `BM_TAG_VALID` 已经成立但 `BM_VALID` 还没有成立，调用者能安全做什么、不能做什么？
3. ERROR 发生在 I/O 进行中时，为什么 ResourceOwner 必须知道 buffer I/O owner？
4. local buffer 没有跨 backend 并发，为什么仍然需要 identity、valid 和 refcount 边界？
5. 诊断 buffer identity 问题时，哪些状态能从 SQL 或扩展视图看到，哪些只能用断点推断？

## 8. 本节小结

本节核心链路是：

```text
ReadBuffer_common()
  -> PinBufferForBlock()
  -> BufferAlloc() / LocalBufferAlloc()
  -> mapping table 或 local hash
  -> pin
  -> BM_TAG_VALID
  -> StartBufferIO()
  -> BM_VALID
```

核心状态和边界是：

- `BufferTag` 是磁盘身份。
- `BufferDesc.buf_id` 是 slot 身份。
- `Buffer` 是调用者手里的 encoded handle。
- `BufferBlocks` / `LocalBufferBlockPointers` 是 page bytes。
- `BufferDesc.state` 把 refcount、usage_count、flags、content lock bits 压在一起。
- `PrivateRefCountEntry` / `LocalRefCount` 记录本 backend 的重复 pin。
- `ResourceOwner` 负责 ERROR / abort 时释放 pin 和 I/O owner。

ownership 规则是：

```text
mapping table 持有 tag -> buf_id 的全局索引语义；
pin 持有 slot identity 的短期稳定性；
content lock 持有 page bytes 的并发访问权；
ResourceOwner 持有清理责任。
```

异常路径的共同模式是：

```text
先保持 identity 唯一
  -> 再等待或修复 content validity
  -> 最后通过 ResourceOwner / invalidation 清理残留状态
```

能观测的是 shared buffer 快照、hit/read 统计、I/O 统计、wait event 和断点状态。

看不到的是完整的 private refcount、local hash table 和极短的 state transition 窗口。

可迁移规律：

```text
在可复用缓存中，identity、content、lifetime、concurrency 必须拆开；
否则系统无法同时做到唯一命名、异步装载、并发等待和安全复用。
```

这个规律会直接进入下一节：

既然 identity 要通过 `BufferTag -> buf_id` 映射保持唯一，PostgreSQL 如何在高并发下维护这张 shared mapping table？
