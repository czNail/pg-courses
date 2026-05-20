# PostgreSQL ResourceOwner tree 与外部资源 ownership 边界

## 课程定位

前置知识：已经理解 `MemoryContext` 如何用 backend-local tree 承载 `palloc()` 内存生命周期，也已经知道 `ERROR` 会通过 longjmp 跳过普通 C 返回路径。

本节唯一主问题：

```text
为什么内存可以交给 MemoryContext 批量 reset，而 buffer pin、lock、snapshot、文件句柄、cache ref 这类资源必须挂到独立的 ResourceOwner tree 上？
```

核心矛盾：PostgreSQL 希望深层执行代码能用统一的生命周期 owner 兜底清理资源；但这些资源不是普通内存块，释放它们往往意味着修改共享内存 pin/refcount、通知 lock manager、撤销 snapshot 注册、关闭虚拟文件或降低 cache tuple 引用计数。它们不能靠 `pfree()` 解决，也不能混进 `MemoryContextReset()` 的内存释放顺序里。

学完后应能判断：一段代码获取的对象到底只是 backend-local 内存，还是一个必须通过 `ResourceOwner` 跟踪的外部资源；为什么 `CurrentMemoryContext` 和 `CurrentResourceOwner` 经常一起切换，但表达的是两种不同 ownership；以及 commit warning 和 abort cleanup 分别说明什么。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `6b48f5d1a741`。

## 1. 本节在总主线中的位置

前面几节已经建立了 MemoryContext 的判断框架：

```text
palloc chunk:
  属于某个 MemoryContext
  在 context reset/delete 时批量释放
  指针只在当前 backend 内有效
```

这能解释大多数 backend-local 内存 cleanup，但解释不了这些对象：

```text
buffer pin:
  pin 住 shared buffer，影响 buffer replacement 和其他 backend 的等待

heavyweight lock:
  记录在 lock manager 的 backend/transaction 持有关系里

snapshot reference:
  影响 snapshot refcount、active/registered snapshot 生命周期和 xmin 推进

virtual file:
  占用当前 backend 的 VFD entry，背后可能有 OS fd 和临时文件删除语义

catcache / tupdesc ref:
  保护缓存对象不被提前释放或重用
```

这些对象可以有一小块 descriptor 内存在当前 backend 里，但真正危险的部分不是那块内存，而是“我正在占用某个外部状态”的事实。ResourceOwner 解决的就是这层 ownership。

本节只讲第一层边界：为什么需要 ResourceOwner tree。下一节才进入 `ResourceOwnerEnlarge()`、`ResourceOwnerRemember()`、`ResourceOwnerForget()` 的 acquire-before-ERROR 细节；再往后才展开事务、Portal owner 传播和三阶段 release 顺序。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
MemoryContext 记录 palloc 内存归哪个生命周期节点；
ResourceOwner 记录外部资源归哪个 transaction / subtransaction / portal 生命周期节点；
当 owner release 时，按资源类型调用对应 ReleaseResource 回调，而不是简单 free 一块内存。
```

这里的 tension 是：

```text
深层代码需要低成本获取 pin / lock / snapshot / cache ref
  vs
ERROR、abort、Portal close 或子事务结束时必须可靠撤销这些外部占用
```

如果只靠 MemoryContext，会出现一个根本错位：

```text
MemoryContextReset(ctx)
  -> 能释放 ctx 下的 palloc chunk
  -> 不知道某个 Buffer 的 pin count 该 decrement
  -> 不知道 lock manager 里的 LOCALLOCK 该 release 还是 reassign
  -> 不知道 snapshot refcount 和 active snapshot stack 该怎么处理
  -> 不知道 VFD 对应文件是否要 close / delete
  -> 不知道 catcache tuple 的 refcount 是否该下降
```

所以 PostgreSQL 没有把 ResourceOwner 做成 MemoryContext 的一个 callback 清单。两者相似，但语义边界不同：

```text
MemoryContext:
  释放动作主要是 allocator-local 的内存回收

