# PostgreSQL active stack 与 registered snapshot 生命周期

## 课程定位

前置知识：已经理解 `SnapshotData` 的 `xmin`、`xmax`、`xip`、`subxip`、`curcid`，也已经理解 `PGPROC->xmin` 会把本 backend 的可见性需求暴露给 ProcArray。

本节唯一主问题：

```text
为什么 PostgreSQL 的 snapshot 生命周期必须同时有 active stack 和 registered heap，而不能只依赖 MemoryContext 在事务结束时统一释放？
```

本节围绕的核心矛盾：

```text
executor 需要一个随着调用栈进入和退出而变化的“当前可见性上下文”；
portal、cursor、executor state、catalog snapshot、exported snapshot 又需要一个能跨调用栈、跨 ERROR cleanup、跨资源 owner 边界存活的“长期持有引用”。
如果只用 MemoryContext 管内存，系统知道什么时候能释放内存，却不知道这个 snapshot 是否仍在语义上保护 tuple、TOAST、catalog 或 cursor 的可见性。
```

一句话运行模型：

```text
active snapshot 表示当前调用栈正在用哪个 snapshot 做可见性判断；
registered snapshot 表示某个 ResourceOwner 或 snapshot manager 内部对象仍持有这个 snapshot；
`active_count + regd_count + RegisteredSnapshots(min xmin) + SnapshotResetXmin()` 共同决定 snapshot 何时能释放，以及 `MyProc->xmin` 何时能前移。
```

学完后应能独立判断：

- 为什么 `GetTransactionSnapshot()` 返回的静态 snapshot 不能随手长期保存。
- 为什么 `PushActiveSnapshot()` 会在必要时复制 snapshot，而不是只保存指针。
- 为什么 executor 启动前要求 `GetActiveSnapshot() == queryDesc->snapshot`。
- 为什么 cursor 每次 fetch 时会重新 push active snapshot。
- 为什么 registered snapshot 需要进入 `ResourceOwner`，而 active snapshot 不一定进入 `ResourceOwner`。
- 为什么 subtransaction abort 要按 nesting level 清理 active stack。
- 为什么 `SnapshotResetXmin()` 只在 active stack 为空时根据 registered heap 推进 `MyProc->xmin`。
- 为什么 portal/cursor 的 snapshot 边界会表现成 VACUUM 无法回收旧版本。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

本节不展开完整 `GetSnapshotData()` 的 ProcArray 扫描，也不展开 VACUUM horizon 的 relation-aware 计算。这些内容分别属于前面的 snapshot 字段课和下一节 cleanup horizon 课。

本节只关心 snapshot 对象在一个 backend 内部被谁持有、谁正在使用、谁负责兜底释放，以及这些本地生命周期如何影响 `MyProc->xmin`。

## 1. 本节在总主线中的位置

前几节已经回答了 snapshot 里保存什么。`xmin` 和 `xmax` 给出范围。

`xip` 和 `subxip` 给出运行事务集合。`curcid` 让同一事务内的不同 command 有不同可见性。

这些字段解释“一个 tuple 对这个 snapshot 是否可见”。但是它们还没有解释“这个 snapshot 自己什么时候还算活着”。

这是本节的入口。一个查询拿到 snapshot 后，代码里会出现几类看似类似的行为。

第一类是当前 executor 正在扫描 heap。这时 heap AM、index AM、expression evaluation、TOAST detoast 和 trigger 函数都需要能问到当前 active snapshot。

第二类是 cursor 已经启动 executor，但当前调用栈并不在 executor 里。这时 snapshot 不能因为某次 `FETCH` 返回给客户端就消失。

下一次 `FETCH` 还必须沿用同一个可见性世界。第三类是 transaction snapshot、catalog snapshot、exported snapshot 或 logical decoding historic snapshot。

它们不是普通 executor 当前调用栈的一帧，却仍可能长期阻止 `MyProc->xmin` 前移。第四类是 ERROR、subtransaction abort、procedure 内部 COMMIT/ROLLBACK、portal drop。

这些路径经常不是原始 push/register 的函数按原样返回。如果只用 `MemoryContext`，最多能保证一批内存在 context reset 时回收。

它回答不了：

- 当前调用栈顶部应该使用哪个 snapshot。
- 某个 snapshot 是否还被 cursor 的 executor state 持有。
- 某个 snapshot 是否还被 `ResourceOwner` 保护。
- subtransaction abort 应该清掉哪些 active snapshot。
- 没有 active snapshot 时，`MyProc->xmin` 是否可以根据 registered snapshot 的最小 `xmin` 前移。

所以本节把 snapshot 看成一个有双重生命周期的本地对象。内存生命周期回答“这块内存何时释放”。

可见性生命周期回答“这个 snapshot 何时仍然保护某些 tuple 不能被视为对所有人死亡”。PostgreSQL 把这两个问题拆开。

这就是 active stack 和 registered heap 共存的根本原因。本节贯穿一个 runtime 现场：

```text
Session A 声明一个普通 cursor，执行少量 FETCH，然后停在事务中。
Session B 删除大量旧版本并 VACUUM。
Session B 发现 dead tuple 不能完全回收。
回到源码后，原因不是 cursor 内存没有释放，而是 executor snapshot 仍 registered，`MyProc->xmin` 仍被这个 snapshot 的 xmin 固定。
```

这个现场会一直贯穿到 `PortalStart()`、`ExecutorStart()`、`PortalRunSelect()`、`ExecutorEnd()`、`UnregisterSnapshot()` 和 `SnapshotResetXmin()`。

## 2. 核心矛盾与一句话运行模型

本节唯一主问题已经给出。为了让它变成可运行的模型，先看 `snapmgr.c` 文件顶部注释给出的边界。

`GetTransactionSnapshot()`、`GetLatestSnapshot()`、`GetCatalogSnapshot()`、`GetNonHistoricCatalogSnapshot()` 返回的是静态分配的 snapshot。这些静态 snapshot 会在后续 snapshot 相关调用或 command counter 更新时被改写。

所以调用者不能把返回指针当成长生命周期对象。调用者必须做两类事之一。

如果马上进入一段需要“当前 snapshot”的调用栈，就 `PushActiveSnapshot()`。如果要让某个资源对象长期持有，就 `RegisterSnapshot()` 或 `RegisterSnapshotOnOwner()`。

这两类动作改变的是不同语义。`PushActiveSnapshot()` 让 `GetActiveSnapshot()` 返回它。

它服务的是“当前调用栈”。`RegisterSnapshotOnOwner()` 增加 `regd_count`，把 snapshot 交给 `ResourceOwner`，并把第一次 registered 的 snapshot 挂进按 `xmin` 排序的 `RegisteredSnapshots` pairing heap。

它服务的是“长期持有者”。这两类引用最终落在同一个 `SnapshotData` 上。

`snapshot.h` 里有三个 bookkeeping 字段：

```text
active_count
regd_count
ph_node
```

`active_count` 只说明 active stack 上有多少帧引用这个 snapshot。`regd_count` 只说明 registered snapshots 中有多少长期引用。

