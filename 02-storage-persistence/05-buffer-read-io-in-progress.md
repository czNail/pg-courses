# PostgreSQL Buffer read I/O、BM_IO_IN_PROGRESS 与等待协议

## 课程定位

前置知识：已经理解 `BufferTag`、buffer mapping table、replacement、pin 与 content lock，知道 `BM_TAG_VALID` 只表示 buffer identity 已建立。

本节唯一主问题：

```text
多个 backend 同时读取同一个 relation fork block 时，谁负责把 page 从磁盘读入 shared buffer，其他 backend 为什么不会重复读或提前访问未完成的 page？
```

核心矛盾：目标 `BufferTag` 必须在 I/O 开始前进入 mapping table，才能保证同一个磁盘 block 只有一个 shared buffer 副本；但此时 page bytes 尚未准备好，普通访问者又绝不能把“找到 buffer”误当成“page 已经可读”。

一句话运行模型：

```text
BM_TAG_VALID 建立唯一 identity；BM_IO_IN_PROGRESS 标记唯一 I/O owner；BM_VALID 开放 page contents；BM_IO_ERROR 记录失败并允许后续重试。
```

学完后应能判断：为什么 `BM_TAG_VALID` 可以早于 `BM_VALID`；`found=false` 为什么不一定是 mapping miss；`StartBufferIO()` 的三个返回值分别要求调用者做什么；I/O owner 正常完成或 ERROR 时如何清状态并唤醒等待者。

源码基线：本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`。

## 1. 本节在总主线中的位置

前四节已经回答：

```text
一个磁盘 block 如何映射到 buffer slot？
mapping table 如何保证 tag 唯一？
miss 时如何选择 replacement victim？
调用者如何用 pin 稳定 buffer identity？
```

本节继续追问：

```text
buffer identity 已经稳定之后，page bytes 何时才真正可用？
```

这段时间可能很短，但它是 buffer manager 的关键并发窗口：

```text
tag 已安装
  -> page 尚未读入
  -> 某个 backend 正在读
  -> 读完成或失败
```

`BM_IO_IN_PROGRESS` 就处在“identity 已建立”和“contents 已可用”之间。

下一节会讨论 dirty page 与写出。本节虽然以读为主，但要先知道：`BM_IO_IN_PROGRESS` 同时服务 shared buffer 的读 I/O 和写 I/O。状态位相同，判断“工作是否已经完成”的条件不同。

## 2. 核心矛盾与一句话运行模型

假设 backend A 和 backend B 同时读取 relation R 的 block 42。

错误方案一：等磁盘读完后再把 tag 放入 mapping table。

```text
A: lookup miss -> 分配 buffer X -> 读 block 42
B: lookup miss -> 分配 buffer Y -> 读 block 42
```

结果是同一个磁盘 block 同时出现在 X 和 Y 两个 shared buffer 中。后续 dirty、WAL、writeback 和 replacement 都无法维持“一个 block 一个缓存身份”的基本不变量。

错误方案二：先放入 tag，但把“lookup 找到”直接当成 page 可读。

```text
A: 安装 tag -> 开始覆盖 buffer page bytes
B: lookup hit -> 读取尚未完成的 page bytes
```

B 可能看到旧 victim 的残留内容、半次 read、尚未验证的 page，或失败 I/O 留下的无效内容。

PostgreSQL 把这两个问题拆成三道门：

```text
identity gate: BM_TAG_VALID
I/O ownership gate: BM_IO_IN_PROGRESS
content readiness gate: BM_VALID
```

普通单页读的概念流程是：

```text
ReadBuffer_common()
  -> PinBufferForBlock()
  -> BufferAlloc()
       -> 找到或安装唯一 BufferTag
       -> 返回 pinned buffer
  -> StartBufferIO()
       -> 已 valid：无需 I/O
       -> 别人在读：加入或等待
       -> 无人在读：认领 I/O
  -> smgr/md/AIO read
  -> completion 验证 page
  -> TerminateBufferIO()
       -> 成功：设置 BM_VALID
       -> 失败：设置 BM_IO_ERROR
       -> 清 BM_IO_IN_PROGRESS
       -> 唤醒等待者
```

一句话记忆：

```text
先让所有人找到同一个 buffer，再决定一个人读，最后才允许所有人消费 page。
```

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节读什么 |
| --- | --- | --- |
| 1 | `src/backend/storage/buffer/buf_init.c` | 为什么 buffer lookup 必须先于 I/O 可见，为什么 I/O 期间必须保持 pin。 |
| 2 | `src/include/storage/buf_internals.h` | `BM_TAG_VALID`、`BM_VALID`、`BM_IO_IN_PROGRESS`、`BM_IO_ERROR`、`StartBufferIOResult`、`io_wref` 和 I/O condition variable。 |
| 3 | `src/backend/storage/buffer/bufmgr.c` | `ReadBuffer_common()`、`BufferAlloc()`、`AsyncReadBuffers()`、`WaitReadBuffers()`、`StartSharedBufferIO()`、`WaitIO()`、`TerminateBufferIO()`、`AbortBufferIO()`。 |
| 4 | `src/backend/storage/smgr/smgr.c` | buffer manager 如何通过 `smgrstartreadv()` 把读请求交给 storage manager。 |
| 5 | `src/backend/storage/smgr/md.c` | `mdstartreadv()`、`md_readv_complete()` 如何进入文件层并报告 success、partial 或 error。 |
| 6 | `src/backend/storage/buffer/localbuf.c` | local buffer 的简化对照：没有跨 backend condition variable，也暂不使用 `BM_IO_IN_PROGRESS`。 |

建议第一遍只跟这条线：

```text
ReadBuffer_common()
  -> StartReadBuffer()
  -> StartReadBuffersImpl()
  -> PinBufferForBlock()
  -> BufferAlloc()
  -> AsyncReadBuffers()
  -> StartBufferIO()
  -> smgrstartreadv()
  -> buffer_readv_complete_one()
  -> TerminateBufferIO()
  -> WaitReadBuffers()
