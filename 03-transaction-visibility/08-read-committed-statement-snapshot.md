# PostgreSQL READ COMMITTED 的语句级 Snapshot

## 课程定位

前置知识：

你已经理解 `SnapshotData` 为什么是 `xmin`、`xmax`、`xip`、`subxip`、`suboverflowed` 和 `curcid` 的组合。你已经理解 `HeapTupleSatisfiesMVCC()` 如何把 snapshot running 判断翻译成 tuple 可见性。

你已经知道 PostgreSQL 默认隔离级别是 READ COMMITTED。你已经能区分事务生命周期、语句生命周期、executor 生命周期和 portal 生命周期。

本节唯一主问题：

```text
READ COMMITTED 如何同时做到“每条语句看见语句开始前已经提交的新数据”和“单条语句内部使用稳定的可见性视角”？
```

本节核心矛盾：

默认隔离级别希望在一个事务内尽量新鲜，后一条语句能看到前一条语句之后提交的数据。但 executor 扫描、修改、触发器、返回结果和 cursor fetch 不能在同一条语句内部被并发 commit 撕裂。

学完本节后，你应该能独立判断：

- 为什么 READ COMMITTED 的新鲜度边界是 statement，而不是 transaction，也不是 tuple。
- 为什么 `GetTransactionSnapshot()` 在 READ COMMITTED 下非首次调用也会重新 `GetSnapshotData()`。
- 为什么调用者必须 `PushActiveSnapshot()` 或 `RegisterSnapshot()`，不能裸用返回的静态 snapshot。
- 为什么 `ExecutorStart()` 要求 query snapshot 已经 active，并把它注册进 `EState`。
- 为什么 cursor/portal 可能让一个 snapshot 活得比普通语句长。
- 为什么 EPQ 会在并发 UPDATE/DELETE 后重检新版本，但不等于刷新整条语句的 snapshot。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

本节不展开：

- REPEATABLE READ 的事务级 snapshot 保存规则。
- SERIALIZABLE predicate lock。
- ActiveSnapshot / RegisteredSnapshot 的完整课程。
- 可更新 cursor 的全部语义。

## 1. 本节在总主线中的位置

上一节把 `SnapshotData` 的字段语义拆开了。本节把同一份结构放到默认隔离级别中。

READ COMMITTED 是 PostgreSQL 日常 workload 最常见的隔离级别。它的名字容易让人误解。

很多人把它理解成“每次读都看最新提交”。这句话不够精确。

PostgreSQL 的 READ COMMITTED 是语句级 snapshot。一条新语句开始时，通常会获取一个新的 MVCC snapshot。

这让它能看到此前已经提交的事务。但一旦语句把 snapshot 交给 executor，单条语句内部就不能随着并发 commit 变化。

否则同一次 scan 可能前半段看不到某个事务，后半段又看到它。聚合、排序、连接、UPDATE quals、RETURNING 和触发器都会变得无法解释。

因此本节只追一条主链路。客户端提交一个 SQL statement。

tcop/pquery 建立或运行 portal。READ COMMITTED 路径调用 `GetTransactionSnapshot()`。

snapshot 被压入 active stack 或注册到 query descriptor。`ExecutorStart()` 把 snapshot 注册到 `EState`。

`ExecutorRun()` 在这个 snapshot 下读写 tuple。语句结束或 portal 生命周期结束时释放 snapshot 引用。

并发 UPDATE 的 EPQ 只作为这条链路上的特殊重检点。它不把整个语句 snapshot 换掉。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
READ COMMITTED 在每个需要 query snapshot 的语句入口调用 GetTransactionSnapshot()；
tcop/pquery 把这份 snapshot 压成 active snapshot 或注册进 QueryDesc；
executor 在同一份 snapshot 下完成扫描、DML、trigger 和 RETURNING；
语句结束、portal 关闭或 executor cleanup 时释放引用。
```

本节的核心矛盾是：

```text
事务内部希望后一条语句能看到更新鲜的已提交数据；
但单条语句内部又必须使用同一个可见性世界，否则 scan、join、aggregation、UPDATE qual 和 RETURNING 会被并发 commit 撕裂。
```

所以 READ COMMITTED 的边界不是“每行最新”，也不是“整个事务固定”。

它的边界是 statement / portal / executor 生命周期共同形成的语句级 snapshot。

## 3. 核心文件分工与阅读顺序

| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/access/xact.h` | 先看 `IsolationUsesXactSnapshot()` 如何把 READ COMMITTED 分到语句级 snapshot。 |
| 2 | `src/backend/utils/time/snapmgr.c` | 再读 `GetTransactionSnapshot()` 的 READ COMMITTED 分支。 |
| 3 | `src/backend/tcop/pquery.c` | 看 portal 如何取得、压栈和释放语句 snapshot。 |
| 4 | `src/backend/executor/execMain.c` | 看 executor 如何要求并注册 query snapshot。 |
| 5 | `src/include/executor/execdesc.h` | 对照 `QueryDesc->snapshot` 的传递边界。 |
| 6 | `src/include/nodes/execnodes.h` | 对照 `EState->es_snapshot` 的执行期保存位置。 |
| 7 | `src/backend/executor/nodeModifyTable.c` | 最后看 DML 与 EvalPlanQual 的重检边界。 |
| 8 | `src/backend/utils/mmgr/portalmem.c` | 核对 portal 生命周期和 snapshot cleanup。 |

## 4. runtime 现象：新鲜但不撕裂

先用两个现象把边界定住。第一个现象是语句之间新鲜。

会话 A 修改一行并提交。会话 B 已经处在 READ COMMITTED 事务里。

B 的下一条语句能看到 A 的提交。这说明 READ COMMITTED 没有把第一个 snapshot 固定到整个事务结束。

第二个现象是语句内部稳定。B 执行一条带 `pg_sleep()` 的长查询。

A 在 B 查询运行期间提交新行。B 的这条查询不会半路把新行读进来。

B 的下一条查询才可能读到。这说明 READ COMMITTED 也不是每个 tuple 重新读取最新事务状态。

这两个现象看似矛盾。源码里的答案是两个边界：

语句开始时获取 snapshot。语句执行期间注册并固定 snapshot。

如果只记住“每条语句新 snapshot”，容易漏掉 executor/portal 的生命周期。如果只记住“单条语句稳定”，又容易把 READ COMMITTED 误写成 REPEATABLE READ。

本节要把这两个边界放到同一条调用链里。

