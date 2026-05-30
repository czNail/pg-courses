# PostgreSQL shm_mq 单生产者单消费者 ring buffer

## 课程定位

前置知识：已经理解 DSM segment 生命周期、`shm_toc` 如何在 DSM 内发现对象，以及 latch / memory barrier 的基本语义。

本节唯一主问题：

```text
为什么共享消息队列限定为 single-reader / single-writer，
mq_bytes_read、mq_bytes_written、ring wrap、memory barrier 和 latch 唤醒
如何在无锁数据搬运、背压和消息边界之间折中？
```

核心矛盾：并行执行、logical apply、repack worker 等场景需要在两个 backend 之间传递 tuple 或协议消息；消息搬运应该尽量少加锁、少复制、能背压、能等待、能保留消息边界。但一旦支持任意多生产者或多消费者，ring buffer 的 head/tail 更新、消息边界、唤醒和 detach 语义都会变成复杂的共享并发协议。

PostgreSQL 的选择是把 `shm_mq` 做成非常明确的双端通道：

```text
一个 sender 写 ring 和 mq_bytes_written；
一个 receiver 读 ring 和 mq_bytes_read；
双方用 atomic counter 发布进度，用 memory barrier 约束 ring 数据可见性；
ring 满或空时用 latch 进入可观测等待。
```

学完后应能判断：

```text
为什么 shm_mq 不支持多个 writer 或多个 reader；
为什么 mq_bytes_read / mq_bytes_written 可以用 atomic write，而不是 fetch-add；
为什么 message 需要 length word，而不是只暴露字节流；
为什么 receive 有时能返回 shared memory pointer，有时必须复制到本地 buffer；
为什么 force_flush 和 1/4 ring threshold 会影响 latency 与 SetLatch 成本；
为什么 ring full / ring empty 是 flow control，不是异常。
```

本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `2c4bd2bf5700`。

## 1. 本节在总主线中的位置

上一节讲的是 `shm_toc`：

```text
DSM 里有很多对象；
不同 backend 映射地址不同；
所以用 key -> offset 找到对象。
```

本节进入其中一个典型对象：`shm_mq`。

在 parallel query 中，leader 会在 DSM 里为 worker 准备 tuple queue、error queue、message queue 等对象，并通过 TOC 暴露给 worker。worker attach DSM 后，找到 queue，再通过 `shm_mq` 把 tuple 或协议消息传回 leader。

本节只讲消息搬运主协议：

```text
共享 ring buffer
  -> 两个单调 counter
  -> message length + payload
  -> barrier 发布
  -> latch 背压和唤醒
```

下一节会把焦点转到 attach、detach、worker 失败和 `SHM_MQ_DETACHED` 的传播。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
shm_mq 是 DSM 中的一段 single-reader / single-writer ring buffer；
sender 把 length word 和 payload 写入未消费区域，批量推进 mq_bytes_written；
receiver 根据 mq_bytes_written 读取完整消息，批量推进 mq_bytes_read；
ring 满时 sender 睡在 latch 上，ring 空时 receiver 睡在 latch 上。
```

关键 tension 是：

```text
低锁数据搬运
  vs
跨进程消息边界、可见性顺序、背压和等待诊断
```

如果只追求低锁，可以让双方共享一段字节数组，各自乱读乱写。但这样无法回答：

```text
这条消息是否完整？
reader 能不能看到 writer 刚 memcpy 一半的数据？
writer 什么时候可以覆盖旧数据？
队列满了谁睡，谁唤醒？
consumer 退出后 producer 会不会永远卡住？
```

`shm_mq` 的答案不是引入一个大锁，而是收紧模型：

```text
只有 sender 能增加 mq_bytes_written；
只有 receiver 能增加 mq_bytes_read；
只有 sender 写 ring 的未发布区域；
只有 receiver 读 ring 的已发布区域。
```

这个限制让 hot path 大部分时间不需要 mutex。它不是 API 简化，而是正确性边界。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 作用 |
| --- | --- | --- |
| 1 | `src/include/storage/shm_mq.h` | 对外语义：single-reader / single-writer、result enum、send / receive / attach API。 |
| 2 | `src/backend/storage/ipc/shm_mq.c` | ring layout、counter 协议、barrier、wait、detach callback 的核心实现。 |
| 3 | `src/backend/executor/tqueue.c` | parallel executor 如何把 tuple 作为消息写入 queue，以及 reader 如何取回 `MinimalTuple`。 |
| 4 | `src/backend/libpq/pqmq.c` | worker 如何把 frontend/backend protocol message 重定向到 `shm_mq`。 |
| 5 | `src/backend/access/transam/parallel.c` | parallel context 如何创建 DSM、放置 queue、启动 worker 并连接两端。 |
| 6 | `src/test/modules/test_shm_mq/` | `shm_mq` 的专用测试模块，适合观察 pipeline、非阻塞和大消息行为。 |

推荐阅读顺序：

```text
先读 shm_mq.h 的注释和 result enum
  -> 再读 shm_mq.c 顶部的 synchronization 注释
  -> 再读 struct shm_mq / shm_mq_handle
  -> 跟 shm_mq_sendv() -> shm_mq_send_bytes()
  -> 跟 shm_mq_receive() -> shm_mq_receive_bytes()
  -> 最后看 tqueue.c / pqmq.c 的真实调用方式
