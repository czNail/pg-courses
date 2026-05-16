# PostgreSQL 事务、Portal 与长生命周期状态

## 课程定位

前置知识：已经理解 MemoryContext tree 表达 backend-local 内存 ownership，也已经理解 executor 中 per-query、per-tuple、per-expression 这类短生命周期状态为什么要靠 reset 批量回收。

本节唯一主问题：

```text
为什么有些 backend-local 状态必须活过单条语句，但不能无限期挂在 TopMemoryContext 下？
```

核心矛盾：PostgreSQL 的一个 backend 会连续处理很多客户端消息、很多 SQL 语句、一个事务块中的多条命令、可暂停和继续执行的 cursor，以及可能跨事务继续读取的 holdable cursor。很多状态不能在单条语句结束时释放；但如果把这些状态都放进 `TopMemoryContext`，长连接会把“本该随事务或 cursor 消失的状态”变成 session 级 retention。

学完后应能判断：一块 backend-local 状态应该放在 `MessageContext`、`TopTransactionContext`、`CurTransactionContext`、`portal->portalContext`、`portal->holdContext`，还是极少数情况下放在 `TopMemoryContext`；也能判断为什么锁、buffer pin、snapshot 注册、cached plan refcount 不能只靠 MemoryContext 释放。

本课基于本地 `~/postgres` 源码，分支 `master`，提交 `8eba2edb8010`。

## 1. 本节在总主线中的位置

前两节建立了两个内存生命周期边界：

```text
MemoryContext tree:
  backend-local 内存归属于某个生命周期节点

executor per-tuple reset:
  每一行 tuple 的表达式临时结果不能跨 ResetExprContext()
```

但真实 backend 中还有更长的状态：

```text
一条协议消息:
  parse/bind/execute 消息中的临时字符串、parsetree、参数格式等

一个事务:
  snapshot、invalidation message、deferred trigger、GUC nest stack、combo cid 等

一个 Portal:
  可执行 query 的状态、cursor 位置、参数、QueryDesc、result tupdesc 等

一个 holdable cursor:
  事务提交后仍可 FETCH 的 materialized result

一个 backend session:
  cache、GUC 定义表、编码转换函数信息、portal manager 基础设施等
```

本节讨论的是中间层：

```text
比一条语句长；
比整个 backend session 短；
必须有明确的 end-of-transaction、portal drop、subtransaction abort 边界。
```

它不是 `TopMemoryContext` 的替代介绍。相反，本节要建立一个判断：

```text
TopMemoryContext 是最后手段，不是默认长生命周期容器。
```

如果一个对象能说清楚“活到事务结束”“活到 cursor 关闭”“活到 holdable cursor drop”，它就应该挂到对应 owner 下，而不是直接挂到 backend 根 context。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
事务内状态挂在 TopTransactionContext / CurTransactionContext；
可执行 cursor 状态挂在 Portal 的 portalContext 和 resowner；
跨事务 cursor 先 materialize 到 holdContext，再切断事务资源；
事务结束、PortalDrop、abort cleanup 分别负责释放不同层的状态。
```

这里的 tension 是：

```text
状态需要跨过单条语句边界，支持事务块、cursor、pipeline、CALL/DO 内部事务
  vs
长连接不能把这些状态无限期保留在 TopMemoryContext
```

一个简单例子：

```sql
BEGIN;
DECLARE c CURSOR FOR SELECT * FROM pg_class;
FETCH 10 FROM c;
FETCH 10 FROM c;
COMMIT;
```

cursor `c` 必须活过第一条 `FETCH`，否则第二条 `FETCH` 无法继续当前位置。它也不能只是 executor per-query 内存，因为 executor 可能处于暂停状态，`QueryDesc` 和位置状态要保留在 Portal 中。

但它也不能无条件活到 backend 退出。普通 cursor 在事务结束时必须关闭，因为它持有的 snapshot、plan/executor 状态、锁和 buffer pin 语义都属于当前事务。

如果写成更抽象的规则：

```text
能跨一条 statement 的对象，需要比 MessageContext 更长的 owner；
不能跨 transaction / portal close 的对象，需要比 TopMemoryContext 更短的 owner。
```

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 作用 |
| --- | --- | --- |
| 1 | `src/backend/tcop/postgres.c` | backend 主循环、`MessageContext`、`start_xact_command()` / `finish_xact_command()`、simple query 与 extended protocol 如何创建 Portal。 |
| 2 | `src/backend/access/transam/xact.c` | `TopTransactionContext`、`CurTransactionContext`、`TransactionAbortContext`、`TopTransactionResourceOwner` 的创建、commit、abort、cleanup 顺序。 |
| 3 | `src/include/utils/portal.h` | `PortalData` 的状态字段：`portalContext`、`resowner`、`createSubid`、`activeSubid`、`queryDesc`、`holdStore`、`holdContext`、`holdSnapshot`。 |
| 4 | `src/backend/utils/mmgr/portalmem.c` | Portal manager、`TopPortalContext`、`CreatePortal()`、`PortalDrop()`、`PreCommit_Portals()`、`AtAbort_Portals()`、subtransaction portal cleanup。 |
| 5 | `src/backend/tcop/pquery.c` | `PortalStart()`、`PortalRun()`、`PortalRunMulti()`、`FillPortalStore()`；这里能看到 `PortalContext` 和 `CurrentResourceOwner` 如何临时切换。 |
| 6 | `src/backend/commands/portalcmds.c` | SQL cursor 的 `PortalCleanup()` 与 `PersistHoldablePortal()`；holdable cursor 如何从 executor 状态转换为 tuplestore。 |
| 7 | `src/include/utils/resowner.h` / `src/backend/utils/resowner/resowner.c` | ResourceOwner 为什么和 MemoryContext 并列存在；锁、pin、snapshot、AIO handle 等外部资源如何分 phase 释放。 |
| 8 | `src/backend/utils/time/snapmgr.c` | snapshot copy / registration 多数分配到 `TopTransactionContext`，但 release 依赖 snapshot manager 与 ResourceOwner。 |
| 9 | `src/backend/catalog/system_views.sql` | `pg_cursors`、`pg_backend_memory_contexts` 的 SQL 观测入口。 |

阅读这组文件时，优先追问四个问题：

```text
这个状态要跨过哪个边界？
它的内存 owner 是谁？
它是否还持有内存之外的资源？
正常 commit、abort、subtransaction abort、PortalDrop 时分别由谁清理？
```

## 4. 关键数据结构与状态

### `TopTransactionContext` 与 `CurTransactionContext`

`xact.c` 的 `AtStart_Memory()` 在事务开始时做三件关键事：

```text
保存事务开始前的 CurrentMemoryContext
创建或复用 TransactionAbortContext
创建或复用 TopTransactionContext，并把 CurTransactionContext 指向它
```

顶层事务中：

```text
CurTransactionContext == TopTransactionContext
```

子事务开始时，`AtSubStart_Memory()` 创建新的 `CurTransactionContext`，并把它挂到父事务当前 context 下面：

```text
parent CurTransactionContext
  -> child CurTransactionContext
