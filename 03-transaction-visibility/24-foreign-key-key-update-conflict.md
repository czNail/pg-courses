# PostgreSQL foreign key 行锁与 key update 冲突

## 课程定位

前置知识：已经理解 `xmax`、row lock 模式、MultiXact 成员，以及 UPDATE / DELETE 冲突后 READ COMMITTED 的 EPQ 边界。

本节唯一主问题：

```text
外键检查为什么必须锁住 referenced row 的 key，而不能只用 MVCC snapshot 查到 parent row 后就结束？
```

本节围绕的核心矛盾：

```text
外键检查希望尽量不阻塞 parent 表的普通更新；
但它必须保证 child row 引用的 parent key 不会在检查和提交之间被并发 DELETE 或 key UPDATE 搬走。
```

学完后应能独立判断：

- 为什么 `RI_FKey_check()` 生成或执行的检查会拿 `FOR KEY SHARE` 语义。
- 为什么 `FOR KEY SHARE` 能避免 referenced key 消失。
- 为什么它不应该阻塞 parent row 的非 key 更新。
- 为什么 parent key update / delete 必须和 child insert / update 冲突。
- 为什么外键检查需要触发器、SPI 或直接索引 fast path，但正确性边界仍然落到 tuple lock。
- 为什么 READ COMMITTED 下还要处理 parent row 被并发更新后的重查。
- 为什么事务级 snapshot 下 RI crosscheck 可能转成 serialization failure。
- 为什么 foreign key 不是单纯 planner / catalog 约束，而是 executor runtime 并发协议。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

第 20-23 节建立了 row lock 的底层模型。

现在进入一个具体应用：

```text
foreign key 如何利用 row lock 保护引用完整性？
```

外键看起来是 SQL 约束。

但在并发系统里，它不是一个静态 catalog 规则。

它必须处理这样的时间线：

```text
Session A 插入 child(parent_id = 1)
  -> 看到 parent(id = 1)

Session B 同时删除 parent(id = 1)
  -> 如果两边都提交，引用完整性可能破坏
```

只查 snapshot 不够。

因为 snapshot 只能说明：

```text
检查这一刻，当前语句能看到 parent row。
```

它不能保证：

```text
parent key 在当前事务提交前不会被另一个事务删除或改掉。
```

PostgreSQL 的解法是把外键检查落到 row lock：

```text
child INSERT / UPDATE
  -> RI trigger 检查 parent key
  -> 找到 parent row
  -> 对 parent row 申请 LockTupleKeyShare
  -> parent DELETE / key UPDATE 必须等待
  -> parent no-key UPDATE 可以继续
```

本节不展开所有 ON DELETE / ON UPDATE action。

CASCADE、SET NULL、RESTRICT、NO ACTION 都有各自触发器。

