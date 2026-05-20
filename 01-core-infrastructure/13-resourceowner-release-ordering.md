# PostgreSQL 三阶段 release 与锁释放顺序

## 课程定位

前置知识：已经理解 `ResourceOwner` tree 的 ownership 边界、`Remember()` / `Forget()` 的 acquire-before-ERROR 安全，以及 `CurrentResourceOwner` 如何在事务、子事务和 Portal 之间传播。

本节唯一主问题：

```text
为什么 ResourceOwnerRelease() 要分成 before-locks、locks、after-locks 三段，buffer pin、relcache ref、DSM、JIT、catcache ref、snapshot、文件等资源的释放顺序如何服务并发可见性和 backend-local cleanup？
```

核心矛盾：事务结束时，PostgreSQL 要尽快释放锁，唤醒正在等待的 backend；但一旦锁释放，别人就可能立刻继续执行，并观察我们刚刚提交、回滚或释放的对象。锁释放之前，必须先让跨 backend 可见的占用状态消失；锁释放之后，才能做只影响本 backend 的引用计数、文件句柄、snapshot 引用等 cleanup。

学完后应能判断：一个新资源类型应该放到 `RESOURCE_RELEASE_BEFORE_LOCKS` 还是 `RESOURCE_RELEASE_AFTER_LOCKS`；为什么 buffer pin、relcache ref、DSM、JIT 在锁前释放，而 catcache ref、snapshot、VFD 文件、tupledesc、plancache 在锁后释放；为什么 `xact.c` 在三次 `ResourceOwnerRelease()` 之间还要插入 relcache、invalidation、pending delete 等事务级 cleanup；以及为什么 release callback 只能做低层、不可失败的清理。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `6b48f5d1a741`。

## 1. 本节在总主线中的位置

前几节把 ResourceOwner 的三个层次铺开了：

```text
第 10 课:
  为什么外部资源要挂 ResourceOwner tree？

第 11 课:
  获取资源时怎样用 Enlarge -> Remember -> Forget 收住 ERROR 窗口？

第 12 课:
  资源会挂到 top transaction、subtransaction 还是 Portal owner？
```

本节进入第四层：

```text
当 owner 结束时，资源按什么顺序释放？
```

这不是清理代码的审美问题。顺序错了，会产生真实的并发错误：

```text
先释放锁，再释放 buffer pin:
  等锁的 backend 被唤醒后可能仍看到我们 pin 着它想回收、截断或复用的 buffer。

先释放锁，再发送 catalog invalidation:
  等待 DDL 锁的 backend 可能继续使用过期 relcache / catcache 语义。

在锁前关闭所有 backend-local cache ref:
  可能徒增锁持有时间，还可能在 abort 中触发不适合发生的高层逻辑。
```

所以 `ResourceOwnerRelease()` 的三段不是为了把代码分层好看，而是为了定义一个事务结束时的可见性边界：

```text
锁释放之前:
  释放其他 backend 可能间接感知到的资源。

锁释放这一刻:
  让等待者认为本事务已经离开它们关心的并发边界。

锁释放之后:
  释放只属于当前 backend 的引用和句柄。
```

本节不再重复 `CurrentResourceOwner` 的传播规则；我们只看 owner 已经确定之后，release 如何在 owner tree、资源 priority、事务清理函数和锁管理器之间协作。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
xact.c 在事务结束时按 before-locks -> locks -> after-locks 三次调用 ResourceOwnerRelease()；
resowner.c 对整棵 owner tree 每个 phase 递归 child-first，并按 ResourceOwnerDesc 的 priority 释放资源；
before-locks 清掉跨 backend 可见或会影响并发判断的占用；
locks 阶段统一释放或转移 heavyweight / predicate lock；
after-locks 清掉 backend-local 引用、snapshot、文件句柄和等待对象。
```

本节 tension 是：

```text
锁应该尽早释放，减少等待者延迟
  vs
锁释放后等待者必须看到一个已经清理到安全边界的世界
```

PostgreSQL 的选择不是“事务结束时扫一遍所有资源”，而是把释放拆成三段：

```c
ResourceOwnerRelease(owner, RESOURCE_RELEASE_BEFORE_LOCKS, isCommit, isTopLevel);
/* xact.c 在这里做 relcache / typecache / invalidation / multixact 等事务级工作 */
ResourceOwnerRelease(owner, RESOURCE_RELEASE_LOCKS, isCommit, isTopLevel);
ResourceOwnerRelease(owner, RESOURCE_RELEASE_AFTER_LOCKS, isCommit, isTopLevel);
```

再把每类资源自己的位置写进 `ResourceOwnerDesc`：

```c
.release_phase = RESOURCE_RELEASE_BEFORE_LOCKS,
.release_priority = RELEASE_PRIO_BUFFER_PINS,
```

这形成两个层次的 ordering：

```text
phase ordering:
  before-locks -> locks -> after-locks

priority ordering:
  同一个 phase 内，不同资源类型按 release_priority 排序
```

还要加上一个容易忽略的层次：

```text
tree ordering:
  每个 phase 内，先 release child ResourceOwner，再 release parent ResourceOwner
```

因此真实顺序不是单个 owner 的线性数组，而是：

```text
phase 1:
  child before-locks resources
  parent before-locks resources
  xact.c phase 之间的事务级 cleanup

phase 2:
  child locks
  parent locks

phase 3:
  child after-locks resources
  parent after-locks resources
