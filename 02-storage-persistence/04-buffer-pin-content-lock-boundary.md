# PostgreSQL Buffer pin 与 content lock 的职责边界

## 课程定位

前置知识：已经知道 shared buffer slot 可以被 replacement 复用，也知道 lookup hit 后必须 pin buffer，miss 后 victim selection 也会先 pin candidate。

本节唯一主问题：

```text
为什么 PostgreSQL 必须把 buffer pin 和 buffer content lock 分成两个机制，而不是用一个锁同时保护 buffer 身份、生命周期和 page 内容？
```

核心矛盾：调用者需要一个低成本、可嵌套、能跨函数持有的引用，保证 buffer frame 不被替换；但 page contents 又需要短临界区的读写互斥、cleanup 强度和 wait queue，不能把所有 pin 都变成内容锁。

一句话运行模型：

```text
pin 稳定 buffer frame 的身份和生命周期；content lock 稳定该 frame 中 page bytes 的并发访问；cleanup lock 是在 exclusive content lock 下确认没有其他 pin 的更强条件。
```

学完后应能判断：pin 防止什么、不防止什么；content lock 防止什么、不防止什么；为什么持有 content lock 前必须持有 pin；ResourceOwner 如何在 ERROR 时兜底释放 lock 和 pin。

源码基线：本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`。

## 1. 本节在总主线中的位置

前面三节分别讲了 identity、mapping uniqueness 和 replacement candidate。拿到 buffer 之后，访问方法真正要面对的是 page bytes 的并发读写。本节把“slot 不被替换”和“page 内容不被并发破坏”拆开讲。

这条边界会贯穿后续 dirty write、heap/btree page 修改、cleanup lock 和 eviction：replacement 看 refcount，page 修改看 content lock，二者不能互相替代。

## 2. 核心矛盾与一句话运行模型

如果只用 content lock 保护一切，scan、executor 和访问方法会把大量长期 buffer 使用变成 page 内容互斥。如果只用 pin，又没有机制防止 page header、line pointer、hint bit 或索引结构在错误窗口被并发修改。

所以本节的模型是：

```text
BufferAlloc() hit/miss
  -> PinBuffer() 或 StrategyGetBuffer() pin candidate
  -> caller optional LockBuffer()
  -> page access
  -> UnlockBuffer()
  -> ReleaseBuffer()/UnpinBuffer()
  -> ResourceOwner 在 ERROR 时兜底