```

这表达了一个重要语义：

```text
子事务提交:
  子事务内存可以并入父事务生命周期

子事务 abort:
  子事务内存必须能被单独删除或失效
```

`TopTransactionContext` 不是每次事务结束都 delete，而是 reset：

```text
AtCommit_Memory()
  -> MemoryContextReset(TopTransactionContext)

AtCleanup_Memory()
  -> MemoryContextReset(TransactionAbortContext)
  -> MemoryContextReset(TopTransactionContext)
```

这让事务级基础 context 可以复用，同时保证其中的 transaction-local chunk 和子 context 全部消失。

### `TransactionAbortContext`

`TransactionAbortContext` 也是 `TopMemoryContext` 的子 context，但它不是普通业务数据容器。

它的作用是：

```text
在 abort cleanup 时提供一块预留内存；
即使事务内其它 context 已经处于错误状态或 OOM，也能运行清理逻辑。
```

`AtAbort_Memory()` 会切到它：

```text
if (TransactionAbortContext != NULL)
    MemoryContextSwitchTo(TransactionAbortContext);
else
    MemoryContextSwitchTo(TopMemoryContext);
```

这里的重点不是“abort 状态的数据放在这里长期保存”，而是：

```text
abort cleanup 自己也需要分配少量内存；
这个清理路径不能依赖已经失败的 transaction context。
```

### `PortalData`

`portal.h` 中的 `PortalData` 是本节最重要的状态结构。不要把它理解成“cursor 名字对应的 struct”，它更准确地说是：

```text
一个可运行、可暂停、可关闭、可跨消息引用的 query execution envelope。
```

关键字段可以分成几组：

| 字段 | 语义 |
| --- | --- |
| `name` | Portal hash table 中的名字；unnamed portal 名字是空字符串。 |
| `portalContext` | Portal 的主体内存；query 文本、参数、format、QueryDesc 等会挂在这里或其子树中。 |
| `resowner` | Portal 持有的资源 owner；用于锁、snapshot 注册、buffer pin 等非内存资源。 |
| `cleanup` | Portal drop 或失败时的 cleanup hook，通常是 `PortalCleanup()`。 |
| `createSubid` / `activeSubid` / `createLevel` | Portal 创建和最近使用所在的 subtransaction 层级；决定 subcommit/subabort 时如何处理。 |
| `stmts` / `cplan` | Portal 要执行的计划列表和 cached plan 引用。 |
| `portalParams` / `queryEnv` | 执行时参数和 query environment。 |
| `status` | `PORTAL_NEW`、`PORTAL_DEFINED`、`PORTAL_READY`、`PORTAL_ACTIVE`、`PORTAL_DONE`、`PORTAL_FAILED`。 |
| `queryDesc` | executor 正在活着时的执行描述；如果非空，关闭 Portal 时必须 eventually `ExecutorEnd()`。 |
| `portalSnapshot` | Portal 级 active snapshot，用于特殊场景下重建 SQL 执行所需 snapshot。 |
| `holdStore` / `holdContext` / `holdSnapshot` | materialized 结果集；用于 holdable cursor 或 returning/util-select 的 tuplestore。 |
| `atStart` / `atEnd` / `portalPos` | cursor 位置状态。 |

Portal 的本体不直接分配在 `portalContext` 里，而是分配在 `TopPortalContext`：

```text
portal = MemoryContextAllocZero(TopPortalContext, sizeof *portal);
portal->portalContext = AllocSetContextCreate(TopPortalContext, "PortalContext", ...);
```

这是一个很典型的两层结构：

```text
TopPortalContext:
  portal hash table
  PortalData 本体
  PortalContext:
    query/executor/portal 局部状态
  PortalHoldContext:
    跨事务 holdable cursor materialized result