```

不要从“队列 API”角度读它。正确入口是：

```text
谁能写哪个字段？
哪个字段是共享状态？
哪个状态只存在于本 backend 的 handle？
counter 推进前后需要哪些 ordering？
```

## 4. 关键数据结构与状态

### 4.1 DSM 内的 `shm_mq`

`shm_mq` 本体放在 DSM 中，所有 attach 该 DSM 的 backend 都能看到：

| 字段 | 更新者 | 语义 |
| --- | --- | --- |
| `mq_mutex` | 双方短暂持有 | 只保护 sender / receiver 指针初始化和 detach 取对端。 |
| `mq_receiver` | receiver 设置一次 | 接收端 `PGPROC`，设置后不再变化。 |
| `mq_sender` | sender 设置一次 | 发送端 `PGPROC`，设置后不再变化。 |
| `mq_bytes_read` | receiver | receiver 已经公开消费的字节数。 |
| `mq_bytes_written` | sender | sender 已经公开写入的字节数。 |
| `mq_ring_size` | 初始化后只读 | ring 可用容量。 |
| `mq_detached` | 任一端 | 表示某一端已经失去兴趣，对端不应无限等待。 |
| `mq_ring_offset` | 初始化后只读 | ring 起始位置对齐偏移。 |
| `mq_ring[]` | sender 写，receiver 读 | 真实消息字节区。 |

最重要的不变量：

```text
used = mq_bytes_written - mq_bytes_read
0 <= used <= mq_ring_size
write offset = mq_bytes_written % mq_ring_size
read offset  = mq_bytes_read    % mq_ring_size
```

这里的 counter 是单调增长的逻辑位置，不是 ring 内数组下标。wrap 只发生在取模时。

### 4.2 backend-local 的 `shm_mq_handle`

`shm_mq_handle` 不在 DSM 中。它是当前 backend 的私有状态：

| 字段 | 语义 |
| --- | --- |
| `mqh_queue` | 指向 DSM 中的 `shm_mq`。 |
| `mqh_segment` | 如果非 NULL，detach DSM 时自动触发 queue detach callback。 |
| `mqh_handle` | 可选 `BackgroundWorkerHandle`，用于判断未 attach 的 worker 是否已经死亡。 |
| `mqh_buffer` / `mqh_buflen` | receiver 本地 buffer，用于重组跨 ring wrap 或过大的消息。 |
| `mqh_consume_pending` | receiver 已读但尚未发布到 `mq_bytes_read` 的字节数。 |
| `mqh_send_pending` | sender 已写 ring 但尚未发布到 `mq_bytes_written` 的字节数。 |
| `mqh_partial_bytes` | 非阻塞或大消息场景下，当前 message 已处理的字节数。 |
| `mqh_expected_bytes` | 读完 length word 后期望的 payload 长度。 |
| `mqh_length_word_complete` | length word 是否已经完整处理。 |
| `mqh_counterparty_attached` | 当前 backend 是否已确认对端 attach。 |
| `mqh_context` | 本地 buffer 分配所在的 MemoryContext。 |

注意这两个 pending 字段：

```text
sender 可以先 memcpy 多段数据，再批量发布 mq_bytes_written；
receiver 可以先连续读多条消息，再批量发布 mq_bytes_read。
```

批量发布减少 atomic write 和 `SetLatch()` 次数，但会改变 latency。源码里的默认阈值是 ring size 的 1/4。

### 4.3 message format

`shm_mq` 不是裸字节流。每条消息在 ring 中编码为：

```text
MAXALIGN(sizeof(Size)) 的 length word
  -> MAXALIGN(payload length) 的 payload