ResourceOwner:
  释放动作是模块语义回调，可能触碰 shared state、refcount、lock table、file table 或 snapshot manager
```

ResourceOwner 对象本身确实用 `TopMemoryContext` 分配，但这是实现细节。它记录的资源不是 `TopMemoryContext` 的内存 ownership，而是“这些外部占用必须在 owner 生命周期结束时被语义释放”。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 作用 |
| --- | --- | --- |
| 1 | `src/backend/utils/resowner/README` | 解释 ResourceOwner 的设计动机：可靠跟踪 query-related resources，并说明它借鉴但不合并 MemoryContext。 |
| 2 | `src/include/utils/resowner.h` | 暴露 `ResourceOwner`、`CurrentResourceOwner`、release phase、`ResourceOwnerDesc` 和 public API。 |
| 3 | `src/backend/utils/resowner/resowner.c` | 实现 owner tree、资源数组/hash、release recursion、callback 调用和 delete 不变量。 |
| 4 | `src/backend/access/transam/xact.c` | 创建 top transaction / subtransaction owner，并在 commit/abort 中 release/delete。 |
| 5 | `src/backend/tcop/pquery.c` | Portal 执行时切换 `CurrentResourceOwner`，说明一条 query 的资源为何常归 Portal。 |
| 6 | `src/backend/storage/buffer/bufmgr.c` | buffer pin 作为 `BEFORE_LOCKS` 资源被 owner 跟踪，释放时真正 unpin。 |
| 7 | `src/backend/storage/lmgr/lock.c` | lock ownership 的特殊路径：commit 子 owner 时转移给父 owner，abort 时释放。 |
| 8 | `src/backend/utils/time/snapmgr.c` | snapshot reference 挂到 owner，释放时撤销 snapshot ref。 |
| 9 | `src/backend/storage/file/fd.c` | VFD / file 资源挂到 owner，避免 ERROR 后文件句柄遗留。 |
| 10 | `src/backend/utils/cache/catcache.c` | catcache ref 通过 owner 兜底降低引用计数，避免 cache tuple 长期 pinned。 |

阅读时不要从 `resowner.c` 的数组/hash 实现开始。先抓住这条线：

```text
谁设置 CurrentResourceOwner
  -> 深层模块获取资源时记到哪个 owner
  -> 正常路径显式 forget
  -> ERROR / abort / Portal close 时 owner bulk release
```

## 4. 关键数据结构与状态

### `ResourceOwner`

公开头文件里 `ResourceOwner` 是不透明指针：

```text
typedef struct ResourceOwnerData *ResourceOwner;
```

真正结构在 `resowner.c`，本节只需要关注这些状态组合：

| 字段 | 语义 |
| --- | --- |
| `parent` / `firstchild` / `nextchild` | ResourceOwner tree；release parent 会递归处理 child。 |
| `name` | 诊断用名称，例如 `TopTransaction`、`SubTransaction`。 |
| `releasing` / `sorted` | release 开始后的状态边界；开始 bulk release 后不能再 retail remember/forget。 |
| `arr` / `hash` | 普通资源记录区，保存 `(resource datum, ResourceOwnerDesc)`。 |
| `locks` | lock 的小型专用缓存；lock 路径足够热，且 commit 语义和普通资源不同。 |
| `aio_handles` | AIO handle 专用链表；有些资源必须在 critical section 中登记，不能走普通数组机制。 |

最重要的不变量是：

```text
ResourceOwner 是 backend-local ownership 账本；
账本里的资源可以指向 shared state、OS fd 或 cache ref；
释放账本时必须调用资源类型自己的 ReleaseResource 语义函数。
```

这和 MemoryContext 的边界正好错开：

```text
MemoryContext tree:
  管“这块内存什么时候 free”

ResourceOwner tree:
  管“这个外部占用什么时候撤销”
```

### `ResourceOwnerDesc`

每种可跟踪资源通过 `ResourceOwnerDesc` 描述：

```text
name
release_phase
release_priority
ReleaseResource
DebugPrint
```

这几个字段说明 ResourceOwner 不是一个通用 `void *` 容器，而是一个带释放语义的调度层：

```text
release_phase:
  资源应该在锁释放前、锁释放时，还是锁释放后处理

