# PostgreSQL 事务、子事务与 Portal owner 传播

## 课程定位

前置知识：已经理解 `ResourceOwner` 为什么独立于 `MemoryContext`，也已经知道 `ResourceOwnerEnlarge()` / `Remember()` / `Forget()` 如何把资源获取路径做成 ERROR-safe。

本节唯一主问题：

```text
CurrentResourceOwner 如何在 top transaction、subtransaction 和 Portal 执行之间切换，为什么子事务提交时锁要转移给父 owner，而 abort 时必须释放？
```

核心矛盾：PostgreSQL 希望深层执行代码不用显式传递 owner，就能把 buffer pin、snapshot、lock、文件、cache ref 等资源挂到“当前生命周期”；但一条 SQL 的执行生命周期并不只是一条直线。它可能处在顶层事务、子事务、Portal 执行、cursor fetch、异常回滚、utility command 内部提交等不同控制流里。`CurrentResourceOwner` 必须随着控制流切换，又不能让资源归属和事务语义错位。

学完后应能判断：一个资源到底会挂到 top transaction owner、subtransaction owner，还是 Portal owner；Portal 在执行时为什么要临时接管 `CurrentResourceOwner`；子事务 commit 为什么不能简单释放所有资源；以及 lock 为什么在子事务 commit 和 abort 下走两条完全不同的路径。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `6b48f5d1a741`。

## 1. 本节在总主线中的位置

前两节已经解决了两个问题：

```text
第 10 课:
  为什么外部资源要挂 ResourceOwner tree，而不是 MemoryContext tree？

第 11 课:
  单次资源 acquire 如何保证 Enlarge -> Remember -> Forget 的 ERROR-safe 窗口？
```

本节进入第三层：

```text
这些 Remember 进去的资源，到底记到哪个 owner？
```

这不是一个小细节。因为 `ResourceOwnerRemember(CurrentResourceOwner, ...)` 的调用点往往藏在很深的模块里：

```text
ReadBuffer()
RegisterSnapshot()
LockAcquire()
OpenTemporaryFile()
SearchSysCache()
GetCachedPlan()
```

这些函数通常不知道自己处在：

```text
普通顶层事务
  vs
SAVEPOINT 里面的子事务
  vs
DECLARE CURSOR / FETCH 的 Portal 执行
  vs
Portal cleanup
  vs
utility command 内部事务切换
```

所以 ResourceOwner 传播的核心不是“有一棵树”这么简单，而是：

```text
谁在进入一个执行生命周期时设置 CurrentResourceOwner；
谁在离开时恢复；
谁在 commit / abort 时决定 child owner 的资源是继承、释放还是诊断。
```

本节不展开三阶段 release 的完整顺序；那是下一节。这里聚焦 owner 在事务、子事务和 Portal 之间的传播，以及 lock 在子事务结束时的特殊语义。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
xact.c 创建 top/subtransaction owner 形成事务 owner tree；
portalmem.c 创建 Portal owner 并把它挂到当前事务 owner 下；
pquery.c 在 PortalStart / PortalRun / PortalRunFetch 期间把 CurrentResourceOwner 临时切到 portal->resowner；
ResourceOwnerRelease 在子事务 commit 时把 lock reassign 给父 owner，在 abort 时 release 当前 owner 的锁。
```

本节 tension 是：

```text
深层代码需要一个隐式 CurrentResourceOwner 来降低接口复杂度
  vs
资源实际生命周期必须精确匹配事务提交、子事务回滚和 Portal 关闭语义
```

如果只用一个全局 owner，子事务失败时就无法只释放子事务内获取的资源：

```text
BEGIN;
SAVEPOINT s;
-- 子事务内读 buffer、拿 lock、注册 snapshot
ROLLBACK TO s;
-- 外层事务还要继续运行
```

如果每个函数都显式传 owner，接口会污染大量存储、执行、cache 和 utility 代码。PostgreSQL 的选择是：

```text
控制流边界负责切换 CurrentResourceOwner；
资源获取点只记住 CurrentResourceOwner；
事务/Portal cleanup 再按 owner tree 做批量处理。
```

这让 deep code 保持简单，但把正确性压力集中在少数边界函数：

```text
AtStart_ResourceOwner()
AtSubStart_ResourceOwner()
PortalStart()
PortalRun()
PortalRunFetch()
PortalDrop()
PreCommit_Portals()
AtSubCommit_Portals()
AtSubAbort_Portals()
ResourceOwnerRelease()
```

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 作用 |
| --- | --- | --- |
| 1 | `src/backend/access/transam/xact.c` | 创建 top/subtransaction owner，并在 commit / abort / cleanup 中 release、delete、恢复父 owner。 |
| 2 | `src/include/utils/portal.h` | 定义 `PortalData` 中的 `resowner`、`createSubid`、`activeSubid`、`portalContext` 等传播状态。 |
| 3 | `src/backend/utils/mmgr/portalmem.c` | 创建 Portal owner、Portal drop cleanup、事务结束和子事务结束时调整 Portal owner 归属。 |
| 4 | `src/backend/tcop/pquery.c` | `PortalStart()`、`PortalRun()`、`PortalRunFetch()` 临时切换 `CurrentResourceOwner` 和 `PortalContext`。 |
| 5 | `src/backend/utils/resowner/resowner.c` | `ResourceOwnerRelease()` 在 release phase 中递归 child owner，并在 lock phase 对 commit / abort 分流。 |
| 6 | `src/backend/storage/lmgr/lock.c` | 实现 `LockReassignCurrentOwner()` 和 `LockReleaseCurrentOwner()`，解释锁为什么在子事务 commit 时继承，在 abort 时释放。 |
| 7 | `src/backend/utils/time/snapmgr.c` | snapshot 的子事务 active stack relabel / cleanup，帮助区分 ResourceOwner 资源和模块自有子事务状态。 |

推荐阅读顺序：

```text
top transaction 创建 owner
  -> subtransaction 创建 child owner
  -> Portal 创建 child owner
  -> Portal 执行时切换 CurrentResourceOwner
  -> 子事务 commit: child owner 的资源如何流向 parent
  -> 子事务 abort: child owner 的资源如何被释放