```

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节读什么 |
| --- | --- | --- |
| 1 | `src/include/storage/buf.h` | `Buffer` handle、shared/local 编码。 |
| 2 | `src/include/storage/bufmgr.h` | `BufferLockMode`、`LockBuffer()` inline wrapper、`LockBufferForCleanup()`、`ConditionalLockBufferForCleanup()`。 |
| 3 | `src/include/storage/buf_internals.h` | `BufferDesc.state` layout、`BM_LOCK_VAL_SHARED`、`BM_LOCK_VAL_SHARE_EXCLUSIVE`、`BM_LOCK_VAL_EXCLUSIVE`、`BM_PIN_COUNT_WAITER`。 |
| 4 | `src/backend/storage/buffer/bufmgr.c` | `PinBuffer()`、`UnpinBuffer()`、`WakePinCountWaiter()`、`BufferLockAcquire()`、`UnlockBuffer()`、`LockBufferForCleanup()`、`ResOwnerReleaseBuffer()`。 |
| 5 | `src/backend/storage/buffer/freelist.c` | `StrategyGetBuffer()` 如何把 refcount 作为 replacement 硬门槛。 |
| 6 | `src/backend/storage/lmgr/lwlock.c` | 只作为 wait queue / shared-exclusive 语义对照；本基线中 buffer content lock 不再是普通 `LWLock *`。 |

## 4. 入口问题：pin 为什么不是 content lock

`ReadBuffer()` 返回的是 pinned buffer。

这句话很容易被误读成：

“我已经拿到 buffer，所以 page contents 安全可读写。”

源码不是这个语义。

pin 保护的是 buffer frame 的身份和生命周期。

它保证这个 `Buffer` 对应的 slot 不会在你持有期间被 replacement 改成别的 `BufferTag`。

它让你可以稳定地说：

```text
这个 Buffer handle 仍然指向我刚刚 lookup / allocation 得到的那个 block。
```

pin 不保证 page bytes 不变。

pin 不阻止另一个 backend 在持有合适 content lock 时修改 page header。

pin 不阻止 hint bit 设置。

pin 不阻止 tuple line pointer 被 page-level protocol 允许的修改影响。

pin 不提供 shared/exclusive 读写互斥。

content lock 保护的是 page contents。

它让多个读者、一个写者、或 share-exclusive 写出路径按模式互斥。

在本基线中，content lock 状态也编码在 `BufferDesc.state` 中。

但这不让它等同于 pin。

它只是与 refcount、usage_count、flags 一起共享一个 atomic state word。

为什么不能只用 content lock？

因为很多路径需要长期持有“我正在使用这个 buffer”的引用，但不需要长期阻塞 page contents。

例如 executor scan 可能 pin 当前 page，读取某些 tuple，再在访问方法协议允许的范围内释放或重新获取 content lock。

如果 pin 等同 content lock，读路径会把大量 buffer residency 变成 page content 互斥。

这会扩大临界区，增加等待，降低并发。

为什么不能只用 pin？

因为 page bytes 需要真实互斥。

两个 backend 同时 compact page、设置 page header、修改 line pointer，不能只靠“都 pin 了”来保证正确。

pin 只说明 page frame 不会被替换。

content lock 才说明 page bytes 当前能按某种模式访问。

所以本节核心边界是：

```text
pin: frame lifetime / identity
content lock: page bytes concurrency
cleanup lock: physical removal safety
```

把这三个边界合并，会在三个方向出错。

把 pin 当 content lock，会读写 page 时缺少互斥。

把 content lock 当 pin，会在释放 pin 后仍持有一个可能已经被 replacement 改身份的 lock 状态。

把 cleanup lock 当普通 exclusive content lock，会在物理删除 item 时忽略其他 backend 手里可能存在的 page 内指针。

## 5. 状态：shared refcount、private refcount、lock mode

`BufferDesc.state` 的低位包含 shared refcount。

`BUF_STATE_GET_REFCOUNT(state)` 读到的是 shared pin count。

这个 count 不等于“每个 backend 对这个 buffer 调用了几次 pin”。

同一 backend 重复 pin 同一个 shared buffer 时，PostgreSQL 不会每次都增加 shared refcount。

它使用 backend-private `PrivateRefCountEntry`。

`bufmgr.c` 中 private refcount entry 同时记录：

- `buffer`。
- 本 backend 的 `refcount`。
- 本 backend 持有的 `lockmode`。

这种设计服务两个目标。

第一，减少 shared atomic 写入。

同一个 backend 内部重复 pin，不需要反复改 shared refcount。

只有从 0 到 1 的第一次 pin 增加 shared refcount。

从 1 到 0 的最后一次 unpin 减少 shared refcount。

第二，把 pin 和 content lock 的本 backend ownership 放在同一 entry 里。

`ResOwnerReleaseBuffer()` 在 ERROR 时能检查：

```text
如果 ref->data.lockmode != BUFFER_LOCK_UNLOCK:
    先 BufferLockUnlock()