release_priority:
  同一 phase 内的相对顺序

ReleaseResource:
  真正撤销外部占用的模块回调

DebugPrint:
  commit 时发现资源没被正常释放，用于 WARNING 诊断
```

例如 `bufmgr.c` 中 buffer pin 的 descriptor 设置为 `RESOURCE_RELEASE_BEFORE_LOCKS`；`snapmgr.c` 中 snapshot reference 设置为 `RESOURCE_RELEASE_AFTER_LOCKS`。这不是偶然顺序，而是说明不同资源的 cleanup 依赖不同 correctness 边界。

### `CurrentResourceOwner`

`CurrentResourceOwner` 是获取 query-lifespan resource 时的隐式 owner。它的角色类似 `CurrentMemoryContext`，但语义完全不同：

```text
CurrentMemoryContext:
  没显式指定 context 的 palloc 去哪里

CurrentResourceOwner:
  没显式指定 owner 的 buffer pin / lock / snapshot / cache ref 去哪里
```

`xact.c` 在事务开始时创建 `TopTransaction` owner，并设置：

```text
TopTransactionResourceOwner = s->curTransactionOwner
CurTransactionResourceOwner = s->curTransactionOwner
CurrentResourceOwner = s->curTransactionOwner
```

`pquery.c` 在执行 Portal 时临时切换：

```text
saveResourceOwner = CurrentResourceOwner
CurrentResourceOwner = portal->resowner
...
CurrentResourceOwner = saveResourceOwner
```

因此一段 deep code 调用 `ReadBuffer()`、`RegisterSnapshot()` 或 `SearchSysCache()` 时，通常不需要显式传 owner。当前执行上下文已经决定了这次外部资源归属于哪个生命周期。

这也带来一个边界要求：

```text
释放资源时，CurrentResourceOwner 通常必须回到获取资源时的同一个 owner。
```

ResourceOwner README 直接说明了这一点。它避免每个资源都携带更重的反向索引，也让忘记切回 owner 的 bug 更容易暴露为“不属于这个 owner”的错误。

## 5. 主流程源码 walkthrough

本节主流程选一条最常见的查询执行路径：一个 Portal 执行时读取 buffer，并在正常结束或 ERROR 后释放资源。

### 5.1 事务开始：建立顶层 owner

`xact.c` 的 `AtStart_ResourceOwner()` 创建 top transaction owner：

```text
StartTransaction()
  -> AtStart_ResourceOwner()
     -> ResourceOwnerCreate(NULL, "TopTransaction")
     -> TopTransactionResourceOwner = owner
     -> CurTransactionResourceOwner = owner
     -> CurrentResourceOwner = owner
```

这个 owner 的生命周期长于单条语句，但短于 backend。它承载事务级资源，也作为子事务和 Portal owner 的父级或回收锚点。

ResourceOwner 对象本身由 `ResourceOwnerCreate()` 分配在 `TopMemoryContext`：

```text
ResourceOwnerCreate(parent, name)
  -> MemoryContextAllocZero(TopMemoryContext, sizeof(ResourceOwnerData))
  -> 挂到 parent->firstchild
```

这常被误读成“ResourceOwner 也是 MemoryContext 的一部分”。更准确地说：

```text
owner 的账本内存需要显式 delete；
owner 记录的外部资源必须先 release；
release 和 delete 是两个不同动作。
```

`ResourceOwnerDelete()` 里也有断言：owner 不应再拥有普通资源，且不能删除当前正在使用的 `CurrentResourceOwner`。

### 5.2 Portal 执行：切换当前资源归属

`pquery.c` 中 `PortalStart()` 会保存当前全局状态，然后把执行上下文切到 Portal：

```text
PortalStart(portal)
  -> saveResourceOwner = CurrentResourceOwner
  -> ActivePortal = portal
  -> CurrentResourceOwner = portal->resowner
  -> PortalContext = portal->portalContext
  -> MemoryContextSwitchTo(PortalContext)
  -> ExecutorStart(...)
  -> restore CurrentResourceOwner / PortalContext