## 5. 隔离级别宏决定 snapshot 复用级别

先看 `src/include/access/xact.h`。源码中有两个宏很关键。

`IsolationUsesXactSnapshot()` 等价于 `XactIsoLevel >= XACT_REPEATABLE_READ`。`IsolationIsSerializable()` 等价于 `XactIsoLevel == XACT_SERIALIZABLE`。

READ COMMITTED 的 `XactIsoLevel` 小于 REPEATABLE READ。所以 `IsolationUsesXactSnapshot()` 在 READ COMMITTED 下为 false。

这不是优化细节。它是本节的分水岭。

如果隔离级别使用 transaction snapshot，第一次 query snapshot 会被复制并保存到 `FirstXactSnapshot`。之后 `GetTransactionSnapshot()` 直接返回同一个 `CurrentSnapshot`。

如果隔离级别不使用 transaction snapshot，第一次调用会取 snapshot。之后每次调用也会重新取 snapshot。

READ COMMITTED 就走后者。因此“每条语句看新提交”的核心不是 executor 在扫描时刷新。

核心是每个需要 query snapshot 的语句进入执行前，会通过 `GetTransactionSnapshot()` 获得新的 `CurrentSnapshotData` 内容。这个函数返回的是指向静态 snapshot 存储的指针。

源码注释明确提醒调用者：

返回值会被未来调用和 `CommandCounterIncrement()` 修改。因此调用者在做非平凡工作之前，必须 `RegisterSnapshot()` 或 `PushActiveSnapshot()`。

这条注释解释了为什么 pquery 和 executor 里有看似重复的 snapshot 注册动作。它们不是装饰。

它们是语句内部稳定性的 ownership 边界。

## 6. `GetTransactionSnapshot()` 的 READ COMMITTED 分支

打开 `src/backend/utils/time/snapmgr.c`。`GetTransactionSnapshot()` 先处理 historic snapshot。

那是 logical decoding 相关边界。普通 SQL 不走那里。

然后它看 `FirstSnapshotSet`。事务内第一次 query snapshot 时，函数会先 `InvalidateCatalogSnapshot()`。

原因是 catalog snapshot 不能早于 xact snapshot。然后检查 parallel mode。

如果正在 parallel operation 中尝试拿 query snapshot，会报错。接着进入隔离级别分支。

READ COMMITTED 下 `IsolationUsesXactSnapshot()` 为 false。所以第一次调用执行：

`CurrentSnapshot = GetSnapshotData(&CurrentSnapshotData)`。然后设置 `FirstSnapshotSet = true`。

返回 `CurrentSnapshot`。同一事务中的第二次和之后调用，代码再次检查隔离级别。

READ COMMITTED 不会返回旧 `CurrentSnapshot`。它会再次 `InvalidateCatalogSnapshot()`。

然后执行：

`CurrentSnapshot = GetSnapshotData(&CurrentSnapshotData)`。最后返回新的 `CurrentSnapshot`。

这里的“新”有两个限制。第一，它重建的是静态 `CurrentSnapshotData` 的内容。

如果调用者没有注册或压栈，后续调用可能覆盖它。第二，它新鲜到调用时 ProcArray 和事务完成状态所能表达的边界。

它不是查询执行期间持续变化的 live view。`GetSnapshotData()` 会扫描 ProcArray，填入 `xmin/xmax/xip/subxip`。

之后 executor 使用的是这份复制结果。这就是 READ COMMITTED 的语句级语义来源。

## 7. portal 是语句 snapshot 进入 executor 的门

PostgreSQL 不会让 SQL 直接跳进 executor。tcop/pquery 通过 portal 管理执行。

本节关注三个入口。`PortalStart()` 为 portal 准备执行状态。

`PortalRun()` 运行 portal。`PortalRunSelect()` 和 `PortalRunMulti()` 分别处理 SELECT-like 和一般多 query 情况。

对 `PORTAL_ONE_SELECT`，`PortalStart()` 先压 active snapshot。如果调用者传入 snapshot，就使用传入的 snapshot。

否则调用 `GetTransactionSnapshot()`。然后它用 `GetActiveSnapshot()` 创建 `QueryDesc`。

`CreateQueryDesc()` 会对传入 snapshot 调 `RegisterSnapshot()`。接着 `PortalStart()` 调 `ExecutorStart(queryDesc, myeflags)`。

`ExecutorStart()` 要求 caller 已经把 query snapshot 设为 active。完成启动后，`PortalStart()` 把 `queryDesc` 保存到 `portal->queryDesc`。

然后 `PopActiveSnapshot()`。

注意这里不是释放 query snapshot。`QueryDesc` 已经注册了它。

之后 `PortalRunSelect()` 每次执行或 fetch 时，会再次 `PushActiveSnapshot(queryDesc->snapshot)`。再调用 `ExecutorRun()`。

执行完再 `PopActiveSnapshot()`。这就是“语句内部稳定”的 portal 形态。

snapshot 在 `PortalStart()` 时创建并注册到 `QueryDesc`。`ExecutorRun()` 期间它被压为 active。

多次 fetch 同一个 portal 时，仍使用同一个 `queryDesc->snapshot`。因此 cursor 的边界和普通一次性 SELECT 不完全相同。

普通 SELECT 很快跑完。cursor 的 portal 可以让 snapshot 活到后续 FETCH。

## 8. `CreateQueryDesc()` 和 `ExecutorStart()` 的双重注册

`CreateQueryDesc()` 定义在 `src/backend/tcop/pquery.c`。它把 planned statement、source text、snapshot、dest receiver、参数和 instrumentation 组合成 `QueryDesc`。

其中一行很关键：

`qd->snapshot = RegisterSnapshot(snapshot)`。`crosscheck_snapshot` 也通过同样机制注册。

`FreeQueryDesc()` 会 `UnregisterSnapshot(qdesc->snapshot)`。这保证 `QueryDesc` 持有 snapshot 时，snapshot 不会被静态工作区覆盖或释放。

`ExecutorStart()` 定义在 `src/backend/executor/execMain.c`。`standard_ExecutorStart()` 开头断言：

`GetActiveSnapshot() == queryDesc->snapshot`。这条断言说明 executor 启动时，调用者必须已经把 query snapshot 放在 active stack 顶部。

然后 executor 创建 `EState`。它在 per-query memory context 中初始化参数、range table、权限和执行状态。

真正进入 executor 可见性边界的是：