本节只抓住 key-share 行锁如何保护“parent key 不消失”。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
foreign key check 先用 SPI 查询或直接索引 fast path 找 referenced parent row；
找到后通过 table_tuple_lock(... LockTupleKeyShare ...) 锁住 parent tuple；
这个锁与 parent DELETE / key UPDATE 冲突，但与 no-key UPDATE 兼容；
如果 parent row 被并发更新，READ COMMITTED 下追最新版本并重查 key，事务级 snapshot 下按 RI crosscheck 报 serialization failure。
```

外键需要保护的不变量是：

```text
child row 提交时，它引用的 parent key 必须存在。
```

这个不变量不是普通 SELECT 可以保证的。

普通 SELECT 的 snapshot 是读视图。

它不会阻止另一个事务在检查后删除 parent row。

因此外键检查要把“读到 parent”升级成：

```text
读到 parent，并在并发写入协议里声明我依赖这个 key。
```

`FOR KEY SHARE` 正好表达这个声明。

它阻止：

- DELETE parent row。
- UPDATE parent key。

它允许：

- UPDATE parent 非 key 列。

本节的系统规律是：

```text
约束检查不是只读判断；
只要约束依赖未来提交前仍成立的事实，就必须把读结果转成某种并发占用协议。
```

## 3. 核心文件分工与阅读顺序

| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/commands/tablecmds.c` | `ATAddForeignKeyConstraint()` 创建约束和 RI trigger 的入口。 |
| 2 | `src/backend/utils/adt/ri_triggers.c` | `RI_FKey_check()`、`ri_PerformCheck()`、`ri_LockPKTuple()`。 |
| 3 | `src/include/catalog/pg_proc.dat` | RI trigger 函数注册：`RI_FKey_check_ins`、`RI_FKey_check_upd` 等。 |
| 4 | `src/backend/commands/trigger.c` | trigger 执行框架与 AFTER ROW trigger 调用。 |
| 5 | `src/backend/executor/nodeModifyTable.c` | INSERT / UPDATE 后触发 RI 检查，crosscheck snapshot 边界。 |
| 6 | `src/backend/access/heap/heapam.c` | `heap_lock_tuple()`、key-share 与 key update/delete 冲突。 |
| 7 | `src/backend/access/heap/README.tuplock` | `FOR KEY SHARE` 与 RI checks 的语义说明。 |
| 8 | `src/backend/access/heap/heapam_visibility.c` | parent row 可见性与锁定前检查。 |
| 9 | `src/backend/storage/lmgr/lmgr.c` | tuple lock wait 与 XID wait。 |
| 10 | `src/backend/access/transam/multixact.c` | 多个 child 同时引用同一 parent row 时的 MultiXact。 |

推荐阅读顺序：

```text
先读 README.tuplock 中 FOR KEY SHARE 的一句语义
  -> 读 ri_triggers.c 中 RI_FKey_check()
  -> 找到生成 SELECT ... FOR KEY SHARE 的 query
  -> 读 ri_LockPKTuple() 的 direct fast path
  -> 回到 heap_lock_tuple() 看 key-share 如何和 key update 冲突
  -> 最后读 tablecmds.c 了解约束如何安装触发器
```

不要把本节写成“外键触发器函数列表”。

关键不是函数名多少。

关键是：

```text
parent key 的存在性如何从 snapshot 事实变成 row lock 不变量。
```

## 4. 一个 runtime 现象先定锚

Session 0：

```sql
DROP TABLE IF EXISTS child_fk;
DROP TABLE IF EXISTS parent_fk;

CREATE TABLE parent_fk(id int primary key, payload text);
CREATE TABLE child_fk(id int primary key, parent_id int references parent_fk(id));
INSERT INTO parent_fk VALUES (1, 'a');
```

Session A：

```sql
BEGIN;
INSERT INTO child_fk VALUES (1, 1);
-- 保持事务打开。
```

Session B：

```sql
DELETE FROM parent_fk WHERE id = 1;
```

Session B 会等待。

因为 Session A 的外键检查已经对 `parent_fk(id=1)` 拿了 key-share 类行锁。

现在换一个操作。

Session C：

```sql
UPDATE parent_fk
SET payload = 'b'
WHERE id = 1;
```

这个非 key 更新不应该被同样方式阻塞。

因为它不破坏 child row 引用的 key。

再换成：

```sql
UPDATE parent_fk
SET id = 2
WHERE id = 1;
```

这个 key update 必须等待或在约束语义下失败。

这个实验说明：

```text
foreign key 需要的不是“parent row 不能动”；
它需要的是“referenced key 不能消失或改变”。
```

`FOR KEY SHARE` 正是这个不变量的最小 row lock。

## 5. 外键检查为什么不是普通 SELECT

设想外键检查只执行：

```sql
SELECT 1
FROM parent_fk
WHERE id = $1;
```

在 MVCC 下，这只能证明：

```text
当前 snapshot 中存在 parent row。
```

但并发时间线可能是：

```text
T1 child insert 查到 parent
T2 delete parent
T2 commit
T1 commit
```

如果 T1 没有锁住 parent key，两个事务都可能提交。

最后 child 指向不存在的 parent。

