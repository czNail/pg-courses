# PostgreSQL UPDATE / DELETE 并发冲突判定

## 课程定位

前置知识：已经理解 `xmax` 如何表达锁、更新和删除，理解四种 row lock 模式，也知道 MultiXact 可以记录多个成员和 updater。

本节唯一主问题：

```text
两个事务同时 UPDATE 或 DELETE 同一行时，PostgreSQL 如何在等待、重查可见性、EvalPlanQual 和报错之间选择？
```

本节围绕的核心矛盾：

```text
READ COMMITTED 希望在等待并发事务结束后尽量使用最新已提交版本继续执行；
事务级 snapshot 又要求当前事务不能悄悄换到一个不属于自己 snapshot 的世界。
```

学完后应能独立判断：

- `TM_Ok`、`TM_Updated`、`TM_Deleted`、`TM_SelfModified`、`TM_BeingModified` 分别代表什么边界。
- 为什么 heap 层只返回 tuple modification 结果，不直接决定 SQL 是否继续。
- 为什么 READ COMMITTED 下需要 EvalPlanQual 重新跑 quals。
- 为什么 REPEATABLE READ / SERIALIZABLE 下并发 update/delete 会报 serialization failure。
- 为什么等待 XID 或 MultiXact 后必须重新检查 tuple header。
- 为什么 `t_ctid` 和 `TUPLE_LOCK_FLAG_FIND_LAST_VERSION` 是追新版本的关键。
- 为什么 BEFORE trigger 或 volatile function 修改同一行会进入特殊错误边界。
- 为什么并发冲突诊断不能只看锁等待，还要看隔离级别和 executor 重查。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

第 20 节讲 `xmax` 的多重含义。

第 21 节讲 row lock 模式。

第 22 节讲 MultiXact 如何保存多个成员。

这三节解决了：

```text
行上有什么并发占用状态？
当前请求和已有状态是否冲突？
```

本节进一步问：

```text
冲突发生以后，SQL 语句应该如何收尾？
```

这个问题不能只在 heap AM 里回答。

heap AM 能判断：

- tuple 可以修改。
- tuple 正被别人修改。
- tuple 已被别人更新。
- tuple 已被别人删除。
- tuple 被当前命令或当前事务自己修改过。

但 SQL 层还要判断：

- 当前隔离级别是什么。
- 是否是 READ COMMITTED。
- 是否可以追最新版本。
- 追到最新版本后 WHERE 条件是否仍成立。
- 是否触发 `RETURNING`。
- 是否有 BEFORE / AFTER trigger。
- 是否有 WITH CHECK OPTION。
- 是否是 MERGE。

所以主链路跨越 heap AM 和 executor：

```text
ExecUpdate() / ExecDelete()
  -> table_tuple_update() / table_tuple_delete()
     -> heap_update() / heap_delete()
        -> HeapTupleSatisfiesUpdate()
        -> 等待 XID / MultiXact
        -> 返回 TM_Result
  -> executor 根据 TM_Result、隔离级别和 EPQ 决定继续、跳过或报错
```

本节不展开全部 trigger、partition update 或 MERGE 语义。