```

PortalData 本体要能在 cleanup 过程中仍然被 hash table 找到，也要在删除子 context 后还能继续完成 `PortalDrop()` 的后续步骤，所以它不放在自己的 `portalContext` 里。

### `ResourceOwner`

ResourceOwner 不是 MemoryContext 的重复实现。它解决的是另一类对象：

```text
释放内存不会自动释放的资源。
```

例如：

```text
buffer pin
relation reference
snapshot registration
local lock
temporary file / AIO handle
cached plan refcount
```

`resowner.c` 中 `ResourceOwnerCreate()` 明确把 ResourceOwner 自身放在 `TopMemoryContext`：

```text
All ResourceOwner objects are kept in TopMemoryContext,
since they should only be freed explicitly.
```

这听起来和本节标题相反，但语义不矛盾：

```text
ResourceOwner 对象本身在 TopMemoryContext 中显式 delete；
它记录的资源必须在事务 / Portal / subtransaction 边界 release；
不能靠 pfree(owner) 当作资源释放。
```

也就是说，ResourceOwner 的内存生命周期很长只是实现手段；它表达的资源生命周期仍然是事务、Portal 或子事务。

## 5. 主流程源码 walkthrough

这一节跟两条主线：

```text
普通事务中的 unnamed portal:
  一条客户端命令如何创建、运行、结束、释放

DECLARE CURSOR / FETCH:
  Portal 如何活过单条语句，但在事务结束时关闭
```

### 5.1 backend 主循环：`MessageContext` 只覆盖一条消息

`postgres.c` 在 backend 主循环创建 `MessageContext`：

```text
MessageContext = AllocSetContextCreate(TopMemoryContext, "MessageContext", ...)
```

每轮接收客户端消息前 reset：

```text
MemoryContextSwitchTo(MessageContext);
MemoryContextReset(MessageContext);
```

所以 parse message、bind message、execute message 中的临时对象，如果只需要活到本消息结束，就应该放在 `MessageContext` 或它的子 context。

但 extended protocol 里 parse、bind、execute 可以跨多条消息：

```text
Parse:
  创建 prepared statement 或 unnamed prepared statement

Bind:
  创建 Portal，绑定参数和 result format

Execute:
  运行指定 Portal
```

这说明：

```text
MessageContext 太短，不能承载 Portal 的可执行状态。
```

因此 `exec_bind_message()` 会创建 Portal：

```text
portal = CreatePortal(portal_name, ..., ...);
```

随后把需要跨到 Execute 阶段的对象放进 portal 的生命周期中，或者确保它们被 cached plan / prepared statement 等更长 owner 持有。

### 5.2 事务开始：`StartTransactionCommand()` 切到事务 context

`postgres.c` 的 `start_xact_command()` 会调用：

```text
StartTransactionCommand()
```

`xact.c` 中 `StartTransactionCommand()` 在必要时调用 `StartTransaction()`，后者会调用：

```text
AtStart_Memory()
AtStart_ResourceOwner()
```

最后 `StartTransactionCommand()` 明确切到：

```text
MemoryContextSwitchTo(CurTransactionContext);
```

这让后续没有显式切换 context 的 transaction-local 分配默认进入当前事务 context。

一个典型的事务级对象例子是 snapshot copy。`snapmgr.c` 中很多 snapshot copy 分配在 `TopTransactionContext`：

```text
snapshot copy:
  内存活到事务结束
  但 snapshot 的 active/registered 语义还要由 snapmgr 和 ResourceOwner 管理
```

所以事务 context 解决的是“内存何时释放”，不是“snapshot 何时不再暴露 xmin”。

### 5.3 创建 Portal：`CreatePortal()`

`portalmem.c` 的 `EnablePortalManager()` 在 backend 初始化时创建：

```text
TopPortalContext = AllocSetContextCreate(TopMemoryContext,
                                         "TopPortalContext", ...)
PortalHashTable = hash_create("Portal hash", ...)
```

`CreatePortal()` 做几件事：

```text
如果同名 portal 存在，根据 allowDup 决定报错或 PortalDrop()
在 TopPortalContext 分配 PortalData
创建 portal->portalContext
创建 portal->resowner，parent 是 CurTransactionResourceOwner
记录 createSubid / activeSubid / createLevel
插入 PortalHashTable
```

这里已经出现了两条 ownership 线：

```text
内存线:
  TopPortalContext
    -> PortalData
    -> portalContext

资源线:
  CurTransactionResourceOwner
    -> portal->resowner
```

为什么 Portal 的 ResourceOwner 是当前事务 ResourceOwner 的 child？

因为普通 Portal 的资源不能比当前事务更久。关闭 Portal 时可以先释放自己的资源；如果没有关闭，事务结束也能从父 owner 递归释放。

### 5.4 定义 query：`PortalDefineQuery()`

`PortalDefineQuery()` 把 source text、plan list、cached plan 引用等放进 Portal：

```text
portal->sourceText = sourceText;
portal->stmts = stmts;
portal->cplan = cplan;
portal->status = PORTAL_DEFINED;
```

源码注释特别强调：

```text
如果 cplan 非空，调用者必须已经 GetCachedPlan()；
这个 refcount 会在 portal destroy 时释放。

如果 cplan 为空，调用者必须保证 plan tree 有足够生命周期；
通常做法是 copy 到 portal context。
```

这正是本节的核心判断：

```text
把指针存进 PortalData 不等于 ownership 自动正确；
指针指向的对象必须活到 Portal 使用结束。
```

### 5.5 启动 Portal：`PortalStart()`

`pquery.c` 的 `PortalStart()` 会临时切换全局状态：

```text
ActivePortal = portal
CurrentResourceOwner = portal->resowner
PortalContext = portal->portalContext
MemoryContextSwitchTo(PortalContext)
```

然后根据 strategy 处理：

```text
PORTAL_ONE_SELECT:
  PushActiveSnapshot(...)
  CreateQueryDesc(...)
  ExecutorStart(...)
  portal->queryDesc = queryDesc
  portal->tupDesc = queryDesc->tupDesc
  PopActiveSnapshot()

PORTAL_ONE_RETURNING / PORTAL_ONE_MOD_WITH:
  先只准备 result tupdesc

PORTAL_UTIL_SELECT:
  准备 utility tuple descriptor