```

`shm_mq_send()` 是单 buffer 入口，内部转成 `shm_mq_sendv()`。`shm_mq_sendv()` 支持多个 iovec，但最终仍然把一条逻辑 message 写成：

```text
Size nbytes
payload bytes
alignment padding
```

这就是 `TupleQueueReaderNext()` 能断言：

```text
tuple->t_len == nbytes
```

的原因。消息边界来自 `shm_mq`，不是上层 tuple 格式本身。

## 5. 主流程源码 walkthrough

### 5.1 创建 queue 并绑定两端

`shm_mq_create(address, size)` 在一段 DSM 地址上初始化队列：

```text
SpinLockInit(mq_mutex)
mq_receiver = NULL
mq_sender = NULL
mq_bytes_read = 0
mq_bytes_written = 0
mq_ring_size = size - aligned header
mq_detached = false
mq_ring_offset = aligned ring start
```

然后两端各自设置身份：

```text
shm_mq_set_receiver(mq, MyProc)
shm_mq_set_sender(mq, MyProc)
```

这两个函数只在 `mq_mutex` 下设置一次 `PGPROC *`。如果对端已经设置，会 `SetLatch(&counterparty->procLatch)`，唤醒可能正在等待 attach 的进程。

随后本 backend 调用：

```text
mqh = shm_mq_attach(mq, seg, bgworker_handle)
```

这一步分配 backend-local handle，并在 `seg != NULL` 时注册：

```text
on_dsm_detach(seg, shm_mq_detach_callback, mq)
```

所以 queue 的共享字节在 DSM 中，操作状态在本地 handle 中。这个分层非常重要：shared object 负责跨进程协议，local handle 负责“本次调用进行到哪里”。

### 5.2 sender：先写 length word，再写 payload

发送入口：

```text
shm_mq_send(mqh, nbytes, data, nowait, force_flush)
  -> shm_mq_sendv(mqh, &iov, 1, nowait, force_flush)
```

`shm_mq_sendv()` 先计算总 payload 大小，并拒绝超过 `MaxAllocSize` 的消息。然后按两个阶段推进：

```text
阶段 1:
  把 Size nbytes 写入 ring

阶段 2:
  把 payload 写入 ring
```

两个阶段都会调用 `shm_mq_send_bytes()`。如果 `nowait = true` 并遇到 ring full，函数可能返回 `SHM_MQ_WOULD_BLOCK`。此时本地 handle 中的 `mqh_partial_bytes` 和 `mqh_length_word_complete` 记录了“当前消息写到哪里”，调用者必须用同一组参数重试。

这是一个很容易忽略的语义：

```text
一条消息开始发送后，不能随便换 payload 重试；
否则 ring 中已经发布或等待发布的前半条消息会被破坏。
```

### 5.3 sender：ring full 时形成背压

`shm_mq_send_bytes()` 的循环核心是：

```text
rb = atomic_read(mq_bytes_read)
wb = atomic_read(mq_bytes_written) + mqh_send_pending
used = wb - rb
available = ring_size - used
```

如果 `available > 0`：

```text
offset = wb % ring_size
sendnow = min(available, ring_size - offset)
memory barrier
memcpy(ring + offset, data, sendnow)
mqh_send_pending += MAXALIGN(sendnow)
```

如果 `available == 0`，队列满了：

```text
先把 mqh_send_pending 发布到 mq_bytes_written
SetLatch(receiver)

如果 nowait:
  返回 SHM_MQ_WOULD_BLOCK
否则:
  WaitLatch(MyLatch, WAIT_EVENT_MESSAGE_QUEUE_SEND)
  ResetLatch(MyLatch)
  CHECK_FOR_INTERRUPTS()
  重新计算 rb/wb/available
