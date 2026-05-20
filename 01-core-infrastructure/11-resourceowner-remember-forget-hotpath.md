# PostgreSQL Remember / Forget hot path 与 acquire-before-ERROR 安全

## 课程定位

前置知识：已经理解 `ResourceOwner` tree 为什么独立于 `MemoryContext`，也知道 PostgreSQL 的 `ERROR` 会通过 longjmp 跳过普通 C 调用栈返回路径。

本节唯一主问题：

```text
为什么 ResourceOwnerEnlarge() 必须在真正获取资源前调用，ResourceOwnerRemember() / ResourceOwnerForget() 如何在 buffer pin、tupledesc refcount、临时文件等路径上把“获取成功但随后 ERROR”的窗口收住？
```

核心矛盾：PostgreSQL 希望资源获取路径足够热、足够轻；但一旦获取成功，这个资源就可能已经改变 shared buffer refcount、TupleDesc refcount、VFD 状态或 OS 文件状态。此后如果再因为分配内存、扩容账本或错误分支抛出 `ERROR`，普通调用者就没有机会手工释放资源。

学完后应能判断：一段获取资源的代码为什么要把 `ResourceOwnerEnlarge(CurrentResourceOwner)` 放在真正 acquire 之前；为什么 `ResourceOwnerRemember()` 之后的每一个正常释放路径都必须配对 `ResourceOwnerForget()`；以及 commit warning 和 abort cleanup 分别在检查什么。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `6b48f5d1a741`。

## 1. 本节在总主线中的位置

上一节回答的是：

```text
为什么外部资源不能交给 MemoryContext reset？
```

本节往下一层，回答：

```text
既然需要 ResourceOwner，为什么登记资源还要拆成 Enlarge -> acquire -> Remember -> Forget？
```

如果只看接口名字，很容易把它想成普通容器操作：

```text
Remember(resource)
Forget(resource)
```

但 PostgreSQL 的真实问题不是“把一个值放进数组”这么简单，而是：

```text
某个资源一旦 acquire 成功，释放责任立刻产生；
而登记这个责任本身如果需要分配内存，就也可能 ERROR。
```

所以 hot path 被拆成两个阶段：

```text
ResourceOwnerEnlarge(owner)
  在尚未占用外部资源前，提前准备账本空间

真正获取资源
  pin buffer / ++tdrefcount / open File

ResourceOwnerRemember(owner, resource, kind)
  在已经预留空间的前提下，快速记录 ownership
```

正常路径释放时则反过来：

```text
ResourceOwnerForget(owner, resource, kind)
  从账本中删掉这一笔 ownership

真正释放资源
  unpin buffer / --tdrefcount / FileClose
```

这节课不展开 ResourceOwner tree 如何在事务、子事务和 Portal 之间传播，那是下一节。这里专注一个更小的不变量：

```text
不能让“外部资源已获取、ResourceOwner 尚未记住”这个状态跨过任何可能 ERROR 的点。
```

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
ResourceOwnerEnlarge() 把可能失败的账本扩容提前；
ResourceOwnerRemember() 在资源获取成功后用预留空间登记 ownership；
ResourceOwnerForget() 在正常释放路径上删除 ownership；
如果 ERROR 跳过正常路径，ResourceOwnerRelease() 用剩余记录批量兜底释放。
```

本节 tension 是：

```text
hot path 希望 ResourceOwnerRemember() 足够便宜
  vs
ERROR-safe cleanup 要求每个已获取资源都必须可被 owner 找回
```

为什么不能直接在 `ResourceOwnerRemember()` 里按需扩容？

因为按需扩容可能调用 `MemoryContextAllocZero(TopMemoryContext, ...)`。如果资源已经获取成功，再进入 `Remember()` 时 OOM：

```text
pin buffer 成功
  -> shared buffer refcount 已增加
  -> ResourceOwnerRemember() 试图扩容
  -> OOM ERROR
  -> caller 的 ReleaseBuffer() 路径被 longjmp 跳过
  -> owner 账本里没有这次 pin
  -> abort cleanup 也找不到它