再 UnpinBufferNoOwner()
```

这就是“释放 pin 前不能还持有 content lock”的 cleanup 兜底。

`ResourceOwner` 是第三层 owner。

`ResourceOwnerRememberBuffer()` 记录每一次 pin。

`ResourceOwnerForgetBuffer()` 在正常 unpin 时移除。

ERROR / abort 时，resource owner callback 释放残留 pin。

这不是 MemoryContext。

MemoryContext 负责内存生命周期。

ResourceOwner 负责外部资源、pin、I/O owner、lock-like ownership 的释放。

content lock 模式定义在 `bufmgr.h`：

```text
BUFFER_LOCK_SHARE
BUFFER_LOCK_SHARE_EXCLUSIVE
BUFFER_LOCK_EXCLUSIVE
```

share 只与 exclusive 冲突。

share-exclusive 与 share-exclusive、exclusive 冲突。

exclusive 与所有模式冲突。

`LockBuffer()` 是 inline wrapper。

如果 mode 是 `BUFFER_LOCK_UNLOCK`，它调用 `UnlockBuffer()`。

其他模式调用 `LockBufferInternal()`。

这也是 hot path 的性能细节。

源码把 unlock 分支放在 inline wrapper 中，避免在 `bufmgr.c` 中一个较差的 branch prediction 影响常见路径。

content lock 的 wait state 也在 `BufferDesc.state` 和 `lock_waiters` 中。

`BM_LOCK_HAS_WAITERS` 表示 content lock 有等待者。

`BM_LOCK_WAKE_IN_PROGRESS` 表示 waiter 已被 signal 但还没运行。

`lock_waiters` 是 wait queue。

这些状态服务 page content 互斥。

它们不表达 pin count。

cleanup wait 使用另一组状态。

`BM_PIN_COUNT_WAITER` 和 `wait_backend_pgprocno` 表示有一个 backend 正在等 pin count 归 1。

当前实现只允许一个这种 waiter。

这服务 `LockBufferForCleanup()`。

它不服务普通 shared/exclusive content lock wait。

local buffer 是简化对照。

local buffer 只有当前 backend 可见。

`LocalRefCount` 记录本 backend pin count。

`LockBuffer()` 对 local buffer 直接返回。

因为没有跨 backend content lock 需求。

但 local buffer 仍然需要 pin / refcount。

它仍然要防止本 backend 在使用同一个 local frame 时把它替换掉。

## 6. 主流程：pin、lock、unlock、unpin

从 lookup hit 看 pin 的主流程。

`BufferAlloc()` 在 mapping table hit 后拿到 `existing_buf_id`。

它调用：

```text
valid = PinBuffer(buf, strategy, false)
```

`PinBuffer()` 先查 private refcount。

如果本 backend 还没有 pin 这个 buffer，它走 shared state CAS。

它等待 `BM_LOCKED` 清掉。

它增加 shared refcount。

普通 strategy 下，如果 usage_count 未达上限，它增加 usage_count。

ring strategy 下，它最多把 usage_count 提到 1。

CAS 成功后调用 `TrackNewBufferPin()`。

`TrackNewBufferPin()` 创建 private refcount entry，增加 private refcount，并 `ResourceOwnerRememberBuffer()`。

如果本 backend 已经 pin 过这个 buffer，`PinBuffer()` 不再增加 shared refcount。

它只增加 private refcount，并再次记录 ResourceOwner。

返回值表示 `BM_VALID`。

这条路径说明：

```text
第一次 pin: shared refcount + private refcount + ResourceOwner
重复 pin: private refcount + ResourceOwner
```

`PinBuffer_Locked()` 是另一个入口。

caller 已经持有 buffer header lock。

它用 `UnlockBufHdrExt(..., refcount_change=1)` 在释放 header lock 时顺手增加 shared refcount。

一些路径必须这样做。

因为必须先 pin，才能防止 buffer state 在 caller 脚下变化。

unpin 的主流程在 `UnpinBuffer()`。

正常 release 会先 `ResourceOwnerForgetBuffer()`。

然后进入 `UnpinBufferNoOwner()`。

它减少 private refcount。

只有 private refcount 归 0 时，才减少 shared refcount。

减少 shared refcount 后，如果 `BM_PIN_COUNT_WAITER` 存在，调用 `WakePinCountWaiter()`。

这服务 cleanup lock 等待者。

同时源码断言：

```text
Assert(!BufferLockHeldByMe(buf))
```

也就是说，最后一个 private pin 释放前，本 backend 不应还持有 content lock。

content lock 主流程从 `LockBuffer()` 开始。

`LockBufferInternal()` 断言 buffer 已 pinned。

shared buffer 才真正进入 `BufferLockAcquire()`。

local buffer 直接返回。

`BufferLockAcquire()` 先找 private refcount entry。

这一步要求当前 backend 已经 pin 了 buffer。

然后它断言本 backend 当前没有持有该 buffer 的 content lock。

当前实现不支持同一 backend 对同一 buffer 同时记录多个 content lock ownership。

然后它 `HOLD_INTERRUPTS()`。

这保证进入 content lock 保护的 shared memory 临界区后，不会被 cancel/die interrupt 在中间打断。

获取 lock 时先 `BufferLockAttempt()`。

如果第一次失败，它把自己加入 wait queue，再尝试第二次。

第二次还失败，才报告 wait event 并睡眠。

这个“两次尝试”避免 lost wakeup：

```text
先尝试拿锁
  -> 失败后入队
  -> 入队后再尝试
  -> 仍失败才等待