`estate->es_snapshot = RegisterSnapshot(queryDesc->snapshot)`。`estate->es_crosscheck_snapshot = RegisterSnapshot(queryDesc->crosscheck_snapshot)`。

`ExecutorEnd()` 会 unregister 这两个 snapshot。这看起来像重复注册。

其实它们对应两个 owner。`QueryDesc` 有自己的生命周期。

`EState` 有 executor 生命周期。portal 持有 `QueryDesc`。

executor 运行期间持有 `EState`。ERROR 或资源 cleanup 时，ResourceOwner 和 snapshot manager 必须能正确释放每一层引用。

这也是为什么不能把 READ COMMITTED 简化成“函数返回一个 pointer 给 executor 用”。这个 pointer 的稳定性靠注册和 active stack 维护。

## 9. `PortalRunMulti()`：一个 statement 内的多 PlannedStmt 边界

`PortalRunMulti()` 的注释说，它处理一个 portal 中的 query list。这里容易误读。

这个 list 通常来自一个 parse tree 经 analysis/rewrite 生成的多个 `PlannedStmt`。它不应该被理解成客户端一次发送的所有分号语句共享同一个 snapshot。

对 plannable query，`PortalRunMulti()` 第一次遇到需要 snapshot 的 planned statement 时调用 `GetTransactionSnapshot()`。如果需要 hold snapshot，就注册并放到 `portal->holdSnapshot`。

然后它调用 `PushCopiedSnapshot(snapshot)`。后续同一个 portal 内的 planned statement 不重新取 snapshot。

而是调用 `UpdateActiveSnapshotCommandId()`。这说明一个 SQL statement 内部 rewrite 出来的多个执行单元仍共享语句级 snapshot。

命令之间的可见性变化通过 command id 推进。不是通过重新获取 ProcArray snapshot。

`ProcessQuery()` 会用 `GetActiveSnapshot()` 创建 `QueryDesc`。然后 `ExecutorStart()`、`ExecutorRun()`、`ExecutorFinish()`、`ExecutorEnd()` 按 executor 生命周期工作。

最后 `PortalRunMulti()` 在本 portal 的 plannable query 结束后 `PopActiveSnapshot()`。这段代码回答一个边界问题：

READ COMMITTED 的 statement 不是 executor 节点。也不是每个 rewrite 产物。

它是 PostgreSQL 执行框架中一个 portal/statement 边界。同一条 statement 内部的多个执行步骤不能各自看到不同的并发提交。

否则 rewrite 生成的辅助操作和主操作之间会出现不可解释的可见性裂缝。

## 10. utility 语句和 snapshot

不是所有 utility statement 都需要 MVCC snapshot。`PlannedStmtRequiresSnapshot()` 决定 utility 语句是否需要 snapshot。

`PortalRunUtility()` 在需要时调用 `GetTransactionSnapshot()`。如果这是 holdable 场景，还会 `RegisterSnapshot()` 并放入 `portal->holdSnapshot`。

随后它调用 `PushActiveSnapshotWithLevel(snapshot, portal->createLevel)`。并把 `portal->portalSnapshot` 指向当前 active snapshot。

然后执行 `ProcessUtility()`。执行后，如果 active snapshot 仍在，就弹出。

源码注释提到某些 utility 命令可能从内部弹掉 active snapshot。因此 cleanup 不能简单假设栈顶一定还在。

这段逻辑说明：

READ COMMITTED 的语句级 snapshot 不是只服务 SELECT。某些 utility command 如果需要查询 catalog 或执行表达式，也需要一个 statement snapshot。

但 transaction control、LOCK、SET 等命令不能强行冻结 transaction snapshot。这就是 `PlannedStmtRequiresSnapshot()` 注释中特别列出的边界。

课程里不要把所有 SQL command 都写成“先拿 snapshot”。正确说法是：

需要 query snapshot 的 statement 在进入实际执行前拿 snapshot。不需要 snapshot 的 utility statement 不走这个路径。

## 11. 单条 SELECT 的主流程 walkthrough

把普通 SELECT 串起来：

```text
客户端提交 SELECT
  -> parser/analyzer/planner 形成 PlannedStmt
  -> PortalStart()
     -> PushActiveSnapshot(GetTransactionSnapshot())
     -> CreateQueryDesc(..., GetActiveSnapshot(), ...)
        -> RegisterSnapshot(snapshot)
     -> ExecutorStart(queryDesc)
        -> Assert(GetActiveSnapshot() == queryDesc->snapshot)
        -> estate->es_snapshot = RegisterSnapshot(queryDesc->snapshot)
     -> PopActiveSnapshot()
  -> PortalRun()
     -> PortalRunSelect()
        -> PushActiveSnapshot(queryDesc->snapshot)
        -> ExecutorRun(queryDesc, direction, count)
        -> PopActiveSnapshot()
  -> ExecutorFinish()/ExecutorEnd()/FreeQueryDesc()/PortalDrop()
     -> UnregisterSnapshot()
```

在 READ COMMITTED 下，`GetTransactionSnapshot()` 这次调用通常会重建 `CurrentSnapshotData`。之后 `QueryDesc` 和 `EState` 注册它。

executor 节点通过 `estate->es_snapshot` 做 table scan、join、qual recheck 和 ModifyTable。heap access method 最终调用 `HeapTupleSatisfiesMVCC()`。

它看到的是同一个 statement snapshot。

这条链路上没有“扫描一半重新 `GetSnapshotData()`”的步骤。如果扫描期间别的事务提交，新事务结果可以影响下一条语句。

不会影响已经进入 executor 的这条语句。

## 12. 可复现 SQL 现象一：语句之间看见新提交

会话 A：

```sql
DROP TABLE IF EXISTS rc08;
CREATE TABLE rc08(id int primary key, v text);
INSERT INTO rc08 VALUES (1, 'old');
BEGIN;
INSERT INTO rc08 VALUES (2, 'from A');
-- 暂不提交。
```

会话 B：

```sql
BEGIN ISOLATION LEVEL READ COMMITTED;
SELECT array_agg(id ORDER BY id) FROM rc08;
-- 这里通常只看到 {1}。
```

现在会话 A 执行：

```sql
COMMIT;
```

会话 B 再执行：

```sql
SELECT array_agg(id ORDER BY id) FROM rc08;
COMMIT;
```

B 的第二条 SELECT 通常能看到 `{1,2}`。源码解释：

第一条 SELECT 调用 `GetTransactionSnapshot()`，A 的 XID 仍 running。第二条 SELECT 再次调用 `GetTransactionSnapshot()`。