```

第二遍再补：

```text
WaitIO()
AbortBufferIO()
buffer_stage_common()
mdstartreadv()
md_readv_complete()
ZeroAndLockBuffer()
```

第一遍按业务主线读，第二遍按等待和异常路径读，比按文件中的函数排列顺序读更容易建立整体模型。

## 4. 入口问题：找到 buffer 为什么不等于 page 已可读

`ReadBuffer()` 返回的是 pinned buffer。

正常返回时，调用者拿到的 page 已经可用；但在它内部，“找到 buffer”和“读完 page”是两个不同阶段。

`BufferAlloc()` 的职责不是读取磁盘。

它只负责：

```text
查找目标 BufferTag；
如果没有，选择并准备 victim；
把目标 tag 唯一地安装到 mapping table；
返回一个已经 pinned 的 buffer；
通过 foundPtr 告诉上层 page contents 是否已经 valid。
```

所以 `BufferAlloc()` 的返回结果有两类：

```text
found=true
  mapping 中有目标 tag
  并且 PinBuffer() 看到了 BM_VALID
  page 可以跳过 read I/O

found=false
  page 还不能用
  上层必须进入 I/O ownership / wait 协议
```

这里最容易犯的错误是把 `found=false` 翻译成“hash table 里没找到”。

实际上它至少覆盖两种情况。

第一种，真正的 mapping miss：

```text
mapping table 中没有目标 tag
  -> 选 victim
  -> 插入 tag
  -> 设置 BM_TAG_VALID
  -> 尚未设置 BM_VALID
  -> found=false
```

第二种，mapping hit 但 contents invalid：

```text
mapping table 已有目标 tag
  -> pin 同一个 buffer
  -> BM_VALID=0
  -> found=false