只围绕 UPDATE / DELETE 对同一 heap tuple 的并发冲突。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
heap_update() / heap_delete() 在 buffer lock 下根据 tuple header 和事务状态返回 TM_Result；
如果遇到 in-progress 修改者，就释放 buffer 等待 XID 或 MultiXact，回来后重查；
executor 在 READ COMMITTED 下用 table_tuple_lock(... FIND_LAST_VERSION ...) 追到最新版本并执行 EvalPlanQual；
在事务级 snapshot 下遇到并发 update/delete 则报 serialization failure。
```

这背后的核心矛盾是：

```text
单条 UPDATE/DELETE 想尽可能完成业务动作；
但它不能在并发修改后对一个已经不满足原 WHERE 条件、或不属于当前 snapshot 语义的版本继续执行。
```

READ COMMITTED 的选择是：

```text
等待并发事务结束；
如果行被更新，追到最新版本；
重新检查 WHERE quals；
如果仍匹配，就对最新版本执行；
如果不匹配，就跳过。
```

REPEATABLE READ / SERIALIZABLE 的选择是：

```text
当前事务 snapshot 必须稳定；
目标行被并发事务改写后，不能自动换到新版本；
因此报 serialization failure。
```

本节要建立的系统规律是：

```text
并发写冲突不是“等完继续”这么简单；
等待只解决对方事务是否结束，后续还要重新证明目标 tuple、查询条件和隔离级别仍允许当前语句继续。
```

## 3. 核心文件分工与阅读顺序

| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/access/tableam.h` | `TM_Result`、`TM_FailureData`、`table_tuple_update/delete/lock` contract。 |
| 2 | `src/backend/executor/nodeModifyTable.c` | `ExecUpdate()`、`ExecDelete()` 如何处理 `TM_Result` 和 EPQ。 |
| 3 | `src/backend/executor/execMain.c` | `EvalPlanQual()`、`EvalPlanQualBegin()`、`EvalPlanQualNext()`。 |
| 4 | `src/backend/access/heap/heapam.c` | `heap_update()`、`heap_delete()`、`heap_lock_tuple()` 冲突和等待分支。 |
| 5 | `src/backend/access/heap/heapam_visibility.c` | `HeapTupleSatisfiesUpdate()` 判断 tuple 是否可修改。 |
| 6 | `src/backend/storage/lmgr/lmgr.c` | `XactLockTableWait()`、`ConditionalXactLockTableWait()`。 |
| 7 | `src/backend/access/transam/multixact.c` | MultiXact 成员状态对等待边界的影响。 |
| 8 | `src/backend/executor/nodeLockRows.c` | `SELECT FOR UPDATE` 追新版本和 EPQ 的相同模式。 |
| 9 | `src/include/nodes/execnodes.h` | `EPQState` 保存 recheck 运行状态。 |
| 10 | `src/backend/executor/README` | EvalPlanQual 的设计背景和 READ COMMITTED update checking。 |

推荐阅读顺序：

```text
先读 tableam.h 的 TM_Result
  -> 读 nodeModifyTable.c 中 ExecUpdate() 的 switch
  -> 回到 heap_update() 看 TM_Result 从哪里来
  -> 读 heap_lock_tuple() 的 FIND_LAST_VERSION 路径
  -> 读 EvalPlanQual() 看如何重新执行 quals
```

不要先陷入 `nodeModifyTable.c` 的全部 trigger、partition、MERGE 分支。

本节只抓住：

```text
冲突返回
  -> 隔离级别判断
  -> 锁最新版本
  -> EPQ 重新判断
  -> 再次执行或跳过
```

## 4. 一个 runtime 现象先定锚

Session 0：

```sql
DROP TABLE IF EXISTS upd_conflict;
CREATE TABLE upd_conflict(id int primary key, flag boolean, payload text);
INSERT INTO upd_conflict VALUES (1, true, 'a');
```

Session A：

```sql
BEGIN;
UPDATE upd_conflict
SET payload = 'from A'
WHERE id = 1;
-- 保持不提交。
```

Session B：

```sql
BEGIN;
UPDATE upd_conflict
SET payload = 'from B'
WHERE id = 1 AND flag = true;
```

Session B 会等待 A。

如果 A 提交，READ COMMITTED 下 B 会重新检查最新版本。

如果 A 只是改 `payload`，`flag` 仍然 true，B 可以继续更新最新版本。

如果 A 改成：

```sql
UPDATE upd_conflict
SET flag = false
WHERE id = 1;
COMMIT;
```

B 等待后重新检查 WHERE。

它发现最新版本不再满足 `flag = true`。

于是 B 不更新这行。

在 REPEATABLE READ 下，B 通常不能这样换到新版本继续。

它会报并发 update 导致的 serialization failure。

这个现象把本节主问题固定住：

```text
等待只让 B 知道 A 的结果；
B 还必须决定是否可以在自己的语义世界里继续处理最新版本。
```

## 5. `TM_Result` 是 heap AM 和 executor 的边界

`TM_Result` 是 table AM 返回给 executor 的关键边界。

常见值包括：