READ COMMITTED 不走 transaction snapshot 复用分支。这次 A 的 XID 已经完成，不再作为 running XID 进入新 snapshot。

因此 heap visibility 可以把 A 插入的 tuple 解释为已提交且对新 statement 可见。如果把 B 改成 REPEATABLE READ，第二条 SELECT 不会看到 `{2}`。

因为 `IsolationUsesXactSnapshot()` 为 true，第一次 snapshot 会被保存到事务结束。这个对照可以确认：本节讨论的是 READ COMMITTED 的 statement boundary。

## 13. 可复现 SQL 现象二：单条语句内部稳定

会话 A 先准备数据：

```sql
DROP TABLE IF EXISTS rc08_slow;
CREATE TABLE rc08_slow(id int primary key);
INSERT INTO rc08_slow VALUES (1);
```

会话 B 执行：

```sql
BEGIN ISOLATION LEVEL READ COMMITTED;
SELECT count(*)
FROM rc08_slow
WHERE pg_sleep(5) IS NULL OR true;
```

在 B 的查询睡眠期间，会话 A 执行：

```sql
INSERT INTO rc08_slow VALUES (2);
COMMIT;
```

B 的这条查询不会在同一次 statement 中突然把 `id = 2` 算进去。B 随后再执行：

```sql
SELECT count(*) FROM rc08_slow;
COMMIT;
```

这次才可能看到 2 行。源码解释：

B 第一条 SELECT 进入 executor 前已经把 snapshot 注册进 `QueryDesc` 和 `EState`。`ExecutorRun()` 扫描时使用 `estate->es_snapshot`。

`HeapTupleSatisfiesMVCC()` 对 `id = 2` 的插入 XID 会按旧 snapshot 判断。如果该 XID 在旧 snapshot 的 `xmax` 之后，或按旧 snapshot still-running，就不可见。

提交发生在 statement 开始之后。它不会修改 B 的 `estate->es_snapshot`。

## 14. cursor/portal 生命周期：snapshot 可能长于普通语句

普通 SELECT 的 portal 很快完成并释放。cursor 会把 portal 生命周期拉长。

`PORTAL_ONE_SELECT` 在 `PortalStart()` 时创建 `QueryDesc`。snapshot 被注册进 `QueryDesc`。

之后 `PortalRunSelect()` 每次 FETCH 都把 `queryDesc->snapshot` 压为 active。这意味着一个非 holdable cursor 在同一事务内多次 FETCH 时，通常使用 cursor 启动时的 snapshot。

它不是每次 FETCH 都调用 `GetTransactionSnapshot()`。这点对 READ COMMITTED 很重要。

READ COMMITTED 的“每条语句新 snapshot”不能机械套到“每次 FETCH 都新 snapshot”。cursor 的 query 是在 portal 启动时绑定 snapshot 的。

可以复现实验。会话 A：

```sql
DROP TABLE IF EXISTS rc08_cursor;
CREATE TABLE rc08_cursor(id int primary key);
INSERT INTO rc08_cursor VALUES (1);
BEGIN;
DECLARE c CURSOR FOR SELECT id FROM rc08_cursor ORDER BY id;
FETCH 1 FROM c;
```

会话 B：

```sql
INSERT INTO rc08_cursor VALUES (2);
COMMIT;
```

回到会话 A：

```sql
FETCH ALL FROM c;
SELECT id FROM rc08_cursor ORDER BY id;
COMMIT;
```

后续 FETCH 不应该把 B 后提交的 `id = 2` 当作 cursor 查询的一部分。同一事务里新的普通 SELECT 则可以在 READ COMMITTED 下看到它。

源码对应：

cursor 的 `queryDesc->snapshot` 在 `PortalStart()` 时注册。FETCH 时 `PortalRunSelect()` 重用这个 snapshot。

普通 SELECT 会走新的 portal 和新的 `GetTransactionSnapshot()`。

## 15. holdable cursor：快照变成物化结果边界

`WITH HOLD` cursor 又多一层边界。持有到事务外的 cursor 不能继续依赖创建事务的 locks、snapshot 和 executor state。

`PreCommit_Portals()` 会在提交前处理 portals。对于当前事务创建的 holdable cursor，它调用 `HoldPortal()`。

`HoldPortal()` 创建 hold store，并调用 `PersistHoldablePortal()`。结果被物化到 tuplestore。

之后释放 cached plan reference。portal 不再属于创建事务。

后续事务 FETCH 的是 `holdStore`。`PortalRunSelect()` 看到 `portal->holdStore` 后走 `RunFromStore()`。

这时不再运行原 executor scan。也不重新拿创建语句的 MVCC snapshot。

这个边界解释了两个现象。`WITH HOLD` cursor 提交后还能 FETCH。

但它读的是提交前物化出的结果集。它不是在每次 FETCH 时用新的 READ COMMITTED snapshot 重新执行原 SELECT。

`PreCommit_Portals()` 还拒绝 `PREPARE TRANSACTION` 与 cursor WITH HOLD 的组合。源码里会报 `cannot PREPARE a transaction that has created a cursor WITH HOLD`。

这说明 holdable portal 的语义和事务边界耦合很深。课程里不要把 cursor 简化成“另一条 SELECT”。

portal 生命周期本身就是 snapshot 生命周期的一部分。

## 16. EPQ 的边界：重检行，不刷新整条语句

READ COMMITTED 下 UPDATE/DELETE/MERGE 还会遇到并发修改。典型场景：

会话 A 更新一行但暂不提交。会话 B 在 READ COMMITTED 下也要更新同一行。

B 会等待 A 的结果。A 提交后，B 不能直接用旧版本继续执行。

它需要考虑 A 提交后的最新行版本是否仍满足 B 的 WHERE 条件。这就是 EvalPlanQual，简称 EPQ。

`nodeModifyTable.c` 中，当 `table_tuple_update()`、`table_tuple_delete()` 或相关 lock 路径返回 `TM_Updated` 时，会区分隔离级别。如果 `IsolationUsesXactSnapshot()` 为 true，REPEATABLE READ / SERIALIZABLE 通常报 serialization failure。

如果是 READ COMMITTED，则尝试锁定最新版本。相关 `table_tuple_lock()` 调用会传入 `estate->es_snapshot`、`estate->es_output_cid` 和 `TUPLE_LOCK_FLAG_FIND_LAST_VERSION`。

锁到最新版本后，调用 `EvalPlanQual()` 重跑必要的 qual。EPQ 子执行状态在 `execMain.c`。