```

第二种情况可能意味着：

- 另一个 backend 正在读这个 page。
- 前一次读失败，留下 `BM_IO_ERROR`。
- 某个 read operation 已经准备了 buffer，但 page 还没完成。

因此 `foundPtr` 表达的不是：

```text
tag 是否存在？
```

而是：

```text
调用者是否还需要走“让 page 变 valid”的协议？
```

新 victim 安装目标 identity 时，状态大致是：

```text
BM_TAG_VALID=1
BM_VALID=0
BM_IO_IN_PROGRESS=0
BM_IO_ERROR=0
```

这个状态完全合法。

它表示：

```text
所有 backend 已经可以 lookup 到同一个 buffer；
但还没有 backend 取得 I/O ownership；
任何 backend 都不能消费 page contents。
```

接下来不是由 `BufferAlloc()` 直接“默认自己负责读”，而是调用 `StartBufferIO()` 竞争 I/O ownership。

这一步不可省略。

即使 A 是最早插入 tag 的 backend，在 A 离开 `BufferAlloc()` 到调用 `StartBufferIO()` 的窗口中，B 也可能 pin 到同一 buffer，并先认领 I/O。

所以：

```text
谁安装 tag
```

和：

```text
谁最终发起 I/O
```

不一定是同一个 backend。

这正是 mapping uniqueness 与 I/O ownership 必须分开的原因。

`ReadBuffer_common()` 在普通读中使用 `StartReadBuffer()` 和 `WaitReadBuffers()`。

`StartReadBuffer()` 这个名字也容易误导。

它的返回语义不是“page 已读完”，而是：

```text
false: buffer 已经 valid，不需要 WaitReadBuffers()
true: 本次 operation 还需要等待、发起或加入 I/O，必须调用 WaitReadBuffers()
```

普通 `ReadBuffer_common()` 会设置 `READ_BUFFERS_SYNCHRONOUSLY`，因为调用者马上就要等结果。

这个 flag 不表示绕开 AIO 框架。

在本基线中，普通同步等待路径仍复用 read operation、AIO handle 和 completion callback。flag 只是告诉底层：调用者会立即等待，不必为了后台并发而承担不必要的异步调度开销。

因此入口处应先建立三个判断：

```text
1. mapping 中是否已经有唯一 tag？
2. 这个 tag 对应的 buffer 是否 BM_VALID？
3. 如果还不 valid，是否已经有人持有 BM_IO_IN_PROGRESS？
```

后续所有分支都从这三个问题展开。

## 5. 状态：identity、I/O ownership、completion

本节最重要的四个状态位是：

| 状态位 | 表达的事实 | 不表达什么 |
| --- | --- | --- |
| `BM_TAG_VALID` | `BufferDesc.tag` 已经是可查找的 buffer identity。 | 不保证 page bytes 可用。 |
| `BM_VALID` | buffer 中的 page contents 已完成构造并可由正常访问路径使用。 | 不保证当前没有写 I/O。 |
| `BM_IO_IN_PROGRESS` | shared buffer 当前有一个 I/O owner，其他 backend 不能重复认领。 | 单独看不出是读还是写。 |
| `BM_IO_ERROR` | 最近一轮 I/O 失败。 | 不是永久禁止重试的状态。 |

对于读路径，常见状态转移是：

| 阶段 | `TAG_VALID` | `VALID` | `IO_IN_PROGRESS` | `IO_ERROR` | 含义 |
| --- | ---: | ---: | ---: | ---: | --- |
| 新 identity 已安装 | 1 | 0 | 0 | 0 | page 尚未读，任何 backend 可竞争 I/O ownership。 |
| 读已被认领 | 1 | 0 | 1 | 0 或 1 | 一个 owner 正在读，其他 backend 加入或等待。 |
| 读成功 | 1 | 1 | 0 | 0 | page 可用。 |
| 读失败 | 1 | 0 | 0 | 1 | page 不可用，后续 backend 可以重试。 |
| 失败后重试中 | 1 | 0 | 1 | 1 | 新 owner 已认领；旧 error 会在本轮结束时更新。 |
| 重试成功 | 1 | 1 | 0 | 0 | `TerminateBufferIO()` 清旧 error 并设置 valid。 |

表中“读已被认领”允许旧 `BM_IO_ERROR` 暂时仍存在。

`StartSharedBufferIO()` 认领新一轮 I/O 时，核心动作是设置 `BM_IO_IN_PROGRESS`。旧 `BM_IO_ERROR` 在 `TerminateBufferIO()` 中统一清理；如果本轮仍失败，再重新设置。

这说明：

```text
BM_IO_ERROR 是上一轮结果；
BM_IO_IN_PROGRESS 是当前 ownership；
BM_VALID 才是读者能否消费 page 的最终门槛。
```

`StartBufferIO()` 返回 `StartBufferIOResult`：

| 返回值 | 当前事实 | 调用者下一步 |
| --- | --- | --- |
| `BUFFER_IO_ALREADY_DONE` | 对读来说已经 `BM_VALID`；对写来说已经不 dirty。 | 不要重复发起 I/O。 |
| `BUFFER_IO_IN_PROGRESS` | 已有 owner；可能同时返回可等待的 `io_wref`。 | 加入已有 I/O、稍后等待，或在非等待探测中停止合并。 |
| `BUFFER_IO_READY_FOR_IO` | 当前调用者刚设置 `BM_IO_IN_PROGRESS`，成为 owner。 | 必须发起 I/O，并最终 terminate 或由错误清理 abort。 |

这三个返回值比一个简单 bool 更重要。

因为“不是我来读”至少有两种原因：

```text
page 已经读完
```

和：

```text
page 仍未读完，但别人正在读
```

前者可以立即继续，后者必须等待。

shared buffer I/O 还涉及三种 ownership。

第一，caller 的普通 pin。

调用者从 `BufferAlloc()` 返回时持有 pin。这个 pin保证 I/O 协议执行期间 buffer slot 不能被 replacement 改成另一个 tag。

第二，buffer I/O ownership。

成功设置 `BM_IO_IN_PROGRESS` 后，`StartSharedBufferIO()` 用 `ResourceOwnerRememberBufferIO()` 记录这次 ownership。

如果在真正移交给 AIO 子系统前发生 ERROR，resource owner 能调用 `AbortBufferIO()`。

第三，AIO 子系统的 pin 和 wait reference。

`buffer_stage_common()` 在 I/O staging 时：

```text
把 AIO wait reference 写入 buf->io_wref；
为 AIO 子系统增加一个额外 pin；
把 buffer I/O ownership 从当前 ResourceOwner 转交给 AIO；
```

额外 pin 很关键。

发起 I/O 的 backend 可能在 I/O 完成前 ERROR，并释放自己的普通 pin。底层 I/O 仍可能在向 buffer memory 写数据。如果 AIO 自己不持 pin，这个 slot 可能被 replacement 复用，完成回调就会写坏另一个 page。

completion 最终通过：

```text
TerminateBufferIO(..., release_aio=true)
```

清 `io_wref` 并释放 AIO pin。

等待机制也有两层。

`buf->io_wref` 让 AIO-aware 调用者直接等待已有 AIO handle。

每个 shared buffer 还有一个 I/O condition variable：

```text
BufferDescriptorGetIOCV(buf)
```

当已有 `BM_IO_IN_PROGRESS`，但还没有可用的 `io_wref`，或者调用者不准备异步加入时，`WaitIO()` 会在这个 condition variable 上等待。

因此：

```text
io_wref: 等待具体 AIO operation
buffer I/O CV: 等待 BM_IO_IN_PROGRESS 被清除
```

两者实现不同，但正确性检查相同：

```text
醒来后重新读取 buffer state。
```

condition variable broadcast 和 AIO completion 都只是“状态可能变化”的通知，不是“page 一定成功”的承诺。

## 6. 主流程：lookup、认领、读取、完成、重试

从普通 `ReadBuffer()` 读一个 shared block 开始。

`ReadBuffer()` 最终进入 `ReadBuffer_common()`。

对普通 block，它构造 `ReadBuffersOperation`，调用：

```text
StartReadBuffer()
```

如果需要 I/O，再立即调用：

```text
WaitReadBuffers()
```

所以最外层语义仍然很简单：

```text
ReadBuffer_common() 正常返回时，buffer 已 pinned，page 已 valid。
```

复杂性被封装在 start/wait 之间。

第一阶段：lookup 或安装唯一 identity。

`StartReadBuffersImpl()` 调用 `PinBufferForBlock()`。

shared buffer 进入 `BufferAlloc()`。

如果 mapping lookup 命中：

```text
pin existing buffer
  -> 看到 BM_VALID：found=true
  -> 没看到 BM_VALID：found=false
```

如果 mapping lookup miss：

```text
释放目标 partition lock
  -> GetVictimBuffer()
  -> 处理 victim 的 dirty/writeback/invalidation
  -> 重新取得目标 partition lock
  -> BufTableInsert(target tag)
```

重新插入时可能发生 collision。

因为选择和清理 victim 期间没有一直持有目标 tag 的 partition lock，另一个 backend 可能已经插入相同 tag。

当前 backend 不能再使用自己的 victim 创建第二个副本。

它必须：

```text
放弃当前 victim
  -> pin collision 返回的 existing buffer
  -> 根据 BM_VALID 返回 found=true 或 false
```

如果插入成功，`BufferAlloc()` 写入新 tag，设置 `BM_TAG_VALID`，但不设置 `BM_VALID`，也不负责读磁盘。

返回时 buffer 已 pinned。

这同时守住两个不变量：

```text
I/O 开始前，所有 backend 已能 lookup 到同一个 identity；
I/O 期间，pin 防止这个 identity 被 replacement 改掉。
```

第二阶段：竞争 I/O ownership。

page 还不 valid 时，`AsyncReadBuffers()` 调用：

```text
StartBufferIO(buffer, true, true, &operation->io_wref)
```

`forInput=true` 表示这是让 invalid buffer 变 valid 的 input operation。

`StartBufferIO()` 对 shared buffer 转到 `StartSharedBufferIO()`。

`StartSharedBufferIO()` 在 header state 下循环检查。

如果已经有 `BM_IO_IN_PROGRESS` 且 `buf->io_wref` 有效：

```text
复制 wait reference
  -> 返回 BUFFER_IO_IN_PROGRESS