```

不要从所有 `CurrentResourceOwner = ...` 搜索结果开始横向扫。那样会看到很多局部保存/恢复，却很难形成主线。先抓住 owner tree 的结构，再看执行边界如何临时改“当前指针”。

## 4. 关键数据结构与状态

### 事务状态里的三个 owner 指针

`xact.c` 中最关键的 owner 状态有三层：

```text
TopTransactionResourceOwner:
  顶层事务 owner，全事务结束时 release/delete

CurTransactionResourceOwner:
  当前事务层级的 owner；进入子事务后指向子事务 owner，离开后恢复父 owner

CurrentResourceOwner:
  deep code 默认使用的 owner；Portal 执行期间可能临时指向 portal->resowner
```

这三个名字容易混淆。可以先用这组判断：

```text
TopTransactionResourceOwner:
  “这一整个 top-level transaction 最外层的 owner 是谁？”

CurTransactionResourceOwner:
  “当前事务栈层级，也就是当前 subtransaction 层级的 owner 是谁？”

CurrentResourceOwner:
  “现在 deep code 调用 ResourceOwnerRemember() 时默认记到谁？”
```

普通执行中三者可能相同：

```text
BEGIN 后、未进入 Portal 临时切换前:
  TopTransactionResourceOwner == CurTransactionResourceOwner
  CurrentResourceOwner == CurTransactionResourceOwner
```

进入子事务后：

```text
TopTransactionResourceOwner:
  仍然是顶层 owner

CurTransactionResourceOwner:
  指向当前 SubTransaction owner

CurrentResourceOwner:
  通常也指向当前 SubTransaction owner
```

进入 Portal 执行后：

```text
CurTransactionResourceOwner:
  仍然表示事务层级

CurrentResourceOwner:
  临时指向 portal->resowner
```

这就是本节最重要的状态边界：

```text
CurTransactionResourceOwner 表示事务栈；
CurrentResourceOwner 表示当前资源登记目标。
```

### `PortalData` 里的 owner 和 subtransaction 标记

`portal.h` 中的 `PortalData` 有几组字段需要一起看：

```text
portalContext:
  Portal 私有内存上下文

resowner:
  Portal 执行期间获取资源的 owner

createSubid / activeSubid / createLevel:
  Portal 在哪个子事务创建，最近在哪个子事务活动

status:
  PORTAL_NEW / DEFINED / READY / ACTIVE / DONE / FAILED

cleanup:
  executor / portal command 层的 cleanup hook
```

不要把 `portal->resowner` 理解成“cursor 生命周期独占所有资源”。更准确地说：

```text
Portal owner 是 Portal 执行期间 deep code 的默认资源登记点；
它本身挂在事务 owner tree 下；
在事务或子事务结束时，它会被保留、转移、释放或置空。
```

### lock 的双层 owner 状态

`lock.c` 里 lock ownership 不只在 ResourceOwner 里有记录，还在 `LOCALLOCK` 里有 `lockOwners`：

```text
LOCALLOCK:
  当前 backend 对某个 locktag/mode 的本地记录

LOCALLOCKOWNER:
  某个 ResourceOwner 对这个 LOCALLOCK 持有多少次

locallock->nLocks:
  当前 backend 对这个锁的总持有次数
```

`ResourceOwner` 的 lock 记录用于快速找到“这个 owner 持有哪些 locallock”。`LOCALLOCK` 的 `lockOwners` 则用于描述同一个 backend 内不同 owner 对同一把锁的持有次数。

这解释了为什么子事务 commit 不能只改 ResourceOwner tree：

```text
子事务拿到的 lock 在语义上要继续由父事务持有；
因此 LOCALLOCK.lockOwners 里的 owner/count 也必须改成父 owner。
```

## 5. 主流程源码 walkthrough

### 5.1 顶层事务开始：创建 top transaction owner

事务开始时，`StartTransaction()` 先初始化内存和 ResourceOwner：

```text
StartTransaction()
  -> AtStart_Memory()
  -> AtStart_ResourceOwner()
```

`AtStart_ResourceOwner()` 做三件事：

```text
s->curTransactionOwner = ResourceOwnerCreate(NULL, "TopTransaction")

TopTransactionResourceOwner = s->curTransactionOwner
CurTransactionResourceOwner = s->curTransactionOwner
CurrentResourceOwner = s->curTransactionOwner
```

这建立了顶层基线：

```text
ResourceOwner tree:
  TopTransaction

当前资源登记点:
  TopTransaction
```

之后如果 deep code 获取资源：

```text
ResourceOwnerRemember(CurrentResourceOwner, ...)
```

默认就会记到 `TopTransaction` owner 下。

### 5.2 子事务开始：创建 child owner 并切换当前事务 owner

进入 `SAVEPOINT` / PL 异常块等子事务时，`AtSubStart_ResourceOwner()` 创建 child owner：

```text
s->curTransactionOwner =
  ResourceOwnerCreate(s->parent->curTransactionOwner, "SubTransaction")

CurTransactionResourceOwner = s->curTransactionOwner
CurrentResourceOwner = s->curTransactionOwner
```

这时 owner tree 变成：

```text
TopTransaction
  -> SubTransaction
