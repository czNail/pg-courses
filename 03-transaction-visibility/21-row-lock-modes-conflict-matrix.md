# PostgreSQL row lock 模式与 tuple lock 冲突矩阵

## 课程定位

前置知识：已经理解 `xmax` 的多重语义，知道 tuple header 可以用 `HEAP_XMAX_LOCK_ONLY`、`HEAP_XMAX_IS_MULTI` 和 `HEAP_KEYS_UPDATED` 区分锁定、更新和删除。

本节唯一主问题：

```text
为什么 PostgreSQL 需要 FOR KEY SHARE、FOR SHARE、FOR NO KEY UPDATE、FOR UPDATE 四种行锁模式，而不是一个统一的 row exclusive lock？
```

本节围绕的核心矛盾：

```text
行锁必须保护外键、更新、删除和 SELECT FOR UPDATE 的正确性；
但如果所有修改都互斥，就会让非 key 更新、外键检查和普通共享锁产生不必要等待。
```

学完后应能独立判断：

- 四种 `LockTupleMode` 分别保护什么不变量。
- 为什么 `FOR KEY SHARE` 可以和 no-key update 共存。
- 为什么 `FOR SHARE` 会阻止 UPDATE。
- 为什么 `FOR NO KEY UPDATE` 不允许 DELETE 或 key update 并发通过。
- 为什么 `FOR UPDATE` 是最强模式。
- 为什么冲突矩阵最终复用 heavyweight lock manager 的 `LOCKMODE` 冲突判断。
- 为什么 tuple lock 等待需要“header 记录 + lock manager 排队”两层协议。
- 为什么等待后仍必须回到 heap tuple header 重查。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

上一节解决的是 `xmax` 的解释入口。

它告诉我们：

```text
同一个 xmax 字段可以表示 locker、updater、deleter 或 MultiXact；
真正语义要看 infomask、HEAP_KEYS_UPDATED、事务状态和调用目标。
```

本节把调用目标展开。

SQL 层并不是只有“锁一行”这一种意图。

它至少有四种不同意图：

```text
我只要保证 referenced key 不消失。
我想阻止别人修改这行。
我想更新非 key 列。
我想删除或修改 key。
```

如果这些意图都映射成一个强锁，正确性简单，但并发性很差。

外键检查会阻塞所有 parent row 更新。

非 key 更新会阻塞 child insert。

多个只读锁会被无意义串行化。

因此 PostgreSQL 把 row lock strength 拆成四级：

```text
LockTupleKeyShare
LockTupleShare
LockTupleNoKeyExclusive
LockTupleExclusive
```

本节只讨论这四种模式的语义和冲突。

第 22 节再讨论多个 locker 如何塞进 MultiXact。

第 23 节再讨论 UPDATE / DELETE 遇到并发修改后的等待、重查和 EvalPlanQual。

本节主线是：