```

这就是 acquire-before-ERROR 的危险窗口。PostgreSQL 的修正方式不是“让所有调用者写复杂的 PG_TRY”，而是形成统一纪律：

```text
在 acquire 前执行所有可能为了登记 ownership 而失败的准备动作。
```

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 作用 |
| --- | --- | --- |
| 1 | `src/backend/utils/resowner/resowner.c` | 实现 `ResourceOwnerEnlarge()`、`ResourceOwnerRemember()`、`ResourceOwnerForget()`，本节的核心不变量都在这里。 |
| 2 | `src/include/utils/resowner.h` | 暴露 generic resource owner API，说明 release callback 期间不应再 `Forget()`。 |
| 3 | `src/backend/utils/resowner/README` | 明确普通资源正常 commit 前应被 `Forget()`，abort 时才依赖 owner 兜底 release。 |
| 4 | `src/backend/storage/buffer/bufmgr.c` | buffer pin 的 acquire-before-remember 主路径，包含 `ReservePrivateRefCountEntry()` 和 `TrackNewBufferPin()`。 |
| 5 | `src/include/storage/buf_internals.h` | buffer resource owner wrapper：`ResourceOwnerRememberBuffer()` / `ResourceOwnerForgetBuffer()`。 |
| 6 | `src/backend/access/common/tupdesc.c` | TupleDesc refcount 的最小路径：`IncrTupleDescRefCount()` / `DecrTupleDescRefCount()`。 |
| 7 | `src/include/access/tupdesc.h` | `PinTupleDesc` / `ReleaseTupleDesc` 宏说明 refcounted TupleDesc 才进入 ResourceOwner。 |
| 8 | `src/backend/storage/file/fd.c` | 临时文件的 `OpenTemporaryFile()`、`RegisterTemporaryFile()`、`FileClose()` 与 VFD cleanup。 |

推荐阅读顺序不是从所有 resource kind 横向扫一遍，而是沿同一个状态故事读三遍：

```text
提前预留账本空间
  -> 获取外部资源
  -> 立刻登记 ownership
  -> 正常路径 forget
  -> ERROR / abort 路径由 ResourceOwnerRelease callback 兜底
```

## 4. 关键数据结构与状态

### `ResourceOwnerEnlarge()` 预留的是“下一次登记”的失败边界

`resowner.c` 里 `ResourceOwnerEnlarge()` 的注释直接给出设计目的：

```text
Make sure there is room for at least one more resource in an array.
This is separate from actually inserting a resource because if we run out
of memory, it's critical to do so before acquiring the resource.
```

它做的事情包括：

```text
如果 owner->arr 还有空间：
  立即返回，不分配内存

如果 array 满了且 hash 也需要增长：
  在 TopMemoryContext 分配或扩大 hash table
  把 array 中已有记录搬到 hash
  清空 array，为下一次 Remember 留出位置
```

关键点不是 hash table 本身，而是这条边界：

```text
Enlarge 可能 ERROR；
Remember 在正确调用 Enlarge 之后不应该因为没有空间而 ERROR。
```

因此源码还特别提醒：

```text
ResourceOwnerEnlarge() 和对应的 ResourceOwnerRemember() 之间
不要夹入无关的 ResourceOwnerRemember() 调用。
```

否则预留出来的位置可能被别的资源消耗，`Remember()` 仍然可能报：

```text
ResourceOwnerRemember called but array was full
```

这不是运行时可恢复错误，而是调用者破坏了 acquire 前预留纪律。

### `ResourceOwnerRemember()` 只追加一笔 ownership

`ResourceOwnerRemember()` 做的事非常少：

```text
检查 ResourceOwnerDesc 有 release phase / priority
检查 owner 没有进入 releasing / sorted 状态
确认 array 有空位
把 (value, kind) append 到 owner->arr
```

这里的 `value` 只是资源标识：

```text
buffer pin:
  Int32GetDatum(buffer)

TupleDesc ref:
  PointerGetDatum(tupdesc)

File:
  Int32GetDatum(file)