`ph_node` 只在 `regd_count` 从 0 变成 1 时进入 `RegisteredSnapshots` heap。这三个字段都不是 MVCC 语义字段。

它们是 snapshot manager 为了安全释放和 horizon 推进维护的本地 bookkeeping。关键不变量是：

```text
active_count == 0 && regd_count == 0 && copied == true
  -> FreeSnapshot() 可以释放这份 snapshot 内存。

ActiveSnapshot == NULL && RegisteredSnapshots empty
  -> MyProc->xmin 可以重置为 InvalidTransactionId。

ActiveSnapshot == NULL && RegisteredSnapshots not empty
  -> MyProc->xmin 可以推进到 registered heap 顶部 snapshot 的 xmin。
```

注意第二条和第三条只在 active stack 为空时发生。`SnapshotResetXmin()` 不会在还有 active snapshot 时扫描 active stack 找最老 `xmin`。

源码注释说这是为了简单。active stack 更像调用栈，可能频繁 push/pop。

registered heap 则维护了按 `xmin` 排序的数据结构，适合快速找到最老长期引用。这就是矛盾的具体形状：

```text
active stack 追求调用栈语义清晰、push/pop 便宜；
registered heap 追求长期 holder 可被 ResourceOwner 兜底释放，并能按 xmin 找到最老引用。
```

把它们合并成一个列表会让其中一个目标变差。只靠 `MemoryContext` 则两个目标都回答不完整。

## 3. 核心文件分工与阅读顺序

本节按生命周期读源码，不按文件名排序。

第一步读 `src/include/utils/snapshot.h`。它定义 `SnapshotData`。

本节只关注 bookkeeping 字段：

- `copied`
- `active_count`
- `regd_count`
- `ph_node`
- `snapXactCompletionCount`

这些字段解释 snapshot 对象能否被长期保存、能否复用、能否进入 registered heap。

第二步读 `src/backend/utils/time/snapmgr.c` 顶部注释。这段注释直接说明 PostgreSQL 为什么有两种 tracking。

它还说明了几个例外：VACUUM 会自己管理事务并可能 pop 调用者的 active snapshot；procedure 或 DO block 内部 COMMIT/ROLLBACK 会摧毁 active snapshot stack，然后由 `EnsurePortalSnapshotExists()` 重建。

第三步读 `CopySnapshot()`、`FreeSnapshot()`。它们回答内存从哪里来。

`CopySnapshot()` 在 `TopTransactionContext` 中分配一块连续内存，复制 `xip` 和必要的 `subxip`，并把 refcount 清零。`FreeSnapshot()` 要求 `regd_count == 0`、`active_count == 0`、`copied == true`。

第四步读 `PushActiveSnapshot()`、`PushActiveSnapshotWithLevel()`、`PopActiveSnapshot()`。它们回答当前调用栈如何持有 snapshot。

`ActiveSnapshotElt.as_level` 会记录 transaction nesting level。这就是 subtransaction abort 能按层级清理的原因。

第五步读 `RegisterSnapshotOnOwner()`、`UnregisterSnapshotFromOwner()`、`UnregisterSnapshotNoOwner()`。它们回答长期引用如何持有 snapshot。

`ResourceOwnerRememberSnapshot()` 是 `snapmgr.c` 内的 static inline wrapper，最终调用 `ResourceOwnerRemember()`。`snapshot_resowner_desc` 把 release phase 设为 `RESOURCE_RELEASE_AFTER_LOCKS`，priority 设为 `RELEASE_PRIO_SNAPSHOT_REFS`。

第六步读 `SnapshotResetXmin()`。它回答 snapshot 生命周期如何影响 `MyProc->xmin`。

它不会重新计算 ProcArray snapshot。它只根据本 backend 内 active/registered 状态决定是否清空或推进本 backend 暴露出去的 `xmin`。

第七步读 `AtSubCommit_Snapshot()`、`AtSubAbort_Snapshot()`、`AtEOXact_Snapshot()`。它们回答 commit、abort、ERROR cleanup 时谁兜底。

第八步读 portal 与 executor。`src/backend/tcop/pquery.c` 的 `PortalStart()`、`PortalRunSelect()`、`PortalRunMulti()`、`PortalRunUtility()`、`EnsurePortalSnapshotExists()` 展示 active snapshot 如何跨 portal 边界进入 executor。

`src/backend/executor/execMain.c` 的 `standard_ExecutorStart()` 和 `ExecutorEnd()` 展示 executor 如何注册并释放自己的 snapshot 引用。`src/backend/utils/mmgr/portalmem.c` 的 `PortalDrop()` 和 portal snapshot cleanup 展示 cursor/portal 如何在异常和事务边界释放持有者。

推荐跟读顺序：

```text
snapmgr.c: GetTransactionSnapshot()
  -> snapmgr.c: PushActiveSnapshot()
  -> pquery.c: PortalStart()
  -> execMain.c: standard_ExecutorStart()
  -> execMain.c: ExecutorEnd()
  -> snapmgr.c: UnregisterSnapshot()
  -> pquery.c: PortalRunSelect()
  -> snapmgr.c: PopActiveSnapshot()
  -> snapmgr.c: SnapshotResetXmin()
  -> snapmgr.c: AtSubAbort_Snapshot() / AtEOXact_Snapshot()
```

这条线里最重要的不是函数名。最重要的是同一个 snapshot 在不同时间点处于不同持有状态。

它可能 active 但未 registered。它可能 registered 但当前不 active。

它也可能同时 active 和 registered。它只有两种 refcount 都归零时才可以释放。

## 4. 关键数据结构与状态边界

`SnapshotData` 是 backend-local 对象。其他 backend 不会拿到这个 C 指针。

其他 backend 只能通过 `MyProc->xmin` 看到当前 backend 声称仍需要保留的可见性下界。这解释了一个重要边界：

```text
snapshot 对象本身是本地状态；
snapshot 的影响通过 PGPROC->xmin 变成共享可见的 horizon 输入。
```

`active_count` 表示这个 snapshot 当前被 active stack 的多少个节点引用。同一个 snapshot 可以被 nested executor 或递归调用重复 push。

`PopActiveSnapshot()` 只减少顶部节点对应 snapshot 的 `active_count`。如果 `regd_count` 仍大于 0，它不能释放 snapshot。

`regd_count` 表示这个 snapshot 当前被 registered 持有多少次。registered reference 可以来自普通 `ResourceOwner`。

也可以来自 `FirstXactSnapshot`、exported snapshot、catalog snapshot、historic snapshot 等 snapshot manager 自己管理的引用。这些特殊引用不一定由普通 ResourceOwner 追踪。

`RegisteredSnapshots` 是 `snapmgr.c` 中的 pairing heap。heap 比较函数是 `xmin_cmp()`。

它把 `xmin` 最小的 registered snapshot 放在顶部。这不是为了查询可见性。

这是为了 `SnapshotResetXmin()` 在 active stack 为空时快速找到最老仍需保留的 `xmin`。`ActiveSnapshot` 是 `snapmgr.c` 中的 backend-local stack top。

每个节点是 `ActiveSnapshotElt`。它包含：

- `as_snap`
- `as_level`
- `as_next`