```

这条 parent link 很关键。它不是为了“自动继承所有资源”，而是为了在 release 时能递归处理 child，并在某些资源类型上知道 parent 是谁。

子事务内 deep code 默认记到子 owner：

```text
SAVEPOINT s;
SELECT ...
  -> ReadBuffer()
  -> LockAcquire()
  -> RegisterSnapshot()
  -> ResourceOwnerRemember(CurrentResourceOwner, ...)
     CurrentResourceOwner == SubTransaction owner
```

因此子事务 abort 时可以只释放子 owner 下的资源，而不伤到外层事务。

### 5.3 Portal 创建：Portal owner 挂到当前事务 owner 下

`CreatePortal()` 在 `portalmem.c` 中创建 `PortalData`，并创建 Portal owner：

```text
portal->resowner = ResourceOwnerCreate(CurTransactionResourceOwner, "Portal")
portal->createSubid = GetCurrentSubTransactionId()
portal->activeSubid = portal->createSubid
portal->createLevel = GetCurrentTransactionNestLevel()
```

如果在顶层事务创建 Portal：

```text
TopTransaction
  -> Portal
```

如果在子事务里创建 Portal：

```text
TopTransaction
  -> SubTransaction
       -> Portal
```

注意 parent 是 `CurTransactionResourceOwner`，不是 `CurrentResourceOwner`。这是有意的：

```text
Portal 的生命周期首先属于当前事务层级；
即使当前代码短暂切换过 CurrentResourceOwner，Portal owner 仍应挂到事务 owner tree 的正确层级。
```

### 5.4 Portal 执行：临时把 CurrentResourceOwner 切到 portal->resowner

`PortalStart()`、`PortalRun()` 和 `PortalRunFetch()` 都采用类似模式：

```text
saveActivePortal = ActivePortal
saveResourceOwner = CurrentResourceOwner
savePortalContext = PortalContext

PG_TRY();
{
  ActivePortal = portal
  if (portal->resowner)
    CurrentResourceOwner = portal->resowner
  PortalContext = portal->portalContext

  MemoryContextSwitchTo(PortalContext)

  执行 executor / fetch / utility path
}
PG_CATCH();
{
  MarkPortalFailed(portal)
  ActivePortal = saveActivePortal
  CurrentResourceOwner = saveResourceOwner
  PortalContext = savePortalContext
  PG_RE_THROW()
}
PG_END_TRY();

恢复 ActivePortal / CurrentResourceOwner / PortalContext
```

这个切换把 deep code 的默认资源登记点变成 Portal owner：

```text
PortalRun()
  -> ExecutorRun()
     -> heap scan / index scan / expression evaluation
        -> buffer pin / lock / snapshot / cache ref
           -> ResourceOwnerRemember(CurrentResourceOwner, ...)
              CurrentResourceOwner == portal->resowner
```

为什么 Portal 要这样做？

因为 Portal 可以被多次 fetch、可以 suspend、可以 close，也可能在错误后进入 `PORTAL_FAILED`。如果所有资源都直接挂在事务 owner 上，普通 `CLOSE cursor` 或 Portal cleanup 很难只处理这个 Portal 执行期间遗留的资源。

Portal owner 的作用是增加一个中间生命周期：

```text
transaction owner:
  事务结束才兜底释放

portal owner:
  Portal drop / failed cleanup / fetch 生命周期结束时可以局部释放
```

### 5.5 PortalRun 的特殊恢复：utility command 内部事务切换

`PortalRun()` 有一段看起来很尴尬但很重要的逻辑：

```text
saveTopTransactionResourceOwner = TopTransactionResourceOwner
saveResourceOwner = CurrentResourceOwner
...
if (saveResourceOwner == saveTopTransactionResourceOwner)
  CurrentResourceOwner = TopTransactionResourceOwner
else
  CurrentResourceOwner = saveResourceOwner
```

注释解释了原因：一些 utility command，例如 `VACUUM`、`CLUSTER` 类路径，可能内部启动和提交事务。进入 `PortalRun()` 时保存的 `TopTransactionResourceOwner` 可能在内部提交后被销毁并替换。

因此恢复时不能盲目写回旧指针：

```text
错误做法:
  CurrentResourceOwner = saveResourceOwner
  -- 如果 saveResourceOwner 是旧 TopTransactionResourceOwner，可能已经无效

实际做法:
  如果进入时 CurrentResourceOwner 就是当时的 TopTransactionResourceOwner，
  退出时恢复为新的 TopTransactionResourceOwner。
```

这说明 `CurrentResourceOwner` 不是一个简单 TLS 变量。它承载的是“当前执行控制流的资源登记语义”，而这个语义在 utility command 的内部事务边界上会发生替换。

### 5.6 子事务 commit：非锁资源 release，锁转移给父 owner

`CommitSubTransaction()` 的后半段是本节的核心路径：

```text
AtSubCommit_Portals(...)

ResourceOwnerRelease(s->curTransactionOwner,
                     RESOURCE_RELEASE_BEFORE_LOCKS,
                     true, false)

CurrentResourceOwner = s->curTransactionOwner
XactLockTableDelete(...)

ResourceOwnerRelease(s->curTransactionOwner,
                     RESOURCE_RELEASE_LOCKS,
                     true, false)

ResourceOwnerRelease(s->curTransactionOwner,
                     RESOURCE_RELEASE_AFTER_LOCKS,
                     true, false)