`EvalPlanQualStart()` 创建一个 cut-down EState。它会复制 parent estate 的不变状态。

其中包括：

`rcestate->es_snapshot = parentestate->es_snapshot` `rcestate->es_crosscheck_snapshot = parentestate->es_crosscheck_snapshot`

`rcestate->es_output_cid = parentestate->es_output_cid`这说明 EPQ 没有创建一个全新的 statement snapshot。

它共享父语句的 snapshot。EPQ 的特殊性在于 target row 的最新版本被锁定、放入 EPQ slot，并在同一语句执行语义下重检 quals。

它解决的是 concurrent update 下“这行现在还是否满足我的语句条件”。它不是把整条查询切换到 A 提交之后的新世界。

这点是 READ COMMITTED 最容易写错的地方。

## 17. 可复现 SQL 现象三：EPQ 重检

会话 A：

```sql
DROP TABLE IF EXISTS rc08_epq;
CREATE TABLE rc08_epq(id int primary key, v int);
INSERT INTO rc08_epq VALUES (1, 10);
BEGIN;
UPDATE rc08_epq SET v = 20 WHERE id = 1;
-- 暂不提交。
```

会话 B：

```sql
BEGIN ISOLATION LEVEL READ COMMITTED;
UPDATE rc08_epq SET v = v + 1 WHERE id = 1 AND v = 10;
```

B 会等待 A。现在会话 A：

```sql
COMMIT;
```

B 继续后，通常更新 0 行。因为最新版本 `v = 20` 不再满足 `v = 10`。

会话 B：

```sql
SELECT * FROM rc08_epq;
COMMIT;
```

源码解释：

B 语句开始时的 snapshot 可能看见旧版本。真正更新时发现目标 tuple 已被并发事务更新。

READ COMMITTED 路径不报 serialization failure。它锁定最新版本并用 EPQ 重检 WHERE 条件。

条件失败，所以不更新。如果把 B 的事务改为 REPEATABLE READ，类似并发 update 场景会走 serialization failure 边界。

因为 transaction snapshot isolation 不允许 READ COMMITTED 这种行级追随最新版本的行为。注意 EPQ 只重检相关行。

它不是重新执行整个 SELECT 并获得新的全局 snapshot。

## 18. snapshot、command id 与 DML 输出

`ExecutorStart()` 对 DML 还会设置 output command id。对 `CMD_INSERT`、`CMD_DELETE`、`CMD_UPDATE`、`CMD_MERGE`，它调用 `GetCurrentCommandId(true)`。

对 SELECT FOR UPDATE/SHARE 或 modifying CTE，也可能需要 output CID。这个 CID 写入 tuple header 的 `cmin` 或 `cmax`。

同一 statement 的 active snapshot 有自己的 `curcid`。上一节已经讲过，`HeapTupleSatisfiesMVCC()` 用 `curcid` 判断当前事务自己的写入和删除。

READ COMMITTED 里每条新语句重取 snapshot 时，`GetSnapshotData()` 会设置新的 `curcid`。`CommandCounterIncrement()` 在命令之间推进 command id。

所以后一条语句可以看到前一条语句的写入。但同一条语句内部的基表扫描不会因为自己刚写了 tuple 就突然看到它。

这就是为什么数据修改 CTE、RETURNING、触发器和基础表扫描需要特别小心。它们共享同一条语句的 snapshot/command boundary。

返回流能看到的东西，不一定等价于重新扫描基表能看到的东西。

## 19. ERROR / abort 时谁兜底

READ COMMITTED snapshot 的正确性不只在 happy path。如果 executor 启动后出错，`ExecutorEnd()` 未必完整走到普通结尾。

PostgreSQL 依靠 ResourceOwner、portal cleanup 和 transaction abort cleanup 收尾。`QueryDesc` 注册的 snapshot 在 `FreeQueryDesc()` 中 unregister。

`EState` 注册的 snapshot 在 `ExecutorEnd()` 中 unregister。如果事务 abort，ResourceOwner release 会释放附着的 snapshot 引用。

`portalmem.c` 中 `PortalDrop()` 还会处理 `portal->holdSnapshot`。如果 portal failed，或者普通 drop，要释放相关 resowner、tuplestore 和 memory context。

`PortalDrop()` 先从 hash table 删除 portal。源码注释说明这样做是为了后续 cleanup 出错时避免无限 error recovery loop。

`AtCleanup_Portals()` 处理事务 abort 后仍属于当前事务的 portals。如果 portal 仍 pinned，会强制 unpin。

如果 cleanup hook 没跑，会 warning 并跳过用户代码。然后 drop portal。

`AtSubCleanup_Portals()` 处理子事务 abort 创建的 portal。这说明 snapshot 生命周期不靠“函数自然返回”保证。

它靠 ResourceOwner、portal owner、active stack 和 cleanup 路径共同保证。如果课程只写 `PopActiveSnapshot()`，就会漏掉 ERROR 边界。

## 20. `EnsurePortalSnapshotExists()` 的特殊边界

`pquery.c` 里还有 `EnsurePortalSnapshotExists()`。它用于在某些场景下重新建立 portal-level snapshot。

注释说明，通常执行 SQL 时会有 active snapshot。但某些允许事务控制的过程或 utility command 可能提交/回滚，从而销毁所有 snapshot。

如果之后还需要在 portal 中执行 SQL，就必须重新建立一个 outer snapshot。函数在没有 active snapshot 时，要求存在 active portal。

否则报错：

`cannot execute SQL without an outer snapshot or portal`然后它调用 `GetTransactionSnapshot()`。

用 `PushActiveSnapshotWithLevel(..., portal->createLevel)` 压栈。并把 `portal->portalSnapshot` 指到 active snapshot。

这个函数不是 READ COMMITTED 主路径。但它说明 portal snapshot 是执行框架的显式资源。

当事务边界在过程内部发生变化时，系统不能假装旧 snapshot 还在。它必须重新建立和 portal 层级匹配的 snapshot。

## 21. 成本模型

READ COMMITTED 的成本来自频繁创建 statement snapshot。每条需要 query snapshot 的语句都可能调用 `GetSnapshotData()`。

如果 `GetSnapshotDataReuse()` 条件成立，可以减少 ProcArray 扫描。但高并发短事务下，事务完成计数常常变化。

snapshot reuse 不一定命中。ProcArray scan 成本随 backend 数增长。

同时它会和事务结束路径共享 `ProcArrayLock` 边界。单条长查询虽然只取一次 snapshot。