```text
SQL rowmark 或 DML 意图
  -> LockTupleMode
  -> MultiXactStatus
  -> heavyweight tuple lock mode
  -> DoLockModesConflict()
  -> heap_lock_tuple() 是否等待、跳过或写入 xmax
```

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
PostgreSQL 用四种 LockTupleMode 表达不同 row-level invariants；
heapam.c 把它们映射成 MultiXactStatus 和 heavyweight LOCKMODE；
冲突判断最终通过 lock manager 的 DoLockModesConflict() 完成；
tuple header 负责记录长期占用，LOCKTAG_TUPLE 只在有等待风险时提供排队公平性。
```

四种模式不是为了 SQL 语法好看。

它们是为了保护不同的不变量。

| SQL 意图 | 内部模式 | 保护的不变量 |
| --- | --- | --- |
| `FOR KEY SHARE` | `LockTupleKeyShare` | referenced key 不被删除或 key update 改掉。 |
| `FOR SHARE` | `LockTupleShare` | tuple 不被任何 UPDATE / DELETE 改动。 |
| `FOR NO KEY UPDATE` | `LockTupleNoKeyExclusive` | 当前事务可以改非 key 列，并阻止其它写者。 |
| `FOR UPDATE` | `LockTupleExclusive` | 当前事务可删除或修改 key，阻止所有其它 row lock 冲突者。 |

冲突矩阵可以压缩成：

```text
UPDATE          冲突所有模式
NO KEY UPDATE   冲突 UPDATE / NO KEY UPDATE / SHARE
SHARE           冲突 UPDATE / NO KEY UPDATE
KEY SHARE       只冲突 UPDATE
```

这里的 `UPDATE` 指最强 row lock 模式，不是 SQL 命令名。

SQL `UPDATE` 是否使用最强模式，要看它是否修改 key。

本节要建立的系统规律是：

```text
row lock 不是“谁读谁写”的二元互斥；
它是把上层语义不变量压缩成有限强度，然后让 heap tuple header 与 lock manager 共同执行的冲突协议。
```

## 3. 核心文件分工与阅读顺序

| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/access/heap/README.tuplock` | 四种 tuple lock strength 和冲突矩阵的语义说明。 |
| 2 | `src/include/nodes/lockoptions.h` | `LockTupleMode` 枚举定义。 |
| 3 | `src/backend/access/heap/heapam.c` | `tupleLockExtraInfo[]`、`MultiXactStatusLock[]`、`heap_lock_tuple()`。 |
| 4 | `src/include/access/multixact.h` | `MultiXactStatusForKeyShare` 到 `MultiXactStatusUpdate`。 |
| 5 | `src/backend/storage/lmgr/lmgr.c` | `LockTuple()`、`ConditionalLockTuple()`、`UnlockTuple()`。 |
| 6 | `src/backend/storage/lmgr/lock.c` | lock manager 的 lock tag、wait queue、冲突判断。 |
| 7 | `src/backend/executor/nodeLockRows.c` | SQL rowmark 到 `LockTupleMode` 的映射。 |
| 8 | `src/backend/executor/execMain.c` | `ExecUpdateLockMode()` 判断 UPDATE 是否 key-changing。 |
| 9 | `src/backend/executor/nodeModifyTable.c` | UPDATE / DELETE 消费 `TM_Result` 的边界。 |
| 10 | `src/backend/utils/adt/ri_triggers.c` | FK 检查为什么使用 `LockTupleKeyShare`。 |

推荐阅读顺序：

```text
先读 README.tuplock 的矩阵
  -> 读 lockoptions.h 的四个枚举
  -> 读 heapam.c 顶部的 tupleLockExtraInfo[]
  -> 读 heap_lock_tuple() 如何按 mode 判断等待
  -> 读 lmgr.c 的 LockTuple() 确认 lock table 只是第二层
  -> 最后读 executor 和 RI trigger，确认上层语义如何落到 mode
```

不要把 `LockTupleMode` 和 heavyweight `LOCKMODE` 混成一件事。

`LockTupleMode` 是 heap row lock 的语义强度。

`LOCKMODE` 是 lock manager 的通用冲突矩阵输入。

`tupleLockExtraInfo[]` 把前者映射到后者。

## 4. 从 runtime 现象进入

先做一个外键场景。

Session 0：

```sql
DROP TABLE IF EXISTS child;
DROP TABLE IF EXISTS parent;

CREATE TABLE parent(id int primary key, payload text);
CREATE TABLE child(id int primary key, parent_id int references parent(id));
INSERT INTO parent VALUES (1, 'a');
```

Session A：

```sql
BEGIN;
INSERT INTO child VALUES (1, 1);
-- 保持事务打开。
```

这个 INSERT 会检查 parent row 是否存在。

RI 检查会锁住 parent row 的 key。

不是为了阻止 parent row 的所有更新。

是为了阻止 key 消失。

Session B：

```sql
UPDATE parent SET payload = 'b' WHERE id = 1;
```

这个更新没有修改 key。

它使用 no-key update 语义。

它和 key-share 可以共存。

Session C：

```sql
DELETE FROM parent WHERE id = 1;
```

这个删除会让 referenced key 消失。

它必须等待或失败。

同样：

```sql
UPDATE parent SET id = 2 WHERE id = 1;
```

也必须冲突。

这个 runtime 现象说明：

```text
外键检查不是阻止 parent row 任何变化；
它只阻止 referenced key 被删除或改变。
```

如果只有一个统一 row exclusive lock，Session B 的非 key 更新也会被阻塞。