```

获取成功后，private refcount entry 的 `lockmode` 记录当前模式。

unlock 主流程在 `UnlockBuffer()` / `BufferLockUnlock()`。

它通过 `BufferLockDisownInternal()` 取回本 backend 持有的 mode。

然后从 `BufferDesc.state` 中扣掉对应 lock bits。

接着 `BufferLockProcessRelease()` 唤醒 waiter。

最后 `RESUME_INTERRUPTS()`。

这说明 interrupt holdoff 覆盖的是 content lock 持有区间。

pin 本身不是 interrupt holdoff。

`UnlockReleaseBuffer()` 是常见组合操作。

它释放 content lock，并释放 pin。

这个函数存在是因为很多路径在完成 page access 后马上 unlock + unpin。

本基线的 state layout 允许某些路径把 unlock 和 unpin 合并成更少 atomic 操作。

这也是为什么 content lock 与 refcount 被压在同一个 64-bit state 中有性能意义。

但语义上它们仍是两种机制。

## 7. 正确性边界：replacement、page 修改、cleanup

第一条边界：replacement 看 pin/refcount。

`StrategyGetBuffer()` 遇到 `BUF_STATE_GET_REFCOUNT(state) != 0` 就不能用该 buffer。

它不关心 caller 是否持有 content lock。

因为 replacement 的问题是：

```text
这个 frame 能不能失去旧身份？
```

只要有人 pin，它就不能失去旧身份。

第二条边界：page 修改看 content lock。

修改 page header、line pointer、tuple data、hint bits、page LSN 等，需要按 access method 的协议持有合适 content lock。

pin 只保证 frame 还在。

它不保证 bytes 不变。

第三条边界：dirty writeback 同时碰到两者。

dirty victim 必须已经 unpinned，才能被 `StrategyGetBuffer()` 选中。

但写出 dirty bytes 前，`GetVictimBuffer()` 还要拿 share-exclusive content lock。

所以 dirty victim 是两条边界的交叉点：

```text
refcount == 0 才能成为 candidate；
content lock 成功才可以安全写出 bytes。
```

第四条边界：cleanup lock 比 exclusive content lock 更强。

`LockBufferForCleanup()` 的注释说明，删除 disk page items 需要：

```text
持有 exclusive content lock
并观察到没有其他 backend pin 该 buffer
```

原因是其他 backend 只要 pin 着 page，就可能持有 page 内指针。

即使它没有 content lock，也可能在访问方法协议中暂存 item reference。

物理删除 item 会让这些引用失效。

所以 cleanup lock 必须等 pin count 归 1。

这里的 1 是当前 backend 自己的 pin。

第五条边界：新增 pin 可以在 cleanup 开始后进入。

`LockBufferForCleanup()` 注释说明，如果 cleanup 已经拿到 exclusive content lock 并观察到 pin count 为 1，之后新来的 backend 可以 pin。

但新来的 backend 看 page contents 前必须拿 content lock。

它会被 exclusive lock 阻塞。

因此 cleanup 的安全点是：

```text
在 exclusive content lock 下观察到当时没有其他 pin holder。
```

不是永久禁止后续 pin。

第六条边界：conditional cleanup 失败不能继续修改。

`ConditionalLockBufferForCleanup()` 如果看到本 backend private refcount 不为 1，返回 false。

如果拿不到 exclusive content lock，返回 false。

如果拿到 lock 但 shared refcount 不为 1，它释放 lock 并返回 false。

caller 不能在 false 后继续执行 cleanup 修改。

第七条边界：I/O pin 也算 pin。

本基线中 AIO / buffer I/O owner 可能在 I/O 进行中持有 pin。

`TerminateBufferIO()` 在 release AIO pin 后，如果存在 `BM_PIN_COUNT_WAITER`，也可能唤醒 cleanup waiter。

因此 cleanup wait 不只等普通 backend scan pin。

也可能等 I/O ownership。

第八条边界：ResourceOwner cleanup 先于普通 lock release 阶段。

buffer pin 的 resource owner release phase 是 `RESOURCE_RELEASE_BEFORE_LOCKS`。

这样 ERROR 时，如果本 backend 同时持有 content lock 和 pin，buffer callback 能先解 content lock，再释放 pin。

否则后续 lock cleanup 可能看不到正确 ownership。

第九条边界：pin 成本主要来自 shared 写入和 cleanup 压力。

第一次 pin 一个 shared buffer 需要增加 shared refcount。

这通常是 atomic CAS 或 atomic add/sub 热点。

同一 backend 重复 pin 同一 buffer 时走 private refcount，避免反复写 shared cache line。

所以 private refcount 不是语义装饰。

它是降低 hot path cache-line bouncing 的工程选择。

如果 workload 让很多 backend 反复 pin 同一小组 hot buffers，shared refcount cache line 仍可能成为竞争点。

如果 workload 长时间持有 pin，replacement 和 cleanup 会受到影响。

这两类成本不同。

前者是 CPU / cache contention。

后者是 lifetime 阻塞。

诊断时不要把它们混成一个“buffer lock 慢”。

第十条边界：content lock 成本来自临界区长度和等待队列。

content lock 保护 page bytes。

如果访问方法在持锁区做过多 CPU work，其他 backend 会在 `WAIT_EVENT_BUFFER_SHARED`、`WAIT_EVENT_BUFFER_SHARE_EXCLUSIVE` 或 `WAIT_EVENT_BUFFER_EXCLUSIVE` 对应的 content lock 等待上停住。

如果临界区很短但冲突极高，等待仍可能显著。

如果写路径需要 exclusive 或 share-exclusive，读者和写者的兼容矩阵会直接影响等待形态。

这类问题的归因应回到 page-level protocol。

它不一定是 buffer replacement 问题。

第十一条边界：cleanup wait 会向 VACUUM、recovery 和 index maintenance 传播。

需要 physical cleanup 的路径不只看 page content lock。

它还要等其他 pin holder 消失。

因此一个长 scan 持有 pin，可能让 cleanup 延后。

在 primary 上，这可能表现为 VACUUM cleanup 或 index page cleanup 变慢。

在 standby recovery 中，这可能变成 recovery conflict on buffer pin。

在 allocation path 中，长期 pin 也会减少可替换 candidate。

同一个 pin lifetime，会通过不同模块表现成不同症状。

第十二条边界：统计指标只能给出局部事实。

`pg_stat_activity.wait_event` 如果对应 `WAIT_EVENT_BUFFER_SHARED`、`WAIT_EVENT_BUFFER_SHARE_EXCLUSIVE` 或 `WAIT_EVENT_BUFFER_EXCLUSIVE`，说明当前在等某种 buffer content lock mode。

它不能证明 pin 泄漏。

`BufferCleanup` 说明 cleanup-strength lock 在等 pin count 条件。

它不能证明 page writer 或 bgwriter 慢。

`pg_buffercache.pinning_backends` 是当前快照。

它不能告诉你 private refcount，也不能告诉你 pin 持有了多久。

如果怀疑长期 pin，需要结合 SQL 现场、access method 代码路径、gdb 断点或扩展 instrumentation。

第十三条边界：版本实现可以变，但职责拆分稳定。

本基线中 buffer content lock 已经是 `BufferDesc.state` 的一部分。

旧资料可能仍把 content lock 描述成 per-buffer LWLock。

读旧资料时要把实现细节和语义分开。

稳定语义是：

```text
pin protects frame lifetime
content lock protects page contents
cleanup lock requires no other pins
ResourceOwner releases leaked ownership on ERROR
```

具体 lock bits、AIO pin、wait queue 实现会随版本演进。

本节基于 `/home/highgo/postgres` 的 master `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`。

## 8. 异常：等待、ERROR 和错误用法

异常路径一：cleanup lock 等待其他 pin。

`LockBufferForCleanup()` 先断言当前 backend pin 了 buffer。

它还调用 `CheckBufferIsPinnedOnce()`。

如果当前 backend 自己 pin 了两次，就不能等待 “pin count == 1”。

因为那一个额外 pin 就是自己造成的。

普通 shared buffer 路径会：

```text
LockBuffer(buffer, BUFFER_LOCK_EXCLUSIVE)
LockBufHdr(bufHdr)
if shared refcount == 1:
    success