PORTAL_MULTI_QUERY:
  先不启动 executor
```

这解释了为什么 Portal 不能只是 MessageContext 里的临时对象：

```text
PORTAL_ONE_SELECT 的 executor 可以在 FETCH 之间保持暂停状态；
queryDesc、tupDesc、cursor position 必须活过一次 PortalRun()。
```

但 `PortalStart()` 也有 `PG_TRY/PG_CATCH`：

```text
出错:
  MarkPortalFailed(portal)
  恢复 ActivePortal / CurrentResourceOwner / PortalContext
  PG_RE_THROW()
```

这说明 Portal 是 ERROR-safe cleanup 链的一部分。它不依赖普通 C 返回路径逐层 free。

### 5.6 运行 Portal：`PortalRun()`

`PortalRun()` 也会切到 portal 状态：

```text
MarkPortalActive(portal)
ActivePortal = portal
CurrentResourceOwner = portal->resowner
PortalContext = portal->portalContext
MemoryContextSwitchTo(PortalContext)
```

如果是普通 `PORTAL_ONE_SELECT`：

```text
PortalRunSelect()
  -> ExecutorRun(...)
  -> 可能只返回 count 行
  -> portal->status = PORTAL_READY
  -> result = portal->atEnd
```

如果还没有到末尾，Portal 回到 `PORTAL_READY`，等待下一次 `FETCH` 或 `Execute`：

```text
同一个 portal
  queryDesc 仍存在
  cursor position 更新
  executor 状态可继续
```

如果是 `PORTAL_MULTI_QUERY`，则运行到 completion，之后 `MarkPortalDone()`。

`PortalRunMulti()` 在每个 statement 后会做：

```text
MemoryContextDeleteChildren(portal->portalContext);
```

这是一条细腻的边界：

```text
PortalContext 本身不能删，因为 Portal 还在；
但其中为某个 statement 创建的 subsidiary contexts 可以丢掉。
```

### 5.7 普通事务结束：`PreCommit_Portals()` 与 `AtCommit_Memory()`

`CommitTransaction()` 里会先处理 portals：

```text
for (;;)
{
    AfterTriggerFireDeferred();
    if (!PreCommit_Portals(false))
        break;
}
```

`PreCommit_Portals()` 对每个 Portal 判断：

```text
holdable portal:
  HoldPortal(portal)

already held from previous transaction:
  continue

普通 portal:
  PortalDrop(portal, true)
```

随后事务进入 resource cleanup：

```text
ResourceOwnerRelease(TopTransactionResourceOwner, BEFORE_LOCKS, ...)
...
ResourceOwnerRelease(TopTransactionResourceOwner, LOCKS, ...)
ResourceOwnerRelease(TopTransactionResourceOwner, AFTER_LOCKS, ...)
...
ResourceOwnerDelete(TopTransactionResourceOwner)
AtCommit_Memory()
```

`AtCommit_Memory()` 做最终内存边界：

```text
MemoryContextSwitchTo(s->priorContext)
MemoryContextReset(TopTransactionContext)
CurTransactionContext = NULL
```

注意顺序：

```text
Portal / ResourceOwner / module cleanup
  -> 释放锁、pin、snapshot、plan refcount 等资源
  -> 最后 reset transaction memory
```

不能先 reset `TopTransactionContext`，因为事务清理函数还可能需要读取其中的状态来释放外部资源。

## 6. 生命周期 / ownership / cleanup

### 常见 context 的生命周期

| Context / owner | 典型内容 | 创建位置 | 释放边界 |
| --- | --- | --- | --- |
| `MessageContext` | 单条客户端消息中的解析临时对象、dest receiver、短命字符串 | `postgres.c` backend 主循环 | 每轮主循环 reset |
| `TopTransactionContext` | transaction-local arrays、snapshot copy、invalidation、deferred trigger、GUC stack | `AtStart_Memory()` | commit 时 `AtCommit_Memory()` reset；abort cleanup 时 `AtCleanup_Memory()` reset |
| `CurTransactionContext` | 当前顶层事务或子事务内状态 | `AtStart_Memory()` / `AtSubStart_Memory()` | 顶层事务 reset；子事务 abort 删除或失效；子事务 commit 并入父级语义 |
| `TransactionAbortContext` | abort cleanup 的预留工作区 | `AtStart_Memory()` | abort cleanup 后 reset，backend 生命周期内复用 |
| `TopPortalContext` | Portal manager、Portal hash、PortalData 本体、portal 子 context | `EnablePortalManager()` | backend 退出 |
| `portal->portalContext` | Portal query/executor 状态、参数、format、QueryDesc 等 | `CreatePortal()` | `PortalDrop()` delete；中间可 `MemoryContextDeleteChildren()` 清 subsidiary contexts |
| `portal->holdContext` | holdable cursor materialized result、tuplestore、长期 tupdesc copy | `PortalCreateHoldStore()` | holdable cursor 最终 `PortalDrop()` 时 delete |
| `portal->resowner` | Portal 持有的资源记录 | `CreatePortal()` | `PortalDrop()` 或事务 ResourceOwner cleanup |

### 为什么 PortalData 本体在 `TopPortalContext`

如果 `PortalData` 本体也放在 `portal->portalContext`，那么删除 portalContext 时：

```text
portal 指针自己先失效；
后续还要从 hash table 删除、释放 holdStore、释放 cached plan refcount、pfree PortalData
```

这会让 cleanup 顺序很脆弱。

所以 PostgreSQL 选择：

```text
PortalData 本体:
  放在 TopPortalContext，显式 pfree

Portal 的大部分 subsidiary memory:
  放在 portalContext，drop 时 delete

holdable result:
  放在 holdContext，独立于 portalContext