```

真正释放语义不在 `value` 里，而在 `ResourceOwnerDesc` 里：

```text
kind->ReleaseResource(value)
```

这也解释了为什么 ResourceOwner 是外部资源账本，而不是类型安全容器。它登记的是：

```text
某个 owner 对某类资源持有一笔释放责任。
```

### `ResourceOwnerForget()` 删除一笔 ownership，不代表资源天然安全

`ResourceOwnerForget()` 的语义是：

```text
从 owner 的 array/hash 中找到一条 (value, kind) 记录并删除；
如果同一个资源被登记多次，只删除一条；
如果找不到，ERROR。
```

“只删除一条”很重要。buffer pin 和 TupleDesc refcount 都允许同一资源被同一 backend 持有多次：

```text
同一个 Buffer 被 pin 两次：
  owner 中有两条 buffer 记录
  private refcount 是 2
  ReleaseBuffer() 一次只 Forget 一条，并减少一次 refcount
```

`Forget()` 找不到资源时抛 `ERROR`，这通常说明调用者有 ownership bug：

```text
重复释放
  或
当前 CurrentResourceOwner 不是当初 remember 的 owner
  或
资源从未被 remember
```

所以 `Forget()` 既是正常释放动作的一部分，也是一个强诊断点。

## 5. 主流程源码 walkthrough

### 5.1 Buffer pin：先准备账本和 private refcount，再碰 shared buffer

buffer pin 是本节最典型的例子，因为它同时涉及：

```text
ResourceOwner 账本
backend-local PrivateRefCountEntry
shared buffer header refcount
可能持有 buffer content lock
```

以 `BufferAlloc()` 为入口，源码在真正 lookup / pin buffer 之前先做：

```text
ResourceOwnerEnlarge(CurrentResourceOwner);
ReservePrivateRefCountEntry();
```

这两个动作分别提前处理两类可能失败或不适合在临界路径中处理的问题：

```text
ResourceOwnerEnlarge:
  为后续 ResourceOwnerRememberBuffer() 预留账本空间

ReservePrivateRefCountEntry:
  为后续 NewPrivateRefCountEntry() 预留 backend-local pin 计数槽位
```

`bufmgr.c` 的注释强调 `ReservePrivateRefCountEntry()` 和 `NewPrivateRefCountEntry()` 的拆分目的：

```text
先 reserve，后 fill；
避免 NewPrivateRefCountEntry() 在一些场景里需要分配内存，
因为它可能在持有 spinlock 的路径附近被调用。
```

主链路可以压缩为：

```text
BufferAlloc()
  -> ResourceOwnerEnlarge(CurrentResourceOwner)
  -> ReservePrivateRefCountEntry()
  -> 找到或选择 BufferDesc
  -> PinBuffer() / PinBuffer_Locked()
     -> 增加 shared buffer refcount
     -> TrackNewBufferPin()
        -> NewPrivateRefCountEntry(buffer)
        -> local refcount++
        -> ResourceOwnerRememberBuffer(CurrentResourceOwner, buffer)
```

这里最危险的一瞬间是：

```text
shared buffer refcount 已经增加
但 ResourceOwner 还没登记 buffer pin
```

PostgreSQL 通过两个提前动作把这个窗口变短：

```text
ResourceOwnerRememberBuffer() 不应再需要扩容；
NewPrivateRefCountEntry() 不应再需要寻找或分配槽位。
```

正常释放路径是 `ReleaseBuffer()` 进入 `UnpinBuffer()`：

```text
UnpinBuffer(buf)
  -> ResourceOwnerForgetBuffer(CurrentResourceOwner, buffer)
  -> UnpinBufferNoOwner(buf)
     -> private refcount--
     -> 如果降到 0，decrement shared refcount
     -> 必要时唤醒等待 cleanup lock 的 backend
```

顺序也值得注意：

```text
先 Forget，再真正 unpin。
```

这样如果 `Forget()` 发现 owner 不匹配或重复释放，代码不会先破坏 shared buffer refcount。正常路径必须证明“我确实拥有这笔 pin”，然后才撤销外部占用。

如果 `ERROR` 跳过了 `ReleaseBuffer()`，ResourceOwner release 会走：

```text
ResOwnerReleaseBuffer(value)
  -> 如果还持有 buffer content lock，先解锁
  -> UnpinBufferNoOwner()
```

注意 callback 里调用的是 `NoOwner` 版本，因为此时 owner 正在 bulk release，不能再调用 `ResourceOwnerForget()`。

### 5.2 TupleDesc refcount：最小模型里也必须先 Enlarge

TupleDesc 路径比 buffer 简单很多，但正因为简单，它很适合看 ResourceOwner 的基本纪律。

`IncrTupleDescRefCount()`：

```text
Assert(tupdesc->tdrefcount >= 0);