```

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 作用 |
| --- | --- | --- |
| 1 | `src/include/utils/resowner.h` | 定义 `ResourceReleasePhase`、内置 `RELEASE_PRIO_*`、`ResourceOwnerDesc` 的 release phase / priority 合约。 |
| 2 | `src/backend/utils/resowner/README` | 直接解释三阶段 release 的设计意图、child-first 顺序、commit leak warning 和 callback 限制。 |
| 3 | `src/backend/utils/resowner/resowner.c` | 实现 `ResourceOwnerRelease()`、递归、排序、锁阶段特殊处理和 release callback 调用。 |
| 4 | `src/backend/access/transam/xact.c` | 在 top transaction、abort、prepare、subtransaction commit / abort 中按三阶段调用 ResourceOwnerRelease，并在阶段之间插入事务级 cleanup。 |
| 5 | `src/backend/storage/buffer/bufmgr.c` | `buffer_io_resowner_desc`、`buffer_resowner_desc` 属于 before-locks，解释 buffer IO / pin 为什么要在锁前处理。 |
| 6 | `src/backend/utils/cache/relcache.c` | `relref_resowner_desc` 属于 before-locks，并和 `AtEOXact_RelationCache()`、invalidation 顺序相互约束。 |
| 7 | `src/backend/storage/ipc/dsm.c` 与 `src/backend/jit/llvm/llvmjit.c` | DSM、JIT context 属于 before-locks，用于释放可能关联执行状态、共享段或运行时代码上下文的资源。 |
| 8 | `src/backend/storage/lmgr/lock.c`、`src/backend/storage/lmgr/proc.c`、`src/backend/storage/lmgr/predicate.c` | locks 阶段释放、转移普通锁和 predicate lock。 |
| 9 | `src/backend/utils/cache/catcache.c`、`src/backend/utils/time/snapmgr.c`、`src/backend/storage/file/fd.c` | after-locks 资源：catcache ref、snapshot ref、VFD file 等 backend-local cleanup。 |
| 10 | `src/backend/access/common/tupdesc.c`、`src/backend/utils/cache/plancache.c`、`src/backend/storage/ipc/waiteventset.c` | after-locks 的其他典型引用类资源，用来判断扩展新资源时的落点。 |

推荐阅读顺序：

```text
先读 resowner.h 的 phase / priority 表
  -> 再读 resowner.c 的三段 release 主循环
  -> 再读 xact.c 在三段之间插了什么
  -> 最后按资源类型看 ResourceOwnerDesc 为什么选这个 phase
```

不要一上来横向搜索所有 `ReleaseResource` callback。那会得到一堆“释放某个对象”的函数，却看不出为什么某些对象必须先于锁释放，某些对象可以晚于锁释放。

## 4. 关键数据结构与状态

### `ResourceReleasePhase`

`resowner.h` 定义了三个 phase：

```c
typedef enum
{
    RESOURCE_RELEASE_BEFORE_LOCKS = 1,
    RESOURCE_RELEASE_LOCKS,
    RESOURCE_RELEASE_AFTER_LOCKS,
} ResourceReleasePhase;
```

语义不是“释放顺序 1、2、3”这么简单，而是：

```text
BEFORE_LOCKS:
  释放会影响其他 backend 判断、等待、回收、可见性或共享状态的占用。

LOCKS:
  释放或转移 heavyweight lock、predicate lock 等并发边界。

AFTER_LOCKS:
  释放锁释放后不再影响其他 backend 正确性的 backend-local 引用和句柄。
```

`resowner.h` 对 before-locks 的注释非常关键：

```text
pre-lock phase 必须释放其他 backend 可见的资源，例如 pinned buffers；
这样当我们释放别人等待的锁时，别人会看到我们已经完全离开事务。
post-lock phase 用于 backend-internal cleanup。
```

这句话就是本课的中心模型。

### 内置资源 priority 表

当前源码中的 before-locks priority：

| priority | 资源类型 | 代表文件 | 为什么在锁前 |
| --- | --- | --- | --- |
| `RELEASE_PRIO_BUFFER_IOS` = 100 | in-progress buffer IO | `bufmgr.c` | ERROR / abort 时必须结束或撤销正在进行的 buffer IO 状态，避免共享 buffer 状态悬挂。 |
| `RELEASE_PRIO_BUFFER_PINS` = 200 | buffer pin | `bufmgr.c` | pin 会阻止 buffer 回收、截断、清理或某些等待路径，其他 backend 可间接感知。 |
| `RELEASE_PRIO_RELCACHE_REFS` = 300 | relcache ref | `relcache.c` | relation ref 会影响 relcache invalidation / cleanup，必须早于事务级 relcache / inval 工作。 |
| `RELEASE_PRIO_DSMS` = 400 | dynamic shared memory segment | `dsm.c` | DSM 段关联跨 backend 或并行执行资源，不能拖到锁释放后才断开。 |
| `RELEASE_PRIO_JIT_CONTEXTS` = 500 | LLVM JIT context | `llvmjit.c` | JIT context 关联执行期资源和可能的外部运行时状态，属于执行资源清退。 |
| `RELEASE_PRIO_CRYPTOHASH_CONTEXTS` = 600 | cryptohash context | crypto 相关代码 | 释放低层执行上下文。 |
| `RELEASE_PRIO_HMAC_CONTEXTS` = 700 | HMAC context | crypto 相关代码 | 释放低层执行上下文。 |

当前源码中的 after-locks priority：

| priority | 资源类型 | 代表文件 | 为什么在锁后 |
| --- | --- | --- | --- |
| `RELEASE_PRIO_CATCACHE_REFS` = 100 | catcache tuple ref | `catcache.c` | backend-local cache entry refcount，释放不需要阻塞等待者继续执行。 |
| `RELEASE_PRIO_CATCACHE_LIST_REFS` = 200 | catcache list ref | `catcache.c` | backend-local cache list pin。 |
| `RELEASE_PRIO_PLANCACHE_REFS` = 300 | cached plan ref | `plancache.c` | 当前 backend 的计划对象引用。 |
| `RELEASE_PRIO_TUPDESC_REFS` = 400 | tupledesc ref | `tupdesc.c` | 当前 backend 的描述符引用计数。 |
| `RELEASE_PRIO_SNAPSHOT_REFS` = 500 | snapshot ref | `snapmgr.c` | snapshot 引用由当前 backend 管理，事务可见性边界已由 ProcArray / xact 状态处理。 |
| `RELEASE_PRIO_FILES` = 600 | VFD file | `fd.c` | 当前 backend 的虚拟文件描述符，通常是本地资源 cleanup。 |
| `RELEASE_PRIO_WAITEVENTSETS` = 700 | WaitEventSet | `waiteventset.c` | 当前 backend 的等待集合对象。 |

这些数字不是性能调参 knob，而是资源之间的局部依赖声明。扩展如果注册自己的 `ResourceOwnerDesc`，也要在这张表附近给自己找位置。

### `ResourceOwnerDesc`

每个资源类型通过 `ResourceOwnerDesc` 声明 release 行为：

```c
typedef struct ResourceOwnerDesc
{
    const char *name;
    ResourceReleasePhase release_phase;
    ResourceReleasePriority release_priority;
    void (*ReleaseResource) (Datum res);
    char *(*DebugPrint) (Datum res);
} ResourceOwnerDesc;
```

这几个字段组合起来才是语义：

```text
name:
  leak warning / debug 输出里的资源名。