PostgreSQL 用 `FOR KEY SHARE` / `NO KEY UPDATE` 的兼容性把这个不必要等待消掉。

## 5. 四种 row lock 模式的语义

### 5.1 `LockTupleKeyShare`

SQL 入口：

```text
SELECT ... FOR KEY SHARE
外键检查中的 referenced row 锁
```

保护目标：

```text
这行的 key 不被删除或改掉。
```

它允许：

- 其它 key-share。
- share。
- no-key update。

它冲突：

- key update。
- delete。
- `FOR UPDATE`。

因此它是外键检查需要的最小强度。

child row 插入时，只需要保证 parent key 在检查期间不会消失。

不需要阻止 parent payload 更新。

### 5.2 `LockTupleShare`

SQL 入口：

```text
SELECT ... FOR SHARE
```

保护目标：

```text
这行不被 UPDATE 或 DELETE 改动。
```

它允许：

- key-share。
- share。

它冲突：

- no-key update。
- key update。
- delete。

与 key-share 相比，它更强。

因为它保护整行稳定，而不仅仅保护 key。

### 5.3 `LockTupleNoKeyExclusive`

SQL 入口：

```text
SELECT ... FOR NO KEY UPDATE
UPDATE 没有修改 key 列
```

保护目标：

```text
当前事务要修改 tuple，但不会破坏 referenced key。
```

它允许：

- key-share。

它冲突：

- share。
- no-key update。
- update。

所以两个事务不能同时 no-key update 同一行。

但外键检查可以和它共存。

### 5.4 `LockTupleExclusive`

SQL 入口：

```text
SELECT ... FOR UPDATE
DELETE
UPDATE 修改 key 列
```

保护目标：

```text
当前事务可能删除 tuple 或改变 key，必须阻止所有其它 row lock 语义。
```

它冲突所有模式。

这就是最强 row lock。

## 6. 冲突矩阵如何落到源码

`README.tuplock` 给出矩阵。

`heapam.c` 顶部把四种 `LockTupleMode` 映射成三类东西：

- heavyweight `LOCKMODE`。
- 纯锁的 `MultiXactStatus`。
- update/delete 的 `MultiXactStatus`。

核心映射可以按语义读：

```text
LockTupleKeyShare
  -> AccessShareLock
  -> MultiXactStatusForKeyShare
  -> 不能作为 updater

LockTupleShare
  -> RowShareLock
  -> MultiXactStatusForShare
  -> 不能作为 updater

LockTupleNoKeyExclusive
  -> ExclusiveLock
  -> MultiXactStatusForNoKeyUpdate
  -> MultiXactStatusNoKeyUpdate

LockTupleExclusive
  -> AccessExclusiveLock
  -> MultiXactStatusForUpdate
  -> MultiXactStatusUpdate
```

这里的 heavyweight lock mode 名字容易误导。

它不是 relation lock。

它被用在 tuple lock tag 上。

`LockTuple()` 会构造：

```text
LOCKTAG_TUPLE
  database oid
  relation oid
  block number
  offset number
```

然后交给通用 lock manager。

所以 conflict matrix 的执行分两层：

```text
长期占用:
  tuple header 上的 xmax / infomask / MultiXact

等待排队:
  LOCKTAG_TUPLE 上的 heavyweight lock
```

在没有冲突时，许多路径不会持有 lock table 对象。

只有遇到需要等待或排队公平性时，才进入第二层。

这就是 `README.tuplock` 说的两层机制。

## 7. 主流程 walkthrough：SELECT FOR UPDATE/SHARE

SQL：

```sql
SELECT *
FROM parent
WHERE id = 1
FOR SHARE;
```

executor 主链：

```text
ExecLockRows()
  -> 从 outer plan 取一行
  -> 根据 rowmark 选择 LockTupleMode
  -> table_tuple_lock()
  -> heap_lock_tuple()
  -> 返回 locked slot 或冲突结果
```

`ExecLockRows()` 做的第一件关键事是转换 rowmark：

```text
ROW_MARK_EXCLUSIVE
  -> LockTupleExclusive

ROW_MARK_NOKEYEXCLUSIVE
  -> LockTupleNoKeyExclusive

ROW_MARK_SHARE
  -> LockTupleShare

ROW_MARK_KEYSHARE
  -> LockTupleKeyShare
```