`as_level` 是 transaction nesting level。它不是 executor nesting depth。

它服务 `AtSubCommit_Snapshot()` 和 `AtSubAbort_Snapshot()`。如果一个子事务中 push 了 active snapshot，子事务 abort 时必须把这些 active frame 清理掉。

否则外层事务后续调用 `GetActiveSnapshot()` 可能看到已经属于失败子事务的 snapshot。`CurrentResourceOwner` 是 registered snapshot 的默认 owner。

`RegisterSnapshot()` 只是调用 `RegisterSnapshotOnOwner(snapshot, CurrentResourceOwner)`。portal 有自己的 `resowner`。

因此 portal 可以把 snapshot 绑到 portal 的资源所有者上，而不是当前命令的短生命周期 owner。这对 cursor 很关键。

`PortalData` 中有两个相关字段。`portalSnapshot` 表示 portal 执行期间的 outermost active snapshot。

`holdSnapshot` 表示为了 holdStore 中的 tuple 或 TOAST 引用而保存的 registered snapshot。这两个字段名字相似，但语义不同。

`portalSnapshot` 保护的是当前执行调用栈。`holdSnapshot` 保护的是 materialized result 可能仍引用的可见性世界。

`QueryDesc->snapshot` 是 executor 的 snapshot。`standard_ExecutorStart()` 要求调用者已经把它 push 成 active。

随后 executor 会把它注册进 `EState`。`ExecutorEnd()` 释放 `estate->es_snapshot` 和 `estate->es_crosscheck_snapshot`。

这解释了为什么一个普通 cursor 在两次 `FETCH` 之间没有 active snapshot，但仍然持有 registered snapshot。

## 5. 主流程源码 walkthrough

本节主流程从一个普通 cursor 开始。它不是 holdable cursor。

它也不是 exported snapshot。它只是最容易看清 active 与 registered 差异的路径。

场景如下：

```sql
-- Session A
CREATE TABLE snap_life_demo(id int primary key, payload text);
INSERT INTO snap_life_demo
SELECT g, repeat('x', 200)
FROM generate_series(1, 10000) AS g;

BEGIN;
DECLARE c CURSOR FOR SELECT * FROM snap_life_demo ORDER BY id;
FETCH 1 FROM c;
-- 保持事务打开，不关闭 cursor。
```

Session A 执行 `DECLARE CURSOR` 时，portal 被创建。`PortalStart()` 进入 `PORTAL_ONE_SELECT` 分支。

它先设置 active snapshot。如果调用者传入 snapshot，就 `PushActiveSnapshot(snapshot)`。

否则就 `PushActiveSnapshot(GetTransactionSnapshot())`。这里有第一个关键点。

`GetTransactionSnapshot()` 返回的 snapshot 可能是静态对象。`PushActiveSnapshot()` 会进入 `PushActiveSnapshotWithLevel()`。

如果 snapshot 是 `CurrentSnapshot`、`SecondarySnapshot` 或 `copied == false`，它会调用 `CopySnapshot()`。也就是说，active stack 上放的通常不是会被后续调用改写的静态对象。

它是一份可长期存在到 pop 的拷贝。`PortalStart()` 随后调用 `CreateQueryDesc()`。

传入的 snapshot 是 `GetActiveSnapshot()`。`CreateQueryDesc()` 保存这个 snapshot 到 `QueryDesc`。

然后 `ExecutorStart(queryDesc, eflags)` 被调用。`standard_ExecutorStart()` 一开始就断言：

```text
GetActiveSnapshot() == queryDesc->snapshot
```

这个断言是 active stack 的语义边界。executor 启动时，当前调用栈必须已经有正确 snapshot。

否则 executor 中的初始化、表达式、权限、RLS、index expression 或其他需要 snapshot 的路径可能读到错误可见性世界。`standard_ExecutorStart()` 创建 `EState`。

在 executor 初始化过程中，`estate->es_snapshot` 会持有 registered 引用。后续 `ExecutorEnd()` 会调用：

```text
UnregisterSnapshot(estate->es_snapshot)
UnregisterSnapshot(estate->es_crosscheck_snapshot)
```

这说明 executor snapshot 不只是 active frame。它还会被 executor state 注册持有。

`PortalStart()` 启动 executor 后调用 `PopActiveSnapshot()`。这时 `DECLARE CURSOR` 的启动调用栈结束。

active stack 不再需要这个 snapshot 作为当前 snapshot。但 cursor 没有关闭。

`portal->queryDesc` 仍在。executor state 仍在。

registered snapshot 仍在。所以 snapshot 还不能释放。

这就是 active 和 registered 的第一次分离：

```text
PortalStart() 返回后：
  active_count 可能已经归零
  regd_count 仍然大于零
  QueryDesc/EState 仍持有 snapshot
  MyProc->xmin 仍可能被该 snapshot 的 xmin 固定
```

接下来 Session A 执行 `FETCH 1 FROM c`。`PortalRun()` 进入 `PortalRunSelect()`。

如果 portal 没有 `holdStore`，它会：

```text
PushActiveSnapshot(queryDesc->snapshot)
ExecutorRun(queryDesc, direction, count)
PopActiveSnapshot()
```

这就是第二个关键点。同一个 registered snapshot 在 `FETCH` 的 executor 调用栈中临时变成 active snapshot。

`ExecutorRun()` 期间 heap scan、index scan、TOAST、expression evaluation 都可以通过 `GetActiveSnapshot()` 找到它。`FETCH` 返回后 active frame 被 pop。

但是 registered 引用仍在。下一次 `FETCH` 会再次 push 同一个 `queryDesc->snapshot`。

因此 cursor 的 snapshot 生命周期不是一个普通函数调用。它是：

```text
DECLARE / PortalStart:
  active: push for executor startup
  registered: executor state remembers snapshot
  active: pop after startup

FETCH:
  active: push registered snapshot for this executor run
  executor reads tuples under that snapshot
  active: pop

CLOSE / transaction end:
  ExecutorEnd unregisters snapshot
  snapshot refcount reaches zero
  SnapshotResetXmin may advance MyProc->xmin
```

如果只靠 MemoryContext，系统很难表达这个状态。`TopTransactionContext` 可以让 snapshot 内存活到事务结束。

但 cursor 可能在事务中提前关闭。一旦 `CLOSE c`，executor snapshot 应该释放，`MyProc->xmin` 也可能前移。

等到事务结束才统一 reset 内存，会把 cleanup horizon 固定得太久。反过来，如果只看调用栈，`PortalStart()` pop active snapshot 后 snapshot 会被误认为无人使用。

下一次 `FETCH` 就会拿到悬空指针或错误 snapshot。registered 引用正是为这个间隙存在。

## 6. `PushActiveSnapshot()` 不是简单赋全局变量

早期可以把 active snapshot 想象成一个全局变量。今天源码已经不是这样。

`snapmgr.c` 顶部注释说，active snapshot stack mirrors the process call stack。它必须支持递归、portal 切换、SPI 调用、函数内部执行 SQL、utility command 等场景。