release_phase:
  锁前、锁阶段、锁后。

release_priority:
  同一 phase 内的资源类型顺序。

ReleaseResource:
  bulk release 时真正释放资源，不能再依赖会失败的高层动作。

DebugPrint:
  commit 时发现资源没被 retail forget，用来打印 WARNING。
```

注意：`ReleaseResource` 在 bulk release 中被调用时，资源已经从 owner 的数组或 hash 中隐式移除，callback 不应该再调用 `ResourceOwnerForget()`。这也是为什么很多 callback 名字里带 `NoOwner` 或注释说“像普通释放，但不调用 Forget”。

### `ResourceOwnerData` 中的 release 状态

`resowner.c` 的 `ResourceOwnerData` 有两个 release 相关标记：

```text
releasing:
  这个 owner 已经进入 bulk release 流程。

sorted:
  arr / hash 是否已经按 phase / priority 排好序。
```

一旦开始 release：

```text
不能再 Remember 新资源；
不能再 retail Forget 旧资源；
后续必须让 bulk release 接管。
```

这个限制很重要。否则 release 过程中数组和 hash 一边排序一边被修改，`ResourceOwnerReleaseAll()` 就无法知道自己释放到了哪里。

## 5. 主流程源码 walkthrough

### 5.1 顶层事务 commit：把锁释放放在“别人可以安全继续”的点上

`xact.c` 的顶层 commit 路径先完成真正的事务提交记录，然后进入 post-commit cleanup。关键顺序是：

```text
RecordTransactionCommit()
  -> ProcArrayEndTransaction()
  -> ResourceOwnerRelease(BEFORE_LOCKS)
  -> AtEOXact_Aio(true)
  -> AtEOXact_Buffers(true)
  -> AtEOXact_RelationCache(true)
  -> AtEOXact_TypeCache()
  -> AtEOXact_Inval(true)
  -> AtEOXact_MultiXact()
  -> ResourceOwnerRelease(LOCKS)
  -> ResourceOwnerRelease(AFTER_LOCKS)
  -> smgrDoPendingDeletes(true)
  -> notify / GUC / SPI / namespace / files 等内部 cleanup
```

源码注释点明了这里的目标：

```text
先释放其他 backend 可见的资源，例如文件、buffer pin；
再释放锁；
最后释放 backend-local resources。
```

更具体地说，`ProcArrayEndTransaction()` 已经让别人从 ProcArray 视角看到我们不再是 in-progress transaction。接下来不能马上释放锁，因为有些等待者被唤醒后会继续访问同一 relation、buffer 或 catalog 状态。PostgreSQL 先做 before-locks：

```text
结束 buffer IO
释放 buffer pin
释放 relcache ref
释放 DSM / JIT 等执行期资源
```

然后 `xact.c` 还在锁前插入事务级 cleanup：

```text
AtEOXact_RelationCache(true)
AtEOXact_TypeCache()
AtEOXact_Inval(true)
AtEOXact_MultiXact()
```

其中 `AtEOXact_Inval(true)` 特别关键：如果本事务改了 catalog，等锁的 backend 被唤醒后应该先知道相关 cache 已经过期，而不是继续使用旧 relcache / catcache 语义。

最后进入 locks 阶段：

```text
ResourceOwnerRelease(TopTransactionResourceOwner,
                     RESOURCE_RELEASE_LOCKS,
                     true,
                     true);
```

`resowner.c` 看到 `isTopLevel == true` 且当前 owner 是 `TopTransactionResourceOwner`，调用：

```text
ProcReleaseLocks(isCommit)
ReleasePredicateLocks(isCommit, false)
```

锁释放之后，才进入 after-locks：

```text
catcache ref
catcache list ref
plancache ref
tupledesc ref
snapshot ref
VFD file
WaitEventSet
```

这些 cleanup 仍然必须做，但它们不再定义其他 backend 是否可以继续的并发边界。

### 5.2 顶层事务 abort：更依赖 ResourceOwner bulk release

abort 路径和 commit 的骨架类似，但 `isCommit = false`：

```text
ResourceOwnerRelease(BEFORE_LOCKS, false, true)
AtEOXact_Aio(false)
AtEOXact_Buffers(false)
AtEOXact_RelationCache(false)
AtEOXact_TypeCache()
AtEOXact_Inval(false)
AtEOXact_MultiXact()
ResourceOwnerRelease(LOCKS, false, true)
ResourceOwnerRelease(AFTER_LOCKS, false, true)
smgrDoPendingDeletes(false)
```

commit 和 abort 的差异是：

```text
commit:
  正常路径理论上应已 retail release 大部分资源；
  ResourceOwnerRelease 发现残留资源会打印 WARNING。

abort:
  ERROR 可能打断任意深层代码；
  ResourceOwnerRelease 真正承担兜底释放责任，通常不打印 leak warning。