ResourceOwnerEnlarge(CurrentResourceOwner);
tupdesc->tdrefcount++;
ResourceOwnerRememberTupleDesc(CurrentResourceOwner, tupdesc);
```

这里没有 shared buffer，也没有 OS fd，为什么仍然要先 `Enlarge()`？

因为 `tdrefcount++` 本身已经改变了对象生命周期。如果顺序写反：

```text
tupdesc->tdrefcount++;
ResourceOwnerRememberTupleDesc(...)
  -> 如果这里因为扩容 OOM ERROR
  -> refcount 已经增加
  -> owner 没记录
  -> abort cleanup 无法自动 --refcount
```

正常释放路径是 `DecrTupleDescRefCount()`：

```text
ResourceOwnerForgetTupleDesc(CurrentResourceOwner, tupdesc);
if (--tupdesc->tdrefcount == 0)
    FreeTupleDesc(tupdesc);
```

和 buffer 一样，它先从 owner 账本中删除 ownership，再降低 refcount。这样能捕获错误 owner 或重复释放。

异常释放路径是 `ResOwnerReleaseTupleDesc()`：

```text
/* Like DecrTupleDescRefCount, but don't call ResourceOwnerForget() */
if (--tupdesc->tdrefcount == 0)
    FreeTupleDesc(tupdesc);
```

这个例子沉淀出一个非常通用的判断：

```text
只要“获取资源”表现为 refcount++，
并且 refcount-- 可能被 ERROR 跳过，
就需要在 ++ 之前为 ResourceOwner 登记预留空间。
```

`PinTupleDesc` / `ReleaseTupleDesc` 宏还补了一层边界：

```text
if ((tupdesc)->tdrefcount >= 0)
    IncrTupleDescRefCount(tupdesc);
```

也就是说，不是所有 TupleDesc 都通过 ResourceOwner 管。只有参与 refcounted 生命周期的 TupleDesc，才需要进入这套 Remember / Forget 机制。

### 5.3 临时文件：先预留 owner 记录，再打开 VFD / OS 文件

临时文件路径展示了另一类外部资源：不是 refcount，而是 VFD、OS fd、临时文件删除和 `temp_file_limit` accounting。

`OpenTemporaryFile(bool interXact)` 里首先判断：

```text
if (!interXact)
    ResourceOwnerEnlarge(CurrentResourceOwner);
```

然后才开始选择临时表空间、创建文件：

```text
OpenTemporaryFileInTablespace(...)
```

如果创建成功，随后设置文件状态并登记：

```text
VfdCache[file].fdstate |= FD_DELETE_AT_CLOSE | FD_TEMP_FILE_LIMIT;

if (!interXact)
    RegisterTemporaryFile(file);
```

`RegisterTemporaryFile()` 做三件事：

```text
ResourceOwnerRememberFile(CurrentResourceOwner, file);
VfdCache[file].resowner = CurrentResourceOwner;
VfdCache[file].fdstate |= FD_CLOSE_AT_EOXACT;
have_xact_temporary_files = true;
```

这里的顺序服务同一个目标：

```text
普通事务内临时文件一旦打开成功，
就必须能在事务结束或 ERROR abort 时自动 close / delete。
```

正常关闭路径是 `FileClose()`：

```text
关闭 OS fd
处理 temp_file_limit accounting
如果 FD_DELETE_AT_CLOSE，stat + unlink + log tempfile
如果 vfdP->resowner，ResourceOwnerForgetFile(vfdP->resowner, file)
FreeVfd(file)
```

异常释放路径是：

```text
ResOwnerReleaseFile(value)
  -> vfdP->resowner = NULL
  -> FileClose(file)
```

这里先把 `resowner` 置空，是为了避免 `FileClose()` 在 bulk release 中再次 `ResourceOwnerForgetFile()`。这和 buffer、TupleDesc callback 使用 `NoOwner` / “不调用 Forget”版本是同一个模式：

```text
正常 retail release:
  Forget + 释放外部资源

ResourceOwner bulk release:
  owner 正在扫描自己的记录
  callback 只释放外部资源
  不再修改 owner 账本