`PushActiveSnapshotWithLevel()` 做几件事。第一，它在 `TopTransactionContext` 中分配 `ActiveSnapshotElt`。

active frame 不是调用栈上的 C 局部变量。这让 ERROR cleanup 可以在非局部跳转后仍找到并清理它。

第二，它必要时复制 snapshot。如果传入的是静态 snapshot，或者可能受 command counter update 影响，必须 `CopySnapshot()`。

这避免 active frame 指向一个后续会被覆盖的静态对象。第三，它记录 `as_level`。

默认 level 来自 `GetCurrentTransactionNestLevel()`。某些 portal 路径会显式传入 `portal->createLevel`。

这避免 portal 中保存的 snapshot 在 subtransaction 边界变成 dangling pointer。第四，它增加 `active_count`。

这和 stack node 分离。同一个 snapshot 可以同时在多个 active frame 中出现。

只要 `active_count > 0`，即使 `regd_count == 0`，snapshot 也不能释放。`PopActiveSnapshot()` 反向执行。

它取出 top frame。减少 `active_count`。

如果 `active_count == 0 && regd_count == 0`，调用 `FreeSnapshot()`。释放 stack node。

把 `ActiveSnapshot` 指向下一个节点。最后调用 `SnapshotResetXmin()`。

注意调用顺序。先 pop active frame，再尝试 reset xmin。

因为 active frame 还在时，`SnapshotResetXmin()` 会直接 return。这一点让 `PopActiveSnapshot()` 成为 `MyProc->xmin` 可能前移的关键节点之一。

但它不是唯一节点。`UnregisterSnapshotNoOwner()` 在释放最后一个 registered 引用时也会调用 `SnapshotResetXmin()`。

## 7. registered snapshot 为什么必须接 ResourceOwner

`RegisterSnapshotOnOwner()` 是长期持有入口。它先判断 invalid snapshot。

然后决定是否复制。如果传入的是静态 snapshot，就 `CopySnapshot()`。

如果传入已经 copied 的 snapshot，就直接使用。随后它执行三步。

第一，`ResourceOwnerEnlarge(owner)`。这为 owner 记录新资源预留空间，避免后续记账失败时 refcount 已经改了。

第二，`snap->regd_count++`。第三，`ResourceOwnerRememberSnapshot(owner, snap)`。

这个 wrapper 在 `snapmgr.c` 中是 static inline。它调用通用的 `ResourceOwnerRemember(owner, PointerGetDatum(snap), &snapshot_resowner_desc)`。

`snapshot_resowner_desc` 的 release phase 是 `RESOURCE_RELEASE_AFTER_LOCKS`。priority 是 `RELEASE_PRIO_SNAPSHOT_REFS`。

release callback 是 `ResOwnerReleaseSnapshot()`。`ResOwnerReleaseSnapshot()` 调用 `UnregisterSnapshotNoOwner()`。

这说明 ResourceOwner 不负责理解 snapshot 字段。它负责在 owner 释放时回调 snapshot manager。

snapshot manager 再减少 `regd_count`、从 heap 移除、释放内存、重算 xmin。为什么 registered snapshot 不直接靠 MemoryContext？

因为 MemoryContext 不知道“引用关系”。一个 snapshot 可能在同一个 `TopTransactionContext` 中。

它可能被一个 cursor 的 resource owner 持有。也可能被 executor state 持有。

也可能被 snapshot manager 内部持有。如果 owner 提前释放，snapshot 应该提前 unregister。

如果 owner 因 ERROR 走 abort cleanup，snapshot 也必须 unregister。如果只等 context reset，系统会把 `MyProc->xmin` 固定到事务结束。

这会制造不必要的 vacuum lag。ResourceOwner 的三阶段 release 也很重要。

`resowner.h` 说明 release 分成：

- `RESOURCE_RELEASE_BEFORE_LOCKS`
- `RESOURCE_RELEASE_LOCKS`
- `RESOURCE_RELEASE_AFTER_LOCKS`

snapshot reference 属于 after-locks。它不是 buffer pin。

它不是 lock。它是 backend-internal cleanup。

这避免把 snapshot 当成并发互斥资源理解。snapshot 持有的是可见性语义。

lock 持有的是并发访问权限。两者都可能影响其他 backend，但影响路径不同。

## 8. `RegisteredSnapshots` heap 与 `SnapshotResetXmin()`

registered snapshot 不只是 refcount。它还进入 `RegisteredSnapshots` pairing heap。

heap 的比较函数按 `snapshot->xmin` 排序。`regd_count` 从 0 变成 1 时，`RegisterSnapshotOnOwner()` 把 `ph_node` 加入 heap。

`regd_count` 从 1 变成 0 时，`UnregisterSnapshotNoOwner()` 把 `ph_node` 移出 heap。这个 heap 只包含 registered snapshots。

active-only snapshot 不在 heap 中。`SnapshotResetXmin()` 先看 active stack。

如果 `ActiveSnapshot != NULL`，直接 return。这表示只要还有 active frame，PostgreSQL 不尝试推进 `MyProc->xmin`。

因为 active stack 没有按 `xmin` 排序的数据结构。为了简单和便宜，源码选择在 active stack 清空后再处理。

如果 active stack 已空，并且 `RegisteredSnapshots` 也空：

```text
MyProc->xmin = TransactionXmin = InvalidTransactionId
```

这表示当前 backend 不再公开任何 snapshot xmin。其他 backend 后续计算 cleanup horizon 时可以忽略它的 `xmin`。

如果 active stack 已空，但 registered heap 非空：

```text
minSnapshot = pairingheap_first(&RegisteredSnapshots)
if (TransactionIdPrecedes(MyProc->xmin, minSnapshot->xmin))
    MyProc->xmin = TransactionXmin = minSnapshot->xmin
```

这里用的是 “advance”，不是重新设置成任意值。如果 `MyProc->xmin` 已经比 heap 顶部更老，就推进到 heap 顶部。

如果不是，就不后退。这一点和 ProcArray horizon 的保守性一致。

`MyProc->xmin` 一旦公开给其他 backend，随意后退会破坏正在进行的判断。所以课程里不要把 `SnapshotResetXmin()` 理解成“重新计算当前事务 xmin”。

它更像：

```text
在本 backend 的 snapshot holder 变化后，尽可能安全地释放或推进公开 xmin。
```

这也是为什么 registered heap 的顺序有价值。当一个 cursor 关闭时，如果它持有的是最老 snapshot，heap 顶部会改变。

`SnapshotResetXmin()` 可能让 `MyProc->xmin` 前移。VACUUM 下一次计算 horizon 时就能看到更近的下界。

## 9. portal/cursor 边界

portal 是本节最适合观察的对象。它把 snapshot 从单个函数调用拉长为一个交互式生命周期。

`PortalData` 中的 `portalSnapshot` 注释说，它是 portal 查询执行的 outermost ActiveSnapshot。大多数 utility command 之外的 portal 都要求存在这样的 snapshot。

原因之一是 query result 中的 TOAST 引用可能还需要 detoast。原因之二是减少 `MyProc->xmin` 暴露值频繁抖动。