```

这里 `PortalContext` 和 `portal->resowner` 经常一起出现，但边界不同：

```text
PortalContext:
  QueryDesc、参数、executor 工作对象等 palloc 内存归属

portal->resowner:
  执行期间获取的 buffer pin、snapshot ref、cache ref、file、lock 等外部资源归属
```

一个 Portal 失败时，`PG_CATCH` 会把 `ActivePortal`、`CurrentResourceOwner`、`PortalContext` 恢复到进入前状态，再 `PG_RE_THROW()`。这说明 ResourceOwner tree 不是替代 `PG_TRY` 的控制流恢复机制；它负责资源账本，`PG_CATCH` 仍要恢复全局指针。

### 5.3 读取 buffer：外部资源被记到当前 owner

`bufmgr.c` 的文件头已经说明 `ReadBuffer()` 的语义：

```text
ReadBuffer()
  -> 找到或创建保存目标 page 的 buffer
  -> pin it so that no one can destroy it while this process is using it
```

pin 的后果不是“分配一块内存”，而是：

```text
当前 backend 的 private refcount 增加
shared buffer replacement 不能随意淘汰这个 buffer
其他路径可能因为 pin 存在而等待或退避
```

所以 buffer pin 的 ResourceOwner descriptor 是：

```text
name: buffer
release_phase: RESOURCE_RELEASE_BEFORE_LOCKS
release_priority: RELEASE_PRIO_BUFFER_PINS
ReleaseResource: ResOwnerReleaseBuffer
```

正常路径中，调用者用完 buffer 后调用 `ReleaseBuffer()`：

```text
ReleaseBuffer(buffer)
  -> ResourceOwnerForgetBuffer(CurrentResourceOwner, buffer)
  -> UnpinBuffer(...)
```

如果深层代码在 pin 后遇到 `ERROR`，普通函数尾部的 `ReleaseBuffer()` 可能不会执行。此时 owner release 会调用 `ResOwnerReleaseBuffer()` 兜底 unpin。

这正是 ResourceOwner 的核心价值：

```text
不是替调用者 free 内存；
而是在调用者没有机会正常返回时，仍能撤销外部占用。
```

### 5.4 正常提交：剩余资源是诊断信号

在 `CommitTransaction()` 的 post-commit cleanup 中，`xact.c` 先释放对其他 backend 可见的资源，再释放锁，再释放 backend-local 资源：

```text
CurrentResourceOwner = NULL
ResourceOwnerRelease(TopTransactionResourceOwner, BEFORE_LOCKS, true, true)
...
ResourceOwnerRelease(TopTransactionResourceOwner, LOCKS, true, true)
ResourceOwnerRelease(TopTransactionResourceOwner, AFTER_LOCKS, true, true)
...
ResourceOwnerDelete(TopTransactionResourceOwner)
```

`isCommit = true` 很重要。ResourceOwner 注释说，commit 时如果还剩普通资源，通常表示 executor 或调用者没有按正常路径清理干净，因此会打印 WARNING，而不是静默吞掉。

这和 abort 不同：

```text
commit:
  正常路径本应 retail release / forget 大多数资源
  剩余资源倾向于是 bug 或异常 ownership 边界

abort:
  ERROR 已经跳过普通 cleanup
  剩余资源由 ResourceOwner bulk release 是预期行为