然后它设置 lock flags。

READ COMMITTED 下，如果 tuple 被更新过，通常需要寻找最新版本。

所以会带上：

```text
TUPLE_LOCK_FLAG_FIND_LAST_VERSION
```

`heap_lock_tuple()` 进入后：

```text
读 buffer
锁 buffer
调用 HeapTupleSatisfiesUpdate()
如果可锁，计算新 xmax
如果冲突，释放 buffer 后等待 XID 或 MultiXact
等待回来后重查
写 tuple header
写 WAL
释放 buffer
释放临时 tuple lock
```

关键点：

```text
SQL row lock 的语义不在 executor 里完成；
executor 只选择 lock mode；
真正冲突判断和 header 写入在 heap AM 中完成。
```

## 8. 主流程 walkthrough：UPDATE 如何选择锁强度

SQL：

```sql
UPDATE parent
SET payload = 'new'
WHERE id = 1;
```

如果 `payload` 不属于 key 列，这通常是 no-key update。

SQL：

```sql
UPDATE parent
SET id = 2
WHERE id = 1;
```

如果 `id` 是 primary key，这就是 key update。

executor 会通过 `ExecUpdateLockMode()` 判断要使用哪种 tuple lock mode。

核心不是“UPDATE 一律最强”。

而是：

```text
修改 key:
  LockTupleExclusive

不修改 key:
  LockTupleNoKeyExclusive
```

heap update 里再把这个 mode 传给：

```text
compute_new_xmax_infomask(..., is_update=true)
```

如果是 key update，会在 tuple header 的 `infomask2` 中设置：

```text
HEAP_KEYS_UPDATED
```

这个 bit 会影响后续 key-share 判断。

外键检查持有 key-share 时，可以和 no-key update 共存。

但不能和 key update 共存。

这条边界是 PostgreSQL 行锁设计的核心之一。

## 9. 等待协议：为什么不只靠 XID wait

tuple header 已经记录了 locker XID。

看起来等待者可以直接：

```text
XactLockTableWait(xmax)
```

为什么还要 `LockTuple()`？

原因是公平性。

如果多个 waiter 都等同一个 XID 结束，它们会一起醒来。

醒来后谁先拿到 buffer、谁先写 header，不受行级排队控制。

共享锁还会放大问题。

连续不断的 share locker 可能让 exclusive locker 长期饥饿。

所以 PostgreSQL 使用两层协议：

```text
LockTuple()
XactLockTableWait() 或 MultiXactIdWait()
mark tuple as locked by me
UnlockTuple()
```

`LockTuple()` 不是长期行锁记录。

它提供：

- wait queue。
- 排队顺序。
- 与 `NOWAIT` / `SKIP LOCKED` 结合的非阻塞判断。

长期状态仍是：

```text
xmax + infomask
或
xmax as MultiXactId + members
```

这就是为什么 `pg_locks` 不是 row lock 真相的完整来源。

有些 tuple 已被 header 标记为锁住，但当前不一定有持久的 lock table row。

## 10. MultiXactStatus 与 LockTupleMode 的边界

`LockTupleMode` 是调用方想要的强度。

`MultiXactStatus` 是记录在 MultiXact 成员里的状态。

二者不是同一个 enum。

映射关系大致是：

| `LockTupleMode` | 纯锁状态 | 更新状态 |
| --- | --- | --- |
| `LockTupleKeyShare` | `MultiXactStatusForKeyShare` | 无效 |
| `LockTupleShare` | `MultiXactStatusForShare` | 无效 |
| `LockTupleNoKeyExclusive` | `MultiXactStatusForNoKeyUpdate` | `MultiXactStatusNoKeyUpdate` |
| `LockTupleExclusive` | `MultiXactStatusForUpdate` | `MultiXactStatusUpdate` |

为什么要区分 pure lock 和 update status？

因为事务结束后的语义不同。

纯 locker 结束后，锁语义消失。

updater 提交后，更新语义仍然存在。

同样是成员 XID：

```text
ForNoKeyUpdate
  -> 事务结束后只是锁没了

NoKeyUpdate
  -> 事务提交后旧 tuple version 已被更新
```