```

当前 backend 只是加入已有 I/O，不发起第二次读。

如果已经有 `BM_IO_IN_PROGRESS`，调用者不愿等待：

```text
返回 BUFFER_IO_IN_PROGRESS
```

批量读用这个分支探测相邻 block。某个 block 已有人读时，不值得为了把它并入本次 `readv` 而停下来。

如果已经有 `BM_IO_IN_PROGRESS`，调用者愿意等待，但暂时拿不到 `io_wref`：

```text
提交本 backend 已 staged 的 I/O
  -> WaitIO(buf)
  -> 醒来后重新循环检查
```

“已设置 `BM_IO_IN_PROGRESS`，但 AIO staging 还没写入 wait reference”是一个很窄但必须正确处理的窗口。

等待结束后，函数不能直接认为自己可以读。

它重新判断：

```text
别人成功了：BM_VALID=1 -> BUFFER_IO_ALREADY_DONE
别人失败了：BM_VALID=0 -> 当前 backend 可再次竞争 ownership
```

如果当前没有 active I/O，对 input operation 检查 `BM_VALID`。

已经 valid：

```text
返回 BUFFER_IO_ALREADY_DONE
```

还不 valid：

```text
设置 BM_IO_IN_PROGRESS
  -> ResourceOwnerRememberBufferIO()
  -> 返回 BUFFER_IO_READY_FOR_IO
```

从这个返回点开始，当前调用者承担明确义务：

```text
要么让 I/O 完成并调用 TerminateBufferIO()；
要么在 ERROR unwind 中由 AbortBufferIO() 清理。
```

第三阶段：准备并发起实际读。

`AsyncReadBuffers()` 在调用 `StartBufferIO()` 前先取得 AIO handle。

顺序必须是：

```text
先取得可能阻塞的 AIO handle
  -> 再设置 BM_IO_IN_PROGRESS
  -> 尽快调用 smgrstartreadv()
```

如果先设置 `BM_IO_IN_PROGRESS`，再阻塞等待 handle，其他 backend 会看到“有人正在 I/O”，但 owner 实际上还没把请求提交到底层。

这不会自动造成 correctness bug，却会把一个本可避免的资源等待放大成整个 hot buffer 的等待。

对本次 read operation 的第一个 block，`AsyncReadBuffers()` 允许等待或加入 foreign I/O。

如果得到：

```text
BUFFER_IO_ALREADY_DONE
```

说明竞争期间别人已读完。当前 backend 把它作为 hit 继续。

如果得到：

```text
BUFFER_IO_IN_PROGRESS
```

operation 记录：

```text
foreign_io=true
io_wref=已有 I/O 的 wait reference
```

后续由 `WaitReadBuffers()` 等别人。

如果得到：

```text
BUFFER_IO_READY_FOR_IO
```

当前 backend 成为 owner，并尝试把后续连续 blocks 合并进同一次 `readv`。

对后续 block 使用：

```text
StartBufferIO(buffer, true, false, NULL)
```

这里 `wait=false`。

原因是批量合并只是优化，不是正确性要求。后续 block 已 valid 或已有 owner 时，停止扩大本次 I/O 即可，不能为了凑更大的 readv 而同步等待。

准备好 buffer/page 数组后，调用：

```text
smgrstartreadv()
  -> mdstartreadv()
  -> FileStartReadV()
```

storage manager 负责 relation/fork/block 抽象。

`md` 层负责 segment file、offset 和文件 I/O。

一次合并读不能越过 storage manager 给出的边界，例如 relation segment boundary。上层通过 `smgrmaxcombine()` 限制可合并 block 数。

第四阶段：AIO staging 稳定 buffer lifetime。

真正执行 I/O 前，`buffer_stage_common()` 逐个检查目标 shared buffer：

```text
BM_TAG_VALID=1
BM_VALID=0
BM_DIRTY=0
BM_IO_IN_PROGRESS=1
refcount >= 1
```

然后为 AIO 增加 pin，写入 `io_wref`，转移 ownership。

此后即使原 backend cancel 或 ERROR，AIO completion 仍可以安全访问 buffer memory。

第五阶段：completion 验证并发布 page。

底层读完成后，`buffer_readv_complete_one()` 处理每个 buffer。

成功拿到 `BLCKSZ` 字节还不等于 page 可以发布。

completion 还要调用 `PageIsVerified()` 检查 page header、checksum 等。

如果启用了相应 zero-on-error 行为，验证失败可以把 page 清零并继续作为 valid page 返回。

否则该 buffer 保持 invalid，并把本轮结果标成失败。

最终：

```text
成功：
  TerminateBufferIO(..., set_flag_bits=BM_VALID, release_aio=true)

失败：
  TerminateBufferIO(..., set_flag_bits=BM_IO_ERROR, release_aio=true)
```

`TerminateBufferIO()` 在 buffer header state 下：

```text
清 BM_IO_IN_PROGRESS；
清旧 BM_IO_ERROR；
根据本轮结果设置 BM_VALID 或重新设置 BM_IO_ERROR；
需要时释放 AIO pin 并清 io_wref；
需要时从 ResourceOwner 忘记 I/O ownership；
```

然后广播 buffer I/O condition variable。

如果释放 AIO pin 后可能满足 cleanup lock 的 sole-pin 条件，还会检查是否需要唤醒 pin-count waiter。

最关键的发布顺序是：

```text
page bytes 已写完并验证
  -> 设置 BM_VALID
  -> 清 BM_IO_IN_PROGRESS
  -> 唤醒等待者
```

不能先唤醒，再设置 valid。

第六阶段：等待者处理 own I/O 或 foreign I/O。

`WaitReadBuffers()` 等待整个 `ReadBuffersOperation` 完成。

如果 operation 等的是自己发起的 I/O，完成后通过 `ProcessReadBuffersResult()` 处理 success、warning、partial 或 error。

如果 operation 等的是 foreign I/O，它没有自己的 I/O result 可消费。

它只能重新读取目标 buffer state：

```text
BM_VALID=1
  -> 别人读成功
  -> 当前 backend 把该 block 作为 hit

