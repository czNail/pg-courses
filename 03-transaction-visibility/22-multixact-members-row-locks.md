# PostgreSQL MultiXact 成员与共享行锁

## 课程定位

前置知识：已经理解 `xmax` 的多重语义，知道四种 `LockTupleMode` 的冲突矩阵，也知道 tuple header 只能保存一个 raw `xmax`。

本节唯一主问题：

```text
多个事务同时锁住同一行时，为什么单个 xmax 不够用，MultiXact 如何记录成员、模式和冲突关系？
```

本节围绕的核心矛盾：

```text
行锁需要允许多个兼容 locker 共存；
但 heap tuple header 只有一个 xmax slot，不能直接塞下多个 XID 和各自锁模式。
```

学完后应能独立判断：

- 什么情况下 `xmax` 从 XID 变成 MultiXactId。
- MultiXact 成员为什么要保存 `xid + status`。
- 为什么 `MultiXactIdExpand()` 创建新 MultiXact，而不是修改旧 MultiXact。
- 为什么 MultiXact 可以包含 updater。
- 为什么每个事务要设置 `OldestMemberMXactId`。
- 为什么读 MultiXact 之前要设置 `OldestVisibleMXactId`。
- 为什么 MultiXact 需要 WAL 和 SLRU。
- 为什么 `pg_multixact` 截断依赖 `relminmxid` / `datminmxid`。
- 为什么高并发 row lock 会带来 MultiXact wraparound 压力。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

第 20 节讲的是：

```text
xmax 不是单一语义；
它可能表示 locker、updater、deleter 或 MultiXact。
```

第 21 节讲的是：

```text
四种 row lock 模式保护不同不变量；
冲突矩阵决定哪些模式可以共存。
```

本节把这两个结论合起来。

如果两个事务都以兼容模式锁同一行：

```text
Session A: FOR SHARE
Session B: FOR SHARE
```

tuple header 不能同时写两个 XID。

如果覆盖 `xmax`，就会丢掉第一个 locker。

如果只保留第一个 locker，就无法表达第二个 locker。

如果把锁都放在 lock manager，锁数量会随被锁 tuple 数增长，shared memory 不可控。

PostgreSQL 的解法是：

```text
tuple header 的 xmax 保存一个 MultiXactId；
MultiXactId 指向 pg_multixact 中的一组成员；
每个成员保存一个 XID 和一个 MultiXactStatus；
冲突判断时展开成员，逐个判断是否仍有效、是否冲突。
```

本节不讲 MultiXact freeze 的全部边界。

那是第 25 节。

本节只讲创建、扩展、成员解释、等待和观测。

主线是：