`PortalStart()` 对 `PORTAL_ONE_SELECT` 的处理很清楚。它 push active snapshot。

创建 `QueryDesc`。启动 executor。

保存 `portal->queryDesc`。然后 pop active snapshot。

这时 cursor 已经可 fetch。`PortalRunSelect()` 每次真正取行时再 push `queryDesc->snapshot`。

这让 executor 运行期间拥有正确 active snapshot。fetch 结束后 pop。

所以普通 cursor 的核心状态是：

```text
portal->queryDesc != NULL
queryDesc->snapshot registered by executor state
active snapshot only exists during executor startup or fetch
```

`PortalDrop()` 则是释放边界。它先调用 portal cleanup hook，通常会关 executor。

然后处理 `portal->holdSnapshot`。如果 portal 有自己的 `resowner`，就 `UnregisterSnapshotFromOwner(portal->holdSnapshot, portal->resowner)`。

最后释放 portal 资源。这里的注释承认 cleanup hook 可能运行用户代码并失败。

如果失败，abort cleanup 还会回来再次 drop。所以 hook 必须能处理重复 cleanup。

这就是内核生命周期代码常见的现实形状：

不是每条边都完美线性。必须能承受 ERROR 后重复进入 cleanup。

holdable cursor 需要单独看。`WITH HOLD` cursor 会把结果 materialize 到 tuplestore。

提交后 cursor 可以继续 fetch。普通 executor state 不能跨事务继续存在。

因此持有策略会从“executor snapshot + active fetch”转向“holdStore + detoast/holdSnapshot 边界”。`portal.h` 对 `holdSnapshot` 的注释很谨慎。

如果 holdStore 中的 tuple 可能包含 TOAST 引用，就必须保持一个 registered snapshot，防止相关 TOAST rows 被 vacuum 掉。对于 held cursor，PostgreSQL 会尽量强制 detoast，避免提交后仍保留 snapshot。

这也是一个重要设计取舍。cursor 要跨事务返回数据，但不应该无限期固定全局 cleanup horizon。

所以系统宁可在 materialization 时付出 detoast 和 tuplestore 成本，也要缩短 snapshot 对 `xmin` 的影响。

## 10. subtransaction 与 ERROR cleanup

active stack 必须记录 nesting level。否则 subtransaction abort 后无法知道哪些 active snapshot 属于失败层级。

`AtSubCommit_Snapshot(level)` 做的事情很简单。它遍历 active stack。

凡是 `as_level >= level` 的 active frame，都把 `as_level` 改成 `level - 1`。这表示子事务成功提交后，这些 active snapshot 的 ownership 归并到父层级。

它们不是释放。它们只是层级归属改变。

`AtSubAbort_Snapshot(level)` 则不同。它从 stack top 开始清理。

只要 `ActiveSnapshot && ActiveSnapshot->as_level >= level`，就弹出这个 frame。对每个 frame：

- 保存 next。
- 减少 snapshot 的 `active_count`。
- 如果 `active_count == 0 && regd_count == 0`，释放 snapshot。
- 释放 stack node。
- 移动到 next。

最后调用 `SnapshotResetXmin()`。这条路径解释了为什么 active stack node 分配在 `TopTransactionContext`。

如果 ERROR 跳过正常 `PopActiveSnapshot()`，abort cleanup 仍能找到这些节点。但 active stack 只处理 active 引用。

registered 引用仍由 ResourceOwner release 或 snapshot manager 内部 cleanup 处理。这就是两套生命周期的另一个必要性。

subtransaction abort 可能只需要清理当前调用栈里属于失败子事务的 active frames。它不应该随便释放外层 cursor 或 catalog snapshot 的 registered reference。

`AtEOXact_Snapshot(isCommit, resetXmin)` 是顶层兜底。它先处理 `FirstXactSnapshot`。

transaction-snapshot mode 下，第一份事务 snapshot 有一个 snapshot manager 内部注册引用。它不通过普通 ResourceOwner 追踪。

EOXact 时必须从 `RegisteredSnapshots` 中移除。然后处理 exported snapshots。

导出的 snapshot 文件 unlink 失败只是 WARNING。因为事务已经太晚，不能为了 unlink 失败 abort 已完成的事务语义。

随后 invalidates catalog snapshot。如果是 commit，源码会对残留 registered snapshot 和 active snapshot 发 WARNING。

这不是正常 release 机制。这是泄漏检测和兜底。

最后它清空 active stack、reset registered heap、清空 `CurrentSnapshot` 和 `SecondarySnapshot`。如果 `resetXmin` 为 true，再调用 `SnapshotResetXmin()`。

源码注释特别说明，在普通 commit 处理中，`ProcArrayEndTransaction()` 已经在更早位置重置 `MyProc->xmin`。所以某些 commit 调用 `AtEOXact_Snapshot()` 时不需要再次 touch xmin。

这说明 snapshot cleanup 和 transaction ProcArray cleanup 是相邻但不同的层。不要把它们合并成一个“事务结束释放所有东西”。

## 11. 为什么 MemoryContext 不够

`CopySnapshot()` 确实把 snapshot 复制到 `TopTransactionContext`。这很容易让人误以为 MemoryContext 已经解决生命周期。

但 MemoryContext 的语义是内存批量释放。它不表达引用计数。

它不表达当前 active snapshot。它不表达 `xmin` 最小值。

它不接 ResourceOwner 的 ERROR cleanup 优先级。它也不区分“内存还活着”和“snapshot 还应该保护 tuple”。

举一个 cursor 例子。`PortalStart()` 返回后，active snapshot 已经 pop。

如果只看调用栈，snapshot 不该再活。但 executor 还没有结束。

cursor 下次 `FETCH` 还要继续用同一个 `queryDesc->snapshot`。所以需要 registered reference。

再看相反例子。cursor 提前 `CLOSE`。

snapshot 内存仍在 `TopTransactionContext` 中。如果只靠 context reset，它会到事务结束才释放。

但是 cursor 已经不会再使用它。它不应该继续固定 `MyProc->xmin`。

所以需要 `UnregisterSnapshot()` 和 `SnapshotResetXmin()`。再看 subtransaction ERROR。

一个子事务中 push 了 active snapshot，随后 ERROR 跳出。正常 C 函数返回路径没有机会 `PopActiveSnapshot()`。

如果只靠 MemoryContext，指针内存可能仍在，但 active stack 上会残留错误 frame。外层继续执行时 `GetActiveSnapshot()` 可能拿到失败子事务的 snapshot。

所以需要 `AtSubAbort_Snapshot(level)` 按 nesting level 清理 stack。再看 ResourceOwner。

executor state、portal、large object、catalog scan 可能把 snapshot 作为资源持有。这些资源有自己的 owner 和释放时机。

MemoryContext 不能替代 owner。否则 ERROR 后资源 release 顺序会失去控制。

这就是本节的核心抽象：

```text
MemoryContext 管 bytes；
active stack 管 current visibility context；
ResourceOwner/registered heap 管 long-lived semantic hold；
SnapshotResetXmin 把本地 holder 状态压缩成 shared `PGPROC->xmin`。
```

只有这四层合在一起，snapshot 生命周期才完整。