else:
    set BM_PIN_COUNT_WAITER
    unlock content lock
    wait
    retry
```

等待时，普通 backend 用 `ProcWaitForSignal(WAIT_EVENT_BUFFER_CLEANUP)`。

hot standby startup process 还会处理 recovery conflict on buffer pin。

这就是 `HoldingBufferPinThatDelaysRecovery()` 的诊断意义。

异常路径二：多个 cleanup waiter。

当前实现每个 buffer 只允许一个 pin-count waiter。

如果已经有 `BM_PIN_COUNT_WAITER`，另一个 backend 也试图等待，源码报：

```text
multiple backends attempting to wait for pincount 1
```

这不是普通 lock queue。

它是 cleanup pin-count waiter 的特殊限制。

异常路径三：conditional content lock 失败。

`ConditionalLockBuffer()` 只尝试一次。

如果本 backend 已经持有该 buffer 的 content lock，它也返回 false。

即使新模式与旧模式不冲突，也返回 false。

原因是 private refcount entry 当前只记录一个 lock ownership。

caller 必须把 false 当作“没有 lock”。

异常路径四：ERROR 时持有 content lock。

正常代码应该显式 unlock 后再 release pin。

但 ERROR 可能跳过正常路径。

`ResOwnerReleaseBuffer()` 检查 private refcount entry。

如果 `lockmode != BUFFER_LOCK_UNLOCK`，它 `HOLD_INTERRUPTS()` 后调用 `BufferLockUnlock()`。

然后再 `UnpinBufferNoOwner()`。

这保证 content lock 不会因为 ERROR 泄漏。

也保证最后一个 pin 释放时不会违反“还持有 content lock”的断言。

异常路径五：forgotten `ResourceOwnerEnlarge()`。

很多 pin path 在拿锁或改变 shared state 前先调用 `ResourceOwnerEnlarge()`。

如果先成功 pin，再发现 ResourceOwner 没空间记录，ERROR cleanup 会变困难。

所以源码把 enlarge 放在 pin 前。

这不是性能无关的样板。

它是 ownership cleanup 的预分配边界。

异常路径六：把 pin 当 content lock。

典型 bug 是：

```text
ReadBuffer()
  -> 不 LockBuffer()
  -> 直接检查或修改 page contents