```

这里的 full 不是错误，而是 backpressure。receiver 不推进 `mq_bytes_read`，sender 就不能覆盖旧数据。

### 5.4 sender：什么时候发布 `mq_bytes_written`

sender 写 ring 后不会每次都立刻更新共享 counter。`shm_mq_sendv()` 在这些情况下发布：

```text
force_flush = true
mqh_send_pending > mq_ring_size / 4
ring full 需要等待前
shm_mq_detach() 前
```

发布由 `shm_mq_inc_bytes_written()` 完成：

```text
pg_write_barrier()
atomic_write(mq_bytes_written, old + n)
```

write barrier 的意义是：

```text
先让 ring 中的 memcpy 对 receiver 可见；
再让 receiver 看到 mq_bytes_written 变大。
```

如果反过来，receiver 可能根据新 counter 读到尚未写完的 ring 字节。

`force_flush` 是 latency knob。`pqmq.c` 用 `force_flush = true`，因为它发送协议消息后还会给 parallel leader 发 proc signal，需要确保 leader 醒来时能看到消息。`tqueue.c` 发送 tuple 时用 `force_flush = false`，允许批量发布以减少唤醒成本。

### 5.5 receiver：先读 length word，再读 payload

接收入口：

```text
shm_mq_receive(mqh, &nbytes, &data, nowait)
```

如果还没确认 sender attach，receiver 会先处理 attach 等待。确认后，它按两个阶段读：

```text
阶段 1:
  读取 Size length word，得到 expected payload length

阶段 2:
  读取 payload，返回 nbytes 和 data pointer
```

底层由 `shm_mq_receive_bytes()` 计算可读范围：

```text
written = atomic_read(mq_bytes_written)
read = atomic_read(mq_bytes_read) + mqh_consume_pending
used = written - read
offset = read % ring_size
```

如果已经有足够数据，或者数据到 ring 尾部发生 wrap：

```text
*nbytesp = min(used, ring_size - offset)
*datap = ring + offset
pg_read_barrier()
return SHM_MQ_SUCCESS
```

read barrier 的意义是：

```text
先看到 sender 发布的 mq_bytes_written；
再读取 ring 中对应的消息字节。
```

这与 sender 发布 `mq_bytes_written` 前的 write barrier 成对。

### 5.6 receiver：零拷贝和 wrap 后复制

`shm_mq_receive()` 尽量避免复制：

```text
如果 length word 后已经有完整 payload 且在 ring 中连续：
  *data 直接指向 DSM 中的 ring
```

这就是 `tqueue.c` 中 `TupleQueueReaderNext()` 可以直接把 `data` 当作 `MinimalTuple` 返回的原因。注释也提醒调用者：

```text
返回指针在下一次 receive 前有效；
可能指向 shared memory，也可能指向本地临时 buffer。
```

如果消息跨 ring 尾部，或者当前连续块不够完整 payload：

```text
分配/扩展 mqh_buffer
把多个 ring chunk 复制到本地 buffer
返回 mqh_buffer
```

所以 ring wrap 不是协议错误，只是把 receive 从 zero-copy 路径带到 copy 路径。

### 5.7 receiver：ring empty 时等待

当 `used` 不足以满足当前读取需求时，receiver 先发布已经消费的字节：

```text
if mqh_consume_pending > 0:
  shm_mq_inc_bytes_read(mq, mqh_consume_pending)
  mqh_consume_pending = 0
```

`shm_mq_inc_bytes_read()` 做两件事：

```text
pg_read_barrier()
atomic_write(mq_bytes_read, old + n)
SetLatch(sender)
```

这里的 barrier 保证 receiver 读取 ring 的动作发生在发布 `mq_bytes_read` 之前。否则 sender 可能过早覆盖 receiver 尚未真正读完的数据。

如果仍然没有足够数据：

```text
nowait = true:
  返回 SHM_MQ_WOULD_BLOCK

nowait = false:
  WaitLatch(MyLatch, WAIT_EVENT_MESSAGE_QUEUE_RECEIVE)
  ResetLatch(MyLatch)
  CHECK_FOR_INTERRUPTS()
  重新读取 mq_bytes_written
```

empty 也是 flow control。它意味着 sender 还没发布足够数据，而不是 queue 损坏。

## 6. 生命周期 / ownership / cleanup

`shm_mq` 的生命周期分成三层：

| 层次 | owner | cleanup |
| --- | --- | --- |
| DSM 内 queue 字节 | 创建 DSM 的上层模块 | 随 DSM detach / destroy 消失。 |
| sender / receiver `PGPROC` 指针 | 各端 set 一次 | 不重置；detach 用 `mq_detached` 表达断开。 |
| `shm_mq_handle` | 当前 backend 的 MemoryContext | `shm_mq_detach()` 释放本地 buffer 和 handle。 |

`shm_mq_detach()` 先做一个关键动作：

```text
如果 mqh_send_pending > 0:
  发布 mq_bytes_written