CurrentResourceOwner = s->parent->curTransactionOwner
CurTransactionResourceOwner = s->parent->curTransactionOwner
ResourceOwnerDelete(s->curTransactionOwner)
```

这里有两个细节。

第一，Portal owner 先处理：

```text
AtSubCommit_Portals(mySubid, parentSubid, parentLevel, parentXactOwner)
  如果 portal->createSubid == mySubid:
    portal->createSubid = parentSubid
    portal->createLevel = parentLevel
    ResourceOwnerNewParent(portal->resowner, parentXactOwner)
```

也就是说，子事务创建的 Portal 如果子事务成功提交，就变成父事务创建的 Portal，Portal owner 也重新挂到父事务 owner 下：

```text
commit 前:
  TopTransaction
    -> SubTransaction
         -> Portal

commit 后:
  TopTransaction
    -> Portal
```

第二，lock phase 不是真释放所有锁，而是根据 `isCommit=true` 转移：

```text
ResourceOwnerRelease(..., RESOURCE_RELEASE_LOCKS, isCommit=true, isTopLevel=false)
  -> LockReassignCurrentOwner(...)
```

这就是“子事务提交时锁要转移给父 owner”的源码落点。

为什么？

因为 PostgreSQL 的子事务 commit 不是对外独立可见的 commit。它只是把子事务成果合并进父事务：

```text
子事务内创建/修改的对象:
  父事务后续还可能使用

子事务内拿到的事务级锁:
  必须继续保护这些对象直到外层事务结束

子事务 owner:
  可以删除，但它持有的成功成果要变成父事务责任
```

如果子事务 commit 时释放锁，会破坏父事务后续执行和并发可见性。例如：

```text
BEGIN;
SAVEPOINT s;
CREATE TABLE t(...);
RELEASE SAVEPOINT s;
-- 外层事务尚未提交，t 的 catalog change 还需要锁保护
COMMIT;
```

`RELEASE SAVEPOINT` 后如果把子事务获得的 lock 都释放，其他 backend 可能过早观察或竞争尚未完成的事务状态。正确语义是：

```text
子事务成功:
  成果合并到父事务
  保护这些成果的锁也合并到父 owner
```

### 5.7 子事务 abort：子 owner 资源必须释放

`AbortSubTransaction()` 先做错误恢复准备：

```text
AtSubAbort_Memory()
AtSubAbort_ResourceOwner()
LWLockReleaseAll()
UnlockBuffers()
LockErrorCleanup()
...
```

然后如果子事务 owner 已创建：

```text
AtSubAbort_Portals(...)

ResourceOwnerRelease(s->curTransactionOwner,
                     RESOURCE_RELEASE_BEFORE_LOCKS,
                     false, false)

ResourceOwnerRelease(s->curTransactionOwner,
                     RESOURCE_RELEASE_LOCKS,
                     false, false)

ResourceOwnerRelease(s->curTransactionOwner,
                     RESOURCE_RELEASE_AFTER_LOCKS,
                     false, false)

AtSubAbort_Snapshot(...)
```

在 `RESOURCE_RELEASE_LOCKS` phase：

```text
ResourceOwnerRelease(..., isCommit=false, isTopLevel=false)
  -> LockReleaseCurrentOwner(...)
```

这次不是转移，而是释放。

原因也正好相反：

```text
子事务失败:
  子事务内的 tuple changes / catalog changes / temp namespace state 等要回滚
  子事务内获取、且只服务这些失败操作的锁不能继续由父事务持有
  子事务 owner 下的资源是失败分支遗留物，应释放而不是继承
```

如果 abort 时把锁转移给父 owner，会产生两个问题：

```text
父事务会继续持有失败操作留下的锁，扩大锁占用时间，甚至导致不必要阻塞

更严重的是，owner 语义会撒谎：
  父事务从未成功继承子事务成果，却继承了保护这些成果的锁
```

所以子事务结束的基本规律是：

```text
commit:
  成功成果向父事务传播
  锁 ownership 向父 owner 传播
  child owner 可以删除

abort:
  失败成果撤销
  child owner 资源释放
  locks 不向父 owner 传播
```

## 6. 生命周期 / ownership / cleanup

### top transaction owner

生命周期：

```text
StartTransaction()
  -> AtStart_ResourceOwner()
     创建 TopTransactionResourceOwner

CommitTransaction()
  -> ResourceOwnerRelease(TopTransactionResourceOwner, BEFORE_LOCKS, true, true)
  -> ResourceOwnerRelease(TopTransactionResourceOwner, LOCKS, true, true)
  -> ResourceOwnerRelease(TopTransactionResourceOwner, AFTER_LOCKS, true, true)
  -> ResourceOwnerDelete(TopTransactionResourceOwner)

AbortTransaction()
  -> ResourceOwnerRelease(TopTransactionResourceOwner, BEFORE_LOCKS, false, true)
  -> ResourceOwnerRelease(TopTransactionResourceOwner, LOCKS, false, true)
  -> ResourceOwnerRelease(TopTransactionResourceOwner, AFTER_LOCKS, false, true)

CleanupTransaction()
  -> ResourceOwnerDelete(TopTransactionResourceOwner)
```

commit 路径中，如果某些本该正常释放的资源仍留在 owner 下，`ResourceOwnerRelease()` 可以发 WARNING。abort 路径下，剩余资源通常是错误恢复预期的一部分，所以以兜底释放为主。

### subtransaction owner

生命周期：

```text
AtSubStart_ResourceOwner()
  -> ResourceOwnerCreate(parent->curTransactionOwner, "SubTransaction")
  -> CurTransactionResourceOwner = child
  -> CurrentResourceOwner = child