```

这就是 `ResourceOwnerReleaseAll(owner, phase, isCommit)` 中 `printLeakWarnings = isCommit` 的来源。

在 abort 中，三阶段顺序更重要。因为 ERROR 可能发生在：

```text
buffer content lock 仍持有
buffer pin 尚未 unpin
relcache 构造到一半
临时文件打开
snapshot 已注册
catcache tuple ref 未释放
```

`ResOwnerReleaseBuffer()` 甚至专门处理“ERROR 时 pin 着 buffer 且 content lock 还没释放”的情况：

```text
如果 private refcount 记录显示仍持有 buffer content lock:
  先释放 content lock
  再 UnpinBufferNoOwner()
```

这就是 ERROR-safe cleanup 的实际落点：不是要求每条路径都写完美的 `PG_TRY`，而是让 ResourceOwner 在 abort 时按正确顺序兜底。

### 5.3 子事务 commit：锁阶段不是释放，而是转移

子事务 commit 路径中的顺序：

```text
AtSubCommit_Portals(...)
...
ResourceOwnerRelease(subowner, BEFORE_LOCKS, true, false)
AtEOSubXact_RelationCache(true, ...)
AtEOSubXact_TypeCache()
AtEOSubXact_Inval(true)
AtSubCommit_smgr()
XactLockTableDelete(subxid)
ResourceOwnerRelease(subowner, LOCKS, true, false)
ResourceOwnerRelease(subowner, AFTER_LOCKS, true, false)
```

这里 `isTopLevel = false`，所以 locks 阶段不会调用 `ProcReleaseLocks()`。而是：

```text
if (isCommit)
    LockReassignCurrentOwner(locks, nlocks);
else
    LockReleaseCurrentOwner(locks, nlocks);
```

子事务 commit 的 lock 语义是：

```text
子事务成功了，它拿到的事务级锁仍属于外层事务；
不能释放，否则外层事务后续逻辑会失去隔离和 DDL / DML 并发保护；
所以锁从子 owner 转移给父 owner。
```

这与 before-locks / after-locks 资源不同。buffer pin、catcache ref、snapshot ref 这些资源如果到子事务结束还挂在子 owner 上，commit 时大多应该已经被正常释放；残留会触发 warning 或按 phase cleanup。lock 则是显式的例外：正常 commit 下也要继承。

### 5.4 子事务 abort：锁阶段必须释放当前 owner 的锁

子事务 abort 顺序类似，但 `isCommit = false`：

```text
AtSubAbort_Portals(...)
RecordTransactionAbort(true)
ResourceOwnerRelease(subowner, BEFORE_LOCKS, false, false)
AtEOXact_Aio(false)
AtEOSubXact_RelationCache(false, ...)
AtEOSubXact_TypeCache()
AtEOSubXact_Inval(false)
ResourceOwnerRelease(subowner, LOCKS, false, false)
ResourceOwnerRelease(subowner, AFTER_LOCKS, false, false)
AtSubAbort_smgr()
...
AtSubAbort_Snapshot(...)
```

这时 `LockReleaseCurrentOwner()` 会释放属于 `CurrentResourceOwner` 的锁。原因很直接：

```text
子事务失败了；
它内部获取的锁保护的是失败路径中的操作；
如果继续转移给父事务，外层事务会持有不该继承的锁，造成过度阻塞甚至错误语义。
```

这也是上一课提到的“子事务 commit 转移，abort 释放”的具体落点。

### 5.5 Portal drop：普通关闭时释放非锁资源，锁归事务

`portalmem.c` 的 `PortalDrop()` 也调用三阶段 release：

```text
ResourceOwnerRelease(portal->resowner, BEFORE_LOCKS, isCommit, false)
ResourceOwnerRelease(portal->resowner, LOCKS, isCommit, false)
ResourceOwnerRelease(portal->resowner, AFTER_LOCKS, isCommit, false)
ResourceOwnerDelete(portal->resowner)
```

但注释强调：

```text
普通 portal drop:
  如果 Portal 不是 FAILED，不释放它的锁；
  锁通过 owner parent 关系变成当前事务的责任。

FAILED portal:
  视作非正常结束，要清掉资源，避免 transaction commit 时出现 leak warning。
```

这说明三阶段 release 是通用机制，但具体语义仍由 `isCommit`、`isTopLevel`、owner parent 和 Portal 状态一起决定。

## 6. 生命周期 / ownership / cleanup

### 创建

资源 owner 的创建在前几节已经讲过：

```text
top transaction:
  AtStart_ResourceOwner() 创建 TopTransaction owner。

subtransaction:
  AtSubStart_ResourceOwner() 创建 child owner。

Portal:
  CreatePortal() 创建 portal->resowner，并挂到当前事务 owner 下。
```

本节关心的是结束时：

```text
ResourceOwnerRelease():
  释放 owner 持有的资源，但不释放 owner 对象本身。

ResourceOwnerDelete():
  删除 owner 对象，要求资源已经 release 完。
```

### 持有

资源通过两类结构持有：

```text
普通资源:
  ResourceOwnerData.arr / hash 记录 Datum + ResourceOwnerDesc。

锁资源:
  ResourceOwnerData.locks 小数组缓存 LOCALLOCK 指针；
  lock.c 的 LOCALLOCKOWNER 记录 owner -> nLocks。
```

锁单独处理，是因为它们有“commit 时不一定释放”的事务语义。普通资源多数在 commit 时应已被 retail release，ResourceOwner bulk release 是异常兜底；锁则经常要持有到事务结束，甚至在子事务 commit 时转移给父 owner。

### release 的 child-first 规则

`ResourceOwnerReleaseInternal()` 每个 phase 都先递归 children：

```text
for each child:
  ResourceOwnerReleaseInternal(child, phase, ...)