```text
第一事务锁行
  -> xmax = first XID
第二事务兼容锁行
  -> compute_new_xmax_infomask()
  -> MultiXactIdCreate()
  -> xmax = new MultiXactId
第三事务申请冲突锁
  -> DoesMultiXactIdConflict()
  -> MultiXactIdWait()
  -> 等待成员结束后重查 tuple header
```

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
MultiXact 是 tuple xmax 的间接引用层；
它把多个 row lock / update member 记录成 xid + MultiXactStatus 数组；
tuple header 只保存 MultiXactId 和提示位；
heapam.c 在加锁、更新、等待、freeze 时展开成员，按成员状态和事务结果重建语义。
```

这个模型有四个关键不变量。

第一，MultiXact 成员不可原地追加。

旧 MultiXact 一旦创建，就不修改成员集合。

新增成员会创建新的 MultiXactId。

这样正在等待旧 MultiXact 的 backend 不会看到集合变化。

第二，一个 MultiXact 可以有多个 locker。

兼容 lock 可以共存。

冲突 lock 必须等待。

第三，一个 MultiXact 最多有一个 updater 语义。

源码创建成员时会检查不能有两个 updating member。

否则版本链就无法解释。

第四，成员结束不等于 tuple header 立即清理。

成员事务结束后：

- locker 的锁语义可以消失。
- updater 提交后仍然影响可见性。
- header 中的 MultiXactId 可能继续存在。
- VACUUM / freeze 后续负责清理或替换。

本节的系统规律是：

```text
当一个小字段无法直接表达集合状态时，系统会引入间接 ID；
但间接 ID 的生命周期、成员稳定性和截断边界会成为新的正确性成本。
```

## 3. 核心文件分工与阅读顺序

| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/access/heap/README.tuplock` | MultiXact 为什么存在、成员含义、`relminmxid` 基本说明。 |
| 2 | `src/include/access/multixact.h` | `MultiXactStatus`、`MultiXactMember`、公开入口。 |
| 3 | `src/backend/access/heap/heapam.c` | `compute_new_xmax_infomask()`、`DoesMultiXactIdConflict()`、`MultiXactIdWait()`。 |
| 4 | `src/backend/access/transam/multixact.c` | `MultiXactIdCreate()`、`MultiXactIdExpand()`、`MultiXactIdCreateFromMembers()`、SLRU 写入。 |
| 5 | `src/include/access/multixact_internal.h` | offsets / members 文件布局相关内部宏。 |
| 6 | `src/backend/utils/adt/multixactfuncs.c` | `pg_get_multixact_members()`、`pg_get_multixact_stats()` 观测入口。 |
| 7 | `src/backend/storage/lmgr/lmgr.c` | `XactLockTableWait()` 作为成员等待基础。 |
| 8 | `src/backend/commands/vacuum.c` | `OldestMxact` 和 `MultiXactCutoff` 的消费入口。 |
| 9 | `src/backend/access/heap/vacuumlazy.c` | lazy VACUUM 后续如何推进 `relminmxid`。 |
| 10 | `src/backend/access/rmgrdesc/mxactdesc.c` | MultiXact WAL 描述，便于读日志。 |

推荐阅读顺序：

```text
先读 README.tuplock 的 MultiXacts 段
  -> 读 multixact.h 的 status 枚举
  -> 读 heapam.c 的 compute_new_xmax_infomask()
  -> 读 multixact.c 的 Create / Expand / CreateFromMembers
  -> 回到 heapam.c 看冲突和等待
  -> 最后读 multixactfuncs.c 做观测
```

不要把 MultiXact 当成普通 lock manager 对象。

它持久化到 `pg_multixact`。

它能被 WAL 重放。

它能在 crash 后继续被 tuple visibility 解释。

## 4. 一个 runtime 现象先定锚

Session 0：

```sql
DROP TABLE IF EXISTS mx_demo;
CREATE TABLE mx_demo(id int primary key, payload text);
INSERT INTO mx_demo VALUES (1, 'a');
CREATE EXTENSION IF NOT EXISTS pageinspect;
```

Session A：

```sql
BEGIN;
SELECT * FROM mx_demo WHERE id = 1 FOR SHARE;
```

Session B：

```sql
BEGIN;
SELECT * FROM mx_demo WHERE id = 1 FOR SHARE;
```

两个 `FOR SHARE` 兼容。

它们都应该能拿到锁。

Session C：

```sql
SELECT lp, t_xmin, t_xmax, t_ctid, t_infomask, t_infomask2
FROM heap_page_items(get_raw_page('mx_demo', 0));
```

观察到的 `t_xmax` 可能不再是某一个 locker XID。

它可能是 MultiXactId。

如果是 MultiXact，可以进一步：

```sql
SELECT *
FROM pg_get_multixact_members('<mxid>'::xid);
```

然后 Session D：

```sql
UPDATE mx_demo SET payload = 'b' WHERE id = 1;
```

`FOR SHARE` 与 no-key update 冲突。

Session D 应等待 A / B 释放。

这个实验说明：

```text
兼容 locker 可以共存；
共存状态不能塞进单个 XID；
MultiXact 把集合状态间接挂到 xmax 上；
后续冲突者必须展开成员判断和等待。
```

## 5. MultiXact 成员状态

`MultiXactMember` 的核心语义是：

```text
member.xid
member.status
```

`xid` 表示哪个事务或子事务持有这个成员。

`status` 表示这个成员持有哪种锁或更新语义。

`MultiXactStatus` 不是只有四个锁模式。

它有六个值：