但它会让 `MyProc->xmin` 保持较老。这会拖住 cleanup horizon。

所以 READ COMMITTED 不等于不会造成 bloat。长 statement、慢 cursor、长时间不 fetch 的 cursor 都可能持有旧 snapshot。

cursor 成本还包括 portal 生命周期。非 holdable cursor 可以保留 executor state 和 snapshot 到事务结束或 close。

holdable cursor 会在 commit 前物化结果。物化可能消耗内存、temp file 和 I/O。

EPQ 成本来自并发冲突。遇到 `TM_Updated` 后，需要等待、锁定最新版本、构造 EPQ slot，并重跑部分 qual。

复杂 join 或 MERGE 场景下，EPQ 可能比普通 UPDATE path 贵得多。这些成本都服务同一个语义目标：

语句之间尽量新鲜。语句内部保持可解释。

## 22. 观测与诊断入口

第一类观测是 snapshot 边界。在 SQL 中使用 `pg_current_snapshot()`、`pg_snapshot_xmin()`、`pg_snapshot_xmax()`、`pg_snapshot_xip()`。

它能展示 READ COMMITTED 下不同语句 snapshot 的变化。它不能展示 `curcid`、active stack、registered refcount 或 `EState` 引用。

第二类观测是长 snapshot 对 cleanup 的影响。查 `pg_stat_activity.backend_xmin`。

如果某个 backend 长时间持有旧 `backend_xmin`，它可能拖住 VACUUM。但这只是 horizon 输入之一。

还要考虑 replication slot、prepared transaction、catalog xmin 等。第三类观测是 cursor。

查 `pg_cursors` 可以看到当前 session 的 cursor 信息。它不能直接告诉你内部 `QueryDesc->snapshot` 内容。

但可以帮助确认是否存在长时间未关闭的 cursor。第四类观测是并发 update wait 和 EPQ。

`pg_stat_activity.wait_event_type` 可能显示 Lock。`wait_event` 可能落在 transactionid 或 tuple 相关等待上，取决于具体冲突点。

`pg_locks` 可以辅助确认事务 ID lock 或 tuple/page/relation lock。EPQ 是否发生通常不能直接从普通系统视图看到。

可以用 `EXPLAIN (ANALYZE, BUFFERS)` 观察 update 行数、过滤、触发器和 buffer 行为。更直接的方法是在 `nodeModifyTable.c` 的 `TM_Updated` 分支、`EvalPlanQual()`、`table_tuple_lock()` 打断点。

第五类观测是 portal cleanup。如果怀疑 cursor 或 portal 泄漏，先查应用是否显式关闭 cursor。

再看事务是否长时间 idle in transaction。最后才进入 `PortalDrop()`、`PreCommit_Portals()` 和 `AtCleanup_Portals()` 调试。

## 23. 常见误区

误区一：

READ COMMITTED 每行都看最新提交。正确说法是每条需要 query snapshot 的语句开始时获取新的 snapshot，语句内部稳定。

误区二：

READ COMMITTED 一个事务内只有一个 snapshot。这是 REPEATABLE READ / SERIALIZABLE 的方向。

READ COMMITTED 下非 transaction-snapshot 分支会重复调用 `GetSnapshotData()`。误区三：

FETCH cursor 等同于一条新 SELECT。cursor 的 query snapshot 可能在 `PortalStart()` 时已经绑定。

FETCH 只是继续运行或读取该 portal。误区四：

EPQ 刷新了整个语句 snapshot。EPQ 重检并发更新后的目标行。

它的 child EState 共享 parent `es_snapshot`。误区五：

`GetTransactionSnapshot()` 返回的指针可以一直保存。返回值指向静态 storage，可能被未来调用和 command counter 修改。

调用者必须注册或压栈。误区六：

看到 `backend_xmin` 就能知道具体语句看见哪些行。`backend_xmin` 是 cleanup horizon 相关公开状态。

不是完整 `SnapshotData`。误区七：

WITH HOLD cursor 提交后还会用 READ COMMITTED 新 snapshot 重新执行查询。它提交前物化结果。

后续 FETCH 读 tuplestore。

## 24. 课堂实验

实验一：对比 READ COMMITTED 与 REPEATABLE READ。用第 12 节的两会话脚本。

先在 B 中使用 READ COMMITTED。再改成 REPEATABLE READ。

观察第二条 SELECT 是否看到 A 的提交。回到 `IsolationUsesXactSnapshot()` 和 `GetTransactionSnapshot()` 解释差异。

实验二：确认单语句稳定。用第 13 节的 `pg_sleep()` 查询。

在 `GetTransactionSnapshot()`、`PortalStart()`、`ExecutorRun()` 打断点。确认长查询执行期间不会再次进入 `GetSnapshotData()`。

实验三：确认 cursor snapshot。用第 14 节 cursor 脚本。

在 `PortalStart()` 和 `PortalRunSelect()` 打断点。观察 cursor 声明时的 `queryDesc->snapshot`。

后续 FETCH 时确认它被 `PushActiveSnapshot(queryDesc->snapshot)` 重用。实验四：确认 EPQ。

用第 17 节并发 UPDATE 脚本。在 `nodeModifyTable.c` 的 `TM_Updated` 分支打断点。

继续到 `EvalPlanQual()`。观察 `EvalPlanQualStart()` 中 child EState 的 `es_snapshot` 指向 parent `EState` 的 snapshot。

实验五：确认 holdable cursor materialization。声明 `DECLARE c CURSOR WITH HOLD FOR SELECT ...`。

提交前后在 `PreCommit_Portals()`、`HoldPortal()`、`PersistHoldablePortal()` 打断点。确认提交后 FETCH 走 `holdStore`，不是重新执行原 query。

## 25. 讨论题

为什么 READ COMMITTED 的“新鲜”边界不能细到每个 tuple？为什么 `GetTransactionSnapshot()` 的返回值必须由调用者注册或压栈？

`PortalStart()` 已经 push 过 snapshot，为什么 `PortalRunSelect()` 运行时还要 push？为什么同一个 cursor 的后续 FETCH 不应该简单理解成新语句 snapshot？

EPQ 为什么在 READ COMMITTED 下继续执行，而在 REPEATABLE READ 下可能报 serialization failure？`EvalPlanQualStart()` 共享 parent `es_snapshot` 说明了什么边界？