release current owner in this phase
```

所以一棵 owner tree 的释放顺序是：

```text
先清理 Portal / subtransaction owner
再清理 parent transaction owner
```

这与 ownership 语义一致：

```text
child owner 是更短生命周期；
parent owner 是更长生命周期；
释放长生命周期之前，必须先把短生命周期残留处理完。
```

### phase 之间不能再 retail Forget

`resowner.c` 在第一次 release 时设置：

```text
owner->releasing = true
owner->sorted = true
```

之后：

```text
ReleaseResource callback 不能 Remember / Forget 其他资源；
调用者也不能再把这个 owner 当作正常 owner 使用；
如果 phase 之间发生 ERROR，AbortTransaction 可能再次进入 release，但只能沿 bulk release 语义继续。
```

这条规则解释了为什么 release callback 必须非常低层：

```text
不能分配可能 ERROR 的内存；
不能执行用户可见动作；
不能依赖高层事务状态仍然完整；
不能制造新的 ResourceOwner 资源。
```

## 7. 正确性机制层次

### 层次一：ProcArray / transaction status 不是全部

顶层 commit 里，`ProcArrayEndTransaction()` 在释放锁之前执行。它让其他 backend 从快照角度看到本事务不再进行中。

但这不等于所有资源都可以留到后面慢慢清：

```text
ProcArray:
  解决 XID in-progress / visibility 判断。

ResourceOwner before-locks:
  解决 buffer pin、relcache ref、DSM、JIT 等运行时占用。

LOCKS:
  解决 heavyweight lock / predicate lock 等并发等待边界。

AFTER_LOCKS:
  解决当前 backend 的引用计数和句柄。
```

一个 backend 被锁唤醒时，不只会做 visibility 判断，还可能：

```text
访问 relation descriptor
处理 relcache invalidation
尝试清理或复用 buffer
继续 DDL / DML 执行
读取 catalog cache
```

所以锁释放点必须排在“跨 backend 相关状态已经安全”的后面。

### 层次二：buffer pin 是共享状态的间接占用

buffer pin 本身是 backend-local refcount 加 shared buffer descriptor 状态的组合。其他 backend 不会直接读你的 `ResourceOwner`，但会感知到：

```text
buffer 仍被 pin，不能安全驱逐或复用；
某些 cleanup / truncation / IO 状态可能等待 pin 消失；
ERROR 时如果 content lock 也残留，会阻塞其他访问者。
```

因此 `buffer_resowner_desc` 在 before-locks：

```c
.release_phase = RESOURCE_RELEASE_BEFORE_LOCKS,
.release_priority = RELEASE_PRIO_BUFFER_PINS,
```

这不是因为 buffer pin “比 lock 更高级”，而是因为：

```text
释放 lock 会唤醒别人；
别人醒来前，我们必须已经不再 pin 它可能要处理的 buffer。
```

### 层次三：relcache ref 和 invalidation 的顺序

relcache ref 也在 before-locks：

```c
.release_phase = RESOURCE_RELEASE_BEFORE_LOCKS,
.release_priority = RELEASE_PRIO_RELCACHE_REFS,
```

`AtEOXact_RelationCache()` 注释说明，它必须在处理 invalidation messages 前调用。原因是 abort 时不能安全地重建 invalidated cache entries，所以要先 reset / cleanup refcount，再处理 pending invalidations。

顶层 commit 注释还强调：

```text
AtEOXact_Inval(true) 必须在释放锁前；
如果有人正在等待我们修改过的 relation 上的 lock，
它开始使用 relation 前应该先知道 catalog change。
```

这形成一条更细的链：

```text
ResourceOwnerRelease(BEFORE_LOCKS)
  -> relcache ref 下降
AtEOXact_RelationCache()
  -> relcache 事务级检查和 cleanup
AtEOXact_Inval()
  -> 发送或处理 cache invalidation
ResourceOwnerRelease(LOCKS)
  -> 等待者继续执行
```

### 层次四：catcache ref 为什么反而在锁后

catcache ref 在 after-locks：

```c
.release_phase = RESOURCE_RELEASE_AFTER_LOCKS,
.release_priority = RELEASE_PRIO_CATCACHE_REFS,
```

这看起来容易误解：

```text
catcache 也和 catalog 有关，为什么不在锁前？
```

答案是：这里释放的是当前 backend 持有的 catcache tuple/list 引用计数，不是向其他 backend 宣告 catalog 变化的 invalidation 边界。

catalog 变化对其他 backend 的可见性通过：

```text
事务提交状态
AtEOXact_Inval()
lock release
```

协作完成。catcache ref cleanup 本身主要是当前 backend 内部缓存对象的 refcount 归还，可以在锁后做。

### 层次五：snapshot ref 在锁后，但 snapshot 语义不只靠 ResourceOwner

snapshot ref 也在 after-locks：

```c
.release_phase = RESOURCE_RELEASE_AFTER_LOCKS,
.release_priority = RELEASE_PRIO_SNAPSHOT_REFS,
```

这不表示 snapshot 对并发不重要。它表示：

```text
ResourceOwner 追踪的是已注册 snapshot 对象的引用；
事务可见性和 xmin horizon 还由 ProcArray、active snapshot stack、snapmgr 的事务钩子共同维护。
```

换句话说：

```text
snapshot correctness:
  不只在 ResOwnerReleaseSnapshot() 里。

snapshot ref cleanup:
  可以作为 backend-local 引用在锁后释放。
```

这也是读源码时必须区分的边界：

```text
ResourceOwner 管“这个对象我还拿着”；
snapmgr 管“这个 snapshot 对全局 xmin / active stack 有什么语义”。
```

### 层次六：文件和 pending deletes 不是一回事

`fd.c` 的 VFD 文件资源在 after-locks：

```c
.name = "File",
.release_phase = RESOURCE_RELEASE_AFTER_LOCKS,
.release_priority = RELEASE_PRIO_FILES,
```

`ResOwnerReleaseFile()` 做的是：

```text
vfdP->resowner = NULL
FileClose(file)
```

这和 `xact.c` 里的 `smgrDoPendingDeletes()` 不是同一层：

```text
VFD File:
  当前 backend 打开的虚拟文件描述符 cleanup。

smgr pending deletes:
  事务提交 / 回滚后真正删除或保留 relation storage 文件。