```

## 6. 生命周期 / ownership / cleanup

### 谁创建

常见 owner 创建点：

| owner | 创建位置 | 生命周期语义 |
| --- | --- | --- |
| `TopTransaction` | `AtStart_ResourceOwner()` | 当前顶层事务。 |
| `SubTransaction` | `AtSubStart_ResourceOwner()` | 当前子事务；parent 是父事务 owner。 |
| Portal owner | Portal 创建路径 | 当前 Portal 执行生命周期；执行时 `CurrentResourceOwner` 指向它。 |
| `AuxiliaryProcess` | `CreateAuxProcessResourceOwner()` | 辅助进程中需要 owner 跟踪资源时使用。 |

本节重点是 top transaction 和 Portal：它们解释了为什么同一条 SQL 里经常同时有事务 owner 和 Portal owner。

### 谁持有

ResourceOwner tree 是 backend-local。其他 backend 不能直接访问你的 owner 指针，也不会替你遍历 owner tree。

但 owner 中的 resource datum 可能指向或代表外部状态：

```text
Buffer:
  shared buffer id + 当前 backend private refcount

LOCALLOCK:
  lock manager 的本地锁对象

Snapshot:
  当前 backend 的 snapshot 对象和注册引用

File:
  VFD 编号，背后可能映射 OS fd

HeapTuple / CatCList:
  catcache 内部缓存对象引用
```

因此不要把 ResourceOwner 理解成“资源对象所在的内存容器”。它更像是当前 backend 的 release ledger。

### 谁释放

释放分两层：

```text
retail release:
  正常路径显式 ReleaseBuffer / UnlockReleaseBuffer / ReleaseCatCache / UnregisterSnapshot 等
  同时 ResourceOwnerForget

bulk release:
  Portal close、transaction commit/abort、subtransaction end 时调用 ResourceOwnerRelease
  对剩余资源逐个调用 ResourceOwnerDesc.ReleaseResource
```

`ResourceOwnerDelete()` 只删除 owner 对象和子 owner 对象。它要求资源已经 release 完成。把 delete 当成 release 是常见误区。

### ERROR / abort 时谁兜底

ERROR 路径中，deep code 可能跳过这些普通 cleanup：

```text
ReleaseBuffer()
ReleaseCatCache()
UnregisterSnapshot()
FileClose()
LockReleaseCurrentOwner()
```

事务 abort 路径会通过 `ResourceOwnerRelease(..., isCommit = false, ...)` 释放剩余资源。此时不应该把每个剩余资源都当成 leak；这正是 ResourceOwner 的存在理由。

但 ResourceOwner 也不是万能兜底：

```text
LWLock:
  通常必须在 PG_TRY/PG_CATCH 或 abort hook 中显式释放，不能依赖普通 ResourceOwner entry

MemoryContext switch:
  需要 PG_CATCH 恢复 CurrentMemoryContext

ActivePortal / CurrentResourceOwner:
  需要 Portal 或顶层 handler 恢复全局指针

shared-memory transient flag:
  可能需要 before_shmem_exit / on_shmem_exit 兜底
```

本节要形成的边界感是：

```text
ResourceOwner 管可登记、可语义释放的外部资源；
它不自动恢复所有 backend 全局执行状态。
```

## 7. 正确性机制层次

ResourceOwner 不单独保证 correctness。它只是把 cleanup 责任集中起来，让其他机制能在正确时刻解除占用。

| 机制 | 本节中的作用 | 不要误解为 |
| --- | --- | --- |
| `MemoryContext` | 管 `palloc` 内存生命周期。 | 自动释放 pin、lock、snapshot、fd。 |
| `ResourceOwner` | 记录外部资源 ownership，并在生命周期结束时调用 release 回调。 | 资源本身的并发控制或语义有效性。 |
| pin / refcount | 表示对象正在被当前 backend 使用。 | 对象内容一定没有语义过期。 |
| heavyweight lock | 管逻辑对象并发访问和等待。 | 替代 buffer pin 或 cache refcount。 |
| invalidation | 通知缓存语义过期。 | 自动释放缓存对象引用。 |
| snapshot manager | 管 snapshot 的 active / registered 引用和 xmin 影响。 | 自动知道哪个 Portal 或事务应该释放它。 |

以 buffer 为例：

```text
Buffer pin:
  防止 buffer 被淘汰或重用

Buffer content lock:
  保护 page 内容的并发读写

Relation lock:
  保护 relation 级 DDL/DML 语义

ResourceOwner:
  确保 ERROR 后 pin 不会永久遗留