一个长时间 idle in transaction 的 READ COMMITTED 会话为什么仍可能拖住 VACUUM？WITH HOLD cursor 为什么需要物化结果，而不是保留创建事务的 snapshot 到后续事务？

## 26. 一次 READ COMMITTED UPDATE 的时间线复盘

把 SELECT、UPDATE 和 EPQ 放进同一条时间线。会话 A 已经提交一行 `id = 1, v = 10`。

会话 B 开始 READ COMMITTED 事务。B 执行：

`UPDATE t SET v = v + 1 WHERE id = 1 AND v = 10`。这条语句进入 `GetTransactionSnapshot()`。

因为是 READ COMMITTED，函数重建 `CurrentSnapshotData`。`PortalRunMulti()` 或对应 query path 把 snapshot 压入 active stack。

`ProcessQuery()` 创建 `QueryDesc`。`CreateQueryDesc()` 注册 snapshot。

`ExecutorStart()` 再把 snapshot 注册到 `EState`。此时这条 UPDATE 的基础可见性已经固定。

executor 扫描到 `id = 1`。`HeapTupleSatisfiesMVCC()` 按 `estate->es_snapshot` 看到旧版本满足条件。

如果没有并发冲突，ModifyTable 会更新它。现在加入会话 C。

C 在 B 更新前抢先更新同一行，并提交成 `v = 20`。B 在尝试更新时发现 tuple 已被更新。

heap access method 返回 `TM_Updated`。READ COMMITTED 路径不会直接报 serialization failure。

`nodeModifyTable.c` 进入并发更新处理。它锁定最新版本。

锁定时传入的仍是 `estate->es_snapshot` 和 `estate->es_output_cid`。`TUPLE_LOCK_FLAG_FIND_LAST_VERSION` 允许它追到最新版本。

然后 B 启动 EPQ。`EvalPlanQual()` 把最新 tuple 放进 EPQ slot。

EPQ 子 EState 复制 parent 的 `es_snapshot`。它不会调用 `GetTransactionSnapshot()`。

接着重新计算 WHERE qual。如果最新版本 `v = 20` 不满足 `v = 10`，B 更新 0 行。

如果最新版本仍满足条件，B 可以继续更新最新版本。这条时间线说明 READ COMMITTED 有两个不同层次的新鲜性。

语句级 snapshot 决定普通扫描世界。EPQ 决定并发修改目标行的最新版本是否仍符合本语句。

EPQ 不是全表重新扫描。也不是新 snapshot。

它是 target-row-level 的补偿机制。这个补偿机制让 READ COMMITTED 能处理常见并发 UPDATE。

同时避免一条语句内部任意节点看到不同全局事务集合。如果隔离级别变成 REPEATABLE READ，`IsolationUsesXactSnapshot()` 为 true。

同样 `TM_Updated` 场景通常转成 serialization failure。这是因为事务级 snapshot 不允许这种行级追随最新版本的语义。

## 27. portal 状态与 snapshot owner 对照

READ COMMITTED 课程经常写错，是因为把 snapshot 当成一个临时变量。在源码里它有多个 owner。

第一个 owner 是 snapshot manager 的静态工作区。`CurrentSnapshotData` 可以被下一次 `GetTransactionSnapshot()` 改写。

裸指针只适合立刻交给注册或 active stack。第二个 owner 是 active snapshot stack。

`PushActiveSnapshot()` 或 `PushCopiedSnapshot()` 表示当前执行路径正在使用这个 snapshot。`GetActiveSnapshot()` 依赖这个栈。

`ExecutorStart()` 明确断言 query snapshot 是 active 的。第三个 owner 是 `QueryDesc`。

`CreateQueryDesc()` 注册 snapshot。这让 portal 可以在 `PortalStart()` 弹出 active snapshot 后仍然持有 query snapshot。

第四个 owner 是 `EState`。`ExecutorStart()` 把 `queryDesc->snapshot` 再注册到 `estate->es_snapshot`。

这让 executor 生命周期有自己的引用。第五个 owner 是 portal 的 hold snapshot。

`PortalRunMulti()` 或 `PortalRunUtility()` 在 holdable 场景下会把 snapshot 存入 `portal->holdSnapshot`。这个引用属于 portal 的 ResourceOwner。

第六个 owner 是 holdStore。一旦 WITH HOLD cursor 被物化，后续 FETCH 的核心资源不再是原 query snapshot，而是 tuplestore。

这些 owner 的边界解释了 cleanup 顺序。普通 SELECT 结束后，executor unregister `EState` snapshot。

`FreeQueryDesc()` unregister `QueryDesc` snapshot。portal drop 删除 portal context。

cursor 没有关闭时，portal 保留 `QueryDesc` 和 snapshot。commit 时，非 holdable 当前事务 portal 被 drop。

holdable portal 被 `HoldPortal()` 转成可跨事务读取的 materialized store。abort 时，`AtCleanup_Portals()` 和 ResourceOwner release 兜底。

如果你在源码中看到同一个 snapshot 多次注册，不要急着当成重复工作。先问这次注册属于哪个 owner。

owner 不同，ERROR 路径也不同。这就是内核代码愿意承担 refcount 复杂度的原因。

没有这些边界，READ COMMITTED 的语句级稳定性在 ERROR、cursor 和 holdable portal 下都会漏。

## 28. 为什么 READ COMMITTED 不能每个 executor node 重取 snapshot

设想每个 scan node 或每次 `ExecProcNode()` 都调用 `GetTransactionSnapshot()`。看起来这样最“新鲜”。

但语义会立即崩坏。一个 hash join 的 build side 和 probe side 可能看到不同并发提交集合。

一个 aggregate 的前半段和后半段可能不在同一数据集上。一个 UPDATE 的 WHERE 条件和 RETURNING 可能基于不同世界。

一个 statement 内的 trigger 和主 DML 可能互相看见不一致状态。一个 cursor 的前几次 FETCH 和后几次 FETCH 会变成不同查询。

这会破坏用户对“一条 SQL statement”的基本理解。也会让 executor 无法做许多本地假设。

例如 plan node 可以缓存 tuple slot、参数、qual 结果和 scan position。这些状态都默认同一 executor invocation 处在同一可见性视角下。

如果 snapshot 在 executor 内部自由变化，许多节点需要重新定义语义。成本也会爆炸。

每次重取 snapshot 都可能扫描 ProcArray。每个 tuple 再问全局状态，会把 ProcArray 和 CLOG 压成 hot bottleneck。

PostgreSQL 的选择是：