```

`smgrDoPendingDeletes()` 被放在锁释放之后，是因为删除文件可能很慢。源码注释说，这样其他 backend 已经能通过 catalog change 知道不要访问相关文件，不必继续被我们的锁阻塞。

所以不要简单地说“文件都在锁后释放”。要区分：

```text
ResourceOwner tracked VFD file:
  backend-local handle cleanup。

storage pending delete:
  事务级文件系统动作，由 smgr 的 pending delete 机制处理。
```

## 8. 错误路径 / 异常路径 / fallback

### commit 残留资源：WARNING，而不是 ERROR

`ResourceOwnerReleaseAll()` 在 `isCommit = true` 时发现普通资源还在，会调用 `DebugPrint` 并打印：

```text
resource was not closed: ...
```

这不是 ERROR。原因是：

```text
事务已经提交或处于不可回头的 cleanup 阶段；
这里不能用 ERROR 再试图回滚；
正确动作是打印 WARNING，并尽力释放资源。
```

因此 commit leak warning 的语义是：

```text
代码路径没有按预期 retail release；
ResourceOwner 兜底救场了；
需要修 bug，但当前 backend 尽量继续保持一致。
```

### abort 残留资源：正常现象

`isCommit = false` 时不打印普通 leak warning，因为 ERROR 可能在任意位置打断执行。比如：

```text
ReadBuffer() 后尚未来得及 ReleaseBuffer()
SearchSysCache() 后尚未来得及 ReleaseSysCache()
RegisterSnapshot() 后尚未来得及 UnregisterSnapshot()
OpenTemporaryFile() 后尚未来得及 FileClose()
```

这时 ResourceOwner 的目标不是报警，而是恢复到可继续服务下一个命令的 backend 状态。

### release phase 之间再次 ERROR

`ResourceOwnerReleaseInternal()` 注释提到：

```text
如果 release phases 之间发生错误，
AbortTransaction 可能再次对同一个 ResourceOwner 调用 ResourceOwnerRelease。
```

所以 `owner->releasing` 已经为 true 时，代码不能假定这是 bug。它会继续使用已经排序过的资源表，按剩余 phase 清理。

这解释了为什么 callback 不能做容易 ERROR 的工作。release 已经在错误恢复路径上，再次 ERROR 会让清理逻辑更脆弱。

### 锁 cache 溢出 fallback

`ResourceOwnerData` 里有一个小的 `locks[MAX_RESOWNER_LOCKS]` 缓存，用于快速处理当前 owner 持有的 lock。

locks 阶段：

```text
如果 owner->nlocks <= MAX_RESOWNER_LOCKS:
  直接把 LOCALLOCK* 数组传给 lock.c，避免扫整个 LockMethodLocalHash。

如果溢出:
  传 NULL，让 lock.c 扫描本 backend 的 local lock hash。
```

这就是 ResourceOwner 在 hot path 和 worst-case 之间的折中：

```text
少量锁:
  O(number of locks in owner)

锁缓存溢出:
  O(number of local locks in backend)
```

典型场景如 `pg_dump` 访问大量 relation，会让锁数量很大。源码注释也专门提到传数组能在大量锁时显著加速。

### Portal FAILED 特例

`PortalDrop()` 中：

```text
top transaction commit:
  通常不需要在 PortalDrop 释放资源，事务 owner 机制会处理。

FAILED portal:
  要主动 release portal->resowner，避免后续 commit 阶段打出资源泄漏 warning。

main/sub transaction abort:
  portal->resowner 已经置 NULL，资源已由事务 abort 清理。
```

这个特例说明：三阶段 release 是统一工具，但调用者仍然要根据自己的生命周期决定何时交给它。

## 9. 成本、资源与跨模块传播

### 排序成本

ResourceOwner release 开始时会把 `arr` 或 `hash` 中的资源按 phase / priority 排序。这样做的收益是：

```text
release 时可以按 phase 分段推进；
同一 owner 内资源类型顺序稳定；
phase 之间如果需要插入 xact.c 工作，不必重新组织资源表。
```

成本是：

```text
owner 中资源越多，release 开始时排序越贵；
一旦 sorted，不能再 retail Remember / Forget；
bulk release callback 必须服从低层、不可失败约束。
```

正常 commit 下，这个成本通常应该很低，因为大多数非锁资源已经在执行路径中 retail release。abort 下成本可能更高，但 abort 已经是错误恢复路径，正确性优先。

### 锁释放成本

顶层事务：

```text
ResourceOwnerRelease(LOCKS, isTopLevel=true)
  -> ProcReleaseLocks(isCommit)
  -> LockReleaseAll(...)
  -> ReleasePredicateLocks(...)
```

子事务或 Portal owner：

```text
ResourceOwnerRelease(LOCKS, isTopLevel=false)
  -> 如果 commit: LockReassignCurrentOwner()
  -> 如果 abort:  LockReleaseCurrentOwner()
```

成本取决于：

```text
当前 owner 持有多少 lock
锁 cache 是否溢出
本 backend local lock hash 有多大
是否涉及 predicate lock
```

这也是为什么很多 schema-heavy workload、`pg_dump`、大量 partition / relation 的事务，会把锁管理和 ResourceOwner release 推到诊断视野里。

### xact.c 的夹层 cleanup 是跨模块传播点

三阶段不是只在 `resowner.c` 里完成。`xact.c` 在 before-locks 和 locks 之间插入：

```text
AtEOXact_Aio()
AtEOXact_Buffers()
AtEOXact_RelationCache()
AtEOXact_TypeCache()
AtEOXact_Inval()
AtEOXact_MultiXact()
```

这些函数不是 ResourceOwner 资源项，但它们和释放顺序同样相关：

```text
AtEOXact_Buffers:
  检查 buffer pins 是否已经清掉。

AtEOXact_RelationCache:
  relcache 事务级 cleanup / cross-check。

AtEOXact_Inval:
  提交时发送 invalidation；abort 时本地处理或丢弃 pending messages。