CommitSubTransaction()
  -> ResourceOwnerRelease(child, BEFORE_LOCKS, true, false)
  -> ResourceOwnerRelease(child, LOCKS, true, false)
     locks reassign to parent
  -> ResourceOwnerRelease(child, AFTER_LOCKS, true, false)
  -> CurrentResourceOwner = parent
  -> ResourceOwnerDelete(child)

AbortSubTransaction()
  -> ResourceOwnerRelease(child, BEFORE_LOCKS, false, false)
  -> ResourceOwnerRelease(child, LOCKS, false, false)
     locks release from current owner
  -> ResourceOwnerRelease(child, AFTER_LOCKS, false, false)

CleanupSubTransaction()
  -> AtSubCleanup_Portals()
  -> CurrentResourceOwner = parent
  -> ResourceOwnerDelete(child)
```

这里要区分 abort 和 cleanup：

```text
AbortSubTransaction:
  做语义释放，释放资源、撤销锁、清理模块状态

CleanupSubTransaction:
  在 abort 后做结构性清理，删除 portal/memory/owner 等对象
```

### Portal owner

生命周期：

```text
CreatePortal()
  -> ResourceOwnerCreate(CurTransactionResourceOwner, "Portal")

PortalStart / PortalRun / PortalRunFetch
  -> CurrentResourceOwner = portal->resowner
  -> deep code 的资源记到 Portal owner
  -> 退出时恢复旧 CurrentResourceOwner

PortalDrop()
  -> cleanup hook
  -> release cached plan / hold snapshot
  -> ResourceOwnerRelease(portal->resowner, ...)
  -> ResourceOwnerDelete(portal->resowner)
```

Portal 在事务结束时有多种命运：

```text
普通非 holdable Portal:
  PreCommit_Portals() 中 PortalDrop(portal, true)

holdable cursor:
  HoldPortal() 转换为跨事务可访问状态

abort 中的 Portal:
  AtAbort_Portals() 让 portal->resowner = NULL
  资源由事务范围 cleanup 释放
  AtCleanup_Portals() 再删除 Portal 结构
```

子事务 commit 时：

```text
AtSubCommit_Portals()
  Portal 的 createSubid / activeSubid 改成父层级
  portal->resowner 重新挂到 parentXactOwner
```

子事务 abort 时：

```text
AtSubAbort_Portals()
  子事务创建的 Portal 标记 FAILED
  cleanup hook / cached plan cleanup
  portal->resowner = NULL
  资源交给 upcoming transaction-wide cleanup 释放

AtSubCleanup_Portals()
  删除失败子事务创建的 Portal
```

## 7. 正确性机制层次

### 层次一：CurrentResourceOwner 只是一根当前指针，不是资源本身

`CurrentResourceOwner` 的切换不释放资源，也不修改 shared state。它只是决定下一次 `Remember()` 记到哪里：

```text
CurrentResourceOwner = portal->resowner
  -> 后续 buffer pin / snapshot / lock 记到 Portal owner

CurrentResourceOwner = s->curTransactionOwner
  -> 后续资源记到当前事务层级
```

真正释放发生在：

```text
ResourceOwnerRelease()
LockReleaseCurrentOwner()
LockReassignCurrentOwner()
各资源类型的 ReleaseResource callback
```

### 层次二：ResourceOwner tree 表示 cleanup 传播，不等同于事务可见性

ResourceOwner tree 和事务状态栈相关，但不是 MVCC 可见性的本体：

```text
xact.c:
  事务状态、XID/SubXID、commit/abort 流程

ResourceOwner:
  外部资源 ownership 账本

snapmgr.c:
  ActiveSnapshot stack、registered snapshot refcount、xmin 影响

lock.c:
  lock manager shared state、LOCALLOCK owner/count
```

比如 `AtSubCommit_Snapshot()` 不是简单靠 ResourceOwner tree 完成，它会把 active snapshot 的 `as_level` 从子事务层级 relabel 到父层级：

```text
active->as_level = level - 1
```

而 `AtSubAbort_Snapshot()` 会弹出子事务层级的 active snapshots，并降低 active count。

这说明 PostgreSQL 的 cleanup 是分层的：

```text
ResourceOwner:
  处理通用外部资源 ownership

模块自己的 AtEOXact / AtEOSubXact:
  处理模块内部状态机、stack、refcount、invalidation、stats 等
```

### 层次三：锁的继承语义由事务语义决定

锁和 buffer pin、temp file 这类资源不同。子事务 commit 时：

```text
buffer pin:
  正常情况下本应已经 released；如果遗留，commit warning / cleanup

lock:
  可能是成功子事务成果仍需的事务级保护
  所以 reassign 给 parent
```

子事务 abort 时：

```text
lock:
  与失败操作绑定的占用应释放
  不能继续让父事务持有
```

这也是为什么 `ResourceOwnerRelease()` 在 `RESOURCE_RELEASE_LOCKS` phase 对 lock 走特殊分支：

```text
if (isTopLevel)
  ProcReleaseLocks(isCommit)
else if (isCommit)
  LockReassignCurrentOwner(...)
else
  LockReleaseCurrentOwner(...)
```

### 层次四：Portal cleanup 需要避免执行不安全代码

Portal 有 cleanup hook，可能会关闭 executor。正常 `PortalDrop()` 可以调用它；但 abort cleanup 中，PostgreSQL 非常谨慎：

```text
AtAbort_Portals():
  可以调用还没执行的 portal cleanup，但会先把危险 Portal 标记 FAILED
  资源本身由 transaction-wide ResourceOwner cleanup 释放

AtCleanup_Portals():
  如果 cleanup hook 还没跑，发 WARNING 并跳过
  因为 post-abort cleanup 不应再调用用户定义代码或复杂 executor 路径