BM_VALID=0
  -> 别人读失败
  -> 不增加 nblocks_done
  -> retry loop 再调用 AsyncReadBuffers()
```

这解释了为什么等待者不能把“foreign I/O 已结束”直接等同于“page 已成功”。

`WaitReadBuffers()` 还处理 partial read。

如果一次 `readv` 只完成了部分 blocks，它推进 `nblocks_done`，再对剩余部分调用 `AsyncReadBuffers()`。

重试期间还可能遇到其他 backend 已经读完某些 block，因此下一次实际 I/O 范围可能继续缩小。

对调用者来说，`WaitReadBuffers()` 的 contract 不变：

```text
整个 operation 完成，或者报告 ERROR。
```

内部可以经历多次 I/O、foreign join 和 partial retry。

把上述流程压缩成两个 backend 的时间线：

```text
backend A                              backend B
---------                              ---------
BufferAlloc(): mapping miss
安装 tag，BM_TAG_VALID=1
                                       BufferAlloc(): mapping hit
                                       pin 同一个 buffer
                                       看到 BM_VALID=0
StartBufferIO()
设置 BM_IO_IN_PROGRESS
发起 read
                                       StartBufferIO()
                                       看到已有 owner
                                       取得 io_wref 或进入 WaitIO
completion 写完并验证 page
TerminateBufferIO()
设置 BM_VALID
清 BM_IO_IN_PROGRESS
广播
                                       醒来，重新检查 BM_VALID
                                       成功后使用 page
```

如果 A 失败，最后三行改为：

```text
A 设置 BM_IO_ERROR，保持 BM_VALID=0，清 BM_IO_IN_PROGRESS；
B 醒来看到 page 仍 invalid；
B 再次调用 StartBufferIO()，可能成为新的 owner。
```

## 7. 正确性边界：唯一 identity、唯一 owner、valid 发布

第一条边界：mapping identity 必须早于 I/O 可见。

`BM_TAG_VALID` 在 read I/O 前设置，不是因为 page 已经可用，而是为了让并发 backend 收敛到同一个 `BufferDesc`。

如果把 tag 安装推迟到读完成之后，就无法防止两个 backend 分别选择 victim、分别读取同一个磁盘 block。

第二条边界：I/O 期间必须持有 pin。

`BM_IO_IN_PROGRESS` 只表达“这个 buffer 上有 I/O owner”。

真正阻止 replacement 改变 slot identity 的仍是 pin/refcount。

普通 caller 在进入协议时有 pin；AIO staging 又取得自己的 pin，保证异步 completion 不依赖原 caller 的正常生命周期。

第三条边界：`BM_IO_IN_PROGRESS` 只允许一个 owner。

对同一 shared buffer，`StartSharedBufferIO()` 在 header state 下检查和设置该位。

其他 backend 要么发现工作已经完成，要么加入已有 I/O，要么等待。它们不会同时返回 `BUFFER_IO_READY_FOR_IO`。

第四条边界：`BM_VALID` 是 page consumption gate。

`BM_TAG_VALID` 不能开放 page contents。

`BM_IO_IN_PROGRESS` 清除也不能开放 page contents。

读失败时完全可能出现：

```text
BM_TAG_VALID=1
BM_IO_IN_PROGRESS=0
BM_VALID=0
BM_IO_ERROR=1
```

只有 `BM_VALID=1` 才说明普通读路径可以继续。

第五条边界：等待通知不是成功结果。

condition variable 和 AIO wait reference 都只告诉等待者：

```text
你关心的 operation 或状态发生了变化。
```

等待者醒来必须重新检查 `BM_IO_IN_PROGRESS`、`BM_VALID` 或 operation result。

这与 PostgreSQL 中常见的 condition-variable 用法一致：先 prepare，循环检查 predicate，睡眠，醒来后重新检查。

第六条边界：`BM_IO_ERROR` 不封死 buffer identity。

读失败后保留 tag，可以让已经 pin 住或随后 lookup 的 backend 继续围绕同一个 buffer 协调。

后续 `StartSharedBufferIO()` 看到：

```text
没有 active I/O
并且 BM_VALID=0
```

就能认领新一轮 input I/O。

重试成功时，`TerminateBufferIO()` 清旧 `BM_IO_ERROR`，设置 `BM_VALID`。

第七条边界：I/O handle 必须在 ownership 前准备。

成功设置 `BM_IO_IN_PROGRESS` 后，应尽量少做可能阻塞或失败的准备工作。

所以 `AsyncReadBuffers()` 先 acquire AIO handle，再认领 buffer。

这是“发布正在工作”前先准备必要资源的通用并发原则。

第八条边界：page verification 属于 valid 发布之前。

磁盘 read syscall 成功，只说明字节被搬到内存。

page header 或 checksum 验证失败时，不能设置 `BM_VALID`，除非 read mode 明确允许 zero-on-error 并已经把 page 替换为合法的 zero page。

第九条边界：`BM_IO_IN_PROGRESS` 与 content lock 不能互相替代。

I/O protocol 解决：

```text
谁把 invalid buffer 变成 valid？
其他人如何等待？
```

content lock 解决：

```text
valid page 的 bytes 如何并发读写？
```

普通读 miss 期间，其他 backend 不能靠 content lock 抢先查看 invalid page。

page valid 之后，访问方法仍要按自己的协议取得 shared/exclusive content lock。

第十条边界：zero-and-lock 虽不读磁盘，也要复用 input ownership。

`RBM_ZERO_AND_LOCK` 和 `RBM_ZERO_AND_CLEANUP_LOCK` 在 buffer 尚未 valid 时调用 `ZeroAndLockBuffer()`。

它通过：

```text
StartSharedBufferIO(buf, true, true, NULL)
```

取得“把 invalid page 变 valid”的唯一资格。

如果当前 backend 成为 owner：

```text
memset(page, 0, BLCKSZ)
  -> 在发布 BM_VALID 前取得 exclusive content lock
  -> TerminateBufferIO(..., BM_VALID, ...)