| 结果 | heap 层含义 | executor 典型处理 |
| --- | --- | --- |
| `TM_Ok` | 修改或锁定成功。 | 继续 trigger、index、RETURNING 等收尾。 |
| `TM_Updated` | tuple 已被其它事务更新。 | READ COMMITTED 走 EPQ；事务级 snapshot 报错。 |
| `TM_Deleted` | tuple 已被其它事务删除。 | READ COMMITTED 跳过；事务级 snapshot 报错。 |
| `TM_SelfModified` | 当前命令或当前事务已改过。 | 防 Halloween；可能忽略或报 trigger 相关错误。 |
| `TM_BeingModified` | 另一个事务正在修改，且调用者不等待。 | 非等待策略返回给上层处理。 |
| `TM_WouldBlock` | `SKIP LOCKED` 等策略下不能立即锁。 | executor 跳过或报错。 |
| `TM_Invisible` | 对当前命令不可见。 | 通常是内部错误或特殊 ON CONFLICT 边界。 |

这张表的重点是：

```text
heap AM 不直接判断 SQL 语句应该重新执行 WHERE；
它只告诉 executor 目标 tuple 的修改状态。
```

executor 才知道：

- 当前语句的 quals。
- 当前隔离级别。
- `RETURNING`。
- trigger。
- rowmark。
- EPQ 状态。

所以 `TM_Result` 是一个模块边界，而不是最终用户语义。

## 6. `heap_update()` 的冲突判断

`heap_update()` 的核心流程可以压缩成：

```text
读 old tuple 所在 buffer
锁 buffer
HeapTupleSatisfiesUpdate()
  -> TM_Ok:
       可以更新
  -> TM_BeingModified:
       需要等待或返回
  -> TM_Updated / TM_Deleted:
       已经被别人结束
  -> TM_SelfModified:
       当前事务自己改过
等待时释放 buffer
等待 XID 或 MultiXact
重拿 buffer
重查 xmax / infomask / ctid
根据结果写旧 tuple xmax、插入新 tuple version
```

等待分支有两个来源。

第一，旧 `xmax` 是普通 XID。

使用：

```text
XactLockTableWait()
```

第二，旧 `xmax` 是 MultiXactId。

使用：

```text
MultiXactIdWait()
```

等待后必须重查：

```text
xmax_infomask_changed()
raw xmax 是否还等于 xwait
t_ctid 是否仍指向同一链路
```

如果状态变化，重新回到判断入口。

这说明 `heap_update()` 不是一次性判断。

它是一个 retry loop。

并发正确性依赖这个 loop。

## 7. `heap_delete()` 的冲突判断

DELETE 和 UPDATE 相似。

区别是 DELETE 不插入新 tuple version。

但它仍要处理：

- 当前 tuple 是否可删除。
- 是否被别的事务锁住。
- 是否被别的事务更新。
- 是否被别的事务删除。
- 已有 MultiXact 中是否有冲突成员。

DELETE 成功时会把旧 tuple 标记为被当前事务删除。

它写入：

```text
xmax = current XID 或新 MultiXactId
HEAP_KEYS_UPDATED 语义上代表 key gone
```

如果 DELETE 发现 tuple 已经被别人更新，它返回 `TM_Updated`。

如果发现 tuple 已经被别人删除，它返回 `TM_Deleted`。

executor 再决定：

```text
READ COMMITTED:
  可能追新版本并 EPQ，或跳过。

事务级 snapshot:
  报 serialization failure。
```

DELETE 的重点是：

```text
删除不是只看当前 snapshot 不可见；
它要在并发写入者之间建立谁终结旧版本的顺序。
```

## 8. READ COMMITTED 下为什么需要 EPQ

READ COMMITTED 每条语句使用语句级 snapshot。

但一个 UPDATE 语句可能先扫描到旧版本。

在它真正尝试更新时，另一个事务已经提交了新版本。

如果 PostgreSQL 直接更新旧版本，会破坏版本链。

如果直接跳过，会错过本可以继续匹配的新版本。

所以 READ COMMITTED 使用 EvalPlanQual。

流程是：