```

这条边界很重要：

```text
ERROR 恢复路径优先保证 backend 回到一致状态；
不追求把所有正常 shutdown hook 重新执行一遍。
```

## 8. 错误路径 / 异常路径 / fallback

### Portal 执行中的 ERROR：恢复当前指针

`PortalStart()`、`PortalRun()`、`PortalRunFetch()` 都使用 `PG_TRY` / `PG_CATCH`：

```text
进入:
  save ActivePortal / CurrentResourceOwner / PortalContext
  CurrentResourceOwner = portal->resowner

ERROR:
  MarkPortalFailed(portal)
  restore ActivePortal / CurrentResourceOwner / PortalContext
  PG_RE_THROW()
```

这和 ResourceOwner cleanup 是两层机制：

```text
PG_CATCH:
  恢复全局执行状态，避免 longjmp 后 CurrentResourceOwner 悬在错误 Portal 上

ResourceOwnerRelease:
  后续事务/Portal cleanup 释放已经记到 owner 的资源
```

不要把它们混成一个机制。`PG_CATCH` 不负责逐个释放 buffer pin；它负责让控制流回到能执行 abort cleanup 的一致状态。

### 子事务 abort 早期失败：owner 可能还不存在

`PushTransaction()` 注释说明：一旦把 subtransaction state 压栈，abort/cleanup 就必须能处理“还没有 transaction context、resource owner 或 XID”的半初始化状态。

所以 `AbortSubTransaction()` 里有判断：

```text
if (s->curTransactionOwner)
{
  ResourceOwnerRelease(...)
  ...
}
```

这类代码保护的是启动子事务过程中的失败窗口：

```text
子事务状态已入栈
  -> 后续创建内存 context 或 ResourceOwner 前 ERROR
  -> abort path 仍必须可重入、可清理
```

### Portal abort：置空 resowner 防止二次释放

`AtAbort_Portals()` 和 `AtSubAbort_Portals()` 都会把某些 portal 的 `resowner` 置为 `NULL`：

```text
portal->resowner = NULL
```

原因不是资源不需要释放，而是释放责任转移到了事务范围 cleanup：

```text
Any resources belonging to the portal will be released
in the upcoming transaction-wide cleanup.
```

如果不置空，后续 `PortalDrop()` 可能再次尝试释放同一个 owner，造成重复释放或 assert。

### FAILED Portal 的特殊处理

`PortalDrop()` 里有一段特殊逻辑：

```text
Top transaction commit:
  通常让事务结束的 ResourceOwner release 处理资源
  但 FAILED portal 要主动清理，避免 resource leak warning

Ordinary portal drop:
  release portal resources
  如果 portal 不是 FAILED，则不要释放它的 locks
  locks 会成为父 transaction ResourceOwner 的责任
```

这解释了为什么 Portal owner 不总是“一 drop 就释放所有锁”。Portal owner 是事务 owner 的 child；正常关闭一个非 FAILED Portal 时，有些锁仍应延续到事务结束。

## 9. 成本、资源与跨模块传播

### CurrentResourceOwner 切换的成本很低，但边界要求很高

Portal 执行时切换的是几个 backend-local 指针：

```text
ActivePortal
CurrentResourceOwner
PortalContext
CurrentMemoryContext
```

这很便宜，适合包住 executor 这种热路径外层。但它要求所有入口都严格成对保存/恢复。一个漏恢复会导致后续资源记到错误 owner，最终表现可能很晚才出现：

```text
commit warning
unexpected lock retention
portal close 后资源仍被事务持有
abort cleanup 中 assertion failure
```

### 子事务数量会放大 owner tree 和 lock owner 成本

大量 SAVEPOINT / PL exception block 会创建大量 subtransaction owner。每个子事务 commit / abort 都要处理：

```text
ResourceOwnerRelease child owner
AtSubCommit_Portals / AtSubAbort_Portals 扫 Portal hash table
lock reassign 或 release
snapshot active stack relabel 或 cleanup
模块 AtEOSubXact cleanup
```

如果 workload 大量使用短子事务，并且子事务内持有很多 locks，成本可能出现在：

```text
LockReassignCurrentOwner():
  如果 ResourceOwner 的 lock 列表未 overflow，可按 owner->locks 快速处理
  如果 overflow，就需要扫 LockMethodLocalHash

AtSubCommit_Portals():
  扫 PortalHashTable，更新 createSubid / activeSubid / owner parent