```

如果只是读取某些 access method 明确允许无锁读的字段，可能由更高层协议保证。

但一般 page contents 不能靠 pin 保护。

诊断这种 bug 时，关注点不是 replacement。

关注点是 page-level concurrent modification。

异常路径七：把 content lock 当 pin。

如果代码释放 pin 后仍保存 `Buffer` 并认为 content lock 或历史 lock 足以保护它，这是错误的。

content lock ownership 本身依赖 private refcount entry。

没有 pin，slot identity 可以被 replacement 改掉。

任何后续使用都必须重新 pin 或重新 lookup。

异常路径八：长期 pin 阻塞 cleanup / recovery / replacement。

长期 pin 会让 `StrategyGetBuffer()` 跳过该 buffer。

它还可能让 `LockBufferForCleanup()` 等待。

在 hot standby 中，startup process 可能因为 buffer pin conflict 请求取消阻塞者。

这类问题在 SQL 层不总是直接暴露为 lock wait。

它可能表现为 vacuum cleanup 变慢、recovery conflict、buffer allocation 压力或 wait event `BufferCleanup`。

## 9. 诊断与实验

本节可观测的 runtime truth 是：

```text
pin wait、content lock wait、cleanup wait 是三个不同现象；它们都围绕同一个 BufferDesc.state，但解释方向完全不同。
```

能直接观察：

- `pg_stat_activity.wait_event` 可看到 content lock mode wait、`BufferCleanup`、`WAIT_EVENT_BUFFER_IO` 等相关等待场景。
- `pg_buffercache` 可以看 `pinning_backends` 的快照。
- gdb 可以看 `PrivateRefCountEntry.refcount`、`lockmode`、`BufferDesc.state` 中 refcount 与 lock bits。
- perf 可以看 content lock acquire / release、CAS retry、buffer cleanup wait 是否成为热点。

只能推断：

- 某个 backend 内部对同一 buffer pin 了几次。
- private refcount hash / array 的完整状态。
- content lock wait queue 的历史。
- cleanup waiter 被哪个 pin holder 阻塞的完整因果链。

诊断时不要把 wait event 当完整归因。

content lock mode wait 说明 page content lock 上等待。

它不说明 buffer 被 replacement 选中。

`BufferCleanup` 说明 cleanup-strength lock 在等 pin count 归 1。

它不说明普通 content lock 冲突。

`pinning_backends` 是快照。

它不能告诉你谁刚刚释放，也不能告诉你本 backend 内重复 pin 次数。

实验 1：观察同 backend 重复 pin。

目标：确认 shared refcount 与 private refcount 分层。

断点：

```text
b PinBuffer
b TrackNewBufferPin
b UnpinBufferNoOwner
```

SQL：

```sql
CREATE TABLE bm_pin_demo(id int);
INSERT INTO bm_pin_demo SELECT generate_series(1, 10000);
BEGIN;
SELECT count(*) FROM bm_pin_demo;
SELECT count(*) FROM bm_pin_demo;
COMMIT;
```

观察点：

- 第一次 pin 某 buffer 时创建 private refcount entry。
- 重复 pin 同一 buffer 时 private refcount 增加。
- 最后一次 unpin 时 shared refcount 才减少。

解释：

这说明 `pinning_backends` 与“本 backend pin 了几次”不是同一指标。

实验 2：区分 content lock wait 与 pin。

目标：确认 pin 不等于 page content 互斥。

断点：

```text
b LockBufferInternal
b BufferLockAcquire
b UnlockBuffer
```

操作：

用两个 session 对同一表执行会触发 page read / update 的操作。

在一个 backend 中暂停在持有 content lock 的位置。

观察另一个 backend 的 wait event。

观察点：

- 等待出现在 content lock acquire。
- 目标 backend 可能早已 pin 到 buffer。
- pin 成功不代表能进入 page modification 临界区。

解释：

pin 只稳定 frame。

content lock 才控制 page bytes 的并发访问。

实验 3：观察 cleanup lock 的 pin count 条件。

目标：确认 cleanup lock = exclusive content lock + no other pins。

断点：

```text
b LockBufferForCleanup
b WakePinCountWaiter
b UnpinBufferNoOwner
```

操作：

在一个 backend 中让长事务扫描并停住，使它持有 buffer pin。

另一个 backend 触发需要 cleanup-strength lock 的路径，例如 VACUUM 或相关访问方法 cleanup。

观察点：

- cleanup backend 先拿 exclusive content lock。
- 如果 refcount 不为 1，设置 `BM_PIN_COUNT_WAITER`。
- pin holder unpin 后，`WakePinCountWaiter()` 发送信号。

解释：

exclusive content lock 不能保证没有其他 pin holder。

cleanup 必须额外等待 pin count 条件。

常见误区：

- 把 pin 叫成“buffer 锁”。
- 认为 `ReadBuffer()` 返回后就能任意读写 page bytes。
- 认为 content lock 可以防 replacement。
- 认为 cleanup lock 是第四种 content lock mode。
- 认为 `pinning_backends = 0` 的快照能证明刚才没有 pin。
- 认为 ERROR 后只要 MemoryContext reset 就能释放 pin。
- 认为 local buffer 完全不需要 pin。

排查时按这个顺序问：

```text
我是否持有 pin？
我是否需要读取或修改 page bytes？
如果需要，我持有什么 content lock mode？
我是否要物理删除 page item？
如果要，我是否满足 cleanup lock 条件？
ERROR 时 ResourceOwner 是否知道这个 pin？
释放最后一个 pin 前 lockmode 是否已回到 BUFFER_LOCK_UNLOCK？
```

这组问题围绕职责边界，而不是函数背诵。

## 10. 讨论题

1. 为什么 pin 只能保护 buffer frame identity，不能保护 page bytes？
2. cleanup lock 为什么不是一个新锁类型，而是 content lock 与 pin count 条件的组合？
3. 如果同一 backend 重复 pin 一个 buffer，shared refcount 和 private refcount 分别怎样变化？
4. ERROR cleanup 必须按什么顺序释放 content lock 和 pin？
5. 哪些 bug 来自把 pin、content lock、ResourceOwner 或 MemoryContext 的职责混在一起？

## 11. 本节小结

本节核心链路是：

```text
PinBuffer()
  -> private/shared refcount
  -> optional LockBuffer()
  -> page access
  -> UnlockBuffer()
  -> UnpinBuffer()
  -> ResourceOwner cleanup on ERROR