这就是 MVCC snapshot 和约束正确性之间的空洞。

因此检查 query 要带：

```sql
FOR KEY SHARE OF x
```

它把只读检查变成：

```text
我依赖这个 key 继续存在；
任何会删除或改变 key 的事务必须和我协调。
```

这不是为了可见性。

这是为了提交前的不变量保护。

## 6. `RI_FKey_check()` 的主流程

child 表 INSERT 或 UPDATE 时，会执行 RI check trigger。

入口包括：

```text
RI_FKey_check_ins()
RI_FKey_check_upd()
  -> RI_FKey_check()
```

`RI_FKey_check()` 先做几个边界判断。

第一，取约束信息：

```text
ri_FetchConstraintInfo()
```

第二，确认当前触发 tuple 仍然有效。

源码会按 `SnapshotSelf` 检查该 child row。

如果它已经被当前事务后续动作删除或替换，延迟触发器不需要再检查旧版本。

第三，处理 NULL。

MATCH SIMPLE 下，某些 NULL 可以直接通过。

MATCH FULL 下，混合 NULL / non-NULL 会报错。

第四，选择执行路径。

本地源码中有直接索引 fast path：

```text
ri_fastpath_is_applicable()
  -> ri_FastPathBatchAdd()
  -> ri_FastPathCheck()
```

也有传统 SPI path：

```text
SPI_connect()
  -> 构造 SELECT ... FOR KEY SHARE
  -> ri_PerformCheck()
```

两条路径不同，但正确性边界相同：

```text
找到 parent row 后，必须取得 key-share 行锁。
```

## 7. SPI path：生成 `FOR KEY SHARE`

在 SPI path 中，`RI_FKey_check()` 构造的核心 query 形态是：

```sql
SELECT 1
FROM [ONLY] pktable x
WHERE pkatt1 = $1
  [AND ...]
FOR KEY SHARE OF x
```

如果是 temporal FK，query 形态会更复杂。

但关键仍然是：

```text
匹配 parent row 后，对 parent row 加 key-share。
```

`pk_rel` 以 `RowShareLock` 打开。

这是因为最终的 SELECT 会在 parent row 上拿 key-share。

`ri_PerformCheck()` 负责执行 prepared plan，并根据结果决定是否报 foreign key violation。

这里有一个重要边界：

```text
SQL query 的 FOR KEY SHARE 不是为了让 executor 返回不同数据；
它是为了让 heap tuple lock 协议参与约束保护。
```

如果没有这个 row lock，query 仍可能查到 parent。

但查到不等于提交前仍存在。

## 8. fast path：直接索引探测后锁 parent tuple

本地源码还包含 RI fast path。

对适用的非分区、非 temporal 外键，它可以绕过 SPI executor 开销。

主流程是：

```text
ri_FastPathCheck()
  -> 打开 parent relation 和唯一索引
  -> 构造 scan key
  -> ri_FastPathProbeOne()
     -> index_getnext_slot()
     -> ri_LockPKTuple()
```

批量路径：

```text
ri_FastPathBatchAdd()
  -> ri_FastPathBatchFlush()
  -> ri_FastPathFlushArray() 或 ri_FastPathFlushLoop()
  -> ri_FastPathProbeOne()
```

`ri_LockPKTuple()` 是关键。

它调用：

```text
table_tuple_lock(
  pk_rel,
  &slot->tts_tid,
  snap,
  slot,
  GetCurrentCommandId(false),
  LockTupleKeyShare,
  LockWaitBlock,
  lockflags,
  &tmfd
)
```

也就是说：

```text
fast path 只是减少执行器和 SPI 成本；
不是减少正确性锁。
```

如果 parent row 被并发更新，`tmfd.traversed` 会告诉调用者追过更新链。

fast path 随后要 recheck key。

这和第 23 节的 EPQ 思路一致：

```text
找到的行被并发更新后，必须确认最新版本仍满足引用 key。
```

## 9. parent DELETE / key UPDATE 如何冲突