```

这些机制互相配合，但没有一个能替代全部。

## 8. 错误路径 / 异常路径 / fallback

### 8.1 `ERROR` 跳过普通函数尾部

典型风险：

```text
buf = ReadBuffer(rel, blkno)
LockBuffer(buf, BUFFER_LOCK_SHARE)
...
ereport(ERROR, ...)
UnlockReleaseBuffer(buf)  -- 不会执行
```

如果这里没有 ResourceOwner，`ERROR` 后 buffer pin 可能遗留到事务结束甚至更久，影响 buffer replacement 和后续等待。ResourceOwner 让 abort 路径仍能发现这个 pin 并调用 buffer manager 的 release 回调。

### 8.2 commit 时发现资源未释放

commit 上仍有普通资源，ResourceOwner 会给 WARNING。这类 WARNING 的意义不是“PostgreSQL 已经帮你安全处理，所以不用管”，而是：

```text
正常路径没有按 ownership 协议释放资源；
虽然 ResourceOwner 能兜底，但代码可能隐藏了生命周期错误。
```

扩展或内核 patch 中，如果 commit warning 出现在自定义资源上，优先检查：

```text
获取资源后是否记到了正确 owner
正常释放时是否调用 Forget
释放时 CurrentResourceOwner 是否和获取时一致
Portal / subtransaction 结束时资源是否该转移给父 owner
```

### 8.3 release 开始后不能继续 retail 操作

`resowner.c` 在 release 开始后设置 `releasing`，并排序资源数组/hash。此后 release callback 不能再调用普通 `ResourceOwnerRemember()` / `ResourceOwnerForget()` 操作同一个 owner。

这不是实现洁癖，而是为了保证 bulk release 遍历不会一边扫描一边改变集合，导致漏释放或重复释放。若资源释放需要额外资源，应该在资源类型自己的低层 cleanup 中处理，且 callback 必须尽量不失败。

## 9. 成本、资源与跨模块传播

ResourceOwner 位于 hot path，因此它的成本模型很具体：

```text
每次获取 query-lifespan resource:
  可能需要 Enlarge + Remember

每次正常释放:
  需要 Forget，从小数组或 hash 中移除

事务 / Portal 结束:
  需要递归 owner tree，并按 phase / priority release 剩余资源
```

它的开销随这些变量扩张：

| 变量 | 成本传播 |
| --- | --- |
| 单 query 持有的 buffer pin / cache ref 数量 | `arr` 溢出到 hash，forget/release 成本增加。 |
| 子事务层级 | owner tree 变深，commit/abort release 递归增加。 |
| Portal 数量 | cursor / extended query protocol 中 owner 数量增加。 |
| lock 数量 | lock 专用缓存可能 overflow，retail release/reassign 退化。 |
| ERROR 频率 | abort bulk release 从异常路径变成高频成本。 |

这解释了为什么 ResourceOwner 不能是一个过度抽象、每个资源都 malloc 一个 wrapper 的机制。`resowner.c` 使用固定小数组、开放寻址 hash、lock 专用缓存和 AIO 专用链表，是在 correctness 和 hot path 成本之间折中。

跨模块边界也要看清：

```text
executor / buffer manager:
  获取 buffer pin，ResourceOwner 兜底释放

lock manager:
  子事务 commit 时锁可能 reassign 给父 owner，abort 时释放

snapshot manager:
  owner release 降低 snapshot ref，影响 xmin 相关清理边界

catcache / relcache:
  cache ref 必须按引用计数释放，invalidation 只说明语义过期

storage/file:
  VFD 与 OS fd 管理不能交给 MemoryContextReset