所以 MultiXact 成员状态不仅是冲突矩阵输入。

它也是可见性和 freeze 的长期语义输入。

## 11. 正确性机制层次

行锁冲突不是一个机制单独保证的。

它由几层配合完成。

第一层是 SQL 到 rowmark 的语义转换。

`nodeLockRows.c`、`nodeModifyTable.c` 和 RI trigger 选择合适模式。

如果这一层选错，底层无法知道真正不变量。

第二层是 heap tuple header。

它保存长期状态：

```text
xmax
HEAP_XMAX_LOCK_ONLY
HEAP_XMAX_IS_MULTI
HEAP_XMAX_*_LOCK
HEAP_KEYS_UPDATED
```

第三层是 MultiXact。

多个成员共享同一行时，单个 `xmax` 指向 MultiXactId。

成员状态保存每个事务的锁强度或更新语义。

第四层是 lock manager。

`LOCKTAG_TUPLE` 负责等待排队。

它避免饥饿和冲突者无序竞争。

第五层是事务状态。

`TransactionIdIsInProgress()`、`TransactionIdDidCommit()`、`TransactionIdDidAbort()` 决定成员是否仍有意义。

第六层是重查。

等待释放 buffer 后，别人可能改过 header。

所以必须重读 `xmax` 和 infomask。

## 12. 错误路径 / 异常路径 / fallback

### 12.1 `NOWAIT`

`NOWAIT` 对应不愿等待。

如果 tuple lock 或 XID wait 会阻塞，系统返回错误。

这不是 visibility 错误。

这是 lock acquisition policy。

### 12.2 `SKIP LOCKED`

`SKIP LOCKED` 遇到不能立即锁住的行时跳过。

executor 看到 `TM_WouldBlock` 后取下一行。

这会改变查询结果集合。

它适合队列表 workload。

但不能用于需要完整结果的业务语义。

### 12.3 READ COMMITTED 追最新版本

READ COMMITTED 下，如果目标行被并发更新，锁定路径可能追到最新版本。

这依赖：

```text
TUPLE_LOCK_FLAG_FIND_LAST_VERSION
```

后续如果 traversed，需要 EPQ 重新判断 quals。

本节不展开 EPQ。

第 23 节会专门讲。

### 12.4 事务级 snapshot 下报错

REPEATABLE READ / SERIALIZABLE 使用事务级 snapshot。

如果目标行被并发更新或删除，很多路径不能像 READ COMMITTED 一样接受最新版本。

它会报：

```text
could not serialize access due to concurrent update
could not serialize access due to concurrent delete
```

这是 snapshot 稳定性和写冲突之间的边界。

### 12.5 自己已经锁住

同一事务可能再次申请相同或更弱锁。

如果已持有足够强的锁，`heap_lock_tuple()` 可以直接成功。

如果要升级，源码会小心跳过某些 `LockTuple()` 排队，以避免自死锁。

## 13. 成本、资源与跨模块传播

### 13.1 不同模式减少不必要等待

四种模式的收益是并发性。

典型收益：

```text
child insert key-share
  不阻塞 parent 非 key update
```

如果没有 no-key update，所有 parent update 都会阻塞外键检查。

高写入系统里这会明显放大等待。

### 13.2 冲突判断成本

单个 locker 时，判断主要看 infomask 和 XID 状态。

MultiXact 时，要读 members。

成员多时：

- `GetMultiXactIdMembers()` 成本增加。
- `DoesMultiXactIdConflict()` 要遍历成员。
- `MultiXactIdWait()` 可能等待多个成员。

成本随同一行上的并发 locker 数增加。

### 13.3 lock manager 成本

`LOCKTAG_TUPLE` 不为每个锁长期存在。

但等待时会进入 lock manager。

高冲突热点行会把成本传到：

- lock table hash。
- wait queue。
- deadlock detector。
- `pg_locks` 可见状态。

### 13.4 WAL 成本

锁写 tuple header。

tuple header 写入可能需要 WAL。

因此大量 `SELECT FOR UPDATE` 或 FK 检查不只是 CPU 等待问题。

也可能带来 WAL 带宽和 checkpoint 压力。

### 13.5 VACUUM 成本