```

然后：

```text
shm_mq_detach_internal(mq)
  -> mq_detached = true
  -> SetLatch(counterparty)

cancel_on_dsm_detach(...)
pfree(mqh_buffer)
pfree(mqh)
```

为什么 detach 前要 flush pending writes？

```text
sender 可能已经 memcpy 了最后一段消息，但还没推进 mq_bytes_written；
如果直接标记 detached，receiver 可能误以为没有剩余数据。
```

`shm_mq_receive_bytes()` 也配合这个语义：看到 `mq_detached` 后，不会立刻丢弃 ring 中已发布的数据，而是先确认 `mq_bytes_written` 没有新的推进。这样 sender detach 后，receiver 仍能 drain 已经发布的消息。

## 7. 正确性机制层次

| 层次 | 机制 | 保护的问题 |
| --- | --- | --- |
| 端点唯一性 | single sender / single receiver | 避免多个进程并发修改同一个 counter 或同一区 ring。 |
| endpoint 初始化 | `mq_mutex` | 保护 `mq_sender` / `mq_receiver` 从 NULL 到非 NULL 的一次性发布。 |
| 数据发布 | `mq_bytes_written` + write barrier | receiver 只能在 ring 数据可见后看到 writer 进度。 |
| 空间回收 | `mq_bytes_read` + read barrier | sender 只能在 receiver 真正读完后覆盖旧字节。 |
| 等待唤醒 | `SetLatch()` / `WaitLatch()` | full / empty / attach 等待有可观测 wait event。 |
| 消息边界 | length word + payload | receiver 按完整 message 返回，不暴露半条消息。 |
| 本地续传 | `mqh_partial_bytes` 等 handle 字段 | 非阻塞或跨 wrap 情况下能从中断点继续。 |
| detach 协议 | `mq_detached` + latch | 对端退出时不让另一端永久阻塞。 |

`shm_mq` 的核心正确性来自“字段更新者唯一”：

```text
mq_bytes_written 只有 sender 改；
mq_bytes_read 只有 receiver 改。
```

因此源码可以用：

```text
atomic_write(old + n)
```

而不是：

```text
atomic_fetch_add()
```

这减少了总线锁和 cache coherency 成本。这个优化成立的前提正是 single-writer / single-reader 限制。

## 8. 错误路径 / 异常路径 / fallback

### 8.1 `SHM_MQ_WOULD_BLOCK`

`nowait = true` 时，send 或 receive 可能返回 `SHM_MQ_WOULD_BLOCK`：

```text
send:
  ring full，无法继续写当前消息

receive:
  ring 中还没有足够字节组成当前需要的部分
```

这不是失败。它表示调用者应该等 latch 被设置后，用同一个 handle 和同一条消息继续。

`TupleQueueReaderNext()` 的注释特别提醒：即使 `nowait` 返回没有 tuple，也可能已经积累了部分 message bytes，所以轮询调用仍可能推进内部状态。

### 8.2 超大消息

发送端拒绝：

```text
nbytes > MaxAllocSize
```

接收端也防御性检查 length word：

```text
invalid message size
```

这防止 receiver 为损坏或恶意 length word 分配过大的本地 buffer。

### 8.3 ring wrap

wrap 的 fallback 是复制：

```text
如果完整消息不在一个连续 ring chunk 中：
  receiver 用 mqh_buffer 重组
```

这保证 API 总是返回一条连续 message，而不是让上层处理 scatter/gather。

### 8.4 对端尚未 attach

如果 queue 满或 receiver 想读，但对端还没 attach，`shm_mq` 会等待 endpoint 指针变成非 NULL。若调用者提供了 `BackgroundWorkerHandle`，还能判断 worker 是否已经不可能启动或已经死亡。

本节只把它作为 flow control 的边界条件；完整 failure 传播留给下一节。

## 9. 成本、资源与跨模块传播

`shm_mq` 的成本主要来自五处：

| 成本 | 触发场景 | 影响 |
| --- | --- | --- |
| `memcpy` 到 ring | 每次发送 | payload 越大成本越高。 |
| ring wrap 后复制到本地 buffer | 消息跨 ring 尾部或大于连续可读块 | receiver 多一次 copy 和本地内存分配。 |
| atomic counter publish | pending 超阈值、force flush、full / detach | 影响 cache line 竞争和 CPU ordering 成本。 |
| `SetLatch()` | 发布 written / read 或 detach | 系统调用和跨进程唤醒成本，不能太频繁。 |
| `WaitLatch()` | ring full / empty / attach 等待 | 形成可观测 wait event，也增加 latency。 |

ring size 的影响非常直接：

```text
ring 太小:
  更容易 full，sender 经常等待；
  wrap 更频繁，receiver 更容易复制；
  但 DSM 占用小。