```text
MultiXactStatusForKeyShare
MultiXactStatusForShare
MultiXactStatusForNoKeyUpdate
MultiXactStatusForUpdate
MultiXactStatusNoKeyUpdate
MultiXactStatusUpdate
```

前四个是纯锁。

后两个是 updater。

这一区分非常重要。

事务结束后：

```text
纯锁成员:
  事务结束后锁语义消失。

updater 成员:
  如果提交，旧 tuple version 的更新语义继续存在。
  如果 abort，可以忽略。
```

因此 `DoesMultiXactIdConflict()` 和 freeze 路径都要区分：

```text
ISUPDATE_from_mxstatus(status)
```

一个 MultiXact 可以包含多个 locker。

但不能包含两个 updater。

`MultiXactIdCreateFromMembers()` 会检查这一点。

因为一个 heap tuple version 只能被一个后继更新链路终结。

如果存在多个 updater，`t_ctid` 和 visibility 语义就无法唯一解释。

## 6. 从单 XID 到 MultiXact

考虑一个 tuple 初始状态：

```text
xmax invalid
```

Session A：

```sql
SELECT * FROM mx_demo WHERE id = 1 FOR SHARE;
```

`compute_new_xmax_infomask()` 看到旧 `xmax` invalid。

它可以直接写：

```text
xmax = A 的 XID
HEAP_XMAX_LOCK_ONLY
HEAP_XMAX_SHR_LOCK
```

现在 Session B 也要 `FOR SHARE`。

旧 `xmax` 是 A。

A 仍在运行。

B 的锁模式与 A 兼容。

但 tuple header 不能同时保存 A 和 B。

所以分支进入：

```text
旧 xmax 是 in-progress XID
  -> 根据旧 infomask 推导 old_status
  -> 根据新 mode 推导 new_status
  -> MultiXactIdCreate(xmax, old_status, add_to_xmax, new_status)
  -> GetMultiXactIdHintBits()
```

新 header 变成：

```text
xmax = new MultiXactId
HEAP_XMAX_IS_MULTI
HEAP_XMAX_LOCK_ONLY
HEAP_XMAX_SHR_LOCK
```

如果后来 Session C 加入相同或兼容锁：

```text
旧 xmax 是 MultiXact
  -> MultiXactIdExpand(old_multi, C, new_status)
  -> 创建新 MultiXactId
  -> tuple header 指向新 MultiXactId
```

注意：

```text
不是修改 old_multi。
```

旧 MultiXact 仍然稳定。

这对等待者正确性非常关键。

## 7. `MultiXactIdCreateFromMembers()` 写了什么

`MultiXactIdCreateFromMembers()` 做三件事。

第一，查 local cache。

如果当前 backend 已经创建过完全相同成员集合，可以复用。

这减少重复 SLRU 写入。

第二，分配新的 MultiXactId 和 member offset。

MultiXact 存储拆成两个 SLRU：

```text
offsets:
  MultiXactId -> members 起始 offset

members:
  offset -> xid + status flags
```

为什么要拆？

因为每个 MultiXact 的成员数量可变。

offsets 给出某个 MultiXact 的成员起点。

下一个 MultiXact 的 offset 给出成员终点。

第三，写 WAL 和 SLRU。

创建 MultiXact 不是临时内存动作。

tuple header 可能持久化这个 ID。

崩溃恢复后，系统仍必须能展开它。

所以它要写：

```text
RM_MULTIXACT_ID
XLOG_MULTIXACT_CREATE_ID
```

然后把 offsets 和 members 写入 SLRU。

这解释了一个性能现象：

```text
大量共享行锁不只是 lock wait；
它还会制造 pg_multixact SLRU 和 WAL 压力。
```

## 8. `MultiXactIdSetOldestMember()` 的 ownership 边界

在 `heap_lock_tuple()`、`heap_update()`、`heap_delete()` 中，源码会调用：

```text
MultiXactIdSetOldestMember()
```

这件事看起来奇怪。

当前操作可能最后只写普通 XID。

为什么也要设置？

原因是：

```text
只要当前事务可能参与 row lock / update / delete，
另一个 backend 就可能把当前 XID 加入一个 MultiXact。
```

因此当前事务必须发布：