```

核心状态和边界是：

- shared refcount：跨 backend / I/O 的 pin count。
- private refcount：本 backend 对同一 buffer 的重复 pin count。
- `ResourceOwner`：ERROR / abort 时释放 pin 和可能残留的 content lock。
- content lock bits：page contents 的 shared/share-exclusive/exclusive 互斥。
- `BM_PIN_COUNT_WAITER`：cleanup lock 等待 pin count 归 1。
- local buffer：没有跨 backend content lock，但仍有 local pin lifecycle。

ownership 规则是：

```text
先 pin，才能稳定 Buffer identity；
持 pin 后，才允许获取 content lock；
释放最后一个 pin 前，必须释放 content lock；
ERROR 时 ResourceOwner 先解 content lock，再 unpin。
```

异常路径的共同模式是：

```text
如果只缺 page content 互斥，就等待 content lock；
如果要物理 cleanup，就等待其他 pin 消失；
如果 ERROR 打断正常路径，就让 ResourceOwner 按 ownership 顺序释放。
```

能观测的是 wait event、`pg_buffercache` 快照、gdb 中的 state/refcount/lockmode。

看不到的是完整 private refcount 历史和短时 lock queue 转换。

可迁移规律：

```text
共享缓存中的“正在使用”和“可以读写内容”是两种不同资格；
前者保护对象身份，后者保护对象内部状态。
```

这条边界能迁移到很多内核结构：

refcount / pin 解决 lifetime。

lock 解决并发访问。

cleanup / reclamation 往往还需要“没有其他引用者”的更强条件。