```

这些模块不会因为一块内存被 free 就自动撤销状态。ResourceOwner 提供的是统一调度入口，不是替代各模块的资源语义。

## 10. 观测与诊断入口

本节最适合锚定的 runtime truth 是：

```text
ERROR 后 MemoryContext 能回收内存，但外部资源必须依赖 ResourceOwner 或模块 abort cleanup 才能撤销。
```

可以从三类入口观察。

### server log

如果 commit 路径发现资源没有按正常路径释放，ResourceOwner 可能打印 WARNING。WARNING 中的资源名来自 `ResourceOwnerDesc.name`，更详细文本来自 `DebugPrint`。

能看到：

```text
某类资源在 commit 时仍挂在 owner 上
```

看不到：

```text
最初是谁获取了这个资源
为什么 CurrentResourceOwner 当时指向这个 owner
资源泄露前完整调用栈
```

通常需要配合 gdb 断点或临时日志。

### SQL 视图

`pg_locks` 能看到当前 backend 或其他 backend 的 lock 状态，但它不是 ResourceOwner tree 的直接视图。

能看到：

```text
某个 backend 当前持有什么 lock
哪些 backend 在等待
```

看不到：

```text
lock 是哪个 Portal owner 或 subtransaction owner 记录的
commit 时会 release 还是 reassign
```

buffer pin、catcache ref、snapshot ref 也没有一个完整的 SQL 视图能直接映射到 ResourceOwner entry。很多情况下只能从等待、WARNING、断点和源码路径推断。

### gdb / 临时日志

更直接的源码实验入口：

```text
break ResourceOwnerRemember
break ResourceOwnerForget
break ResourceOwnerRelease
break ResOwnerReleaseBuffer
break ResOwnerReleaseSnapshot
```

观察重点不是单个函数参数，而是：

```text
CurrentResourceOwner->name 是什么
资源是在 Portal owner 还是 TopTransaction owner 上
ERROR 后是否走到 isCommit=false 的 ResourceOwnerRelease
commit warning 出现前资源是否没有正常 Forget
```

注意推断边界：ResourceOwner 是 backend-local 账本，别的 backend 不能通过 SQL 直接看到你的 owner tree。你看到的等待或 lock 状态是外部结果，不是 owner tree 本身。

## 11. 常见误区

### 误区 1：把 ResourceOwner 当成 MemoryContext 的别名

两者都形成 tree，也都有 current 指针，但释放语义完全不同：

```text
MemoryContextReset:
  释放内存块

ResourceOwnerRelease:
  调用资源类型回调撤销外部占用
```

ResourceOwner 对象放在 `TopMemoryContext`，不等于资源也归 `TopMemoryContext` 管。

### 误区 2：认为 `pfree()` 一个 descriptor 就释放了资源

如果一个对象代表 buffer pin、snapshot ref 或 file handle，只释放本地 descriptor 内存通常会丢失真正 cleanup 所需的信息，反而更危险。

正确判断是：

```text
释放这东西是否需要通知另一个模块、降低 refcount、unpin、unlock、close 或 unregister？
```

如果需要，它通常不能只靠 MemoryContext。

### 误区 3：认为 abort 时剩余资源就是 leak

abort 时剩余资源是 ResourceOwner 的正常工作场景。commit 时剩余资源才更像协议错误信号。

### 误区 4：把 invalidation 当成引用释放

catcache / relcache invalidation 只说明缓存语义可能过期。只要你仍持有 ref，缓存对象就不能按普通路径释放。ResourceOwner 兜底的是 refcount 释放，不是 invalidation 传播。

### 误区 5：忽略 owner 切换恢复

`CurrentResourceOwner` 是全局 backend-local 指针。Portal、SPI、扩展 hook 或子事务路径中忘记恢复它，会让后续资源挂到错误生命周期上，症状可能延迟到 commit warning、abort cleanup 或“不属于 owner”的 ERROR。

## 12. 课堂实验

### 实验 1：跟读 Portal owner 与 MemoryContext 同步切换

源码入口：

```text
src/backend/tcop/pquery.c
  PortalStart()