```text
我不可能成为早于某个 MultiXactId 的成员。
```

这个值存放在 per-backend slot：

```text
OldestMemberMXactId[]
```

它用于防止 SLRU 被过早截断。

如果没有这个边界，可能出现：

```text
事务 T 还可能是某个旧 MultiXact 的成员
  -> VACUUM / checkpoint 截断 pg_multixact
  -> 后续 tuple header 仍引用该 MultiXact
  -> 无法展开成员
```

所以 MultiXact 的 ownership 不只在 tuple header。

它还通过 shared memory slot 发布活动事务的最老成员边界。

## 9. `OldestVisibleMXactId` 的读取边界

写入者需要 `OldestMemberMXactId`。

读取者也有边界。

`multixact.c` 中有：

```text
MultiXactIdSetOldestVisible()
```

它在读取 MultiXact 成员前建立：

```text
当前事务可能看见的最老 MultiXactId。
```

这个值来自：

- `nextMXact`。
- 所有 valid `OldestMemberMXactId[]`。

设置后可以保证：

```text
当前事务要查看的 MultiXact SLRU 数据不会被截断。
```

这也是为什么 MultiXact 不是普通内存对象。

它跨事务、跨 backend、跨 crash。

读写双方都要通过 shared memory slot 和 truncation lock 建立边界。

本节先记住：

```text
MultiXact 生命周期不只由引用它的 tuple 决定；
还由当前正在创建、正在读取、可能成为成员的 backend 决定。
```

第 25 节会把这个边界推进到 wraparound 和 freeze。

## 10. 冲突判断：展开成员逐个解释

当一个事务想申请某种 row lock，tuple 上已有 MultiXact。

`heap_lock_tuple()` 会调用：

```text
DoesMultiXactIdConflict(multi, infomask, wanted_mode, current_is_member)
```

判断步骤可以压缩成：

```text
GetMultiXactIdMembers()
  -> 遍历每个 member
  -> 当前事务自己的成员跳过
  -> 把 member.status 映射到 LOCKMODE
  -> 把 wanted mode 映射到 LOCKMODE
  -> DoLockModesConflict()
  -> 对 updater 和 locker 分别检查事务状态
```

纯 locker 的规则：

```text
如果成员事务不再 in progress，锁语义消失。
```

updater 的规则：

```text
如果 updater abort，可以忽略。
如果 updater commit，更新语义仍然存在。
如果 updater in progress，要按冲突矩阵等待。
```

这说明 MultiXact 冲突判断不是简单集合交集。

它是：

```text
成员 status
  + member XID 当前状态
  + 调用方想要的 mode
  + 当前事务是否已经是成员
```

共同决定。

## 11. 等待：`MultiXactIdWait()`

如果已有 MultiXact 与当前请求冲突，等待入口是：

```text
MultiXactIdWait()
ConditionalMultiXactIdWait()
```

内部实现是：

```text
Do_MultiXactIdWait()
  -> GetMultiXactIdMembers()
  -> 遍历成员
  -> 对冲突成员调用 XactLockTableWait()
  -> 对 NOWAIT/SKIP LOCKED 使用 ConditionalXactLockTableWait()
```

这里的关键注释是：

```text
等完 MultiXact 后，tuple 的 xmax 可能已经被别人改了；
调用者必须重新迭代。
```

所以等待返回不表示“我现在可以写 header”。

它只表示：

```text
旧 MultiXact 中与我冲突的其它 backend 成员已经结束或不可阻塞。
```

接下来必须：

```text
重新拿 buffer lock
重新读取 tuple header
检查 xmax 和 infomask 是否变化
必要时从头开始
```

这条规律和第 20、21 节一致。

等待不是证明。

等待后重查才是证明。

## 12. 生命周期 / ownership / cleanup

MultiXact 的生命周期可以分成六段。

### 12.1 创建

创建者是 heap 路径。

入口包括：

- `heap_lock_tuple()`。
- `heap_update()`。
- `heap_delete()`。
- freeze 过程中为了保留 surviving members 创建新 MultiXact。

### 12.2 持有

持有状态分两部分。

tuple header 持有：