child insert 持有 parent row 的 key-share。

parent DELETE 需要最强语义。

它等价于 referenced key 消失。

因此冲突矩阵中：

```text
KEY SHARE vs UPDATE
  -> conflict
```

这里的 UPDATE 是 `LockTupleExclusive` 语义。

parent key UPDATE 也使用 `LockTupleExclusive`。

因为它会设置：

```text
HEAP_KEYS_UPDATED
```

parent no-key UPDATE 使用：

```text
LockTupleNoKeyExclusive
```

它与 key-share 兼容。

这正是外键系统的性能关键：

```text
child 引用 parent key
  不应该阻止 parent payload 更新
  但必须阻止 parent key 消失
```

`heap_lock_tuple()` 的 key-share 分支会特别看：

```text
HEAP_KEYS_UPDATED
```

如果已有 update 没改 key，key-share 可以不等待。

如果改了 key，就必须处理冲突。

## 10. 并发 parent update 后为什么要重查 key

READ COMMITTED 下，外键检查可能遇到 parent row 被并发更新。

如果只是 non-key update，key 仍然相同。

检查可以继续。

但系统必须证明这一点。

SPI path 通过 executor row lock / EvalPlanQual 处理。

fast path 中 `ri_LockPKTuple()` 返回：

```text
concurrently_updated = tmfd.traversed
```

随后 `recheck_matched_pk_tuple()` 用实际找到的 parent key 重新核对 index match。

原因是：

```text
index scan 最初命中的 TID 可能不是最终锁住的版本；
沿 update chain 追到新版本后，必须确认它仍满足原外键 key。
```

这正是外键和 MVCC 的关键交界：

```text
snapshot 能找到一个版本；
row lock 可能追到另一个版本；
约束必须对最终被保护的版本成立。
```

## 11. 事务级 snapshot 与 RI crosscheck

`nodeModifyTable.c` 中 `table_tuple_update()` 调用带有：

```text
estate->es_crosscheck_snapshot
```

注释说明这是 referential integrity updates 在 transaction-snapshot mode 下需要的特殊行为。

它用于确认被更新行对 crosscheck snapshot 是否可见。

如果不可见，会抛出 serialization failure。

为什么需要这个边界？

在事务级 snapshot 下，系统不能把并发提交后的新现实悄悄混入当前事务。

外键动作涉及 parent / child 两张表。

如果一个事务级 snapshot 的约束检查使用了不一致的可见性世界，就可能破坏约束判断。

所以 PostgreSQL 在相关路径中把 RI crosscheck 显式传下去。

本节不展开所有 crosscheck 场景。

只记住：

```text
RI 不是普通 UPDATE；
它有额外 snapshot consistency 要求。
```

## 12. 约束安装：触发器从哪里来

外键约束创建入口在 `tablecmds.c`。

核心函数是：

```text
ATAddForeignKeyConstraint()
```

它负责：

- 打开 referenced table。
- 检查 referenced relation 类型。
- 找到或验证唯一索引。
- 创建 pg_constraint 元数据。
- 创建 action triggers。
- 创建 check triggers。
- 处理 partitioned table 的递归约束和触发器。

外键不是 planner 每次临时想起来检查。

它被编译成系统触发器。

`pg_proc.dat` 中注册了 RI trigger 函数：

- `RI_FKey_check_ins`。
- `RI_FKey_check_upd`。
- `RI_FKey_cascade_del`。
- `RI_FKey_cascade_upd`。
- `RI_FKey_noaction_del`。
- `RI_FKey_noaction_upd`。
- `RI_FKey_restrict_del`。
- `RI_FKey_restrict_upd`。
- `RI_FKey_setnull_*`。
- `RI_FKey_setdefault_*`。

本节关注 check trigger。

action trigger 负责 parent row delete/update 时对 child row 做限制、级联或置空。

它们共享同一个核心事实：

```text
外键是 runtime 触发器 + row lock + snapshot 共同保证的。
```