MultiXact 和 old `xmax` 会进入 freeze / cleanup。

行锁热点 workload 可能推高 `relminmxid` 压力。

这会传播到 autovacuum 和 `pg_multixact` 截断。

## 14. 观测与诊断入口

### 14.1 SQL 侧观察

可以观察：

```sql
SELECT pid, wait_event_type, wait_event, query
FROM pg_stat_activity
WHERE wait_event_type = 'Lock';
```

再看：

```sql
SELECT locktype, mode, granted, relation, page, tuple, transactionid, pid
FROM pg_locks
WHERE NOT granted OR locktype IN ('tuple', 'transactionid');
```

注意：

`pg_locks` 看到的是 lock manager 当前状态。

它不等于 tuple header 中所有 row lock 痕迹。

### 14.2 tuple header 观察

用 `pageinspect` 看：

```sql
SELECT lp, t_xmin, t_xmax, t_ctid, t_infomask, t_infomask2
FROM heap_page_items(get_raw_page('parent', 0));
```

如果 `xmax` 是 MultiXactId，可以看成员：

```sql
SELECT *
FROM pg_get_multixact_members('<mxid>'::xid);
```

这里仍要小心：

- hint bit 可能已更新。
- tuple 可能被 HOT pruning 改写 line pointer。
- 观察时机会影响结果。
- `xmax` 数字本身不能说明锁模式。

### 14.3 源码断点

建议断点：

```text
ExecLockRows
ExecUpdateLockMode
heap_lock_tuple
heap_acquire_tuplock
DoesMultiXactIdConflict
Do_MultiXactIdWait
compute_new_xmax_infomask
```

观察变量：

```text
mode
wait_policy
lockflags
infomask
infomask2
xwait
result
have_tuple_lock
skip_tuple_lock
```

诊断顺序：

```text
先确认 SQL 意图
  -> 确认 LockTupleMode
  -> 确认 tuple header 当前状态
  -> 如果 MultiXact，展开成员
  -> 判断是否符合矩阵冲突
  -> 看等待策略是 block、skip 还是 error
```

## 15. 常见误区

误区一：

```text
FOR KEY SHARE 是弱读锁，不重要。
```

错误。

它是外键正确性的关键锁。

它弱，是因为它只保护 key 不消失。

不是因为它可以被忽略。

误区二：

```text
UPDATE 一定等价于 FOR UPDATE。
```

错误。

不修改 key 的 UPDATE 使用 no-key update 语义。

这允许它和 key-share 共存。

误区三：

```text
FOR SHARE 和 FOR KEY SHARE 差不多。
```

错误。

`FOR SHARE` 阻止所有 UPDATE。

`FOR KEY SHARE` 只阻止 key-changing update 和 delete。

误区四：

```text
pg_locks 里没有 tuple lock，就说明行没被锁。
```

错误。

长期状态可能只在 tuple header / MultiXact。

lock manager 对象主要用于等待排队。

误区五：

```text
冲突矩阵是 SQL 层规则。
```

不完整。

SQL 层选择模式。

冲突执行在 heap AM、MultiXact 和 lock manager 中完成。

## 16. 课堂实验

### 实验一：key-share 与 no-key update 兼容

目标：

```text
验证外键检查保护 key，而不阻止非 key 更新。
```

步骤：

```sql
DROP TABLE IF EXISTS child;
DROP TABLE IF EXISTS parent;
CREATE TABLE parent(id int primary key, payload text);
CREATE TABLE child(id int primary key, parent_id int references parent(id));
INSERT INTO parent VALUES (1, 'a');

-- Session A
BEGIN;
INSERT INTO child VALUES (1, 1);

-- Session B
UPDATE parent SET payload = 'b' WHERE id = 1;
```

观察：

- Session B 不应因为 key-share 语义而长期等待。
- 如果触发器时机或约束延迟改变，观察点要结合 `pg_locks` 和事务边界解释。

### 实验二：key-share 与 delete 冲突

步骤：

```sql
-- Session A
BEGIN;
SELECT * FROM parent WHERE id = 1 FOR KEY SHARE;

-- Session B
DELETE FROM parent WHERE id = 1;
```

观察：