ring 太大:
  等待少，wrap 少；
  但 DSM 占用大，pending threshold 也变大，force_flush=false 时 latency 可能上升。
```

跨模块传播的例子：

```text
tqueue.c:
  executor worker 发送 MinimalTuple；
  leader 用 TupleQueueReaderNext() 读取；
  queue full 会反向背压 worker 的执行速度。

pqmq.c:
  worker 把协议消息写入 queue；
  使用 force_flush=true，并在发送后 signal leader；
  latency 比批量吞吐更重要。
```

所以 `shm_mq` 的参数不是孤立性能选项，而是会改变上层执行器、协议消息和 worker 协作的节奏。

## 10. 观测与诊断入口

### 10.1 wait event

`shm_mq` 的等待点有明确 wait event：

| wait event | 含义 |
| --- | --- |
| `WAIT_EVENT_MESSAGE_QUEUE_SEND` | sender 因 ring full 等 receiver 消费。 |
| `WAIT_EVENT_MESSAGE_QUEUE_RECEIVE` | receiver 因 ring empty 等 sender 写入。 |
| `WAIT_EVENT_MESSAGE_QUEUE_INTERNAL` | 等待对端 attach。 |
| `WAIT_EVENT_MESSAGE_QUEUE_PUT_MESSAGE` | `pqmq.c` 非阻塞发送协议消息时等待可写。 |

在 parallel query 卡住时，如果 worker 大量处于 send wait，常见解释是 leader 读取不够快或 queue 太小；如果 leader 处于 receive wait，则 worker 还没有产生足够数据或已进入其它等待。

### 10.2 gdb 断点

适合观察的断点：

```text
break shm_mq_send_bytes
break shm_mq_receive_bytes
break shm_mq_inc_bytes_written
break shm_mq_inc_bytes_read
break shm_mq_detach_internal
```

关键字段：

```text
p mq->mq_ring_size
p pg_atomic_read_u64(&mq->mq_bytes_written)
p pg_atomic_read_u64(&mq->mq_bytes_read)
p mqh->mqh_send_pending
p mqh->mqh_consume_pending
p mqh->mqh_partial_bytes
p mq->mq_detached
```

观察重点不是单个字段值，而是：

```text
used = written - read
used 是否接近 ring_size
pending 是否迟迟未发布
detach 前是否 flush 了 pending writes
```

### 10.3 测试模块

`src/test/modules/test_shm_mq/` 提供专用 SQL 函数，用于测试 pipeline、大消息和 worker 通信。适合做两类实验：

```text
小 ring + 大消息:
  更容易触发 wrap、复制和 send wait。

nowait / pipelined 场景:
  更容易看到 SHM_MQ_WOULD_BLOCK 和 partial message 状态。