```text
ExecUpdate()
  -> table_tuple_update()
  -> 返回 TM_Updated
  -> table_tuple_lock(... FIND_LAST_VERSION ...)
  -> 锁住最新版本
  -> EvalPlanQual()
  -> 重新执行原查询 quals
  -> 如果新版本仍匹配，构造新 update tuple
  -> goto redo_act
```

关键调用是：

```text
table_tuple_lock(... TUPLE_LOCK_FLAG_FIND_LAST_VERSION ...)
```

这个 flag 告诉 heap AM 沿 `t_ctid` 追到更新链的最新可锁版本。

`EvalPlanQual()` 则重新运行相关 plan subtree。

它不是只重新判断一个 WHERE 表达式字符串。

它使用 executor 的 EPQ state、test slot、rowmark 等上下文重新执行。

如果 EPQ 返回空 slot：

```text
新版本不再满足 quals；
当前 UPDATE/DELETE 跳过这行。
```

如果 EPQ 返回新 slot：

```text
用新版本作为输入，重新构造 UPDATE 目标 tuple；
再次尝试 table_tuple_update()。
```

所以 READ COMMITTED 的语义不是“读到谁就改谁”。

它是：

```text
等待后对最新 committed version 重新检查语句条件。
```

## 9. 事务级 snapshot 下为什么报错

REPEATABLE READ 和 SERIALIZABLE 使用事务级 snapshot。

它们的核心承诺是：

```text
一个事务内后续语句不因为外部提交而换读视图。
```

如果 UPDATE 遇到并发事务已经更新目标行，READ COMMITTED 可以追最新版本。

但事务级 snapshot 不能无声追新版本。

因为新版本可能在本事务 snapshot 之后才提交。

继续更新它会把当前事务带到一个不属于自己 snapshot 的世界。

所以 `nodeModifyTable.c` 在 `TM_Updated` 分支中检查：

```text
IsolationUsesXactSnapshot()
```

如果为真，报：

```text
could not serialize access due to concurrent update
```

DELETE 对应：

```text
could not serialize access due to concurrent delete
```

这不是 lock timeout。

也不是 deadlock。

它是隔离级别语义要求。

## 10. `TM_SelfModified` 与 Halloween 边界

`TM_SelfModified` 表示目标 tuple 已被当前命令或当前事务自己改过。

最典型的风险是 Halloween problem。

例如 join update 中，多个 join row 指向同一个 target row。

PostgreSQL 的选择是：

```text
同一命令已经更新过:
  忽略后续重复更新。

当前事务中较晚命令或 BEFORE trigger 改过:
  不能安全合并，报错。
```

`nodeModifyTable.c` 会比较：

```text
tmfd.cmax
estate->es_output_cid
```

如果不是当前 command id，说明可能是 trigger 或 volatile function 引起的当前事务内部修改。

系统报：

```text
tuple to be updated was already modified by an operation triggered by the current command
```

这条边界说明：

```text
并发冲突不仅来自其它事务；
同一事务内部的命令边界也会影响可修改性。
```

第 10 节的 CommandId 语义在这里回来了。

## 11. DELETE 的 EPQ 边界

DELETE 遇到 `TM_Updated` 时也可能走 EPQ。

流程类似：

```text
ExecDelete()
  -> table_tuple_delete()
  -> TM_Updated
  -> READ COMMITTED 下锁最新版本
  -> EvalPlanQual()
  -> 如果新版本仍匹配 DELETE 条件，再 delete
  -> 如果不匹配，跳过
```

如果最新版本已经被删除：

```text
TM_Deleted
  -> READ COMMITTED 下跳过
  -> 事务级 snapshot 下 serialization failure
```

DELETE 的特殊点是：

```text
它没有新 tuple slot；
但它仍需要 old tuple、RETURNING、trigger 和 EPQ 之间的状态传递。
```

因此 `nodeModifyTable.c` 中 DELETE 相关代码也很长。

读的时候不要被 trigger epilogue 淹没。

只追：

```text
TM_Updated
  -> EvalPlanQual
  -> 是否继续 delete
```

## 12. 生命周期 / ownership / cleanup