在 statement boundary 处支付一次 snapshot 获取成本。在 executor 内部使用注册后的稳定 snapshot。

并用 EPQ 处理并发更新目标行这个特殊问题。这个选择不是最强新鲜度。

它是默认隔离级别在可解释语义和工程成本之间的平衡。

## 29. 版本、workload 与推断边界

本节基于 `/home/highgo/postgres` 当前分支。核心边界相对稳定。

READ COMMITTED 不使用 transaction snapshot。`GetTransactionSnapshot()` 会在非 transaction-snapshot isolation 下重新取 snapshot。

executor 使用 active/registered snapshot。EPQ 在并发 UPDATE/DELETE 下重检目标行。

但实现细节会随版本变化。Portal strategy、executor instrumentation、EPQ 代码路径、snapshot reuse 优化和 ResourceOwner 细节都可能演进。

诊断时不要只凭函数名。要确认当前版本里调用链是否仍落在同一 owner 边界。

workload 也会改变现象。短小 SELECT 下，snapshot 获取成本可能比 tuple visibility 更显眼。

长扫描下，单条 statement 持有旧 snapshot 的时间可能主导 bloat 风险。大量 cursor 或未关闭 portal 会让 snapshot 生命周期超过普通语句直觉。

高冲突 UPDATE 下，EPQ 和 row lock wait 可能主导延迟。WITH HOLD cursor 会把成本从 MVCC snapshot 转成 tuplestore、内存和临时文件。

系统视图也有粒度限制。`pg_current_snapshot()` 看到的是 SQL 层 snapshot 投影。

`pg_stat_activity.backend_xmin` 看到的是 cleanup 相关公开 xmin。`pg_locks` 看到的是等待和锁。

没有一个视图能直接告诉你“这个 executor 的 `EState->es_snapshot` 是什么时候注册的”。必要时仍要断点或临时日志。

断点顺序也要按生命周期放置。先断 `GetTransactionSnapshot()`，确认本语句是否真的重新取 snapshot。

再断 `CreateQueryDesc()`，确认哪个 snapshot 被注册到 query descriptor。再断 `standard_ExecutorStart()`，确认 active snapshot 与 `queryDesc->snapshot` 是否一致。

最后断 `ExecutorEnd()` 或 `PortalDrop()`，确认引用在哪里释放。如果一开始就断 `HeapTupleSatisfiesMVCC()`，你只能看到消费现场。

很难反推出这个 snapshot 是普通 statement、cursor、holdable cursor 还是 EPQ 子执行状态带来的。

## 30. 与 REPEATABLE READ 的精确分界

本节的结尾要提前划清下一节边界。READ COMMITTED 和 REPEATABLE READ 都使用 `SnapshotData`。

两者不是字段结构不同。差异在于 snapshot 何时创建、何时复用、复用到什么范围。

READ COMMITTED 下，事务内第一次调用 `GetTransactionSnapshot()` 会设置 `FirstSnapshotSet`。但这不表示后续都复用同一 snapshot。

因为 `IsolationUsesXactSnapshot()` 为 false。所以后续需要 query snapshot 时，会再次 `GetSnapshotData()`。

REPEATABLE READ 下，第一次调用会复制 `CurrentSnapshotData`。复制结果保存到 `FirstXactSnapshot`。

`FirstXactSnapshot->regd_count` 增加。并放入 `RegisteredSnapshots` heap。

后续 `GetTransactionSnapshot()` 直接返回 `CurrentSnapshot`。这就是“事务级 snapshot”的形成。

不要把 `FirstSnapshotSet` 本身当成 REPEATABLE READ 的语义。READ COMMITTED 也会设置它。

真正分界是 `IsolationUsesXactSnapshot()`。同样，不要把 `CurrentSnapshotData` 的静态存储当成事务级稳定性。

READ COMMITTED 会反复改写它。只有复制并注册到 `FirstXactSnapshot` 后，才形成事务级持有。

EPQ 也能帮助看清分界。READ COMMITTED 遇到并发更新，可以锁定最新版本并重检 quals。

REPEATABLE READ 使用事务级 snapshot。遇到同类 `TM_Updated` 时不能简单跟随新版本继续。

所以通常报 serialization failure。这不是因为 `HeapTupleSatisfiesMVCC()` 换了字段。

而是因为隔离级别改变了允许的 snapshot 生命周期和并发更新处理策略。cursor 边界同样要小心。

READ COMMITTED 的普通新 SELECT 会重取 snapshot。但 cursor 的 portal snapshot 已在 `PortalStart()` 绑定。

REPEATABLE READ 下新 SELECT 本来就复用事务 snapshot。所以两种隔离级别在 cursor 场景中表面现象可能更接近。

诊断时要先问：

当前是在普通 statement、cursor FETCH、holdable cursor，还是 EPQ recheck。再问当前 isolation 是否使用 transaction snapshot。

最后才解释是否应该看到并发提交。这样可以避免把“READ COMMITTED 看新提交”这句话用到错误生命周期上。

## 31. 本节小结

本节只回答一个问题：

READ COMMITTED 如何同时做到语句之间新鲜和语句内部稳定。答案不是“每次读查最新 CLOG”。

也不是“整个事务固定一个 snapshot”。源码里的模型是：

READ COMMITTED 下 `IsolationUsesXactSnapshot()` 为 false。`GetTransactionSnapshot()` 在每次需要 query snapshot 时重建 `CurrentSnapshotData`。

portal 在语句进入 executor 前把 snapshot 放到 active stack。`CreateQueryDesc()` 注册 snapshot。

`ExecutorStart()` 要求 snapshot 已 active，并把它注册到 `EState`。`ExecutorRun()` 期间 heap visibility 使用 `estate->es_snapshot`。

语句结束、executor 结束、portal drop 或 abort cleanup 时释放引用。cursor 会把 portal 和 snapshot 生命周期拉长。

WITH HOLD cursor 会在提交前把结果物化，后续 FETCH 读 store。并发 UPDATE/DELETE 下，EPQ 在 READ COMMITTED 中重检最新目标行。

但 EPQ child EState 共享 parent snapshot，不刷新整条语句。本节可迁移规律是：

隔离级别不是一个单独开关。它落在“什么时候创建 snapshot、谁持有 snapshot、执行期间是否允许替换、异常时谁释放”这组生命周期边界上。

下一节会把同一套入口对照到 REPEATABLE READ。在那里，第一次 snapshot 会被提升为事务级状态，READ COMMITTED 的语句级新鲜度会被有意关闭。