```

这不是说不要用 SAVEPOINT，而是诊断时要知道：

```text
子事务不是免费控制流；
它会驱动 owner、snapshot、portal、lock、catcache 等多个模块的传播和 cleanup。
```

### 跨模块传播不是统一抽象完全接管

同一个子事务结束事件会分发到多个模块：

```text
AtSubCommit_Portals()
AtEOSubXact_LargeObject()
AtSubCommit_Notify()
AtEOSubXact_RelationCache()
AtEOSubXact_Inval()
AtSubCommit_smgr()
AtEOSubXact_Files()
AtSubCommit_Snapshot()
```

ResourceOwner 是核心 cleanup 账本，但不是唯一状态机。很多模块还有自己的 subtransaction state，需要在同一个事务边界上同步推进。

这是一条可迁移规律：

```text
通用 owner 适合管理“外部资源是否需要释放”；
模块自己的 transaction hook 负责管理“这个模块的语义状态如何提交或回滚”。
```

## 10. 观测与诊断入口

### SQL 层现象：子事务 abort 释放锁

可以用两个会话观察子事务 abort 后锁占用变化。

会话 A：

```sql
BEGIN;
CREATE TABLE ro_demo(id int);
SAVEPOINT s;
LOCK TABLE ro_demo IN ACCESS EXCLUSIVE MODE;
ROLLBACK TO s;
SELECT pg_backend_pid();
```

会话 B：

```sql
SELECT locktype, mode, granted, relation::regclass
FROM pg_locks
WHERE relation = 'ro_demo'::regclass
ORDER BY mode;
```

预期思路：

```text
ROLLBACK TO s 释放子事务内获取的锁；
但外层事务本身因 CREATE TABLE 等操作仍可能持有别的锁。
```

这个实验要小心解释：`pg_locks` 看到的是当前 backend 仍持有的所有 lock，不会直接显示 ResourceOwner 名称。你需要结合操作位置判断哪个锁来自子事务，哪个锁来自外层事务。

### SQL 层现象：子事务 commit 后锁延续到父事务

会话 A：

```sql
BEGIN;
CREATE TABLE ro_demo_commit(id int);
SAVEPOINT s;
LOCK TABLE ro_demo_commit IN ACCESS EXCLUSIVE MODE;
RELEASE SAVEPOINT s;
SELECT pg_backend_pid();
```

会话 B：

```sql
SELECT locktype, mode, granted, relation::regclass
FROM pg_locks
WHERE relation = 'ro_demo_commit'::regclass
ORDER BY mode;
```

预期：

```text
RELEASE SAVEPOINT 后，子事务 owner 被删除；
但 lock 仍由父事务持有，直到 COMMIT / ROLLBACK。
```

源码对应：

```text
CommitSubTransaction()
  -> ResourceOwnerRelease(child, RESOURCE_RELEASE_LOCKS, true, false)
  -> LockReassignCurrentOwner()
```

### gdb 断点：看 CurrentResourceOwner 切换

适合设置的断点：

```gdb
break AtStart_ResourceOwner
break AtSubStart_ResourceOwner
break CommitSubTransaction
break AbortSubTransaction
break PortalStart
break PortalRun
break PortalRunFetch
break ResourceOwnerRelease
break LockReassignCurrentOwner
break LockReleaseCurrentOwner
```

观察点：

```gdb
print CurrentResourceOwner
print CurTransactionResourceOwner
print TopTransactionResourceOwner
print portal->resowner
print owner->name
print phase
print isCommit
print isTopLevel
```

特别建议在 `ResourceOwnerRelease()` 下断时区分：

```text
top transaction commit:
  isCommit = true
  isTopLevel = true

subtransaction commit:
  isCommit = true
  isTopLevel = false

subtransaction abort:
  isCommit = false
  isTopLevel = false
```

只有第二种会走 `LockReassignCurrentOwner()`。

### 日志与 WARNING

可以关注这些现象：

```text
commit 时资源未正常释放:
  ResourceOwnerRelease 可能打印 WARNING

portal cleanup 被跳过:
  AtCleanup_Portals 可能 WARNING "skipping cleanup for portal ..."

锁等待 / deadlock:
  lock manager 日志和 pg_locks 可辅助判断锁是否在子事务结束后仍保留
```

但要记住：

```text
pg_locks 不显示 ResourceOwner；
ResourceOwner tree 主要是 backend-local 内部状态；
很多 owner 传播只能通过 gdb、源码断点或间接行为推断。
```

## 11. 常见误区

### 误区一：CurrentResourceOwner 总是 CurTransactionResourceOwner

不对。Portal 执行期间：

```text
CurTransactionResourceOwner:
  当前事务层级 owner

CurrentResourceOwner:
  portal->resowner
```

这正是 Portal 能局部管理执行资源的原因。

### 误区二：子事务 commit 应该 release 子 owner 的所有资源

要分资源类型。很多资源在正常路径本应已经 `Forget()`；遗留资源可能产生 warning 或 cleanup。但 lock 是特殊的：

```text
子事务 commit:
  lock ownership reassign to parent

子事务 abort:
  lock ownership release
```

### 误区三：PortalDrop 总是释放 Portal 持有的所有锁

普通非 FAILED Portal drop 时，`PortalDrop()` 注释明确说：如果 portal 没失败，不释放它的 locks，locks 会成为事务 ResourceOwner 的责任。

这是因为锁的生命周期常常是事务级，而不是 Portal 级。

### 误区四：ResourceOwner tree 能替代所有 AtEOXact / AtEOSubXact hooks

不对。ResourceOwner 处理通用资源 release；模块自己的 hook 处理模块语义状态。

例如 snapshot：

```text
RegisterSnapshotOnOwner():
  registered snapshot ref 记到 ResourceOwner

AtSubCommit_Snapshot():
  active snapshot stack 层级 relabel

AtSubAbort_Snapshot():
  active snapshot stack 弹出并 decrement active_count
```

这些状态不能只靠 ResourceOwner generic callback 表达。

### 误区五：PG_CATCH 已经释放了资源

通常不对。Portal 的 `PG_CATCH` 主要做：

```text
MarkPortalFailed()
恢复 ActivePortal / CurrentResourceOwner / PortalContext
重新抛出 ERROR
```

资源释放发生在后续 abort / cleanup 的 `ResourceOwnerRelease()` 和各模块 cleanup hook 中。

## 12. 课堂实验

### 实验一：跟踪子事务 commit 的锁转移

目标：确认 `RELEASE SAVEPOINT` 后，子事务 owner 删除，但 lock 仍由父事务持有。

步骤：

```text
1. 编译 debug 版本 PostgreSQL。
2. gdb attach 到 backend。
3. 设置断点:
   break CommitSubTransaction
   break ResourceOwnerRelease
   break LockReassignCurrentOwner