AtEOXact_MultiXact:
  结束 multixact 相关事务状态。
```

所以真实模型应该是：

```text
ResourceOwner phase ordering
  +
xact.c transaction-end hook ordering
  +
lock manager release/reassign semantics
```

不要只看 `ResourceOwnerDesc` 表就以为掌握了事务结束顺序。

### 扩展资源类型如何选 phase

给扩展或新内核资源加 `ResourceOwnerDesc` 时，可以用这组问题判断：

```text
1. 锁释放后，其他 backend 是否可能因为这个资源没释放而等待、误判或访问旧状态？
   是 -> before-locks。

2. 这个资源是否只是当前 backend 的引用计数、内存外句柄或本地 cache ref？
   通常 -> after-locks。

3. 它是否和 lock 本身具有事务继承 / 子事务转移语义？
   可能需要 lock-like 特殊处理，而不是普通 ResourceOwnerDesc。

4. ReleaseResource 是否可能 ERROR、分配内存、访问用户对象或产生用户可见副作用？
   如果是，要重构；release callback 不是做这些事的地方。

5. 它依赖哪个资源先释放？
   用 release_priority 表达同 phase 内顺序。
```

## 10. 观测与诊断入口

### 观察 lock 释放点

可以用两个会话观察锁等待：

会话 A：

```sql
BEGIN;
LOCK TABLE pg_class IN ACCESS EXCLUSIVE MODE;
SELECT pg_sleep(30);
COMMIT;
```

会话 B：

```sql
SELECT * FROM pg_class LIMIT 1;
```

会话 C：

```sql
SELECT pid, locktype, relation::regclass, mode, granted
FROM pg_locks
WHERE relation = 'pg_class'::regclass
ORDER BY granted, pid;
```

你能看到的是 lock 等待和释放，看不到的是：

```text
ResourceOwnerRelease(BEFORE_LOCKS)
AtEOXact_Inval()
ResourceOwnerRelease(AFTER_LOCKS)
```

这正好说明本节的诊断边界：

```text
pg_locks 能看 locks 阶段；
before-locks / after-locks 多数只能通过 gdb、日志、断点或 leak warning 推断。
```

### 用 gdb 跟三阶段

调试自编译 PostgreSQL 时，可以在 backend 进程上设置断点：

```gdb
break ResourceOwnerRelease
commands
  silent
  printf "ResourceOwnerRelease owner=%p phase=%d isCommit=%d isTopLevel=%d\n", owner, phase, isCommit, isTopLevel
  continue
end

break ProcReleaseLocks
break LockReassignCurrentOwner
break LockReleaseCurrentOwner
break AtEOXact_Inval
```

运行普通事务 commit，会看到顶层 owner 的三次 release：

```text
phase=1, isCommit=1, isTopLevel=1
phase=2, isCommit=1, isTopLevel=1
phase=3, isCommit=1, isTopLevel=1
```

运行 `SAVEPOINT` 后提交子事务，可以观察：

```text
subowner phase=1, isCommit=1, isTopLevel=0
LockReassignCurrentOwner()
subowner phase=3, isCommit=1, isTopLevel=0
```

运行 `ROLLBACK TO SAVEPOINT`，可以观察：

```text
subowner phase=1, isCommit=0, isTopLevel=0
LockReleaseCurrentOwner()
subowner phase=3, isCommit=0, isTopLevel=0
```

### 用 leak warning 定位 retail release 漏洞

如果正常 commit 看到类似：

```text
WARNING:  resource was not closed: ...
```

诊断思路不是“ResourceOwner 出错了”，而是：

```text
1. 看 warning 中的资源名和 DebugPrint 输出。
2. 找对应 ResourceOwnerDesc。
3. 找 Remember 入口和正常 Forget 入口。
4. 判断哪条正常路径少了 retail release。
5. 再看 abort 路径是否仍能由 bulk release 兜底。
```

常见资源名对应：

```text
"buffer":
  buffer pin 未 ReleaseBuffer()

"relcache reference":
  RelationClose() 或相应 decrement path 缺失

"catcache reference":
  ReleaseSysCache() 缺失

"snapshot reference":
  UnregisterSnapshot() 缺失

"File":
  FileClose() 缺失
```

### 看源码时的搜索入口

推荐搜索：

```bash
rg -n "ResourceOwnerDesc|release_phase|release_priority" src/backend src/include
rg -n "ResourceOwnerRelease\\(" src/backend/access/transam src/backend/utils/mmgr
rg -n "resource was not closed|DebugPrint" src/backend
rg -n "AtEOXact_Inval|AtEOXact_RelationCache|ProcReleaseLocks" src/backend
```

阅读时把每个结果归入三类：

```text
资源类型声明:
  ResourceOwnerDesc

释放调度点:
  ResourceOwnerRelease(...)

事务级夹层:
  AtEOXact_* / AtEOSubXact_* / smgrDoPendingDeletes
```

## 11. 常见误区

### 误区一：三阶段只是为了让代码更整齐

不是。三阶段定义了锁释放前后的并发可见性边界：

```text
锁前:
  清掉其他 backend 可见或可等待的占用。

锁后:
  清掉本 backend 的引用和句柄。