```

先拿 content lock、再设置 valid 很重要。

等待者看到 `BM_VALID` 后可能立刻尝试访问 page。exclusive content lock 会挡住它们，直到当前调用者完成新 page 的初始化。

这里不能只用 exclusive content lock 代替 I/O ownership。

因为 access method 的 buffer access rules 允许某些读者在持 pin 的同时释放 content lock，并继续保留与 page 内容有关的信息。对 invalid-to-valid 转换，必须先阻止所有竞争者把未初始化内容当成正式 page。

第十一条边界：批量读的合并不能改变单 buffer ownership。

一次 `readv` 可以覆盖多个连续 buffers。

但每个 buffer 都必须分别取得 `BM_IO_IN_PROGRESS`。

遇到已 valid 或已有 owner 的中间 block，就停止本次合并；不能因为底层希望连续 I/O，就绕过单 buffer 的状态机。

第十二条边界：read 和 write 共享状态位，但完成条件不同。

input operation 检查：

```text
BM_VALID 是否已经设置
```

output operation 检查：

```text
BM_DIRTY 是否已经清除
```

写 I/O 通常在 `BM_VALID=1`、`BM_DIRTY=1` 的 buffer 上进行。

所以只看到 `BM_IO_IN_PROGRESS`，不能断言这是 read wait。`wait_event = BufferIO` 也不能独立区分 read 和 write。

第十三条边界：local buffer 只有相同目标，没有相同跨进程机制。

local buffers 只对当前 backend 可见，不需要 shared condition variable 或跨 backend `BM_IO_IN_PROGRESS`。

本基线中 `StartLocalBufferIO()` 主要借助 `io_wref` 协调同一 backend 内的异步操作，`BM_IO_IN_PROGRESS` 暂不用于 local buffer。

但 local AIO 仍需要 pin，仍需要在 completion 时设置 `BM_VALID` 或 `BM_IO_ERROR`。

稳定语义是：

```text
I/O 期间 buffer identity 必须稳定；
同一个 buffer 的 invalid-to-valid 转换必须只有一个 owner；
完成后才能发布 valid 或 error。
```

具体 shared/local 等待实现可以不同。

## 8. 异常：等待、ERROR 和错误用法

异常路径一：已有 I/O，但 wait reference 尚未发布。

owner 已设置 `BM_IO_IN_PROGRESS`，AIO staging 还没把 `io_wref` 写入 `BufferDesc`。

另一个 backend 这时无法异步加入具体 handle。

如果它允许等待，`StartSharedBufferIO()` 会先提交本 backend 已 staged 的 I/O，再进入 `WaitIO()`。

`WaitIO()` 循环：

```text
准备睡眠
  -> 在 header state 下读取 BM_IO_IN_PROGRESS 和 io_wref
  -> 已结束：退出
  -> 有 io_wref：等待 AIO handle
  -> 无 io_wref：睡 buffer I/O condition variable
  -> 醒来重新检查
```

这避免错过 owner 在检查与睡眠之间发出的 broadcast。

异常路径二：foreign I/O 失败。

等待 foreign I/O 的 backend 没有资格把别人的底层 result 当成自己的最终结果。

它等待结束后检查 `BM_VALID`。

如果仍 invalid，`WaitReadBuffers()` 不推进完成位置，而是重新调用 `AsyncReadBuffers()`。

于是新的 backend 可以成为 owner 并重试。

异常路径三：I/O owner 在正常 terminate 前 ERROR。

成功设置 `BM_IO_IN_PROGRESS` 后，shared buffer I/O 被记入 `CurrentResourceOwner`。

在 ownership 尚未转交 AIO 时，如果 ERROR 跳过正常完成路径，resource owner callback 调用：

```text
AbortBufferIO()
```

它保守地把本轮标为 `BM_IO_ERROR`，再通过 `TerminateBufferIO()`：

```text
清 BM_IO_IN_PROGRESS
  -> 保持 page invalid 或 dirty 状态
  -> 广播等待者