## 13. 生命周期 / ownership / cleanup

### 13.1 约束元数据

owner 是 catalog。

`pg_constraint` 保存外键定义。

`pg_trigger` 保存触发器。

relcache / syscache 缓存约束信息。

invalidation 负责让缓存失效。

### 13.2 单次检查状态

单次 RI check 使用：

- trigger data。
- `RI_ConstraintInfo`。
- prepared SPI plan 或 fast path metadata。
- parent relation ref。
- child tuple slot。
- snapshot。
- tuple lock。

这些状态大多是 backend-local。

### 13.3 parent row lock

parent row key-share 记录在 tuple header / MultiXact 中。

如果多个 child 事务同时引用同一个 parent row，可能形成 MultiXact。

事务结束后锁语义消失。

tuple header 里的痕迹可能后续被清理。

### 13.4 ERROR cleanup

如果 RI check 抛出 foreign key violation：

```text
当前语句 ERROR
事务可能 abort
SPI context 清理
ResourceOwner 释放 relation、snapshot、buffer pin
lock manager 释放当前事务锁
tuple header 中当前事务写过的 XID 后续按 aborted 解释
```

这也是 MVCC 设计的收益：

不需要回滚时逐字节撤销 tuple header。

### 13.5 deferred constraint

如果约束 deferrable，检查可能延后。

这会改变检查发生的时间。

但最终仍要在检查时证明 parent key 存在并拿到正确锁。

延迟不等于跳过并发协议。

## 14. 正确性机制层次

第一层：catalog 约束。

它定义哪些列引用哪些 parent key。

第二层：trigger。

它把 INSERT / UPDATE / DELETE 事件转成 runtime 检查。

第三层：snapshot。

它决定检查语句能看到哪些 parent row。

第四层：row lock。

`LockTupleKeyShare` 把“看见 parent”转成“parent key 暂时不能消失”。

第五层：conflict matrix。

key-share 与 delete / key update 冲突，与 no-key update 兼容。

第六层：update-chain recheck。

如果 parent row 被并发更新，必须追最新版本并重查 key。

第七层：MultiXact。

多个 child 事务同时锁同一 parent key 时，成员集合记录在 MultiXact。

第八层：transaction cleanup。

事务提交或 abort 决定锁成员和写入结果的最终语义。

## 15. 异常路径 / fallback

### 15.1 parent row 找不到

如果 check query 或 fast path 找不到 parent row，报 foreign key violation。

这是普通约束失败。

### 15.2 parent row 被并发删除

READ COMMITTED 下，如果最终锁不到有效 parent row，检查失败或跳过到 violation。

事务级 snapshot 下可能转成 serialization failure。

具体结果取决于触发器类型和 snapshot 边界。

### 15.3 parent row 被并发 non-key update

可以继续。

但必须确认最终版本的 key 仍匹配。

fast path 用 `concurrently_updated` 和 recheck。

SPI path 由 executor row locking / EPQ 处理。

### 15.4 parent row 被并发 key update

必须冲突。

因为 referenced key 已改变。

如果 key update 提交，child 原引用不再成立。

### 15.5 权限和 RLS fallback

`RI_Initial_Check()` 做整表初始验证时，如果权限或 RLS 不允许直接大查询，会返回 false，让调用者使用触发器方法。

这不是降低正确性。

这是换一条执行路径保持语义。

### 15.6 partitioned table

分区外键会创建更多 constraint rows 和 triggers。

检查路径可能跨 root、leaf、ancestor。

但每个实际 parent tuple 的 runtime 保护仍然落到 row lock。

## 16. 成本、资源与跨模块传播

### 16.1 每行检查成本

普通 INSERT child row 可能触发 parent lookup。

成本包括：

- constraint cache lookup。
- SPI plan 或 fast path metadata。
- parent index probe。
- tuple lock。
- update-chain recheck。

批量插入时，fast path 可以减少 executor overhead。

但不能省掉 tuple lock。

### 16.2 热点 parent key