```

### 误区二：所有 catalog 相关资源都应在锁前释放

不对。`relcache ref` 在锁前，`catcache ref` 在锁后。原因不是名字里有没有 cache，而是：

```text
这个释放动作是否影响等待者醒来后的安全执行。
```

catalog change 的跨 backend 可见性主要靠 invalidation 和锁顺序，不靠把当前 backend 的所有 catcache ref 都赶在锁前释放。

### 误区三：snapshot 在 after-locks，说明 snapshot 不影响并发

不对。snapshot 对并发和 vacuum horizon 很重要，但 ResourceOwner 释放的是 snapshot 对象引用。snapshot 的全局语义还由 ProcArray、snapmgr active stack、transaction callbacks 等机制维护。

### 误区四：commit 时 ResourceOwnerRelease 应该总是空操作

普通非锁资源在 commit 时理想上应已 retail release，所以残留会 WARNING。但锁是例外：

```text
事务级锁正常持有到事务结束；
子事务 commit 时锁正常转移给父 owner；
顶层 commit/abort 时才统一释放。
```

### 误区五：ReleaseResource callback 可以顺手做复杂修复

不能。callback 发生在 commit/abort cleanup 阶段，必须只做低层、不可失败、无用户可见副作用的清理。需要复杂逻辑时，应该放在正常执行路径或事务级 hook 中，而不是 ResourceOwner bulk release callback。

## 12. 课堂实验

### 实验一：画出本版本内置资源 release 表

在 `/home/highgo/postgres` 中运行：

```bash
rg -n "ResourceOwnerDesc|release_phase|release_priority" src/backend src/include
```

任务：

```text
1. 列出所有 ResourceOwnerDesc。
2. 标注 before-locks / after-locks。
3. 写出每个资源的 ReleaseResource callback。
4. 判断它是跨 backend 可见资源，还是 backend-local cleanup。
```

预期收获：

```text
你会发现 ResourceOwnerDesc 不是“资源注册表”，而是事务结束 ordering contract。
```

### 实验二：观察子事务 commit 与 abort 的 lock 分流

会话 A：

```sql
BEGIN;
SAVEPOINT s;
LOCK TABLE pg_class IN SHARE MODE;
RELEASE SAVEPOINT s;
SELECT pg_backend_pid();
```

用 gdb 断在：

```gdb
break LockReassignCurrentOwner
break LockReleaseCurrentOwner
```

再做 abort 版本：

```sql
BEGIN;
SAVEPOINT s;
LOCK TABLE pg_class IN SHARE MODE;
ROLLBACK TO SAVEPOINT s;
```

观察：

```text
RELEASE SAVEPOINT:
  LockReassignCurrentOwner()

ROLLBACK TO SAVEPOINT:
  LockReleaseCurrentOwner()
```

解释：

```text
同样的 ResourceOwnerRelease(LOCKS)，因为 isCommit 不同，锁语义完全不同。
```

### 实验三：制造并定位一个假想资源泄漏

不要直接改生产环境。可以在本地源码实验分支里临时注释某个正常释放调用，例如一条测试路径中的 `ReleaseSysCache()`，然后运行相关 SQL。

观察：

```text
commit 时是否出现 "resource was not closed" WARNING；
warning 中的资源名来自哪个 ResourceOwnerDesc；
abort 时是否没有同样 WARNING。
```

分析：

```text
commit warning 说明正常路径少了 retail release；
abort 静默 cleanup 说明 ResourceOwner 兜底路径工作正常。
```

实验后必须还原代码。

### 实验四：从一个新资源倒推 phase

假设你给扩展加了一个 backend-local C handle：

```text
每个查询创建；
只在当前 backend 使用；
释放只是关闭本地 handle；
不影响其他 backend 是否能继续访问 relation / buffer / lock。
```

问题：

```text
应该选 before-locks 还是 after-locks？
priority 应该靠近哪个内置资源？
ReleaseResource 里哪些动作不允许？
```

再假设它是一个共享段引用，其他 worker 可能等待它 detach：

```text
phase 选择会不会改变？
为什么？
```

## 13. 讨论题

1. 如果把 buffer pin 放到 after-locks，会出现哪些等待者已经被唤醒但资源仍未释放的场景？

2. 为什么 relcache ref 在 before-locks，而 catcache ref 在 after-locks？这个差异说明“资源名字”不能直接决定 release phase。

3. `AtEOXact_Inval(true)` 为什么要在释放锁之前？如果等待 DDL 锁的 backend 醒来后还没收到 invalidation，会有什么风险？

4. 为什么 `ResourceOwnerRelease()` 每个 phase 都要 child-first？如果 parent 先释放，Portal owner 或 subtransaction owner 的残留资源可能怎样破坏生命周期边界？

5. 为什么 release callback 不能做可能失败的工作？如果 callback 内部 `ereport(ERROR)`，事务结束路径会变成什么状态？

6. 一个扩展资源如果“正常 commit 应该已经释放，abort 需要兜底释放”，它的 DebugPrint 应该提供哪些信息，才能让 leak warning 可诊断？

7. `smgrDoPendingDeletes()` 为什么不是普通 ResourceOwnerDesc？它和 `fd.c` 的 VFD File cleanup 在语义上有什么不同？

## 14. 本节小结

本节的可迁移模型：

```text
事务结束不是“把资源都 free 掉”；
而是先把其他 backend 可见的占用清到安全边界，
再释放并发等待边界，
最后清理当前 backend 的本地引用和句柄。
```

`ResourceOwnerRelease()` 的三阶段顺序可以压缩成：

```text
before-locks:
  我还占着哪些别人可能感知到的东西？

locks:
  谁在等我离开并发边界？

after-locks:
  我自己还需要把哪些引用和句柄归还？
```

源码上要记住三组事实：

```text
1. ResourceOwnerDesc 声明资源类型的 phase / priority。
2. ResourceOwnerReleaseInternal() 每个 phase child-first，并在锁阶段特殊处理 top-level / subtransaction / Portal 语义。
3. xact.c 在 phase 之间插入 relcache、invalidation、buffer、smgr 等事务级 cleanup，真实顺序跨越多个模块。
```

把这个模型带回诊断现场：

```text
看到 commit leak warning:
  找正常路径漏掉的 retail release。

看到 lock wait / release 问题:
  区分 top-level release、subtransaction reassign、abort release。

设计新 ResourceOwner 资源:
  先问锁释放后其他 backend 是否会因为它没释放而观察到错误世界。
```

下一节会继续沿着 ERROR-safe cleanup 往下走：当 release 顺序已经确定，PostgreSQL 如何在 abort、failed Portal、callback、half-initialized resource 等路径上尽量保证 cleanup 本身不再制造新的错误。