```

### 为什么 holdable cursor 需要 `holdContext`

`DECLARE CURSOR WITH HOLD` 的语义是：

```text
事务提交后仍可 FETCH。
```

这意味着不能保留：

```text
executor state
transaction snapshot dependency
transaction locks
buffers / pins
plan execution state
```

`HoldPortal()` 会：

```text
PortalCreateHoldStore(portal)
PersistHoldablePortal(portal)
PortalReleaseCachedPlan(portal)
portal->resowner = NULL
portal->createSubid = InvalidSubTransactionId
portal->activeSubid = InvalidSubTransactionId
```

`PersistHoldablePortal()` 关键动作是：

```text
把 tupdesc copy 到 holdContext
把剩余 executor 输出写入 holdStore
tuplestore receiver 强制 detoast 数据
ExecutorFinish / ExecutorEnd / FreeQueryDesc
MemoryContextDeleteChildren(portal->portalContext)
```

强制 detoast 的意义是：

```text
结果中的 TOAST 指针不能依赖创建事务的 snapshot；
跨事务保存时必须让 tuplestore 持有可独立访问的数据。
```

所以 holdable cursor 不是“把 executor 留到下个事务继续跑”，而是：

```text
在事务提交前，把还能产生的结果 materialize 成一个独立 tuplestore；
然后释放事务资源，只留下可读的结果容器。
```

### 为什么 ResourceOwner 不能省

MemoryContext reset 能释放 palloc chunk，但不能自动做这些事：

```text
释放 lock manager 中的锁
释放 buffer pin
UnregisterSnapshotFromOwner()
ReleaseCachedPlan()
关闭或删除 tuplestore temp files
回退 GUC nest stack 的语义
发送 invalidation / notify cleanup
```

因此完整 cleanup 经常是两条线一起推进：

```text
ResourceOwner:
  释放外部资源和引用计数

MemoryContext:
  释放 backend-local 内存 chunk
```

只做后者会泄漏资源；只做前者会泄漏内存。

## 7. 正确性机制层次

### 事务状态不能只靠内存活着

事务级状态通常依赖多层机制：

| 机制 | 管什么 |
| --- | --- |
| `TopTransactionContext` | transaction-local backend 内存 |
| `TopTransactionResourceOwner` | buffer pin、lock、snapshot registration 等资源 |
| transaction state machine | `TBLOCK_*` / `TRANS_*` 状态推进 |
| snapshot manager | active snapshot、registered snapshot、xmin 暴露 |
| invalidation manager | catalog/relcache 变更对其它 backend 可见 |
| lock manager | 并发互斥与等待 |
| WAL / clog / ProcArray | commit/abort 对其它 backend 和 crash recovery 的语义 |

`TopTransactionContext` reset 只是其中最后一层。

### Portal 状态必须和 subtransaction 绑定

PortalData 记录：

```text
createSubid
activeSubid
createLevel
```

原因是 subtransaction 可以创建或使用 Portal，然后失败。

如果子事务创建了 Portal，随后 abort：

```text
Portal 可能引用子事务中创建或改变的对象；
不能让它继续 READY 并在父事务中运行。
```

`AtSubAbort_Portals()` 会把相关 Portal 标为 FAILED，运行 cleanup hook，释放 cached plan reference，并删除 portalContext 的子 context。

如果子事务 commit：

```text
AtSubCommit_Portals()
  createSubid -> parentSubid
  activeSubid -> parentSubid
  portal->resowner reparent 到父事务 ResourceOwner