很多 child row 引用同一个 parent row 时，会产生：

- 多个 key-share locker。
- MultiXact 成员增长。
- parent key update/delete 等待。
- `pg_multixact` SLRU 压力。

这类 workload 看起来是外键成本。

底层常常是 MultiXact 和 row lock 成本。

### 16.3 WAL 成本

key-share 锁可能写 tuple header。

tuple header 改动可能写 WAL。

所以高频 FK 检查也可能带来 WAL 增量。

### 16.4 autovacuum 传播

MultiXact 引用会影响 `relminmxid`。

大量外键 key-share 可能间接推动 MultiXact freeze 压力。

这会传播到 autovacuum。

### 16.5 schema 设计成本

外键列缺少合适索引时，parent action trigger 检查 child 表会更贵。

本节关注 child -> parent check。

但完整外键性能还包括 parent delete/update 时查 child rows。

诊断外键等待时必须区分这两个方向。

## 17. 观测与诊断入口

### 17.1 查看等待

```sql
SELECT pid, wait_event_type, wait_event, query
FROM pg_stat_activity
WHERE wait_event_type = 'Lock';
```

```sql
SELECT locktype, mode, granted, relation, page, tuple, transactionid, pid
FROM pg_locks
WHERE NOT granted OR locktype IN ('tuple', 'transactionid');
```

如果 parent delete 等 child insert，常见是 transactionid wait 或 tuple wait。

### 17.2 查看外键触发器

```sql
SELECT tgname, tgfoid::regproc, tgenabled, tgconstraint
FROM pg_trigger
WHERE tgrelid = 'child_fk'::regclass
   OR tgrelid = 'parent_fk'::regclass;
```

可以看到 RI trigger 函数。

### 17.3 查看 tuple header

```sql
CREATE EXTENSION IF NOT EXISTS pageinspect;

SELECT lp, t_xmax, t_infomask, t_infomask2
FROM heap_page_items(get_raw_page('parent_fk', 0));
```

如果 parent row 被多个 child 事务 key-share 锁住，`xmax` 可能是 MultiXactId。

### 17.4 查看 MultiXact 成员

```sql
SELECT *
FROM pg_get_multixact_members('<mxid>'::xid);
```

关注 mode 是否为 key-share 类。

### 17.5 源码断点

建议断点：

```text
RI_FKey_check
ri_PerformCheck
ri_FastPathProbeOne
ri_LockPKTuple
heap_lock_tuple
compute_new_xmax_infomask
DoesMultiXactIdConflict
```

观察：

```text
LockTupleKeyShare
lockflags
tmfd.traversed
concurrently_updated
HEAP_KEYS_UPDATED
```

## 18. 常见误区

误区一：

```text
外键检查就是 SELECT parent 是否存在。
```

错误。

它必须锁住 parent key。

误区二：

```text
外键会阻止 parent 行任何更新。
```

错误。

key-share 允许 no-key update。

误区三：

```text
fast path 绕过 SPI，所以也绕过行锁。
```

错误。

fast path 仍调用 `table_tuple_lock(... LockTupleKeyShare ...)`。

误区四：

```text
child insert 慢一定是 child 表问题。
```

不一定。

可能是 parent key 热点、MultiXact、parent lock wait 或 WAL。

误区五：

```text
约束定义在 catalog 里，所以正确性在 DDL 阶段已经解决。
```

错误。

DDL 只安装规则。

运行时正确性靠 trigger、snapshot 和 row lock。

## 19. 课堂实验

### 实验一：child insert 阻止 parent delete

步骤：

```sql
DROP TABLE IF EXISTS child_fk;
DROP TABLE IF EXISTS parent_fk;
CREATE TABLE parent_fk(id int primary key, payload text);
CREATE TABLE child_fk(id int primary key, parent_id int references parent_fk(id));
INSERT INTO parent_fk VALUES (1, 'a');

-- Session A
BEGIN;
INSERT INTO child_fk VALUES (1, 1);

-- Session B
DELETE FROM parent_fk WHERE id = 1;
```