## 12. 可复现 runtime 现象

这个实验用普通 cursor 展示 registered snapshot 如何在 active frame 不存在时仍固定 `backend_xmin`。实验需要两个 psql session。

Session A：

```sql
DROP TABLE IF EXISTS snap_life_demo;
CREATE TABLE snap_life_demo(id int primary key, payload text);
INSERT INTO snap_life_demo
SELECT g, repeat('x', 200)
FROM generate_series(1, 20000) AS g;

BEGIN;
DECLARE c CURSOR FOR SELECT * FROM snap_life_demo ORDER BY id;
FETCH 1 FROM c;

SELECT pg_backend_pid();
```

记下 Session A 的 pid。不要提交。

不要关闭 cursor。Session B：

```sql
SELECT pid, state, backend_xmin, query
FROM pg_stat_activity
WHERE pid = <session_a_pid>;

DELETE FROM snap_life_demo WHERE id <= 10000;
VACUUM (VERBOSE) snap_life_demo;

SELECT n_live_tup, n_dead_tup
FROM pg_stat_user_tables
WHERE relname = 'snap_life_demo';
```

你应该看到 Session A 处于 idle in transaction。`backend_xmin` 通常仍有值。

VACUUM 可能报告无法移除某些 recently dead tuple，或者 `n_dead_tup` 在统计刷新后仍显示死行压力。具体数字受 autovacuum、统计刷新、版本和 timing 影响。

但核心现象稳定：

```text
一个看似没有正在执行 executor 的 session，仍然能通过 cursor 的 registered snapshot 固定 backend_xmin。
```

现在在 Session A 中关闭 cursor：

```sql
CLOSE c;
SELECT pg_backend_pid();
```

然后 Session B 再观察：

```sql
SELECT pid, state, backend_xmin, query
FROM pg_stat_activity
WHERE pid = <session_a_pid>;

VACUUM (VERBOSE) snap_life_demo;
```

如果没有其他 snapshot holder，`backend_xmin` 可能变成空。再次 VACUUM 更有机会移除旧版本。

如果仍然不释放，先检查 Session A 是否还有其他 cursor、portal、transaction snapshot、catalog snapshot 或 active query。这个实验不是为了把 `pg_stat_user_tables.n_dead_tup` 当成精确因果。

它是为了把现象连回源码。`DECLARE CURSOR` 让 executor snapshot registered。

`FETCH` 期间该 snapshot 被 push 为 active。`FETCH` 返回后 active frame pop，但 registered reference 仍在。

`CLOSE` 触发 portal cleanup 和 executor cleanup，`UnregisterSnapshot()` 后 `SnapshotResetXmin()` 才有机会推进 `MyProc->xmin`。这就是 runtime -> source 的闭环。

## 13. 回到源码解释实验

实验里 `backend_xmin` 的来源是 `PGPROC->xmin`。`GetSnapshotData()` 创建 snapshot 时会设置 `MyProc->xmin = TransactionXmin = xmin`，前提是本 backend 还没有 valid xmin。

这个 `xmin` 是当前 snapshot 需要保留的下界。之后 cursor 启动完成，active snapshot pop。

`PopActiveSnapshot()` 调用 `SnapshotResetXmin()`。但是 executor state registered 了 snapshot。

所以 `RegisteredSnapshots` 不为空。`SnapshotResetXmin()` 不会把 `MyProc->xmin` 清成 invalid。

它最多把它推进到 registered heap 顶部 snapshot 的 `xmin`。如果只有这个 cursor snapshot，heap 顶部就是它。

所以 Session A 的 `backend_xmin` 仍在。`FETCH` 时，`PortalRunSelect()` 临时 push `queryDesc->snapshot`。

这不会创建新的可见性世界。它只是把已经持有的 snapshot 变成当前 active。

fetch 结束后 pop。`backend_xmin` 仍由 registered holder 保持。

`CLOSE c` 或事务结束时，portal cleanup 会关闭 executor。`ExecutorEnd()` 释放 `estate->es_snapshot`。

`UnregisterSnapshot()` 进入 `UnregisterSnapshotFromOwner()`。它先 `ResourceOwnerForgetSnapshot()`。

然后 `UnregisterSnapshotNoOwner()` 减少 `regd_count`。如果 `regd_count` 变 0，就从 `RegisteredSnapshots` 移除。

如果 `active_count` 也为 0，就 `FreeSnapshot()`。最后当释放导致最后引用消失时，`SnapshotResetXmin()` 可能清空 `MyProc->xmin`。

VACUUM 看到的是下一节要讲的 horizon。

这里先只需要记住：

```text
cursor 没有运行 executor，不等于 cursor snapshot 已释放；
active frame 不存在，不等于 registered holder 不存在；
registered holder 不存在，才可能让 MyProc->xmin 前移。
```

## 14. portal 特例：utility、procedure 与 active stack 重建

`pquery.c` 里有几个特例能说明 active stack 不是普通 C 调用栈的薄包装。`PortalRunUtility()` 会在 utility statement 需要 snapshot 时 push active snapshot。

但源码注释说，某些 utility command 可能在执行过程中 pop 调用者推入的 active snapshot。例如 VACUUM 自己管理事务。

`PortalRunUtility()` 因此在返回后只在 `portal->portalSnapshot != NULL && ActiveSnapshotSet()` 时 pop。这不是理想化结构。

这是内核代码为了兼容 utility command 历史行为而保留的 awkwardness。`PlannedStmtRequiresSnapshot()` 也展示了边界。

普通 plannable query 一定需要 snapshot。但 transaction control、LOCK、SET、SHOW、FETCH、LISTEN、NOTIFY、CHECKPOINT、WAIT 等 utility 可能不需要由 portal 设置 snapshot。

这个判断不是因为这些命令不重要。而是因为它们的语义不应该在 transaction-snapshot-mode 事务刚开始时冻结一个 snapshot。

特别是 transaction control command 如果为了执行自身而取 snapshot，会改变后续隔离级别语义。`EnsurePortalSnapshotExists()` 是另一个重要特例。

procedure 或 DO block 内部可以在受限条件下 COMMIT/ROLLBACK。这会摧毁 active snapshot stack。

后续同一个 portal 中继续执行 SQL 时，可能需要重新建立 portal-level snapshot。`EnsurePortalSnapshotExists()` 在没有 active snapshot 时，从当前 `ActivePortal` 创建新的 snapshot。

它调用 `PushActiveSnapshotWithLevel(GetTransactionSnapshot(), portal->createLevel)`，并把 `portal->portalSnapshot` 指向 `GetActiveSnapshot()`。这里再次看到 `portal->createLevel` 的作用。

portal 持有的 active snapshot 必须能被 subtransaction cleanup 正确识别。这说明 active stack 不是“谁 push 谁 pop”的完美栈纪律。

在正常 executor 路径里，它接近调用栈。但 utility、procedure、ERROR cleanup 会打破直线控制流。

所以 snapshot manager 必须有自己的 stack、level 和 cleanup 钩子。

## 15. 正确性机制层次

本节涉及四类机制。第一类是 MVCC snapshot 语义。