```

这就是“跨过子事务 commit，但不能跨过子事务 abort”的 ownership 表达。

### cleanup 顺序是一种 correctness 机制

`CommitTransaction()` 的注释给出核心顺序：

```text
先释放其它 backend 可见的资源，例如文件、buffer pin；
再释放 locks；
最后释放 backend-local resources。
```

原因是：

```text
等待锁的其它 backend 一旦被唤醒，就可能观察我们提交后的 catalog/data 状态；
在释放锁前，相关 invalidation、relcache cleanup、buffer pin cleanup 必须处于正确状态。
```

所以不能把 transaction cleanup 简化成：

```text
MemoryContextReset(TopTransactionContext);
```

这个 reset 只能发生在更高层语义都收尾之后。

## 8. 错误路径 / 异常路径 / fallback

### `PortalStart()` / `PortalRun()` 的 `PG_TRY`

Portal 执行路径会修改全局变量：

```text
ActivePortal
CurrentResourceOwner
PortalContext
CurrentMemoryContext
```

如果执行中 `ERROR` longjmp，必须恢复这些全局指针，否则后续 cleanup 会在错误 owner 下继续执行。

`PortalRun()` 的 catch path 会：

```text
MarkPortalFailed(portal)
恢复 CurrentMemoryContext
恢复 ActivePortal
恢复 CurrentResourceOwner
恢复 PortalContext
PG_RE_THROW()
```

这里还有一个很真实的 awkwardness：

```text
VACUUM、CLUSTER、CALL/DO 等 utility command 可能内部提交并重启事务；
PortalRun() 保存 TopTransactionResourceOwner / TopTransactionContext，
恢复时要判断旧指针是否已经被替换。
```

这说明 Portal 生命周期不是理想化层次结构，而是必须兼容 PostgreSQL 历史上存在的内部事务推进路径。

### abort 时不能正常关闭所有 Portal

`AbortTransaction()` 早期会：

```text
AtAbort_Memory()
AtAbort_ResourceOwner()
...
AtAbort_Portals()
```

`AtAbort_Portals()` 不会立刻 `PortalDrop()` 所有 Portal。注释说明：

```text
此时可以运行 cleanup hook；
但不能释放 portal 本体内存，直到 cleanup call 之后。
```

它会：

```text
跳过前一事务留下的 held cursor
跳过 auto-held cursor
把 READY portal 标成 FAILED
调用 cleanup hook
释放 cached plan reference
portal->resowner = NULL
删除 portalContext 的子 context
```

之后 `CleanupTransaction()` 调用：

```text
AtCleanup_Portals()
AtEOXact_Snapshot(false, true)
ResourceOwnerDelete(...)
AtCleanup_Memory()
```

这两阶段设计避免了一个问题：

```text
abort 中某些 cleanup 还需要 PortalData 和部分 portal state；
但事务资源释放后，不能再假设 executor 正常可关闭。
```

### `PortalCleanup()` 在 FAILED portal 上更保守

`portalcmds.c` 的 `PortalCleanup()` 会检查 `portal->queryDesc`。

如果 executor 仍在：

```text
portal->queryDesc = NULL
```

然后只有在 portal 不是 `PORTAL_FAILED` 时才运行：

```text
ExecutorFinish(queryDesc)
ExecutorEnd(queryDesc)
FreeQueryDesc(queryDesc)
```

失败状态下跳过 executor shutdown，是因为：

```text
错误发生后，executor 内部状态可能引用已经失效的事务对象；
强行正常 shutdown 可能再次 ERROR 或 crash；
事务 abort 机制会兜底释放资源。
```

这里的取舍是：

```text
正常路径尽量完整关闭 executor；
错误路径优先保证 cleanup 可继续推进，避免二次失败。
```

### `PortalDrop()` 先从 hash table 删除

`PortalDrop()` 会先调用 cleanup hook，再从 hash table 删除 Portal。

源码注释说明：

```text
hash table 删除之后，如果后续步骤出错，也不会再次回来尝试删除同一个 portal；
宁愿 leak 一点内存，也不要陷入错误恢复循环。
```

这是 PostgreSQL cleanup 代码里很常见的原则：

```text
ERROR recovery 路径要避免无限递归；
cleanup 失败时，小的 backend-local leak 通常比破坏全局状态更可接受。
```

## 9. 成本、资源与跨模块传播

### 为什么不用一个巨大的 transaction malloc 区

如果所有事务内状态都直接挂在 `TopMemoryContext`，长事务或长连接会遇到两个问题：

```text
事务结束不能统一 reset；
无法用 pg_backend_memory_contexts 看出 retention 属于哪个事务 / portal / executor。
```

如果每个对象都逐个 `pfree()`，则会遇到：

```text
commit/abort 路径要知道每个模块的所有对象；
ERROR longjmp 后很难保证每条 retail free 路径都执行；
hot path 中 ownership 传递成本高。
```

PostgreSQL 的折中是：

```text
内存按生命周期批量 reset/delete；
非内存资源按 ResourceOwner 分 phase release；
模块语义通过 AtEOXact_* / AtAbort_* / PreCommit_* 回调补齐。
```

### Portal hash table 的成本

`TopPortalContext` 下的 Portal hash 初始估计值是 `PORTALS_PER_USER = 16`。

事务结束时 `PreCommit_Portals()`、`AtAbort_Portals()`、subtransaction cleanup 都会扫描 portal hash table。

这意味着：

```text
大量未关闭 cursor 会增加事务结束和 subtransaction cleanup 成本；
即使每个 cursor 当前不活跃，也要在边界上被检查。
```

这不是通常 SQL workload 的瓶颈，但对生成大量 cursor、PL/pgSQL 循环、驱动端不关闭 named portal 的场景，会变成可观察的 backend-local 状态膨胀。

### holdable cursor 的成本转移

普通 cursor：

```text
低内存，保留 executor 状态和事务资源；
事务结束时必须关闭。
```

holdable cursor：

```text
提交前 materialize 结果；
释放 executor、snapshot、locks；
提交后 FETCH 从 tuplestore 读取。
```

所以 `WITH HOLD` 不是免费跨事务 cursor。它把成本从：

```text
持续持有执行状态和事务资源
```

转成：

```text
提交前一次性读取剩余结果、写 tuplestore、可能使用临时文件。
```

### SPI、CALL/DO 与非原子上下文

`spi.c` 中有一条很有代表性的边界：

```text
atomic contexts:
  SPI procCxt / execCxt 使用 TopTransactionContext

non-atomic contexts:
  使用 PortalContext 或其子 context