并发 UPDATE / DELETE 涉及多个对象生命周期。

### 12.1 tuple version

旧版本生命周期：

```text
被扫描到
  -> 被尝试更新或删除
  -> 可能等待已有 xmax
  -> 成功后写入当前事务 xmax
  -> 更新时 t_ctid 指向新版本
  -> VACUUM 后续按 cleanup horizon 回收
```

新版本生命周期：

```text
heap_update() 插入新 tuple
  -> xmin = current XID
  -> xmax invalid
  -> index entries 视 HOT / non-HOT 决定
```

### 12.2 executor slot

EPQ 使用 slot 保存：

- 原始 plan 输出。
- 被锁定的最新 tuple。
- EPQ recheck 结果。
- old tuple slot。
- update projection 的输入。

这些是 executor-local 生命周期。

它们不会进入 shared memory。

### 12.3 lock 和 wait

等待时：

```text
buffer lock 必须释放
事务锁或 MultiXact 成员等待
等待回来重拿 buffer lock
```

长期 row lock 记录仍在 tuple header。

### 12.4 ERROR cleanup

如果 EPQ、trigger、constraint 或 serialization failure 抛错：

```text
事务 abort
ResourceOwner 清理 buffer pin、relation ref、snapshot 等资源
lock manager 释放事务锁
heap tuple header 中当前事务写入的 XID 后续被解释为 aborted
```

这说明 heap 层不需要在 ERROR 时手动撤销已经写过的 tuple header。

MVCC 通过事务结果解释它。

## 13. 正确性机制层次

第一层：tuple header。

`xmax`、infomask、`t_ctid` 记录旧版本状态。

第二层：事务状态。

`TransactionIdIsInProgress()`、commit、abort 决定旧 `xmax` 的结果。

第三层：等待。

`XactLockTableWait()` 和 `MultiXactIdWait()` 等待冲突事务完成。

第四层：重查。

等待后必须确认 header 没变。

第五层：隔离级别。

READ COMMITTED 可以追最新版本。

事务级 snapshot 报 serialization failure。

第六层：EvalPlanQual。

EPQ 确认最新版本是否仍满足原查询条件。

第七层：CommandId。

同一事务内部多次修改同一 tuple 的边界由 command id 判断。

第八层：WAL。

成功 UPDATE / DELETE 的 heap page 改动、new tuple、old tuple xmax 都需要 WAL 保护。

## 14. 异常路径 / fallback

### 14.1 等待后 tuple 已被删除

READ COMMITTED 下：

```text
UPDATE:
  通常跳过。

DELETE:
  已经没什么可删，跳过。
```

事务级 snapshot 下：

```text
serialization failure
```

### 14.2 等待后 tuple 被更新且 quals 不再匹配

READ COMMITTED 下：

```text
EPQ 返回空 slot
当前行跳过
```

这不是错误。

这是 READ COMMITTED 的语义。

### 14.3 等待后 tuple 被更新且 quals 仍匹配

READ COMMITTED 下：

```text
重新构造 update target
goto redo_act
再次尝试 table_tuple_update()
```

如果再次遇到冲突，loop 继续。

### 14.4 NOWAIT / SKIP LOCKED

这些策略常见于 `SELECT FOR UPDATE`。

DML 内部也可能通过 table AM contract 表达 would-block。

遇到 would-block 时 executor 不会进入普通等待。

它按 SQL 策略跳过或报错。

### 14.5 trigger 造成自修改

BEFORE trigger 修改同一行，外层 UPDATE 再试图改同一行。

系统没有可靠合并规则。

因此报 triggered data change violation。

建议用 AFTER trigger 或重新设计写入路径。

## 15. 成本、资源与跨模块传播

### 15.1 热点行等待

同一行被多个事务更新时，主要成本是：

- lock wait。
- 事务等待。
- deadlock detector 可能参与。
- 重查 tuple header。
- 反复 EPQ。

吞吐随热点集中度急剧下降。

### 15.2 EPQ 成本

EPQ 不是常数成本。

它可能重新执行 plan subtree。

涉及 join、subquery、rowmark、foreign table 时成本更高。