```

这保证等待者不依赖 owner 正常返回。

没有这条路径，一个 ERROR 就可能留下永久的 `BM_IO_IN_PROGRESS`，所有后续 backend 都会等待一个已经不存在的 owner。

异常路径四：ownership 已转交 AIO 后，原 backend ERROR。

这时 AIO 子系统自己的 pin 保护 buffer。

completion 可以在合适的执行者中继续完成验证和 `TerminateBufferIO()`，不依赖原 backend 的普通 pin。

这就是 `buffer_stage_common()` 转移 ownership 的意义。

异常路径五：底层返回 partial read。

`md_readv_complete()` 可以把结果标为 partial。

`ProcessReadBuffersResult()` 推进已完成 block 数，`WaitReadBuffers()` 对剩余 blocks 重试。

因此：

```text
一次 ReadBuffersOperation
```

不保证只对应：

```text
一次底层 readv
```

异常路径六：文件读成功但 page verification 失败。

completion 在设置 `BM_VALID` 前执行 `PageIsVerified()`。

不能修复或清零时：

```text
BM_VALID 保持 0
BM_IO_ERROR 设置为 1
BM_IO_IN_PROGRESS 清为 0
```

等待者醒来后必须走失败或重试逻辑，不能访问该 page。

异常路径七：把 `BM_IO_ERROR` 当永久故障锁。

`BM_IO_ERROR` 记录最近一次失败。

它不会阻止 `StartSharedBufferIO()` 在 page invalid 且当前无 active I/O 时认领重试。

真正决定 input 是否 already done 的是 `BM_VALID`。

异常路径八：把 `found=false` 当作“我一定负责读”。

多个 backend 都可能从 `BufferAlloc()` 得到同一个 pinned、invalid buffer，并都看到 `found=false`。

只有拿到 `BUFFER_IO_READY_FOR_IO` 的 backend 才是 owner。

其他 backend 不能直接调用 storage manager 读入 page。

异常路径九：等待结束后直接访问 page。

不论等待的是 buffer CV 还是 AIO wait reference，都要重新检查 `BM_VALID` 或 read operation result。

“I/O 不再进行”可能表示成功，也可能表示失败。

异常路径十：把 content lock 当 I/O completion。

持有 exclusive content lock 不会自动设置 `BM_VALID`，也不会清 `BM_IO_IN_PROGRESS` 或唤醒 I/O waiters。

反过来，page 读成功并设置 `BM_VALID`，也不意味着调用者自动持有 content lock。

异常路径十一：在设置 `BM_IO_IN_PROGRESS` 后做长时间无关工作。

从 ownership 发布到 `smgrstartreadv()` 之间，其他 backend 已经会等待。

这段路径应尽量短。

如果要增加日志、统计或资源获取，应先判断它是否可能阻塞，以及能否移到 ownership 之前或 I/O 提交之后。

排查读 I/O 协议时，按以下顺序问：

```text
目标 tag 是否已经在 mapping table 中？
当前 buffer 是否 pinned？
BM_VALID 是否已经设置？
BM_IO_IN_PROGRESS 是否已有 owner？
当前调用者拿到哪个 StartBufferIOResult？
等待的是 io_wref 还是 buffer I/O CV？
completion 最终设置了 BM_VALID 还是 BM_IO_ERROR？
ERROR 时 ownership 仍在 ResourceOwner，还是已经转交 AIO？
等待者醒来后是否重新检查了状态？
```

这比从“磁盘是不是慢”开始排查更接近真实协议。

## 9. 诊断与实验

本节可观测的 runtime truth 是：

```text
BufferIO 等待只说明某个 shared buffer 的 I/O 尚未结束；
它不直接说明这是读还是写，也不说明最终一定成功。
```

能直接观察：

- `pg_stat_activity.wait_event = 'BufferIO'`：backend 正在等待 buffer 的 I/O 状态变化。
- `AioIoCompletion`：backend 正在等待 AIO completion；具体名称以本基线的 wait event 定义为准。
- `DataFileRead`：更低层的 relation data file read 等待。
- `pg_stat_io`：按 backend type、object、context 观察 read 次数、字节和时间。
- `EXPLAIN (ANALYZE, BUFFERS)`：观察 query 级 shared hit/read，但看不到具体 `BM_IO_IN_PROGRESS` ownership。
- gdb：直接查看 `BufferDesc.state`、`io_wref` 和 `StartSharedBufferIO()` 返回值。
- buffer/smgr tracepoints：连接 buffer read start/done 与 md read start/done。

只能结合源码或 instrumentation 推断：

- 哪个 backend 最先安装 tag，哪个 backend 最终成为 I/O owner。
- 等待者是加入了现有 `io_wref`，还是落入 buffer condition variable。
- 某次 shared read 是首次 miss、foreign I/O 后的重试，还是 partial read 的后续片段。
- 短暂 `BM_IO_ERROR` 曾存在多久。
- 同一 block 的竞争是否来自并发 scan、read-ahead 重叠，还是失败重试。

诊断时不要过度解释单个指标。

`shared_blks_read` 增加表示当前 backend 发起了物理读计数，不表示每个等待者都重复读了一次。

等待 foreign I/O 成功的 backend 会把自己的结果按 hit 跟踪，而真正 owner 记录 read。

`BufferIO` 也可能来自 buffer 写出，因为 `BM_IO_IN_PROGRESS` 和 I/O condition variable 同时服务读写。

实验 1：按状态变化跟读源码。

在 `/home/highgo/postgres` 中确认课程基线：

```bash
git rev-parse --abbrev-ref HEAD
git rev-parse HEAD
```

定位状态和枚举：

```bash
rg -n "BM_TAG_VALID|BM_VALID|BM_IO_IN_PROGRESS|BM_IO_ERROR|StartBufferIOResult" \
  src/include/storage/buf_internals.h
```

定位主状态机：

```bash
rg -n "BufferAlloc\\(|StartSharedBufferIO\\(|WaitIO\\(|TerminateBufferIO\\(|AbortBufferIO\\(" \
  src/backend/storage/buffer/bufmgr.c
```

不要先按函数的文件位置顺读。

按以下四次搜索跟状态：

```text
谁设置 BM_TAG_VALID？
谁设置 BM_IO_IN_PROGRESS？
谁设置 BM_VALID？
谁清 BM_IO_IN_PROGRESS 并广播？
```

观察结果：

- `BufferAlloc()` 安装 identity，不读 page。
- `StartSharedBufferIO()` 认领 I/O，不发布 valid。
- completion 决定 `BM_VALID` 或 `BM_IO_ERROR`。
- `TerminateBufferIO()` 统一结束 ownership 并唤醒等待者。

实验 2：观察两个 backend 同读一个冷 block。

目标不是证明每次都能看到某个 wait event，而是确认：

```text
两个 backend pin 到同一个 BufferDesc；
只有一个 backend 拿到 BUFFER_IO_READY_FOR_IO。
```

debug build 中可设置断点：

```gdb
break BufferAlloc
break StartSharedBufferIO
break WaitIO
break buffer_readv_complete_one
break TerminateBufferIO
```

准备一张足够大的表，用较小 `shared_buffers` 或扫描其他数据把目标 block 变冷。

session A 读取目标范围，并在 `StartSharedBufferIO()` 认领后暂停。

session B 读取相同范围。

观察：

- B 的 `BufferAlloc()` 是否命中 A 安装的 tag。
- A、B 的 buffer id 是否相同。
- B 的 `StartSharedBufferIO()` 是返回 `BUFFER_IO_IN_PROGRESS`，还是进入 `WaitIO()`。
- A 完成后，B 是否重新检查 `BM_VALID`。

AIO staging 很快，`WaitIO()` 的窄窗口不一定稳定复现。

复现不到 `BufferIO` 不代表协议不存在。B 可能直接取得 `io_wref` 并等待 `AioIoCompletion`，也可能在运行到该点前 A 已经完成。

实验 3：手工推演失败与重试。

先写下初始状态：

```text
TAG_VALID=1
VALID=0
IO_IN_PROGRESS=1
IO_ERROR=0
```

然后分别推演：

```text
读成功
读 syscall 失败
page verification 失败
foreign I/O 失败后另一个 backend 重试成功
owner 在 ownership 转交 AIO 前 ERROR
```

期望状态：

| 场景 | 结束状态 | 等待者下一步 |
| --- | --- | --- |
| 读成功 | `VALID=1, IO_IN_PROGRESS=0, IO_ERROR=0` | 使用 page。 |
| 读失败 | `VALID=0, IO_IN_PROGRESS=0, IO_ERROR=1` | 报错或重试。 |
| foreign 读失败 | 同上 | 当前 backend 再次竞争 ownership。 |
| 重试成功 | `VALID=1, IO_IN_PROGRESS=0, IO_ERROR=0` | 使用 page。 |
| owner 提前 ERROR | `AbortBufferIO()` 清 in-progress 并设置 error | 等待者被唤醒，不永久睡眠。 |

这个练习的重点是确认：

```text
清 BM_IO_IN_PROGRESS
```

不等于：

```text
设置 BM_VALID
```

实验 4：检查 zero-and-lock 的发布顺序。

定位：

```bash
rg -n "ZeroAndLockBuffer|StartSharedBufferIO|TerminateBufferIO" \
  src/backend/storage/buffer/bufmgr.c