```

原因是 procedure / DO block 可能在执行中提交或回滚事务。此时：

```text
TopTransactionContext 可能被 reset；
PortalContext 代表外层可继续执行的 Portal envelope。
```

这也是本节主题的另一种表达：

```text
不是所有“长于语句”的状态都应该是 transaction-local；
有些状态必须跟随 Portal，而不是跟随当前事务。
```

## 10. 观测与诊断入口

### `pg_backend_memory_contexts`

在当前 session 中可以看 context tree：

```sql
SELECT name, ident, parent, level, total_bytes, used_bytes
FROM pg_backend_memory_contexts
WHERE name IN (
  'TopTransactionContext',
  'TransactionAbortContext',
  'TopPortalContext',
  'PortalContext',
  'PortalHoldContext',
  'MessageContext'
)
ORDER BY level, name, ident;
```

观察要点：

```text
普通事务开始后会看到 TopTransactionContext；
DECLARE CURSOR 后会看到 PortalContext；
DECLARE CURSOR WITH HOLD 并 COMMIT 后可能看到 PortalHoldContext；
关闭 cursor 后对应 PortalContext / PortalHoldContext 应消失。
```

输出大小会随版本、编译选项、session 状态变化，不要把精确 byte 数当成稳定结论。重点是 context 是否存在、parent 是谁、是否在边界后消失。

### `pg_cursors`

Portal 的 SQL 级观察入口是：

```sql
SELECT name, statement, is_holdable, is_binary, is_scrollable
FROM pg_cursors
ORDER BY name;
```

普通 cursor：

```sql
BEGIN;
DECLARE c CURSOR FOR SELECT * FROM pg_class;
SELECT name, is_holdable FROM pg_cursors;
COMMIT;
SELECT name, is_holdable FROM pg_cursors;
```

`COMMIT` 后普通 cursor 不应再出现。

holdable cursor：

```sql
BEGIN;
DECLARE hc CURSOR WITH HOLD FOR SELECT oid, relname FROM pg_class;
FETCH 1 FROM hc;
COMMIT;
SELECT name, is_holdable FROM pg_cursors;
FETCH 1 FROM hc;
CLOSE hc;
```

`COMMIT` 后 `hc` 还能出现并继续 `FETCH`，但它已经不再持有创建事务的 executor 状态，而是读取 materialized tuplestore。

### `pg_log_backend_memory_contexts(pid)`

另一个 session 可以触发目标 backend 打印 memory context tree：

```sql
SELECT pg_log_backend_memory_contexts(<pid>);
```

适合观察：

```text
TopPortalContext 下是否有大量 PortalContext
PortalHoldContext 是否在 CLOSE 后消失
TopTransactionContext 是否在事务结束后 reset 到较小状态
```

注意这个函数把信息写入 server log，不直接返回完整树。

### gdb 断点

适合本节的断点：

```gdb
break AtStart_Memory
break AtCommit_Memory
break AtCleanup_Memory
break CreatePortal
break PortalDrop
break PreCommit_Portals
break AtAbort_Portals
break PersistHoldablePortal
break ResourceOwnerRelease
```

建议观察变量：

```gdb
print TopTransactionContext
print CurTransactionContext
print TopTransactionResourceOwner
print portal->name
print portal->status
print portal->createSubid
print portal->activeSubid
print portal->portalContext
print portal->holdContext
print portal->resowner
```

不要只看指针是否非空。真正的语义来自：

```text
指针所在 context
Portal status
subtransaction id
ResourceOwner parent
commit/abort cleanup phase
```

## 11. 常见误区

### 误区 1：活过一条语句就应该放 `TopMemoryContext`

更准确的判断是：

```text
活过一条语句:
  可能属于 transaction、Portal、prepared statement、cache 或 session

只有真正 backend 生命周期内都有效的状态:
  才考虑 TopMemoryContext 或 TopMemoryContext 下的长期子 context
```

把 transaction-local 状态放到 `TopMemoryContext`，在短连接测试中可能看不出问题；在连接池和长 session 中会变成稳定增长的 retention。

### 误区 2：Portal 是事务内对象，所以都放 `TopTransactionContext`

普通 Portal 的资源确实由当前事务兜底，但 Portal 本体由 Portal manager 管理。

原因是：

```text
Portal 可能跨多条消息和多条语句；
abort cleanup 需要先操作 PortalData，再释放事务内存；
holdable Portal 还可能跨事务继续存在。
```

因此 Portal 有自己的 `TopPortalContext`、`portalContext`、`holdContext` 和 Portal hash table。

### 误区 3：MemoryContext reset 后资源也自动释放

MemoryContext 只知道 palloc chunk。它不知道：

```text
一个 snapshot 是否 registered
一个 cached plan refcount 是否增加
一个 buffer pin 是否还在
一个 local lock 是否属于当前事务
一个 tuplestore 是否有跨事务临时文件
```

这些必须由 ResourceOwner、PortalDrop、tuplestore、snapmgr、plancache 等模块显式处理。

### 误区 4：holdable cursor 是把 cursor 执行状态保留到下个事务

不是。

holdable cursor 在提交前会：

```text
把剩余结果写入 tuplestore
关闭 executor
释放 cached plan reference
清掉 transaction resource owner
标记 createSubid / activeSubid 为 Invalid
```

提交后继续 `FETCH` 的不是原 executor，而是 materialized result。

### 误区 5：Portal status 只是展示信息

`PORTAL_READY`、`PORTAL_ACTIVE`、`PORTAL_DONE`、`PORTAL_FAILED` 直接影响 cleanup 行为。

特别是：

```text
PORTAL_FAILED:
  cleanup 更保守，避免在错误路径中再次正常 shutdown executor

PORTAL_ACTIVE:
  不能普通 drop

PORTAL_READY:
  可继续执行，但 subtransaction abort 中可能被强制 failed
```

## 12. 课堂实验

### 实验 1：普通 cursor 在事务结束后消失

一个 session 执行：

```sql
BEGIN;
DECLARE c CURSOR FOR SELECT oid, relname FROM pg_class;
FETCH 2 FROM c;

SELECT name, statement, is_holdable, is_scrollable
FROM pg_cursors
ORDER BY name;

SELECT name, ident, parent, level, used_bytes
FROM pg_backend_memory_contexts
WHERE name LIKE '%Portal%' OR name = 'TopTransactionContext'
ORDER BY level, name;

COMMIT;

SELECT name, statement, is_holdable
FROM pg_cursors;
```

预期现象：

```text
COMMIT 前能看到 cursor；
COMMIT 后普通 cursor 消失；
相关 PortalContext 不应继续作为可见活跃 cursor 的 owner 存在。
```

源码回扣：

```text
CommitTransaction()
  -> PreCommit_Portals(false)
     -> PortalDrop(portal, true)
  -> ResourceOwnerRelease(...)
  -> AtCommit_Memory()
```

### 实验 2：holdable cursor 跨事务存在

```sql
BEGIN;
DECLARE hc CURSOR WITH HOLD FOR
  SELECT oid, relname FROM pg_class ORDER BY oid;
FETCH 1 FROM hc;