READ COMMITTED 下高冲突 update 会把成本从 heap wait 扩散到 executor recheck。

### 15.3 MultiXact 成本

如果冲突前已有多个 locker，UPDATE / DELETE 还要展开 MultiXact。

这增加：

- SLRU 访问。
- 成员遍历。
- 多成员等待。

### 15.4 index 和 HOT 成本

UPDATE 成功后是否需要更新索引，取决于 HOT 条件和 key 改动。

并发冲突本身不直接决定 index 成本。

但 EPQ 后的新版本可能改变 update projection。

最终 index insert 发生在 executor update epilogue。

### 15.5 VACUUM 传播

UPDATE / DELETE 制造旧版本。

旧版本回收又受 xmin horizon、replication slot、prepared xact、MultiXact freeze 边界影响。

热点更新不仅造成前台等待。

也会制造后续 VACUUM 和 bloat 压力。

## 16. 观测与诊断入口

### 16.1 等待观察

```sql
SELECT pid, wait_event_type, wait_event, state, query
FROM pg_stat_activity
WHERE wait_event_type = 'Lock';
```

再看：

```sql
SELECT locktype, mode, granted, transactionid, relation, page, tuple, pid
FROM pg_locks
WHERE NOT granted OR locktype IN ('transactionid', 'tuple');
```

注意：

看到 transactionid wait 只能说明在等事务结束。

不能说明等待后会继续更新、跳过还是报错。

### 16.2 隔离级别确认

```sql
SHOW transaction_isolation;
```

同一个冲突，在 READ COMMITTED 和 REPEATABLE READ 下结果不同。

诊断必须先确认隔离级别。

### 16.3 EPQ 现象

可以通过实验观察：

```text
Session B 等待 Session A；
Session A 提交后，B 没有更新任何行；
不是因为锁失败；
是因为 EPQ 后 quals 不再匹配。
```

`UPDATE 0` 是常见外在表现。

### 16.4 源码断点

建议断点：

```text
ExecUpdate
ExecDelete
ExecUpdateAct
ExecDeleteAct
EvalPlanQual
heap_update
heap_delete
heap_lock_tuple
XactLockTableWait
MultiXactIdWait
```

观察变量：

```text
result
tmfd.ctid
tmfd.xmax
tmfd.cmax
IsolationUsesXactSnapshot()
lockmode
tupleid
```

### 16.5 日志和错误

区分错误：

```text
could not serialize access due to concurrent update
  -> 隔离级别冲突。

deadlock detected
  -> lock wait graph 成环。

canceling statement due to lock timeout
  -> 等待时间策略。

tuple to be updated was already modified by an operation triggered by the current command
  -> 当前事务内部自修改边界。
```

## 17. 常见误区

误区一：

```text
第二个 UPDATE 等第一个事务提交后一定继续更新。
```

错误。

READ COMMITTED 还要 EPQ。

事务级 snapshot 可能报错。

误区二：

```text
UPDATE 0 表示没有扫描到目标行。
```

不一定。

可能扫描到了旧版本，等待后 EPQ 发现最新版本不再匹配。

误区三：

```text
TM_Updated 就是用户可见错误。
```

不完整。

它是 heap AM 到 executor 的中间结果。

executor 才决定 EPQ 或报错。

误区四：

```text
等待 transactionid 是性能问题，不影响语义。
```

错误。

等待结束后的事务结果决定后续语义分支。

误区五：

```text
READ COMMITTED 没有稳定 snapshot，所以不需要重查。
```

相反。

正因为它允许看到等待后的最新提交，才必须用 EPQ 重查条件。

## 18. 课堂实验

### 实验一：READ COMMITTED 等待后继续更新

Session 0：

```sql
DROP TABLE IF EXISTS epq_demo;
CREATE TABLE epq_demo(id int primary key, flag boolean, payload text);
INSERT INTO epq_demo VALUES (1, true, 'a');
```

Session A：

```sql
BEGIN;
UPDATE epq_demo SET payload = 'A' WHERE id = 1;
```

Session B：

```sql
BEGIN;
UPDATE epq_demo SET payload = 'B' WHERE id = 1 AND flag = true;
```