```text
xmax = MultiXactId
HEAP_XMAX_IS_MULTI
锁强度提示位
LOCK_ONLY 或 updater 语义
```

`pg_multixact` 持有：

```text
members offset
xid + status array
```

### 12.3 使用

使用者包括：

- visibility routine。
- tuple lock conflict 判断。
- UPDATE / DELETE 并发冲突。
- VACUUM freeze。
- `pg_get_multixact_members()`。

### 12.4 成员结束

成员事务结束后：

- pure locker 可以被忽略。
- aborted updater 可以被忽略。
- committed updater 必须保留为更新语义。

但旧 MultiXactId 本身不会自动删除。

### 12.5 tuple 层 cleanup

后续操作可能：

- 把 lock-only MultiXact 变成 invalid `xmax`。
- 把只有 updater 的 MultiXact 替换成普通 XID。
- 把仍有多个有效成员的 MultiXact 替换成新 MultiXact。
- 保持不变。

### 12.6 SLRU 截断

只有当所有 relation 的 `relminmxid` 推进，数据库级 `datminmxid` 推进，checkpoint 记录安全边界后，旧 `pg_multixact` segment 才能截断。

这比 tuple 上成员结束晚得多。

## 13. 正确性机制层次

MultiXact 正确性靠多层机制。

第一层：成员集合不可变。

旧 MultiXact 不原地追加成员。

这避免等待者看到集合变化。

第二层：每个成员有 status。

同一个 XID 是 locker 还是 updater，事务结束后的意义不同。

第三层：事务状态判断顺序。

源码多处强调先看 in-progress，再看 commit / abort。

这是为了避免事务状态 race。

第四层：SLRU 持久化。

tuple header 写到磁盘后，MultiXact 成员必须能在 crash 后恢复。

第五层：WAL。

MultiXact create 有 WAL record。

heap lock/update/delete 的 tuple header 改动也有 WAL。

第六层：truncation 边界。

`OldestMemberMXactId`、`OldestVisibleMXactId`、`relminmxid`、`datminmxid` 一起防止成员被过早删除。

第七层：等待后重查。

所有 wait path 都不能跳过重新解释 tuple header。

## 14. 错误路径 / 异常路径 / fallback

### 14.1 old Multi 已无成员

`MultiXactIdExpand()` 读取旧 MultiXact 成员时可能发现成员已经无意义。

它会创建一个只包含当前成员的新 MultiXact。

这不是常态。

但它避免把 API 复杂性传给调用者。

### 14.2 成员中有 aborted locker

aborted locker 不再阻塞。

冲突判断会忽略。

freeze 路径也可丢掉。

### 14.3 成员中有 committed updater

不能简单丢掉。

如果 updater committed，它代表旧 tuple version 已被更新。

即使其它 locker 都结束，也可能需要保留 updater XID。

### 14.4 NOWAIT / SKIP LOCKED

`ConditionalMultiXactIdWait()` 用于非阻塞策略。

如果有冲突成员仍不能立即等待通过，调用者返回 would-block 或报错。

这不改变 MultiXact 的内容。

### 14.5 SLRU 查不到

如果 tuple header 引用的 MultiXact 早于 relation 的 `relminmxid`，freeze 路径会认为这是 corruption。

因为 relation 元数据已经声明不会再有这么老的 MultiXact 引用。

这说明 `relminmxid` 是 correctness contract，不只是优化指标。

## 15. 成本、资源与跨模块传播

### 15.1 成员数量成本

冲突判断需要遍历 members。

成员越多：

- CPU 遍历越多。
- SLRU 读取越多。
- cache miss 可能越多。
- wait 目标可能越多。

热点行上的大量共享锁会把成本集中到 MultiXact。

### 15.2 SLRU 成本

MultiXact 使用两个 SLRU。

这意味着：

- shared buffers 之外还有 SLRU buffer。
- 可能产生 SLRU page read/write。
- 可能被 checkpoint 和 truncation 影响。

### 15.3 WAL 成本

创建 MultiXact 写 WAL。

heap tuple header 指向新 MultiXact 也写 WAL。

在外键检查密集、`SELECT FOR UPDATE` 密集、队列表密集 workload 中，这部分 WAL 可能可见。