`SnapshotData` 的 `xmin/xmax/xip/subxip/curcid` 决定 tuple 对某个 snapshot 是否可见。这部分不负责生命周期。

第二类是 active stack。它决定当前调用栈中 `GetActiveSnapshot()` 返回什么。

heap AM、index AM、catalog access、TOAST 和 executor 节点依赖这个当前上下文。active stack 不负责长期 holder。

第三类是 registered snapshot 和 ResourceOwner。它决定 snapshot 是否被长期资源持有。

它负责 ERROR cleanup 和 owner release。它还能通过 `RegisteredSnapshots` heap 支持 `xmin` 前移。

registered snapshot 不决定当前调用栈用哪个 snapshot。第四类是 `PGPROC->xmin`。

它把本 backend 的本地 snapshot 需求发布给其他 backend。其他 backend 不知道你的 `SnapshotData *`。

它们只在 ProcArray 扫描和 horizon 计算中看到你的 `xmin`。这四类机制不能互相替代。

active snapshot 不是 lock。registered snapshot 不是 pin。

MemoryContext 不是 ResourceOwner。`PGPROC->xmin` 不是 snapshot 对象。

错误的诊断通常来自把其中两类合并。例如看到 `backend_xmin` 非空，就说“query 正在执行”。

这不一定对。cursor idle in transaction 也能固定 `backend_xmin`。

例如看到 active snapshot 已 pop，就说“snapshot 释放了”。这也不一定对。

registered holder 可能仍在。例如看到内存还在 `TopTransactionContext`，就说“它还在保护 tuple”。

这也不一定对。如果 `active_count == 0 && regd_count == 0`，它可能只是还没等到 context reset 的无语义内存，或者已经被显式 `pfree`。

## 16. 错误路径与异常路径

第一条异常路径是 subtransaction abort。子事务里 push 的 active snapshot 如果不清理，会污染外层事务。

`AtSubAbort_Snapshot(level)` 按 `as_level >= level` 清掉 top frames。它不会扫描 registered heap 删除任意 snapshot。

registered holder 由 ResourceOwner 的子 owner release 处理。这让 active cleanup 和 owner cleanup 各守边界。

第二条异常路径是顶层 commit 后仍有 snapshot 残留。`AtEOXact_Snapshot(isCommit=true, ...)` 会对 leftover registered snapshots 和 active snapshots 发 WARNING。

这通常意味着调用者忘了 unregister 或 pop。在 abort 中，系统更倾向于安静清理。

因为 ERROR 路径里很多资源本来就可能没按 happy path 释放。第三条异常路径是 portal cleanup 失败。

`PortalDrop()` 注释说 cleanup hook 可能运行用户定义代码，失败要被预期。因此 portal 从 hash table 删除的时机、cleanup hook 置空的时机、holdSnapshot release 都要避免无限错误恢复循环。

这不是 snapshot 独有问题。但 snapshot holder 参与其中。

如果 portal 失败后 resowner 已经在 abort cleanup 中释放了 snapshot，后续 `PortalDrop()` 不能再假设 `holdSnapshot` 一定仍有 owner。第四条异常路径是 utility command 自己 pop active snapshot。

`PortalRunUtility()` 不能盲目 assert 它 push 的 snapshot 仍在 top。它必须容忍 active stack 已空。

这解释了为什么 `ActiveSnapshotSet()` 出现在 pop 前。第五条异常路径是 `UpdateActiveSnapshotCommandId()`。

它要求 active snapshot 唯一 active 且没有 registered 引用：

```text
active_count == 1
regd_count == 0
```

否则更新 `curcid` 会改变其他 holder 看到的可见性。parallel mode 下如果 command id 需要变化，它还会报错。

因为 parallel workers 在开始时共享 snapshot，leader 后续修改 active snapshot 的 `curcid` 会造成不一致。这些异常路径共同说明：

snapshot 生命周期错误不是小内存泄漏。它可能表现为错误可见性、过长 cleanup horizon、ERROR 后悬空 active snapshot、cursor 返回错误数据或 parallel consistency 问题。

## 17. 成本、资源与跨模块传播

active stack 的成本主要是 backend-local。每次 push 分配一个 `ActiveSnapshotElt`。

必要时还会 `CopySnapshot()`。copy 成本随 `xcnt` 和 `subxcnt` 增长。

如果 snapshot 很大，频繁 push static snapshot 会有可见成本。不过常见路径中 executor 会注册并复用 snapshot，fetch 时 push copied snapshot 引用，不一定每次复制完整数组。

registered heap 的成本主要在 register/unregister。第一次 registered 时进入 pairing heap。

最后一次 unregister 时从 heap 移除。heap 让找最小 `xmin` 便宜。

代价是 snapshot 持有路径多了一层 bookkeeping。`MyProc->xmin` 的成本是跨模块传播。

一旦它被老 snapshot 固定，VACUUM、HOT pruning、all-visible 标记、index cleanup、freeze 推进都会受到影响。

本节不展开这些算法。下一节会把它们压成 cleanup horizon。

这里要记住传播方向：

```text
cursor/executor registered snapshot
  -> MyProc->xmin
  -> ProcArray / ComputeXidHorizons()
  -> OldestXmin / GlobalVisState
  -> VACUUM / pruning 不能移除某些 old tuple versions
```

ResourceOwner 的成本是 cleanup 复杂性。每类资源都要声明 release phase 和 priority。

snapshot reference 在 after-locks 阶段释放。这让它和 buffer pin、relcache ref、lock release 区分开。

如果一个扩展或内部模块错误地长期注册 snapshot，它可能不会显示成 wait event。它更可能显示成 bloat、old `backend_xmin`、autovacuum warning、freeze pressure。

portal 的成本是状态跨度。普通查询结束后 executor snapshot 很快 unregister。

cursor 则可能把 executor state 留到下一次 fetch 或 close。holdable cursor 还可能把成本转移到 tuplestore、detoast 和 holdStore 内存/临时文件。

这就是 PostgreSQL 常见取舍：

为了缩短 snapshot horizon，系统愿意在 materialization 上付出 I/O 或内存成本。

## 18. 观测与诊断入口

最直接入口是 `pg_stat_activity.backend_xmin`。它显示 backend 当前暴露的 xmin。

它不是 snapshot 指针。它也不告诉你是 active snapshot 还是 registered snapshot 导致的。

但它能告诉你这个 backend 正在影响 cleanup horizon。常用查询：

```sql
SELECT pid, state, backend_xmin, xact_start, query
FROM pg_stat_activity
WHERE backend_xmin IS NOT NULL
ORDER BY backend_xmin;
```

第二个入口是 `pg_cursors`。它能看到当前 session 中可见 cursor。

注意它只看当前 session。它不能跨 session 列出别人所有 portal 内部状态。

如果一个 session `backend_xmin` 很老，且它处于 idle in transaction，cursor 是需要排查的对象之一。第三个入口是 VACUUM VERBOSE。

它可能报告 dead row 不能移除，或者只移除部分 dead tuples。这不是直接证明某个 snapshot holder。