```

临时文件还有一个细节体现 ERROR-safe 思维：`FileClose()` 处理 `FD_DELETE_AT_CLOSE` 时，会先清掉该 flag，再做 `stat()`、`unlink()` 和日志上报。注释解释了原因：

```text
如果 ereport/elog 又导致错误，abort 会再次回到这里；
先清 flag 可以避免无限循环。
最坏结果是少打一条日志，而不是不尝试 unlink。
```

这说明 ResourceOwner 兜底 cleanup 并不是“随便调用释放函数”。释放函数自己也必须尽量做到低层、幂等边界清楚、失败后不扩大损害。

## 6. 生命周期 / ownership / cleanup

本节可以把三类资源放进同一张 ownership 表：

| 资源 | acquire 前准备 | acquire 成功 | remember | 正常释放 | abort 兜底 |
| --- | --- | --- | --- | --- | --- |
| buffer pin | `ResourceOwnerEnlarge()` + `ReservePrivateRefCountEntry()` | shared refcount 增加，private refcount 建立 | `ResourceOwnerRememberBuffer()` | `ResourceOwnerForgetBuffer()` + `UnpinBufferNoOwner()` | `ResOwnerReleaseBuffer()` |
| TupleDesc ref | `ResourceOwnerEnlarge()` | `tdrefcount++` | `ResourceOwnerRememberTupleDesc()` | `ResourceOwnerForgetTupleDesc()` + `tdrefcount--` | `ResOwnerReleaseTupleDesc()` |
| 临时 File | `ResourceOwnerEnlarge()` | VFD / OS fd 创建成功 | `ResourceOwnerRememberFile()` + `vfdP->resowner` | `FileClose()` 中 `ResourceOwnerForgetFile()` | `ResOwnerReleaseFile()` |

这张表背后的长期不变量是：

```text
每一笔资源 ownership 都有两条离开路径：

1. 正常路径：
   显式 Forget，然后执行模块自己的释放动作。

2. 异常路径：
   记录留在 ResourceOwner 中，由 ResourceOwnerRelease() 调用 callback 释放。
```

commit 和 abort 对“剩余记录”的解释不同：

```text
commit 时普通资源还留在 owner 中：
  通常说明 executor 或调用者漏了正常释放路径
  ResourceOwnerRelease() 会 WARNING

abort 时资源还留在 owner 中：
  这是预期路径
  因为 ERROR 可能跳过了正常释放
```

这就是为什么正常路径不能懒得 `Forget()`，把所有 cleanup 都丢给事务结束：

```text
ResourceOwner 是 ERROR-safe 兜底，不是正常生命周期管理的替代品。
```

## 7. 正确性机制层次

Remember / Forget 这节课容易和其他机制混淆。可以分层理解：

```text
ResourceOwner:
  记录“谁负责释放这一笔资源”

refcount / pin:
  表达“资源当前正在被使用，不能回收或复用”

MemoryContext:
  管理 backend-local 内存块本身

lock / buffer content lock:
  管理并发访问和等待

release callback:
  在 owner 生命周期结束时，把 ownership 转成模块级释放动作
```

以 buffer 为例：

```text
ResourceOwner 记录一条 buffer pin
PrivateRefCountEntry 记录当前 backend pin 了几次
BufferDesc state 中的 refcount 影响全局 buffer replacement
Buffer content lock 保护页内容访问
```

这些状态不能互相替代：

```text
有 ResourceOwner 记录，不等于 shared refcount 已减少；
shared refcount 减少，不等于 owner 账本已删除；
释放 buffer content lock，不等于释放 buffer pin。
```

ResourceOwner cleanup 的正确性依赖一个重要约束：

```text
ReleaseResource callback 运行在 abort cleanup 场景中，
应只做低层 cleanup，不应执行可能产生用户可见副作用或复杂失败的新操作。
```

这也是为什么 callback 里常出现：

```text
NoOwner 版本
不调用 ResourceOwnerForget()
先清本地标志位
只降低 refcount / close file / unlock buffer
```

## 8. 错误路径 / 异常路径 / fallback

### 8.1 OOM 必须发生在 acquire 前

本节最核心的错误路径是 OOM：

```text
ResourceOwnerEnlarge()
  -> 可能分配 hash table
  -> 如果 OOM，ERROR
  -> 此时还没 pin buffer、还没 ++tdrefcount、还没 open temp file
  -> 没有外部资源需要清理