4. SQL 执行:
   BEGIN;
   CREATE TABLE ro_subcommit(id int);
   SAVEPOINT s;
   LOCK TABLE ro_subcommit IN ACCESS EXCLUSIVE MODE;
   RELEASE SAVEPOINT s;
5. 在 LockReassignCurrentOwner 处观察 CurrentResourceOwner 和 parent。
```

预期解释：

```text
LockReassignCurrentOwner() 会找到 CurrentResourceOwner 持有的 LOCALLOCKOWNER slot；
如果 parent 没有 slot，就把 child slot owner 改成 parent；
如果 parent 已有 slot，就合并 nLocks；
最后 ResourceOwnerForgetLock(child, locallock)。
```

### 实验二：跟踪子事务 abort 的锁释放

目标：确认 `ROLLBACK TO SAVEPOINT` 后走 `LockReleaseCurrentOwner()`。

步骤：

```text
1. 设置断点:
   break AbortSubTransaction
   break ResourceOwnerRelease
   break LockReleaseCurrentOwner
2. SQL 执行:
   BEGIN;
   CREATE TABLE ro_subabort(id int);
   SAVEPOINT s;
   LOCK TABLE ro_subabort IN ACCESS EXCLUSIVE MODE;
   ROLLBACK TO s;
3. 在 ResourceOwnerRelease 中观察:
   phase == RESOURCE_RELEASE_LOCKS
   isCommit == false
   isTopLevel == false
```

预期解释：

```text
子事务失败，child owner 下的 lock 不会继承到 parent；
LockReleaseCurrentOwner() 会释放属于 CurrentResourceOwner 的锁记录。
```

### 实验三：观察 Portal 执行期间 owner 切换

目标：确认 Portal 执行期间 deep code 默认记到 `portal->resowner`。

步骤：

```text
1. 设置断点:
   break PortalStart
   break PortalRun
   break PortalRunFetch
   break ResourceOwnerRemember
2. SQL 执行:
   BEGIN;
   DECLARE c CURSOR FOR SELECT * FROM pg_class;
   FETCH 1 FROM c;
3. 在 PortalRunFetch 中观察:
   CurrentResourceOwner
   portal->resowner
4. 在 ResourceOwnerRemember 中观察:
   owner 参数是否等于 portal->resowner
```

预期解释：

```text
PortalRunFetch 进入时保存旧 CurrentResourceOwner；
执行期间 CurrentResourceOwner 切到 portal->resowner；
退出或 ERROR 时恢复。
```

### 实验四：源码阅读练习

沿下面调用链做一次手工标注：

```text
CreatePortal()
  -> portal->resowner = ResourceOwnerCreate(CurTransactionResourceOwner, "Portal")

PortalRunFetch()
  -> CurrentResourceOwner = portal->resowner

RegisterSnapshot()
  -> RegisterSnapshotOnOwner(snapshot, CurrentResourceOwner)
  -> ResourceOwnerRememberSnapshot(owner, snap)

PortalDrop()
  -> UnregisterSnapshotFromOwner(portal->holdSnapshot, portal->resowner)
  -> ResourceOwnerRelease(portal->resowner, ...)
```

标注每一步回答两个问题：

```text
当前资源记到哪个 owner？
如果这里 ERROR，谁负责恢复 CurrentResourceOwner，谁负责释放已登记资源？
```

## 13. 讨论题

1. 为什么 Portal owner 挂到 `CurTransactionResourceOwner`，而不是挂到进入 `CreatePortal()` 时的 `CurrentResourceOwner`？
2. `PortalRun()` 为什么不能简单保存旧 `CurrentResourceOwner` 并无条件恢复？utility command 内部事务切换带来了什么指针有效性问题？
3. 子事务 commit 时，哪些资源应该向父事务传播，哪些资源应该在正常路径中已经释放？lock 为什么是特殊案例？
4. 如果子事务 abort 时把 lock 转移给父 owner，会造成哪些可观察问题？哪些是性能问题，哪些是 correctness 问题？
5. 为什么 `AtSubAbort_Portals()` 不直接删除 Portal，而要等 `AtSubCleanup_Portals()`？
6. ResourceOwner tree 和 MemoryContext tree 都有 parent/child，为什么 Portal cleanup 中还要同时恢复 `PortalContext` 和 `CurrentResourceOwner`？
7. `pg_locks` 看不到 ResourceOwner 名称。实际诊断时如何把 SQL 现象、gdb owner 指针和源码调用链对应起来？

## 14. 本节小结

本节的可迁移模型是：

```text
隐式 current owner 能让 deep code 保持简单；
但每个控制流边界都必须精确保存、切换和恢复 current owner；
子事务结束不是统一“释放 child owner”，而是根据 commit / abort 语义决定资源是否向父生命周期传播。
```

在 PostgreSQL 中，这个模型具体表现为：

```text
xact.c:
  创建 top/subtransaction owner，决定事务边界上的 release/delete

portalmem.c:
  创建 Portal owner，处理 Portal 在事务和子事务边界上的 reparent / cleanup

pquery.c:
  在 Portal 执行期间临时切换 CurrentResourceOwner

resowner.c + lock.c:
  在 subtransaction commit 时 reassign locks to parent
  在 subtransaction abort 时 release locks from current owner
```

把本节压缩成一句话：

```text
ResourceOwner 的难点不只是“记住资源”，而是让资源跟着 PostgreSQL 的真实执行生命周期移动；成功的子事务把责任交给父 owner，失败的子事务把责任就地清掉。
```

下一节会继续沿这条线进入 `ResourceOwnerRelease()` 的三阶段顺序：为什么 buffer pin、relcache ref、invalidation、locks、snapshot、文件和 backend-local cleanup 不能随便换顺序。