```

如果要把 runtime 现象回到源码，优先用这个模块，而不是直接从复杂 parallel query 现象开始。

## 11. 常见误区

误区一：`shm_mq` 是通用多生产者队列。

不是。它的文件头和 API 语义都明确限定 single-reader / single-writer。多个生产者需要上层拆成多个 queue，或引入额外仲裁层。

误区二：atomic counter 足以保证 ring 数据可见。

不够。counter 只告诉对方“逻辑进度”，barrier 才约束 ring 中 memcpy 与 counter 发布的顺序。

误区三：`SetLatch()` 就等于消息已经完整可读。

不是。latch 只是唤醒提示。醒来后必须重新检查 `mq_bytes_written` / `mq_bytes_read` 和当前 message 状态。

误区四：`SHM_MQ_WOULD_BLOCK` 表示没有任何状态变化。

不是。一次 nowait receive 可能已经读完 length word 或复制了部分 payload；一次 nowait send 可能已经写入了部分 message。状态保存在 `shm_mq_handle` 中。

误区五：返回的 `data` 总是 DSM pointer。

不是。连续消息可以直接指向 ring；跨 wrap 或需要重组时会指向本地 `mqh_buffer`。上层只能依赖“下一次 receive 前有效”这个语义。

误区六：queue 满了应该调大锁或加更多 worker。

queue 满通常说明 receiver 侧消费速度跟不上。更多 worker 可能制造更多 producer，但单个 `shm_mq` 仍然只有一个 sender。真正要判断的是上层 pipeline 和消费端瓶颈。

## 12. 课堂实验

### 实验一：画出 counter 和 ring 的状态

在 `shm_mq_send_bytes()` 和 `shm_mq_receive_bytes()` 下断点，记录：

```text
ring_size
mq_bytes_written
mq_bytes_read
mqh_send_pending
mqh_consume_pending
offset = counter % ring_size
```

手工画出：

```text
已读区域
已发布未读区域
已写但未发布区域
可写空闲区域
```

目标：理解 `pending` 为什么属于 backend-local 状态，而不是共享 counter 的一部分。

### 实验二：触发 ring full

使用 `test_shm_mq` 或构造 parallel query，让 sender 产生数据快于 receiver 消费。观察 `pg_stat_activity.wait_event` 是否出现 message queue send wait。

源码解释：

```text
available == 0
  -> shm_mq_inc_bytes_written()
  -> SetLatch(receiver)
  -> WaitLatch(... MESSAGE_QUEUE_SEND)
```

目标：把“执行变慢”解释成 backpressure，而不是 worker 无故挂起。

### 实验三：比较 `force_flush`

对比 `tqueue.c` 和 `pqmq.c`：

```text
tqueue.c:
  shm_mq_send(..., force_flush=false)

pqmq.c:
  shm_mq_sendv(..., force_flush=true)
```

回答：

```text
为什么 tuple 流更偏吞吐，可以等 pending 超过 1/4 ring；
为什么协议消息更偏 latency，需要立即发布并 signal leader。
```

目标：理解同一个 queue primitive 在不同调用者下的 flush 策略。

### 实验四：构造 wrap 后复制

让 ring size 较小，发送长度接近或超过 ring 尾部剩余空间的消息。在 `shm_mq_receive()` 中观察：

```text
rb < nbytes
mqh_buffer 分配或扩展
payload 被复制到 backend-local buffer
```

目标：理解 zero-copy receive 的适用边界。

## 13. 讨论题

1. 如果要支持多个 sender，共享状态中至少要新增哪些仲裁机制？哪些字段不能再由单一进程 atomic write？
2. 为什么 `mq_bytes_written` 和 `mq_bytes_read` 使用单调 counter，而不是直接保存 ring 下标？
3. `force_flush=false` 提高吞吐的同时，会在哪些 workload 下增加 tail latency？
4. 为什么 receiver 看到 `mq_detached` 后仍要检查是否还有已发布数据可读？
5. 如果上层消息天然是 fixed-size tuple，是否可以去掉 length word？这样会失去哪些通用性？
6. `shm_mq` 的 wait event 能告诉你“谁慢”吗？哪些结论仍需要结合执行计划、worker 状态和上层调用者判断？

## 14. 本节小结

`shm_mq` 的核心不是“在共享内存里实现一个队列”，而是把跨进程消息通道压缩成一组非常硬的不变量：

```text
一个 sender，一个 receiver；
sender 唯一推进 mq_bytes_written；
receiver 唯一推进 mq_bytes_read；
counter 发布必须用 barrier 包住 ring 数据可见性；
ring full / empty 通过 latch 形成背压；
length word 保留消息边界；
本地 handle 记录 partial 和 pending 状态。
```

single-reader / single-writer 看起来是限制，实际上是 `shm_mq` 能够在 hot path 避免大锁的前提。它把复杂性从“任意并发队列”收敛为“两端之间的流控协议”。

可迁移规律：

```text
共享内存通信的低锁设计，通常不是靠减少状态完成的；
而是靠明确规定每个状态只能由谁修改、何时发布、对端凭什么观察。
```

下一节会继续沿着这条通道往外看：当 sender 或 receiver 尚未 attach、提前退出、ERROR 或 worker 死亡时，`shm_mq` 如何把“等不到对端”转化为 `SHM_MQ_WOULD_BLOCK` / `SHM_MQ_DETACHED` 和可诊断的失败传播。