```

这就是“先失败”的价值。

相反，如果把扩容藏在 `Remember()` 里，失败点会落在 acquire 之后，owner 账本反而失去兜底能力。

### 8.2 `ResourceOwnerForget()` 的 ERROR 是调用者 bug

`Forget()` 找不到资源时会报：

```text
<resource kind> <pointer> is not owned by resource owner <name>
```

常见原因包括：

```text
释放次数多于获取次数
CurrentResourceOwner 已经被切换
资源被交给了另一个 owner，但释放时用错 owner
ResourceOwnerRemember() 根本没发生
```

这类 ERROR 不是业务异常，而是内核 ownership 不变量被破坏。诊断时应回到：

```text
谁调用 Enlarge？
谁真正 acquire？
Remember 和 acquire 之间是否夹了会 ERROR 的操作？
释放时 CurrentResourceOwner 是否还是同一个？
是否存在同一资源多次 remember，只 forget 了一部分？
```

### 8.3 临时文件打开失败和 fallback

`OpenTemporaryFile()` 还有一个普通 fallback：

```text
如果配置了临时表空间，先尝试对应 tablespace；
如果不可用，可能回退到数据库默认 tablespace；
interXact 文件强制进入默认 tablespace。
```

这个 fallback 和 ResourceOwner 的边界是：

```text
只要普通事务内文件最终可能成功打开，就先 Enlarge；
如果打开失败且没有产生 File，就没有需要 Remember 的资源；
如果打开成功，就 RegisterTemporaryFile() 进入 owner。
```

`interXact` 是另一个边界：

```text
interXact 临时文件要活过当前事务；
因此不能挂在当前事务 ResourceOwner 上自动关闭。
```

这再次说明 `CurrentResourceOwner` 表达的是生命周期，不是“当前函数方便放哪里”。

## 9. 成本、资源与跨模块传播

### 9.1 为什么 ResourceOwner 有 array + hash

`ResourceOwnerRemember()` 的普通路径只 append 到小数组：

```text
owner->arr[idx].item = value
owner->arr[idx].kind = kind
owner->narr++
```

这服务 hot path：

```text
大多数时刻，一个 owner 同时持有的普通资源数量不大；
append 到 array 比每次 hash insert 更便宜；
只有 array 满时，Enlarge 才把资源搬到 hash。
```

`ResourceOwnerForget()` 也先从 array 尾部往前找：

```text
最近 remember 的资源最可能马上被 forget。
```

这符合大量资源使用模式：

```text
pin page
访问 page
release page
```

短生命周期资源通常呈现近似栈式或局部性很强的使用方式，所以 array-first 设计能降低常态成本。

### 9.2 为什么 buffer 还要 `ReservePrivateRefCountEntry()`

ResourceOwner 只解决 owner 账本空间问题。buffer pin 还需要 backend-local private refcount entry：

```text
PrivateRefCountArray:
  常见 pin 数量下的快速路径

PrivateRefCountHash:
  同时 pin 很多 buffer 时的 overflow 路径

ReservedRefCountSlot:
  提前找到或腾出一个可写槽位
```

这和 `ResourceOwnerEnlarge()` 是同构思想：

```text
把可能分配或搬迁的数据结构调整，提前到真正 pin 之前。
```

如果一个路径只调用 `ResourceOwnerEnlarge()`，但后续 `NewPrivateRefCountEntry()` 仍可能在 pin 后失败，那仍然会留下未登记或无法跟踪的 pin。buffer 因此需要两套预留。

### 9.3 Forget 的扫描成本和资源数量相关

`ResourceOwnerForget()` 需要在 array/hash 中找匹配项。对于大量资源：

```text
array:
  从后往前线性扫描

hash:
  按 (value, kind) 哈希查找
```

这解释了为什么锁使用专门路径 `ResourceOwnerRememberLock()` / `ResourceOwnerForgetLock()`，并且 lock list 是 lossy cache。锁的数量和释放语义足够特殊，不能简单套 generic array/hash 成本模型。

本节不展开 lock，重点只记住：

```text
ResourceOwner 的 generic Remember / Forget 是通用但仍然被 hot path 调优过的机制；
极热或语义特殊的资源可以有专用 owner 结构。
```

## 10. 观测与诊断入口

### 10.1 commit WARNING

最容易看到的现象是 commit 时资源没有正常释放。`ResourceOwnerRelease()` 在 commit 场景发现普通资源残留时，会通过 `DebugPrint` 打 WARNING。

典型含义：

```text
正常路径漏了 ResourceOwnerForget()
  或