Session A：

```sql
COMMIT;
```

观察 Session B 继续成功。

### 实验二：READ COMMITTED 等待后跳过

Session A：

```sql
BEGIN;
UPDATE epq_demo SET flag = false WHERE id = 1;
```

Session B：

```sql
BEGIN;
UPDATE epq_demo SET payload = 'B' WHERE id = 1 AND flag = true;
```

Session A：

```sql
COMMIT;
```

观察 Session B 返回 `UPDATE 0`。

回到源码：

```text
TM_Updated
  -> table_tuple_lock(... FIND_LAST_VERSION ...)
  -> EvalPlanQual()
  -> TupIsNull(epqslot)
  -> return NULL
```

### 实验三：REPEATABLE READ 报错

Session B 改成：

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
UPDATE epq_demo SET payload = 'B' WHERE id = 1;
```

在 Session A 并发更新提交后，观察：

```text
could not serialize access due to concurrent update
```

### 实验四：DELETE 冲突

Session A：

```sql
BEGIN;
UPDATE epq_demo SET flag = false WHERE id = 1;
```

Session B：

```sql
DELETE FROM epq_demo WHERE id = 1 AND flag = true;
```

Session A 提交后，观察 DELETE 是否跳过。

### 实验五：断点画状态图

断点：

```text
heap_update
ExecUpdate
EvalPlanQual
```

画出：

```text
扫描旧版本
  -> heap_update 返回 TM_Updated
  -> executor 锁最新版本
  -> EPQ 重查
  -> redo_act 或跳过
```

## 19. 讨论题

1. 为什么 heap AM 不直接执行 EvalPlanQual？

2. `TM_Updated` 和 `TM_BeingModified` 的边界是什么？

3. 为什么 READ COMMITTED 可以等待后追最新版本，而 REPEATABLE READ 不可以？

4. EPQ 重新检查的是哪个版本？为什么不能只检查旧版本？

5. 为什么等待后要重查 `xmax` 和 `t_ctid`？

6. `UPDATE 0` 可能来自哪些路径？

7. `TM_SelfModified` 如何避免 Halloween problem？

8. 如何区分 lock timeout、deadlock、serialization failure 和 EPQ skip？

## 20. 本节小结

本节回答了并发 UPDATE / DELETE 的核心问题：

```text
等待并发事务结束后，PostgreSQL 如何决定继续、跳过还是报错？
```

核心链路是：

```text
ExecUpdate() / ExecDelete()
  -> table_tuple_update() / table_tuple_delete()
  -> heap_update() / heap_delete()
  -> HeapTupleSatisfiesUpdate()
  -> 等待 XID / MultiXact
  -> 返回 TM_Result
  -> READ COMMITTED: 锁最新版本 + EvalPlanQual
  -> 事务级 snapshot: serialization failure
```

核心状态是：

- `xmax`。
- `t_ctid`。
- `TM_Result`。
- `TM_FailureData`。
- `LockTupleMode`。
- `TUPLE_LOCK_FLAG_FIND_LAST_VERSION`。
- `EPQState`。
- command id。
- isolation level。

ownership 和 cleanup 的边界是：

```text
heap AM 拥有 tuple header 修改；
executor 拥有 SQL 语义、slot、EPQ 和 trigger 收尾；
ResourceOwner 和事务 abort 负责 ERROR 后资源释放；
已经写入的 tuple XID 通过事务结果解释，而不是手工撤销。
```

错误路径的核心规律是：

```text
等待只告诉我们对方事务结束；
隔离级别和 EPQ 才决定当前语句是否还能继续。
```

可观测入口包括：

- `pg_stat_activity`。
- `pg_locks`。
- `SHOW transaction_isolation`。
- `UPDATE 0` / serialization failure。
- gdb 断点。

本节的可迁移模型是：

```text
并发写路径需要把“物理目标是否还在”、
“最新版本是否满足查询条件”、
“当前隔离级别是否允许换版本”
分成三个问题；
PostgreSQL 用 TM_Result 和 EvalPlanQual 把这三个问题明确分层。
```