```

任务：

```text
1. 画出进入 PortalStart 前的 CurrentResourceOwner / CurrentMemoryContext。
2. 标出 portal->resowner 和 portal->portalContext 被设为当前状态的位置。
3. 标出 PG_CATCH 中恢复全局变量的位置。
4. 回答：为什么只恢复 MemoryContext 不够，只恢复 ResourceOwner 也不够？
```

### 实验 2：观察 buffer pin 的 owner 登记和兜底释放

源码入口：

```text
src/backend/storage/buffer/bufmgr.c
  ReadBuffer()
  ReleaseBuffer()
  ResOwnerReleaseBuffer()
```

建议断点：

```text
break ResourceOwnerRemember
break ResourceOwnerForget
break ResOwnerReleaseBuffer
```

任务：

```text
1. 执行一条会访问表页的 SELECT。
2. 观察 buffer pin 记到哪个 CurrentResourceOwner。
3. 在正常路径中确认 ReleaseBuffer 会 Forget。
4. 人为在调试环境制造 ERROR，观察 abort 时是否走 ResOwnerReleaseBuffer。
```

### 实验 3：比较 commit warning 与 abort cleanup 的语义

修改源码实验：

```text
在一条受控测试路径中，临时注释掉某个正常 Release/Forget 调用。
```

任务：

```text
1. 正常 commit 时观察是否出现 ResourceOwner WARNING。
2. ERROR / abort 路径中观察同类资源是否被静默 bulk release。
3. 解释为什么同样是“owner 里还有资源”，commit 和 abort 的诊断意义不同。
```

这个实验只在本地调试分支做，不要把故意破坏 cleanup 的 patch 带入正式代码。

## 13. 讨论题

1. 为什么 `ResourceOwnerCreate()` 把 owner 对象分配在 `TopMemoryContext`，但仍然要求 `ResourceOwnerDelete()` 显式删除？
2. 如果 buffer pin 只靠 MemoryContext reset，会漏掉哪一层共享状态变化？
3. `CurrentMemoryContext` 和 `CurrentResourceOwner` 同时切换时，分别改变了什么归属关系？
4. 为什么 commit 时剩余资源应该 warning，而 abort 时剩余资源通常不 warning？
5. 子事务 commit 时，为什么 lock 可能需要转移给父 owner，而不是直接释放？
6. `pg_locks` 能帮助诊断 ResourceOwner 问题的哪一部分？它看不到什么？
7. 如果扩展要登记一种自定义外部资源，`ReleaseResource` 回调为什么不能执行可能失败的复杂操作？
8. invalidation、refcount、ResourceOwner 三者分别解决什么问题，为什么不能互相替代？

## 14. 本节小结

本节只建立一个边界判断：

```text
MemoryContext 管 backend-local 内存块；
ResourceOwner 管必须按模块语义撤销的外部资源占用。
```

主链路是：

```text
事务开始创建 TopTransaction owner
  -> Portal 执行时切换 CurrentResourceOwner
  -> buffer / lock / snapshot / file / cache ref 记到当前 owner
  -> 正常路径 retail release + forget
  -> ERROR / abort / Portal close 时 bulk release 剩余资源
  -> release 完成后 delete owner 对象
```

核心不变量是：ResourceOwner tree 是 backend-local 账本，但账本里的条目通常代表对 shared state、文件、snapshot 或 cache 对象的外部占用。释放 owner 不是 free 一块内存，而是按 `ResourceOwnerDesc` 调用资源类型自己的 cleanup。

可观测上，ResourceOwner tree 本身没有完整 SQL 视图。`pg_locks`、wait event、server log WARNING、gdb 断点和临时日志只能分别看到外部结果或局部状态。诊断时要把“能直接看到的 lock/wait/WARNING”和“只能从源码路径推断的 owner 归属”分开。

可迁移的系统规律是：

```text
只要一个对象的释放会改变内存 allocator 之外的状态，
它就不应该只依赖内存生命周期；
它需要一个能在正常路径和 ERROR 路径都执行语义 cleanup 的 owner。
```

下一节会沿着同一条边界继续问：既然外部资源必须登记到 owner，为什么 PostgreSQL 要求在真正 acquire 之前先 `ResourceOwnerEnlarge()`，以及 `Remember/Forget` 如何收住“资源已获取但登记可能 ERROR”的窗口。