观察 Session B 等待。

用 `pg_locks` 和 `pg_stat_activity` 解释等待。

### 实验二：child insert 不阻止 parent non-key update

步骤：

```sql
-- Session A 仍保持 child insert 未提交

-- Session C
UPDATE parent_fk SET payload = 'b' WHERE id = 1;
```

观察 non-key update 能继续。

回到第 21 节的矩阵解释。

### 实验三：child insert 阻止 parent key update

步骤：

```sql
-- Session D
UPDATE parent_fk SET id = 2 WHERE id = 1;
```

观察等待或后续约束行为。

源码回读：

```text
LockTupleKeyShare
  vs
LockTupleExclusive + HEAP_KEYS_UPDATED
```

### 实验四：多个 child 事务制造 MultiXact

步骤：

```sql
-- 多个 session 同时
BEGIN;
INSERT INTO child_fk VALUES (..., 1);
```

再看 parent tuple header：

```sql
SELECT lp, t_xmax, t_infomask, t_infomask2
FROM heap_page_items(get_raw_page('parent_fk', 0));
```

如果形成 MultiXact：

```sql
SELECT *
FROM pg_get_multixact_members('<mxid>'::xid);
```

### 实验五：断点看 fast path

断点：

```text
RI_FKey_check
ri_FastPathProbeOne
ri_LockPKTuple
heap_lock_tuple
```

观察：

```text
ri_LockPKTuple() 使用 LockTupleKeyShare
tmfd.traversed 如何影响 concurrently_updated
```

如果 fast path 条件不满足，再跟 SPI path。

## 20. 讨论题

1. 为什么外键检查不能只依赖 MVCC snapshot？

2. `FOR KEY SHARE` 保护的最小不变量是什么？

3. 为什么 parent non-key update 不应阻塞 child insert？

4. fast path 和 SPI path 的正确性共同点是什么？

5. parent row 被并发更新后，为什么要重查 key？

6. 多个 child 同时引用同一个 parent key，为什么会产生 MultiXact？

7. deferrable constraint 改变了什么，没有改变什么？

8. 诊断外键等待时如何区分 child -> parent check 和 parent -> child action trigger？

## 21. 本节小结

本节回答了外键行锁的核心问题：

```text
为什么 child row 检查 parent row 存在时，必须把读到 parent 转成 key-share 行锁？
```

核心链路是：

```text
child INSERT / UPDATE
  -> RI_FKey_check()
  -> SPI SELECT ... FOR KEY SHARE 或 fast path index probe
  -> table_tuple_lock(... LockTupleKeyShare ...)
  -> parent DELETE / key UPDATE 冲突
  -> parent no-key UPDATE 兼容
```

核心状态是：

- `RI_ConstraintInfo`。
- parent key values。
- child tuple slot。
- `LockTupleKeyShare`。
- `tmfd.traversed`。
- `HEAP_KEYS_UPDATED`。
- parent tuple `xmax`。
- MultiXact members。

ownership 和 cleanup 的边界是：

```text
constraint 元数据在 catalog；
检查状态在 backend-local trigger / SPI / fast path context；
parent row 锁记录在 tuple header / MultiXact；
事务结束释放语义锁；
tuple header 痕迹和 MultiXact SLRU 由后续 cleanup / freeze 处理。
```

错误路径的核心规律是：

```text
查到 parent 不是最终证明；
最终证明是锁住一个仍匹配 referenced key 的 parent version。
```

可观测入口包括：

- `pg_stat_activity`。
- `pg_locks`。
- `pg_trigger`。
- `pageinspect`。
- `pg_get_multixact_members()`。
- gdb 断点。

本节的可迁移模型是：

```text
约束检查如果依赖“某个事实在提交前仍成立”，就必须把 snapshot 中的读结果提升为并发控制对象；
外键通过 FOR KEY SHARE 把 parent key 的存在性变成了 row lock 不变量。
```