### 15.4 VACUUM 成本

MultiXact 引用会让 VACUUM freeze 更复杂。

它要：

- 展开成员。
- 判断 locker 是否仍 running。
- 判断 updater 是否 committed。
- 决定 invalidate、保留 XID、替换 MultiXact 或 no-op。

### 15.5 wraparound 成本

MultiXactId 也会回卷。

成员 offset 虽然更大，但成员空间也会膨胀。

`MultiXactMemberFreezeThreshold()` 会在 members 空间压力大时让 VACUUM 更激进。

这把行锁热点传播到 autovacuum 调度。

## 16. 观测与诊断入口

### 16.1 `pageinspect`

先看 tuple header：

```sql
SELECT lp, t_xmax, t_infomask, t_infomask2
FROM heap_page_items(get_raw_page('mx_demo', 0));
```

要判断 `t_xmax` 是否 MultiXact，需要结合 infomask。

`pageinspect` 给出原始值。

解释需要源码或辅助函数。

### 16.2 `pg_get_multixact_members()`

如果确认是 MultiXact：

```sql
SELECT *
FROM pg_get_multixact_members('<mxid>'::xid);
```

输出通常包括：

- multixid。
- xid。
- mode。

mode 对应成员 status 的可读表达。

### 16.3 `pg_get_multixact_stats()`

可以观察整体使用量：

```sql
SELECT *
FROM pg_get_multixact_stats();
```

这有助于判断：

- multixacts 数量。
- member offset 增长。
- oldest / next 边界。

具体列以本地版本函数定义为准。

### 16.4 系统视图

等待时看：

```sql
SELECT pid, wait_event_type, wait_event, query
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

等待的是 XID 或 tuple lock tag。

不是直接“等待 MultiXact 对象”。

`MultiXactIdWait()` 会拆成对成员 XID 的等待。

### 16.5 断点

建议断点：

```text
compute_new_xmax_infomask
MultiXactIdCreate
MultiXactIdExpand
MultiXactIdCreateFromMembers
RecordNewMultiXact
GetMultiXactIdMembers
DoesMultiXactIdConflict
Do_MultiXactIdWait
```

要观察：

```text
nmembers
members[i].xid
members[i].status
old_infomask
new_infomask
new_infomask2
multi
offset
```

## 17. 常见误区

误区一：

```text
MultiXact 就是多个事务共享锁。
```

不完整。

它可以包含 updater。

成员 status 决定事务结束后的语义。

误区二：

```text
MultiXact 成员会被原地追加。
```

错误。

扩展会创建新的 MultiXactId。

误区三：

```text
成员事务结束后，MultiXact 马上消失。
```

错误。

tuple header 仍可能引用它。

SLRU 截断还要等 `relminmxid` / `datminmxid`。

误区四：

```text
MultiXact 只是内存状态。
```

错误。

它有 WAL 和 SLRU。

crash recovery 后仍要能解释 tuple header。

误区五：

```text
pg_locks 能完整展示 MultiXact 成员。
```

错误。

`pg_locks` 显示 lock manager 当前等待和持有。

MultiXact 成员要用 `pg_get_multixact_members()` 或源码推断。

## 18. 课堂实验

### 实验一：制造共享 MultiXact

步骤：

```sql
DROP TABLE IF EXISTS mx_demo;
CREATE TABLE mx_demo(id int primary key, payload text);
INSERT INTO mx_demo VALUES (1, 'a');
CREATE EXTENSION IF NOT EXISTS pageinspect;

-- Session A
BEGIN;
SELECT * FROM mx_demo WHERE id = 1 FOR SHARE;

-- Session B
BEGIN;
SELECT * FROM mx_demo WHERE id = 1 FOR SHARE;