- DELETE 需要等待 Session A。
- `pg_locks` 可能看到 transactionid 或 tuple lock 等待。

源码回读：

```text
heap_lock_tuple()
  -> mode == LockTupleKeyShare 分支
  -> HEAP_KEYS_UPDATED 判断
```

### 实验三：share 与 no-key update 冲突

步骤：

```sql
-- Session A
BEGIN;
SELECT * FROM parent WHERE id = 1 FOR SHARE;

-- Session B
UPDATE parent SET payload = 'c' WHERE id = 1;
```

观察：

- `FOR SHARE` 比 key-share 强。
- 它会阻止 no-key update。

### 实验四：NOWAIT 与 SKIP LOCKED

步骤：

```sql
-- Session A
BEGIN;
SELECT * FROM parent WHERE id = 1 FOR UPDATE;

-- Session B
SELECT * FROM parent WHERE id = 1 FOR UPDATE NOWAIT;

-- Session C
SELECT * FROM parent WHERE id = 1 FOR UPDATE SKIP LOCKED;
```

观察：

- NOWAIT 报错。
- SKIP LOCKED 跳过。
- 两者不改变冲突矩阵，只改变等待策略。

### 实验五：断点跟读矩阵执行

断点：

```text
ExecLockRows
heap_lock_tuple
heap_acquire_tuplock
DoesMultiXactIdConflict
```

对四个 SQL 分别运行：

```sql
SELECT * FROM parent WHERE id = 1 FOR KEY SHARE;
SELECT * FROM parent WHERE id = 1 FOR SHARE;
SELECT * FROM parent WHERE id = 1 FOR NO KEY UPDATE;
SELECT * FROM parent WHERE id = 1 FOR UPDATE;
```

画出：

```text
SQL clause
  -> ROW_MARK_*
  -> LockTupleMode
  -> heavyweight LOCKMODE
  -> MultiXactStatus
  -> infomask bits
```

## 17. 讨论题

1. 为什么外键检查选择 `FOR KEY SHARE`，而不是 `FOR SHARE`？

2. 如果 PostgreSQL 只有一个 row exclusive lock，哪些 workload 会产生不必要等待？

3. `LockTupleMode` 和 heavyweight `LOCKMODE` 的边界是什么？

4. 为什么 `LOCKTAG_TUPLE` 不作为长期行锁记录？

5. 为什么等待一个 XID 结束后仍需要 tuple header 重查？

6. `FOR NO KEY UPDATE` 的“不改 key”边界来自 SQL 语义、索引定义还是 executor 判断？

7. 为什么 `FOR SHARE` 会阻止 no-key update，而 `FOR KEY SHARE` 不阻止？

8. `SKIP LOCKED` 适合什么业务，不适合什么业务？

## 18. 本节小结

本节把 `xmax` 的多义字段推进到 row lock 模式。

核心链路是：

```text
SQL rowmark 或 UPDATE/DELETE 意图
  -> LockTupleMode
  -> tupleLockExtraInfo[]
  -> MultiXactStatus
  -> heap_lock_tuple()
  -> LOCKTAG_TUPLE 排队
  -> tuple header / MultiXact 记录长期状态
```

四种模式保护四种不同不变量：

- `FOR KEY SHARE` 保护 key 不消失。
- `FOR SHARE` 保护整行不被修改。
- `FOR NO KEY UPDATE` 允许非 key 更新，同时阻止其它写者。
- `FOR UPDATE` 保护删除或 key update 的最强语义。

正确性依赖多层机制：

- executor 选择模式。
- heap AM 判断 tuple 状态。
- infomask 解释 `xmax`。
- MultiXact 保存多成员。
- lock manager 负责等待排队。
- 事务状态决定成员是否仍有效。
- 等待后重查关闭 race。

可观测入口包括：

- `pg_stat_activity`。
- `pg_locks`。
- `pageinspect`。
- `pg_get_multixact_members()`。
- gdb 断点。

但单个入口都不是完整因果。

本节的可迁移模型是：

```text
并发控制里的“锁强度”应该对应业务不变量；
强度过弱会破坏正确性；
强度过强会制造无谓等待；
PostgreSQL 的四级 tuple lock 就是在这两者之间压缩出的有限语义集合。
```