它只是 cleanup horizon 被固定后的结果。需要结合 `pg_stat_activity.backend_xmin`、replication slot、prepared xact 和 standby feedback 排除其他来源。

第四个入口是 gdb 或临时日志。可以在这些函数打断点：

- `PushActiveSnapshotWithLevel()`
- `RegisterSnapshotOnOwner()`
- `UnregisterSnapshotNoOwner()`
- `SnapshotResetXmin()`
- `AtSubAbort_Snapshot()`
- `AtEOXact_Snapshot()`
- `PortalRunSelect()`
- `ExecutorEnd()`

观察字段：

- `ActiveSnapshot`
- `ActiveSnapshot->as_snap->active_count`
- `snapshot->regd_count`
- `snapshot->xmin`
- `MyProc->xmin`
- `pairingheap_is_empty(&RegisteredSnapshots)`

第五个入口是 `pg_locks` 和 wait event。它们通常不能直接解释 snapshot 生命周期。

snapshot holder 不一定等待 lock。它可能完全 idle。

所以诊断 bloat 时不能只看 lock wait。需要把 long transaction、cursor、replication slot、prepared xact 和 standby feedback 统一放进 horizon 模型。

本节只解释其中的 snapshot holder。

## 19. 常见误区

误区一：把 `GetTransactionSnapshot()` 返回值当成长生命周期对象。它可能指向静态 storage。

后续 snapshot call 或 command counter update 可能改写它。长期使用必须 push 或 register。

误区二：把 active snapshot 和 registered snapshot 当成同义词。active 回答“当前调用栈用什么”。

registered 回答“谁长期持有并负责释放”。cursor 在两次 fetch 之间最能说明两者差异。

误区三：把 MemoryContext 当 ResourceOwner。MemoryContext 管内存。

ResourceOwner 管资源引用和 cleanup 顺序。snapshot 需要两者配合，但不能只靠其中一个。

误区四：看到 `backend_xmin` 非空就认为该 backend 正在执行查询。idle cursor、exported snapshot、catalog snapshot 或其他 registered holder 都可能固定 `backend_xmin`。

误区五：认为 `AtEOXact_Snapshot()` 是唯一释放路径。很多 snapshot 应该在 executor end、portal drop 或 unregister 时提前释放。

等到 EOXact 才清理通常意味着 horizon 被固定得更久。误区六：认为 registered heap 中最小 `xmin` 等于全局 OldestXmin。

它只描述当前 backend 的 registered snapshots。全局 cleanup horizon 还要看其他 backend、replication slots、prepared xacts、standby feedback 和 relation kind。

这正是下一节的问题。

## 20. 课堂实验

实验一：普通 cursor 固定 `backend_xmin`。按第 12 节的 Session A / Session B 步骤执行。

重点观察 `FETCH` 返回后 Session A 没有 active executor 调用栈，但 `backend_xmin` 仍可能存在。回到源码，用 `PortalStart()` 和 `PortalRunSelect()` 解释 active push/pop，用 `ExecutorEnd()` 解释 registered release。

实验二：CLOSE cursor 后观察 `backend_xmin` 前移。在 Session A 执行：

```sql
CLOSE c;
```

然后 Session B 查询 `pg_stat_activity`。如果 `backend_xmin` 消失，回到 `UnregisterSnapshotNoOwner()` 和 `SnapshotResetXmin()` 解释。

如果没有消失，列出可能的其他 holder，不要直接得出源码错误结论。实验三：gdb 跟踪 refcount。

在测试实例上对 Session A backend 打断点：

```text
break PushActiveSnapshotWithLevel
break RegisterSnapshotOnOwner
break UnregisterSnapshotNoOwner
break SnapshotResetXmin
```

声明 cursor、fetch、close。每次断点看 `snapshot->active_count`、`snapshot->regd_count` 和 `MyProc->xmin`。

画出状态时间线。实验四：subtransaction abort 清理 active stack。

写一个 PL/pgSQL 函数，在 exception block 内执行需要 snapshot 的 SQL 并抛错。在 `AtSubAbort_Snapshot()` 打断点。

观察 `as_level` 大于等于当前 level 的 active frame 被清理。这个实验的重点不是业务 SQL，而是 ERROR 后没有正常 `PopActiveSnapshot()` 时谁兜底。

## 21. 讨论题

1. 为什么 `PushActiveSnapshot()` 必须在某些情况下复制 snapshot，而不是只增加一个指针引用？
2. 为什么 registered snapshot 要进入 ResourceOwner，而 `FirstXactSnapshot` 又不通过普通 ResourceOwner 追踪？
3. cursor 在两次 `FETCH` 之间没有 active snapshot，为什么仍可能固定 `backend_xmin`？
4. `SnapshotResetXmin()` 为什么 active stack 非空时直接返回，而不是扫描所有 active snapshot 找最小 `xmin`？
5. `UpdateActiveSnapshotCommandId()` 为什么要求 `active_count == 1` 且 `regd_count == 0`？
6. 如果一个扩展忘记 `UnregisterSnapshot()`，它更可能表现成内存泄漏、错误可见性，还是 vacuum lag？
7. `PortalRunUtility()` 为什么要容忍 utility command 把 active snapshot pop 掉？
8. 为什么 holdable cursor 倾向于 materialize/detoast，而不是简单持有原 executor snapshot 跨事务？

## 22. 本节小结

本节只回答一个问题：snapshot 为什么需要 active stack 和 registered heap 两套生命周期。答案不是“实现复杂”。

答案是 snapshot 同时承担两种角色。它是当前调用栈的可见性上下文。

它也是 cursor、executor、catalog、exported snapshot 等对象的长期语义引用。active stack 用 `ActiveSnapshotElt`、`as_level` 和 `active_count` 表达当前调用栈。

registered heap 用 `regd_count`、`ResourceOwnerRememberSnapshot()` 和 `RegisteredSnapshots` 表达长期 holder。`SnapshotResetXmin()` 把本 backend 的 holder 状态压缩成 `MyProc->xmin`。

MemoryContext 只回答内存何时可以批量释放。它不能替代 active stack。

它不能替代 ResourceOwner。它也不能决定 cleanup horizon。

普通 cursor 是最好的 mental model：

`DECLARE` 启动 executor 时 push active snapshot，executor 注册 snapshot，启动完成后 pop active snapshot。每次 `FETCH` 再临时 push 同一个 snapshot。

`CLOSE` 或 executor end 才 unregister。在这段时间里，即使 session idle，`backend_xmin` 也可能固定 VACUUM cleanup horizon。

ERROR 和 subtransaction abort 让这个设计更必要。`AtSubAbort_Snapshot()` 清理失败层级的 active frames。

ResourceOwner release 兜底 registered refs。`AtEOXact_Snapshot()` 做顶层 cleanup 和残留检测。

可迁移规律是：

```text
内核对象的生命周期不能只看内存是否还活着；
必须区分当前使用、长期持有、资源 owner、共享可见的保守下界。
```

下一节把这个本地 `MyProc->xmin` 扩展成全局 cleanup horizon，解释为什么一个 tuple 对当前查询已经不可见，仍然不能立刻被 VACUUM 或 HOT pruning 移除。