SELECT name, is_holdable FROM pg_cursors ORDER BY name;

COMMIT;

SELECT name, is_holdable FROM pg_cursors ORDER BY name;

SELECT name, ident, parent, level, used_bytes
FROM pg_backend_memory_contexts
WHERE name LIKE '%Portal%'
ORDER BY level, name, ident;

FETCH 1 FROM hc;
CLOSE hc;
```

预期现象：

```text
COMMIT 后 hc 仍在 pg_cursors 中；
它可以继续 FETCH；
CLOSE 后消失。
```

源码回扣：

```text
PreCommit_Portals(false)
  -> HoldPortal()
     -> PortalCreateHoldStore()
     -> PersistHoldablePortal()
        -> ExecutorRun(... tuplestore ...)
        -> ExecutorFinish / ExecutorEnd / FreeQueryDesc
        -> MemoryContextDeleteChildren(portal->portalContext)
```

### 实验 3：subtransaction abort 后 cursor 失效

在 psql 中：

```sql
BEGIN;
SAVEPOINT s;
DECLARE sc CURSOR FOR SELECT * FROM pg_class;
ROLLBACK TO s;
SELECT name FROM pg_cursors ORDER BY name;
COMMIT;
```

预期现象：

```text
在 savepoint 内创建的 cursor 不应在 rollback to savepoint 后继续正常可用。
```

源码回扣：

```text
AtSubAbort_Portals()
  -> 发现 portal->createSubid == mySubid
  -> MarkPortalFailed()
  -> cleanup hook
  -> PortalReleaseCachedPlan()
  -> MemoryContextDeleteChildren(portal->portalContext)
AtSubCleanup_Portals()
  -> 删除该 subtransaction 创建的 portal
```

### 实验 4：gdb 观察普通 Portal 创建和 drop

启动 backend 后设置断点：

```gdb
break CreatePortal
break PortalStart
break PortalRun
break PreCommit_Portals
break PortalDrop
```

执行：

```sql
BEGIN;
DECLARE c CURSOR FOR SELECT * FROM pg_class;
FETCH 1 FROM c;
COMMIT;
```

观察：

```text
CreatePortal 时 portal->createSubid
PortalStart 后 portal->status
FETCH 后 portal->portalPos
PreCommit_Portals 中普通 portal 如何走 PortalDrop
PortalDrop 中先 cleanup，再 hash delete，再 release cplan / resources / contexts
```

### 实验 5：构造“为什么不能只 reset 内存”的推理

源码阅读练习：

```text
1. 在 PortalDefineQuery() 中找到 cplan refcount 交接说明。
2. 在 PortalDrop() 中找到 PortalReleaseCachedPlan()。
3. 在 ResourceOwnerRelease() 中找到三阶段 release。
4. 在 CommitTransaction() 中找到 ResourceOwnerRelease 与 AtCommit_Memory 的相对顺序。
```

回答：

```text
如果只 MemoryContextReset(TopTransactionContext)，cached plan refcount、
snapshot registration、locks、buffer pins 各会发生什么？
```

## 13. 讨论题

1. `TopTransactionContext` 在 commit 时 reset，而不是 delete。这个选择带来什么好处？什么时候 reset 不足以表达资源 cleanup？

2. 普通 cursor 为什么不能在事务提交后继续保留 executor 状态？请分别从 snapshot、lock、buffer pin、catalog invalidation、executor memory 角度解释。

3. `PortalData` 本体为什么不放在 `portal->portalContext`？如果放进去，`PortalDrop()` 的哪些步骤会变得危险？

4. holdable cursor 为什么要强制 materialize？为什么 `PersistHoldablePortal()` 要把 `tupDesc` copy 到 `holdContext`？

5. 子事务 abort 时，为什么有些上层 Portal 只是保守地标记 failed 或重新挂 ResourceOwner，而不是简单地继续使用？

6. 如果扩展在 `TopMemoryContext` 中缓存了 transaction-local 指针，会有哪些症状？如何用 `pg_backend_memory_contexts`、gdb 或 server log 观察？

7. 为什么 ResourceOwner 自身可以分配在 `TopMemoryContext`，但它记录的资源不能拥有 session 生命周期？

## 14. 本节小结

本节的唯一主问题是：

```text
为什么有些 backend-local 状态必须活过单条语句，但不能无限期挂在 TopMemoryContext 下？
```

答案可以压缩成一个模型：

```text
MessageContext:
  一条客户端消息

TopTransactionContext / CurTransactionContext:
  当前事务或子事务

portal->portalContext:
  一个可执行 Portal / cursor envelope

portal->holdContext:
  跨事务 holdable cursor 的 materialized result

TopMemoryContext:
  backend 生命周期内真正 permanent 的基础设施，或需要显式 delete 的管理对象
```

PostgreSQL 不是简单地按“短命 / 长命”二分内存，而是按 runtime 边界分层：

```text
statement boundary
transaction boundary
subtransaction boundary
portal close boundary
holdable cursor boundary
backend session boundary
```

MemoryContext 管 palloc chunk 的批量释放；ResourceOwner 管锁、pin、snapshot、refcount 这类内存之外的资源；Portal 把可暂停执行和 cursor 位置状态包成一个可被事务边界识别的对象。

可迁移的系统规律是：

```text
任何“活过当前调用栈”的状态，都必须回答两个问题：

1. 它跨过哪个 runtime 边界？
2. 它不能跨过哪个 runtime 边界？

只有同时回答这两个问题，ownership 才完整。
```

下一节会继续沿着错误路径展开：`ERROR` longjmp 后，哪些内存能靠 MemoryContext tree 自动回收，哪些资源必须靠 ResourceOwner、callback 和专门的 abort cleanup 兜底。