-- Session C
SELECT lp, t_xmax, t_infomask, t_infomask2
FROM heap_page_items(get_raw_page('mx_demo', 0));
```

观察：

- `t_xmax` 可能成为 MultiXactId。
- infomask 表示 `HEAP_XMAX_IS_MULTI`。
- 两个 `FOR SHARE` 可以共存。

### 实验二：展开成员

步骤：

```sql
SELECT *
FROM pg_get_multixact_members('<mxid>'::xid);
```

观察：

- 两个成员 XID。
- mode 显示 share 类锁。

源码回读：

```text
GetMultiXactIdMembers()
mxstatus_to_string()
pg_get_multixact_members()
```

### 实验三：第三个冲突者等待

步骤：

```sql
-- Session D
UPDATE mx_demo SET payload = 'b' WHERE id = 1;
```

观察：

```sql
SELECT pid, wait_event_type, wait_event, query
FROM pg_stat_activity
WHERE wait_event_type = 'Lock';
```

回到源码：

```text
DoesMultiXactIdConflict()
Do_MultiXactIdWait()
XactLockTableWait()
```

### 实验四：观察 MultiXact 使用量

步骤：

```sql
SELECT *
FROM pg_get_multixact_stats();
```

然后并发运行大量：

```sql
SELECT * FROM mx_demo WHERE id = 1 FOR SHARE;
```

再看统计。

注意：

单机实验可能变化很小。

重点是理解哪些计数会随着 MultiXact 创建和成员写入增长。

### 实验五：gdb 跟读创建路径

断点：

```text
compute_new_xmax_infomask
MultiXactIdCreate
MultiXactIdCreateFromMembers
RecordNewMultiXact
```

要画出：

```text
old xmax = XID
  -> old_status = ForShare
  -> new_status = ForShare
  -> members[0] = old XID
  -> members[1] = current XID
  -> new xmax = MultiXactId
```

## 19. 讨论题

1. 为什么 tuple header 不能直接保存多个 row lock XID？

2. 为什么 MultiXact 成员必须保存 `status`，不能只保存 XID？

3. 为什么 MultiXact 可以包含 updater，但不能包含两个 updater？

4. 为什么 `MultiXactIdExpand()` 不原地追加成员？

5. `OldestMemberMXactId` 和 `OldestVisibleMXactId` 分别保护什么？

6. 为什么 pure locker 结束后可以忽略，而 committed updater 不能忽略？

7. `pg_locks` 和 `pg_get_multixact_members()` 各自能看到什么？

8. 高并发外键检查为什么可能变成 MultiXact 和 WAL 问题？

## 20. 本节小结

本节回答了 MultiXact 的核心问题：

```text
当多个事务需要在同一 tuple 上保存兼容 row lock 时，单个 xmax slot 如何表达集合语义？
```

核心链路是：

```text
first locker
  -> xmax = XID
second compatible locker
  -> MultiXactIdCreate()
  -> xmax = MultiXactId
more lockers
  -> MultiXactIdExpand()
  -> new MultiXactId
conflicting locker/updater
  -> DoesMultiXactIdConflict()
  -> MultiXactIdWait()
  -> 等待后重查 tuple header
```

核心状态是：

- `MultiXactId`。
- `MultiXactMember.xid`。
- `MultiXactMember.status`。
- offsets SLRU。
- members SLRU。
- `OldestMemberMXactId[]`。
- `OldestVisibleMXactId[]`。
- tuple header 的 `HEAP_XMAX_IS_MULTI`。
- relation 的 `relminmxid`。

ownership 和 cleanup 的边界是：

```text
成员事务结束不等于 MultiXact 引用消失；
tuple header 仍可能需要这个 MultiXact 来解释 updater 或 surviving lockers；
VACUUM / freeze 清掉 tuple 引用后，relminmxid 和 datminmxid 才能推进；
checkpoint 后 pg_multixact SLRU 才能截断旧段。
```

异常路径的核心是：

```text
成员结束、等待返回、旧 Multi 无成员、NOWAIT 失败都不能直接改写语义；
必须回到 tuple header 和成员状态重新解释。
```

可观测入口包括：

- `pageinspect`。
- `pg_get_multixact_members()`。
- `pg_get_multixact_stats()`。
- `pg_stat_activity`。
- `pg_locks`。
- gdb 断点。

本节的可迁移模型是：

```text
小字段表达集合状态时，间接 ID 是常见解法；
但一旦引入间接 ID，就必须同时设计成员稳定性、持久化、截断边界、等待协议和观测入口。
```