```

确认 invalid buffer 的顺序：

```text
取得 input ownership
  -> 清零 page
  -> 获取 exclusive content lock
  -> 设置 BM_VALID 并唤醒等待者
```

然后回答：

```text
为什么 content lock 必须在 BM_VALID 之前取得？
```

答案不是“保护 memset 不被并发执行”这么简单。

I/O ownership 已经保证只有一个 backend 清零。content lock 的作用是：page 一旦被发布为 valid，仍不允许其他访问者在调用者完成初始化前看到它。

常见误区：

- 把 `BM_TAG_VALID` 当成 `BM_VALID`。
- 把 `found=false` 当成纯 mapping miss。
- 认为安装 tag 的 backend 一定是 I/O owner。
- 认为等待结束就表示读成功。
- 认为 `BM_IO_ERROR` 会永久阻止重试。
- 认为 AIO completion 可以只依赖发起 backend 的普通 pin。
- 认为 zero-and-lock 没有真实磁盘 I/O，所以不需要 I/O ownership。
- 认为 content lock 能代替 `BM_IO_IN_PROGRESS`。
- 认为 `BufferIO` wait event 一定是磁盘读慢。

## 10. 讨论题

1. 为什么 `BufferAlloc()` 必须先设置 `BM_TAG_VALID`，却不能同时设置 `BM_VALID`？
2. 两个 backend 都从 `BufferAlloc()` 得到 `found=false` 时，什么机制保证只有一个 backend 发起实际读？
3. `BUFFER_IO_ALREADY_DONE` 与 `BUFFER_IO_IN_PROGRESS` 为什么不能合并成同一个返回值？
4. 等待 foreign I/O 的 backend 被唤醒后，为什么必须检查 `BM_VALID`？
5. `BM_IO_ERROR=1`、`BM_VALID=0`、`BM_IO_IN_PROGRESS=0` 对下一次读意味着什么？
6. AIO 子系统为什么要取得额外 pin，并在 staging 后接管 I/O ownership？
7. `ZeroAndLockBuffer()` 为什么先拿 exclusive content lock，再发布 `BM_VALID`？
8. 看到 `wait_event = BufferIO` 时，哪些事实可以直接判断，哪些仍需结合 buffer state、I/O statistics 或断点？

## 11. 本节小结

本节核心链路是：

```text
BufferAlloc()
  -> 安装或找到唯一 BufferTag
  -> pin buffer
  -> StartBufferIO()
  -> join / wait / become owner
  -> smgr/md/AIO read
  -> page verification
  -> TerminateBufferIO()
  -> valid 或 error
  -> wake and recheck
```

核心状态是：

- `BM_TAG_VALID`：buffer identity 已经进入 mapping protocol。
- `BM_IO_IN_PROGRESS`：当前有唯一 shared buffer I/O owner。
- `BM_VALID`：page contents 可以被正常访问。
- `BM_IO_ERROR`：最近一轮 I/O 失败，后续仍可重试。
- `io_wref`：AIO-aware 等待者加入已有 operation。
- buffer I/O condition variable：等待 `BM_IO_IN_PROGRESS` 发生变化。
- caller/AIO pin：I/O 期间稳定 buffer slot identity 和 memory lifetime。

ownership 规则是：

```text
安装 tag 不等于拥有 I/O；
found=false 不等于必须由我读取；
只有 BUFFER_IO_READY_FOR_IO 才授予 I/O ownership；
取得 ownership 后必须 terminate，ERROR 时必须 abort；
AIO staging 后由 AIO pin 和 completion 接管生命周期。
```

等待规则是：

```text
已有 io_wref 就加入具体 AIO；
没有 io_wref 时可以等待 buffer I/O condition variable；
无论等哪一种，醒来后都必须重新检查；
只有 BM_VALID 才代表 input operation 成功。
```

失败规则是：

```text
失败时保持 page invalid；
设置 BM_IO_ERROR；
清 BM_IO_IN_PROGRESS；
唤醒所有等待者；
允许后续 backend 重新认领并重试。
```

可迁移规律：

```text
共享缓存填充通常需要三阶段发布：
先发布唯一 identity，
再发布单一 initializer ownership，
最后发布 initialized/valid 状态。
```

这条规律不只适用于 PostgreSQL buffer manager。

任何“先注册对象、后台填充内容、并发读者等待”的共享缓存，都必须把：

```text
对象存在
正在初始化
初始化成功
初始化失败
```

分成可检查、可等待、可清理的不同状态。