释放动作没有跑到
  或
资源挂错 owner，导致当前 owner 结束时仍然持有
```

abort 时不应按同样标准解读，因为 abort 正是 ResourceOwner 兜底释放资源的场景。

### 10.2 gdb 断点

源码实验时可以在这些函数上下断点：

```text
break ResourceOwnerEnlarge
break ResourceOwnerRemember
break ResourceOwnerForget
break ResOwnerReleaseBuffer
break ResOwnerReleaseTupleDesc
break ResOwnerReleaseFile
```

观察重点不是函数被调用了多少次，而是顺序：

```text
Enlarge 是否在 acquire 前？
Remember 是否紧跟 acquire 后？
正常释放是否先 Forget？
ERROR 后是否进入 ReleaseResource callback？
```

### 10.3 临时文件日志

设置：

```sql
SET log_temp_files = 0;
```

再执行会产生临时文件的排序、hash aggregate 或临时表操作，可以从日志看到临时文件创建和删除记录。这个现象本身不直接打印 ResourceOwner，但能帮助建立：

```text
SQL 产生临时文件
  -> fd.c 打开 VFD / OS file
  -> RegisterTemporaryFile() 挂到 owner
  -> FileClose() / transaction cleanup 删除
```

### 10.4 源码级注入 ERROR

为了验证 acquire-before-ERROR，可以在开发环境里临时注入错误：

```text
在 IncrTupleDescRefCount() 的 tupdesc->tdrefcount++ 之后、
ResourceOwnerRememberTupleDesc() 之前插入 elog(ERROR, ...)
```

预期结果：

```text
这是一个故意破坏不变量的实验；
你会制造 refcount 已增加但 owner 无记录的状态。
```

更安全的实验是把错误放在 `ResourceOwnerEnlarge()` 之后、真正 acquire 之前：

```text
ResourceOwnerEnlarge(CurrentResourceOwner);
elog(ERROR, "inject before acquire");
```

预期：

```text
ERROR 发生时还没有外部资源被获取；
abort cleanup 没有什么可释放；
不会产生 owner 账本和外部状态不一致。
```

这类实验只能在临时源码树里做，不能用于生产实例。

## 11. 常见误区

### 误区一：`ResourceOwnerEnlarge()` 是性能优化

更准确地说，它首先是 correctness 边界：

```text
把可能失败的账本扩容放在资源 acquire 之前。
```

性能收益是副作用：

```text
让 Remember 变成极短的 append 快路径。
```

### 误区二：`Remember()` 成功后就不需要手工释放

错误。`Remember()` 是 abort 兜底，不是正常路径替代品。

正常路径仍然应该：

```text
Forget
释放外部资源
```

否则 commit 时 ResourceOwner 会发 WARNING，说明代码没有按预期把资源生命周期闭合。

### 误区三：`Forget()` 可以随便放在释放之后

很多路径选择先 `Forget()`，再释放外部资源，因为这能先验证 ownership：

```text
如果 owner 不拥有这笔资源，先报错；
不要先修改 refcount / pin / file state。
```

但 bulk release callback 中不能再 `Forget()`，因为 owner 正在扫描并释放自己的记录。

### 误区四：同一个资源只会出现一条记录

不一定。`ResourceOwnerForget()` 明确说明：

```text
如果同一个 resource ID 被同一个 owner 关联多次，只删除其中一条。
```

这对多次 pin 同一 buffer、多次持有同一 refcounted 对象很重要。

### 误区五：所有临时文件都挂当前事务 owner

`interXact` 文件要活过当前事务，因此不能挂到当前事务 `CurrentResourceOwner`。这类边界不能靠“当前函数在哪执行”判断，而要看资源应该活到什么时候。

## 12. 课堂实验

### 实验一：跟踪 buffer pin 的 Enlarge / Remember / Forget 顺序

源码入口：

```text
src/backend/storage/buffer/bufmgr.c
  BufferAlloc()
  PinBuffer()
  TrackNewBufferPin()
  UnpinBuffer()
  ResOwnerReleaseBuffer()
```

练习：

```text
1. 在 gdb 中对 ResourceOwnerEnlarge、ResourceOwnerRemember、ResourceOwnerForget 下断点。
2. 执行 SELECT * FROM 一个足够大的表 LIMIT 1;
3. 观察 buffer pin 路径是否先 Enlarge，再 Remember。
4. 再观察正常 executor 退出时是否有 Forget。
```

思考：

```text
如果查询中途 ERROR，哪些 Forget 不会发生？
哪些 ReleaseResource callback 会接手？
```

### 实验二：阅读 TupleDesc 的最小 acquire-before-ERROR 模型

源码入口：

```text
src/backend/access/common/tupdesc.c
  IncrTupleDescRefCount()
  DecrTupleDescRefCount()
  ResOwnerReleaseTupleDesc()
```

练习：

```text
1. 画出 tdrefcount 从 N 到 N+1 再回到 N 的状态图。
2. 标出 ResourceOwnerEnlarge()、ResourceOwnerRememberTupleDesc()、ResourceOwnerForgetTupleDesc() 的位置。
3. 解释如果 Remember 放在 tdrefcount++ 之后且会分配失败，会遗留什么状态。
```

### 实验三：观察临时文件 cleanup

SQL 方向：

```sql
SET log_temp_files = 0;
SET work_mem = '64kB';
SELECT *
FROM generate_series(1, 200000) g
ORDER BY md5(g::text);
```

源码入口：

```text
src/backend/storage/file/fd.c
  OpenTemporaryFile()
  RegisterTemporaryFile()
  FileClose()
  ResOwnerReleaseFile()
```

观察：

```text
普通完成时，FileClose() 会 forget owner 记录；
如果 ERROR/abort 跳过普通关闭，ResourceOwnerRelease() 会调用 ResOwnerReleaseFile()。
```

### 实验四：检查 commit warning 的含义

开发环境里可以故意制造一条泄漏路径，例如注释掉某个安全可控测试分支里的 `ResourceOwnerForget*()`。预期：

```text
事务 commit 时出现 leaked resource warning。
```

分析时不要只看 warning 文本，而要回到：

```text
这个资源在哪 acquire？
在哪 remember？
正常释放路径应该在哪里 forget？
为什么这次没有走到？
```

## 13. 讨论题

1. 为什么 `ResourceOwnerRemember()` 不自己调用 `ResourceOwnerEnlarge()`？

2. `ResourceOwnerForget()` 为什么在找不到资源时选择 `ERROR`，而不是静默返回？

3. buffer pin 路径为什么除了 `ResourceOwnerEnlarge()`，还需要 `ReservePrivateRefCountEntry()`？

4. 为什么 ResourceOwner release callback 不能再调用 `ResourceOwnerForget()`？

5. 如果一个资源应该活过当前事务，应不应该挂到 `CurrentResourceOwner`？判断依据是什么？

6. commit 时出现 ResourceOwner leaked resource warning，为什么通常说明正常路径有 bug，而 abort 时资源残留却是正常现象？

## 14. 本节小结

本节的可迁移规律是：

```text
当“登记 cleanup 责任”本身可能失败时，
必须在真正 acquire 外部资源之前完成登记所需的失败点；
acquire 成功后只能执行不再分配、不再复杂失败的快速 remember。
```

在 PostgreSQL 里，这条规律落成了固定形态：

```text
ResourceOwnerEnlarge()
  -> acquire external resource
  -> ResourceOwnerRemember()
  -> normal path ResourceOwnerForget()
  -> abort path ResourceOwnerRelease callback
```

buffer pin、TupleDesc refcount 和临时文件看起来属于不同模块，但它们共享同一个 correctness 约束：

```text
不能让外部资源状态已经改变，而 ResourceOwner 账本还不知道。
```

下一节可以在这个基础上继续看 `CurrentResourceOwner` 如何在 top transaction、subtransaction 和 Portal 执行之间切换：同样一条 `Remember()` 记录，为什么有时应该归事务，有时应该归 Portal，有时子事务 commit 还要把资源转移给父 owner。
