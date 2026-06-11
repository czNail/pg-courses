# PostgreSQL CommandId 与事务内 self visibility

## 课程定位

前置知识：

- 已经理解 MVCC snapshot 如何用 `xmin/xmax/xip` 判断其他事务的可见性。
- 已经理解 REPEATABLE READ 会固定外部事务集合，但 `curcid` 会随命令边界推进。
- 已经理解 heap tuple header 中 `xmin`、`xmax`、`t_cid` 共同描述版本来源。
- 已经知道 PostgreSQL 当前源码真实入口是 `CommandCounterIncrement()`。

本节唯一主问题：

```text
同一个事务内部，PostgreSQL 如何用 `CommandId` 和 combo CID 判断哪些自己写过的 tuple 对当前语句可见？
```

本节围绕的核心矛盾：

- 一个事务必须看到自己已经完成的前序命令。
- 同一条语句又不能在普通扫描中反复看到自己刚刚插入或更新的新版本。
- heap tuple header 只有一个 command id 存储位置。
- 当同一事务先插入又删除同一 tuple 时，系统同时需要 `cmin` 和 `cmax`。
- PostgreSQL 用 `currentCommandId`、snapshot `curcid` 和 combo CID 映射解决这个矛盾。

学完本节后，应能独立判断：

- 为什么每条写语句会使用 `GetCurrentCommandId(true)`。
- 为什么命令结束后调用的是 `CommandCounterIncrement()`。
- 为什么 `SnapshotSetCommandId()` 会更新静态 snapshot 的 `curcid`。
- 为什么 `HeapTupleSatisfiesMVCC()` 对当前事务 XID 走特殊分支。
- 为什么 `HeapTupleHeaderAdjustCmax()` 可能把普通 `cmax` 改成 combo CID。
- 为什么 parallel worker 不能随意标记新的 command id 使用。
- 为什么 logical decoding 需要 `XLOG_HEAP2_NEW_CID` 和 reorder buffer 的 tuple CID 映射。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

本节不展开：

- 不讲所有 executor node 如何产生 output tuple。
- 不讲 MultiXact locker/updater 的完整语义。
- 不讲 logical decoding 的完整 snapshot builder 状态机。
- 不讲 SSI 的 predicate lock。
- 这些只在 command id 边界需要时出现。

## 1. 本节在总主线中的位置

上一节讲了事务级 snapshot。那节留下一个关键细节：

- RR 事务看不到别人后来提交的行。
- 但能看到自己上一条语句写入的行。

这个现象不能只靠 `xmin/xmax/xip` 解释。因为当前事务自己的 XID 通常不在 snapshot 的 running XID 数组里按普通外部事务处理。

heap visibility 对当前事务必须有额外规则。最小例子：

```sql
DROP TABLE IF EXISTS cid_demo;
CREATE TABLE cid_demo(id int primary key, v int);

BEGIN ISOLATION LEVEL REPEATABLE READ;
INSERT INTO cid_demo VALUES (1, 10);
SELECT * FROM cid_demo WHERE id = 1;
COMMIT;
```

第二条 `SELECT` 能看到第一条 `INSERT`。但如果一条 `UPDATE` 自己制造了新版本，普通扫描不能在同一条命令中再次把新版本当成可见候选并重复更新。

这就是 self visibility 的核心问题。外部可见性回答：

- 其他事务对我是否 committed。

事务内可见性回答：

- 这是我自己哪个 command 产生或删除的版本。
- 这个 command 相对当前 snapshot 的 `curcid` 是之前、当前，还是之后。

本节的主线是：

```text
statement 使用当前 command id 写 tuple
  -> tuple header 记录 cmin 或 cmax
  -> statement 结束时 CommandCounterIncrement()
  -> SnapshotSetCommandId() 推进 snapshot->curcid
  -> 下一条 statement 看到前序 command 的版本
  -> 同一 statement 通过 cmin/cmax 与 curcid 比较避免自我重读
  -> 插入后又删除需要 combo CID
  -> parallel 和 logical decoding 为 backend-local CID 状态补边界
```

这条链路解释三个 runtime 现象。第一，前序命令可见：

- `INSERT` 后 `SELECT` 能看见自己的行。

第二，同一命令不重复消费自己产生的新版本：

- `UPDATE cid_demo SET v = v + 1 WHERE v < 15` 不会把同一行一路更新到 15。

第三，同一事务插入又删除同一 tuple 时，tuple header 需要同时表达 `cmin` 和 `cmax`。这一点普通 SQL 不总能直接看到。

但 pageinspect、logical decoding 和源码断点能看到 combo CID 路径。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
写路径用 `GetCurrentCommandId(true)` 把当前命令号写入 tuple header；语句边界调用 `CommandCounterIncrement()` 推进 `currentCommandId` 并通过 `SnapshotSetCommandId()` 更新 snapshot `curcid`；`HeapTupleSatisfiesMVCC()` 遇到当前事务自己的 `xmin/xmax` 时，不走普通 CLOG 判断，而是比较 `cmin/cmax` 与 `snapshot->curcid`；当一个 tuple 同时需要保存本事务的 `cmin` 和 `cmax` 时，`HeapTupleHeaderAdjustCmax()` 创建 combo CID 映射。
```

这里有两个必须区分的数字。第一个是 transaction id。

- 它回答哪个事务插入或删除了 tuple。
- tuple header 的 `xmin/xmax` 保存它。

第二个是 command id。

- 它回答同一事务内部哪个 command 做了这件事。
- tuple header 的 `t_cid` 保存它。

如果只靠 transaction id，系统只能判断：

- 这是我自己的事务。

但无法判断：

- 这是上一条语句插入的，还是当前语句刚插入的。
- 这是当前语句删除的，还是前序语句删除的。

`CommandId` 给同一事务内部加了一个局部时间轴。这个时间轴只在事务内有意义。

它不是 shared MVCC 顺序。它不会写入 CLOG。

它也不能跨 backend 直接解释。这带来第二个矛盾：

- tuple header 只有一个 `t_cid`。
- 插入时要存 `cmin`。
- 删除或更新旧 tuple 时要存 `cmax`。
- 如果同一事务先插入又删除同一 tuple，就需要同时保存两个值。

PostgreSQL 没有扩大 tuple header。它用 combo CID：

- header 中存一个 synthetic command id。
- `HEAP_COMBOCID` 标志说明它不是普通 cmin/cmax。
- backend-local combo CID table 把它映射回 `{cmin, cmax}`。

这保住了 tuple header 大小。代价是：

- combo CID 只对创建它的事务本地可解释。
- parallel worker 需要序列化/恢复 combo CID state。
- logical decoding 需要额外 WAL 记录让重放端能解释 catalog tuple 的 cmin/cmax。

## 3. 核心文件分工与阅读顺序

| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/access/xact.h` | 先看 `CommandId`、`GetCurrentCommandId()` 和 `CommandCounterIncrement()` 的公开边界。 |
| 2 | `src/backend/access/transam/xact.c` | 再读 command id 的分配、推进和事务结束 cleanup。 |
| 3 | `src/include/utils/snapshot.h` | 明确 snapshot `curcid` 在可见性判断中的位置。 |
| 4 | `src/backend/utils/time/snapmgr.c` | 看 `SnapshotSetCommandId()` 如何更新当前 snapshot。 |
| 5 | `src/include/access/htup_details.h` | 对照 tuple header 中 `t_cid`、`HEAP_COMBOCID` 和相关宏。 |
| 6 | `src/backend/access/heap/heapam_visibility.c` | 跟当前事务自己的 `xmin/xmax` 可见性分支。 |
| 7 | `src/backend/access/heap/heapam.c` | 看写路径何时记录 `cmin/cmax`。 |
| 8 | `src/backend/utils/time/combocid.c` | 看 combo CID 如何把一个 header 字段映射回 `{cmin, cmax}`。 |
| 9 | `src/backend/access/transam/parallel.c` | 核对 parallel worker 的 command id 状态边界。 |
| 10 | `src/backend/replication/logical/snapbuild.c` / `src/backend/replication/logical/reorderbuffer.c` | 最后看 logical decoding 为什么需要 CID 映射。 |

## 4. 可复现 runtime 现象

### 4.1 前序命令可见

准备：

```sql
DROP TABLE IF EXISTS cid_demo;
CREATE TABLE cid_demo(id int primary key, v int);
```

执行：

```sql
BEGIN;
INSERT INTO cid_demo VALUES (1, 10);
SELECT * FROM cid_demo WHERE id = 1;
ROLLBACK;
```

`SELECT` 能看到 `(1, 10)`。源码解释：

- `INSERT` 写 tuple 时使用 `GetCurrentCommandId(true)`。
- tuple 的 `cmin` 等于当前 `currentCommandId`。
- 语句结束后 `CommandCounterIncrement()` 把 `currentCommandId` 加一。
- `SnapshotSetCommandId()` 更新当前 snapshot 的 `curcid`。
- 下一条 `SELECT` 中 `HeapTupleSatisfiesMVCC()` 遇到自己的 `xmin`。
- 它比较 `HeapTupleHeaderGetCmin(tuple) >= snapshot->curcid`。
- 因为插入命令的 cmin 小于当前 curcid，所以可见。

### 4.2 同一命令不反复更新自己

准备：

```sql
TRUNCATE cid_demo;
INSERT INTO cid_demo VALUES (1, 1);
```

执行：

```sql
UPDATE cid_demo
SET v = v + 1
WHERE v < 5
RETURNING id, v;
```

结果只返回一次，`v` 从 `1` 到 `2`。它不会在同一条 `UPDATE` 中继续把新版本 `2` 更新到 `3`、`4`、`5`。

源码解释：

- 这条 `UPDATE` 的 scan snapshot 有固定 `curcid`。
- 新版本的 `cmin` 是当前 command id。
- 对同一命令的普通 visibility 判断，`cmin >= snapshot->curcid`。
- 所以新版本对同一命令的普通扫描不可见。

这不是 planner 防止循环。这是 heap visibility 的 self visibility 边界。

### 4.3 插入后删除需要 combo CID

执行：

```sql
BEGIN;
INSERT INTO cid_demo VALUES (2, 20);
DELETE FROM cid_demo WHERE id = 2;
COMMIT;
```

这行对事务外当然不可见。但在执行 `DELETE` 时，旧 tuple 同时携带：

- `xmin` 是当前事务。
- `cmin` 是插入命令。
- `xmax` 也是当前事务。
- `cmax` 是删除命令。

header 里只有一个 `t_cid` 位置。因此删除路径会调用 `HeapTupleHeaderAdjustCmax()`。

如果发现该 tuple 是当前事务插入的，就创建 combo CID。header 存 combo cid，并设置 `HEAP_COMBOCID`。

当前 backend 通过 combo table 解出真实 `cmin/cmax`。普通 SQL 不一定直接展示这个细节。

可以用 gdb 在 `HeapTupleHeaderAdjustCmax()` 和 `GetComboCommandId()` 断点观察。也可以在允许安装扩展的环境中用 `pageinspect` 观察 raw tuple header。

但 pageinspect 看到的是原始 bit 和 raw command id。真正语义仍要回到 `combocid.c`。

## 5. 关键状态与边界

### 5.1 `currentCommandId`

`currentCommandId` 是 backend-local transaction state。它不是 subtransaction-local。

`GetCurrentCommandId()` 注释明确说：

- command id global to a transaction。

这意味着：

- 子事务不会拥有独立 command id 序列。
- 当前事务栈共享一个 command counter。

`currentCommandId` 初始从 0 开始。每当有写命令真正使用了它，`currentCommandIdUsed` 被设置。

命令结束时 `CommandCounterIncrement()` 才会推进。如果上一条命令没有使用 command id 标记 tuple，则 CCI 可以不递增。

这避免无意义消耗 command id。也延缓 32 bit command id overflow。

### 5.2 `currentCommandIdUsed`

`GetCurrentCommandId(bool used)` 的 `used` 参数很关键。如果 `used = true`：

- 表示调用者要把这个 command id 写进 tuple header。
- 函数设置 `currentCommandIdUsed = true`。

如果 `used = false`：

- 表示只读用途，例如填 snapshot `curcid`。
- 不需要强制下一个 CCI 递增。

这解释为什么 `GetSnapshotData()` 使用 `GetCurrentCommandId(false)`。构造 snapshot 不应该自己制造一个新 command 边界。

只有真正写 tuple 的路径才应该标记 used。

### 5.3 `CommandCounterIncrement()`

`CommandCounterIncrement()` 是真实命令边界入口。它做四件事。

第一，如果 `currentCommandIdUsed` 为 false，直接返回。这让只读命令的 CCI 很便宜。

第二，如果在 parallel mode 或 parallel worker 中尝试开始新 command，报错。因为 parallel operation 开始后，leader 和 worker 的事务状态已经同步，不能再产生新的 command id 分歧。

第三，递增 `currentCommandId`。如果达到 `InvalidCommandId`，报错：

- 不能在一个事务里有超过 `2^32-2` 个命令。

第四，调用：

- `SnapshotSetCommandId(currentCommandId)`
- `AtCCI_LocalCache()`

前者推进 self visibility。后者让 catalog cache/invalidation 的本地效果在命令边界生效。

### 5.4 `SnapshotData->curcid`

`snapshot.h` 注释给出定义：

- 在当前事务中，`CID < curcid` 可见。

这个比较方向很重要。如果 tuple 是当前命令插入的：

- `cmin == currentCommandId`
- snapshot `curcid` 也是当前命令可见边界
- `cmin >= curcid`
- 对普通 scan 不可见

命令结束后 CCI：

- `currentCommandId` 加一
- `SnapshotSetCommandId()` 更新 `curcid`
- 之前 tuple 的 `cmin < curcid`
- 对下一条命令可见

这就是自写可见性的时间轴。

### 5.5 tuple header 的 `t_cid`

heap tuple header 的 command id 存在一个位置：

- `t_choice.t_heap.t_field3.t_cid`

它可以表示：

- `cmin`
- `cmax`
- combo cid

这取决于 tuple 状态和 infomask。`HeapTupleHeaderSetCmin()` 会写 raw cid 并清除 `HEAP_COMBOCID`。

`HeapTupleHeaderSetCmax()` 要求调用者已经执行 `HeapTupleHeaderAdjustCmax()`。如果 `iscombo` 为 true，它设置 `HEAP_COMBOCID`。

所以 raw field 不是语义。语义是：

- raw cid
- `HEAP_COMBOCID`
- `xmin/xmax` 是否为当前事务
- backend-local combo table
- snapshot `curcid`

组合起来才成立。

### 5.6 combo CID table

combo CID table 位于当前 backend 内存。`combocid.c` 中核心状态包括：

- `comboHash`
- `comboCids`
- `usedComboCids`
- `sizeComboCids`

`GetComboCommandId(cmin, cmax)` 会：

- 首次创建 hash 和数组。
- 查找已有 `{cmin, cmax}` 是否已存在。
- 存在则复用 combo id。
- 不存在则追加一个新 combo id。

这让多个 tuple 可以共享同一个 `{cmin, cmax}` 映射。但它仍然是 transaction-local 状态。

事务结束时 `AtEOXact_ComboCid()` 清空指针和计数。内存同样由 `TopTransactionContext` 回收。

## 6. 主流程源码 walkthrough

### 6.1 statement 获取 output command id

executor 初始化写操作时会记录 output command id。典型路径在 `execMain.c`：

- `estate->es_output_cid = GetCurrentCommandId(true)`

table AM 或 heap AM 写入时也会看到：

- `table_tuple_insert(..., GetCurrentCommandId(true), ...)`
- `heap_insert(..., cid, ...)`

`used = true` 是关键。它表示这个 command id 将进入 tuple header。

从这个瞬间开始，下一次 `CommandCounterIncrement()` 必须推进 command id。否则下一条命令会无法区分前序写入和当前写入。

### 6.2 insert 写 `cmin`

`heap_insert()` 准备 tuple header。关键动作：

```text
HeapTupleHeaderSetXmin(tup, xid)
HeapTupleHeaderSetCmin(tup, cid)
```

`xmin` 是当前事务 XID。`cmin` 是当前 command id。

如果当前事务还没有 XID，写路径会先通过事务 accessor 分配。

本节不展开 XID 分配。这里只强调：

- `xmin` 解决事务间可见性。
- `cmin` 解决同事务内命令顺序。

### 6.3 statement 结束推进 command counter

PostgreSQL 在查询结束或命令边界调用 `CommandCounterIncrement()`。常见入口包括：

- simple query 每条 query 后。
- extended query 在 Sync 边界。
- utility command 内部显式 CCI。
- catalog 操作需要让新对象对后续步骤可见时显式 CCI。

主路径：

```text
CommandCounterIncrement()
  -> if (!currentCommandIdUsed) return
  -> if (IsInParallelMode() || IsParallelWorker()) ERROR
  -> currentCommandId++
  -> currentCommandIdUsed = false
  -> SnapshotSetCommandId(currentCommandId)
  -> AtCCI_LocalCache()
```

这不是简单加一。它是命令边界 publication。

它同时推进 self visibility 和本地 catalog cache invalidation。

### 6.4 snapshot `curcid` 被更新

`SnapshotSetCommandId()` 位于 `snapmgr.c`。它做得很少：

```text
if (!FirstSnapshotSet)
    return
if (CurrentSnapshot)
    CurrentSnapshot->curcid = curcid
if (SecondarySnapshot)
    SecondarySnapshot->curcid = curcid
```

这几行解释上一节的关键现象。RR 的 `CurrentSnapshot` 是同一个 copied snapshot。

外部 `xip/xmax` 不变。但 `curcid` 会随 CCI 更新。

所以 RR 中：

- 不看见其他事务新提交。
- 看得见自己前序命令。

### 6.5 heap visibility 处理自己的 `xmin`

`HeapTupleSatisfiesMVCC()` 看到未标记 committed 的 `xmin` 时，先判断是不是当前事务。如果是当前事务：

```text
if (HeapTupleHeaderGetCmin(tuple) >= snapshot->curcid)
    return false;
```

含义：

- 插入命令号不早于当前 snapshot 可见边界。
- 说明这是当前命令或未来命令的 tuple。
- 当前普通 scan 不能看见它。

如果 `cmin < curcid`：

- 这是前序命令插入的 tuple。
- 继续检查 `xmax`。

如果没有有效删除，tuple 可见。

### 6.6 heap visibility 处理自己的 `xmax`

如果 tuple 已经被当前事务删除或更新，visibility 要看 `cmax`。典型规则：

```text
if (HeapTupleHeaderGetCmax(tuple) >= snapshot->curcid)
    return true;   // deleted after scan started
else
    return false;  // deleted before scan started
```

含义：

- 当前命令刚删除的旧版本，对当前 scan 可能仍要按删除发生后边界处理。
- 前序命令已经删除的旧版本，对当前命令不可见。

这就是为什么 `DELETE` 后同一事务下一条 `SELECT` 看不到该行。也解释更新链中旧版本和新版本如何在当前事务内切换可见性。

### 6.7 update 同时制造旧版本删除和新版本插入

`UPDATE` 可以理解为：

- 旧 tuple 设置 `xmax/cmax`。
- 新 tuple 设置 `xmin/cmin`。

对同一条 UPDATE 的 scan：

- 新 tuple 的 `cmin` 等于当前命令，`cmin >= curcid`，不可见。
- 旧 tuple 的处理取决于 scan 已经拿到它的时机和 EvalPlanQual 等更新路径。

对下一条 statement：

- `curcid` 已经推进。
- 新 tuple `cmin < curcid`，可见。
- 旧 tuple `cmax < curcid`，不可见。

这就是一次 UPDATE 之后下一条 SELECT 看见新版本的原因。

### 6.8 delete 前调整 `cmax`

`heap_delete()` 在实际修改 shared buffer 中的 tuple 前调用：

```text
HeapTupleHeaderAdjustCmax(tp.t_data, &cid, &iscombo)
```

这个调用在 critical section 之前。原因在注释里：

- 创建 combo CID 可能分配内存。
- 内存分配可能失败。
- 不能在已经进入修改 shared buffer 的 critical section 后才冒 OOM。

这体现了错误路径设计：

- 可能失败的本地准备工作先做。
- 真正修改 page header / tuple header 的关键区间尽量不失败。

### 6.9 `HeapTupleHeaderAdjustCmax()` 何时创建 combo

函数逻辑：

- 如果 tuple 的 `xmin` 未被标记 committed。
- 且 `TransactionIdIsCurrentTransactionId(raw xmin)` 为真。
- 说明这个 tuple 是当前事务插入的。
- 删除它时需要同时保存插入命令和删除命令。
- 读取 `cmin`。
- 调 `GetComboCommandId(cmin, cmax)`。
- 把 `*cmax` 替换成 combo id。
- `*iscombo = true`。

否则：

- 不需要 combo。
- `cmax` 原样写入 header。

这就是 combo CID 的唯一主因。不是所有 update/delete 都需要 combo。

只有同一事务需要同时解释 cmin 和 cmax 时才需要。

### 6.10 `TransactionIdIsCurrentTransactionId()`

self visibility 还依赖判断 XID 是否属于当前事务。`TransactionIdIsCurrentTransactionId()` 会检查：

- top transaction XID。
- 当前 subtransaction。
- parents。
- subcommitted children。
- parallel worker 中的 `ParallelCurrentXids`。

它不会把 aborted subtransaction 简单当作 current。这对 savepoint 场景很关键。

如果一个子事务插入后 abort：

- 后续 visibility 不能把它当成当前有效写入。

因此 self visibility 不只是 command id。它还依赖当前事务/子事务状态栈。

## 7. combo CID 生命周期

combo CID 创建时机：

- 同一事务删除或更新自己插入的 tuple。
- 删除路径先调 `HeapTupleHeaderAdjustCmax()`。
- update 路径也会对 old tuple 调整 cmax。

combo CID 保存在哪里？

- raw tuple header 的 `t_cid` 存 combo id。
- `HEAP_COMBOCID` 标志说明它需要映射。
- `{cmin, cmax}` 保存在 backend-local `comboCids` 数组和 hash 中。

谁能解释？

- 创建它的 backend。
- 继承了 combo CID state 的 parallel worker。
- logical decoding 通过 WAL/reorder buffer 保存的映射。

谁不能解释？

- 任意其他普通 backend 不能直接解释当前事务尚未结束的 combo id。
- 它们通常也不需要解释，因为未提交当前事务 tuple 对它们不可见。

何时释放？

- `AtEOXact_ComboCid()` 在事务结束路径清空状态。
- 内存随 `TopTransactionContext` 释放。

为什么不写进 tuple header 两个字段？

- tuple header 是极热路径和磁盘格式。
- 为少数 self-delete 场景扩大每个 tuple header 成本不划算。
- combo CID 把稀有复杂性移到 backend-local side table。

这是一种典型 PostgreSQL 取舍：

- common case 保持 header 紧凑。
- rare case 用本地映射补语义。

## 8. parallel 边界

parallel query 让 command id 状态变得敏感。原因是：

- leader 和 worker 共享一个事务语义。
- worker 不能把自己的写命令状态悄悄带回 leader。
- worker 也不能独立推进 command counter。

`GetCurrentCommandId(true)` 中如果是 parallel worker，会报错：

- cannot modify data in a parallel worker

`CommandCounterIncrement()` 中如果在 parallel mode 或 worker 中尝试开始新 command，也会报错。这不是保守限制。

这是因为：

- `currentCommandIdUsed`
- combo CID table
- current transaction/subtransaction XID set
- snapshot `curcid`

这些状态必须一致。parallel infrastructure 会在启动 worker 前序列化状态：

- `SerializeTransactionState()`
- `SerializeComboCIDState()`
- `SerializeSnapshot()`

worker 启动后恢复：

- transaction state
- combo CID state
- active snapshot
- transaction snapshot

`TransactionIdIsCurrentTransactionId()` 还会检查 `ParallelCurrentXids`。这是因为 worker 没有 leader 完整的 transaction state stack。

它需要一个序列化过来的 current XID 集合。结论：

- parallel 可以共享既有 self visibility 状态。
- parallel worker 不能随意创建新的写 command 边界。

## 9. logical decoding 边界

combo CID 是 backend-local。事务结束后普通 backend 清空 combo table。

logical decoding 却可能在之后解码 WAL。它需要知道 catalog tuple 在事务内的 cmin/cmax。

否则 historic catalog snapshot 无法正确判断 catalog tuple 对解码点是否可见。PostgreSQL 的做法是：

- 在 effective wal_level 为 logical 时。
- 对 catalog tuple 修改记录 `XLOG_HEAP2_NEW_CID`。
- 入口是 `heapam.c` 的 `log_heap_new_cid()`。

这个 WAL record 包含：

- top xid。
- relation locator。
- tuple TID。
- cmin。
- cmax。
- combocid。

解码端：

- `decode.c` 识别 `XLOG_HEAP2_NEW_CID`。
- `SnapBuildProcessNewCid()` 处理它。
- `ReorderBufferAddNewTupleCids()` 把映射加入 reorder buffer。
- `SetupHistoricSnapshot()` 把 tuplecid hash 交给 historic snapshot。
- `ResolveCminCmaxDuringDecoding()` 在需要时解析。

这条路径说明：

- combo CID 不是 WAL redo 所需的物理修改。
- 它是 logical decoding 解释 catalog visibility 所需的语义补丁。

普通 data tuple 不需要所有这些信息。catalog tuple 需要，因为 decoding 会在历史时间点读取 catalog。

## 10. 错误路径 / 异常路径 / fallback

### 10.1 command id overflow

`CommandCounterIncrement()` 递增后如果得到 `InvalidCommandId`，会回退并 ERROR。错误是：

- 一个事务中不能超过 `2^32-2` 个命令。

这通常只可能出现在异常长事务或循环执行大量写命令的场景。它是硬边界。

不是 wraparound 后继续使用。

### 10.2 parallel worker 写入

`GetCurrentCommandId(true)` 在 parallel worker 中报错。原因是没有机制把 worker 设置的 `currentCommandIdUsed` 和写入 tuple header 的 command id 回传给 leader。

这防止出现 leader snapshot `curcid` 与 worker tuple header 不一致。

### 10.3 parallel mode 中 CCI

`CommandCounterIncrement()` 在 parallel mode 中如果要推进 command id，也会报错。因为 parallel operation 开始后事务状态已经被分发。

中途推进 command id 会破坏 worker 的 snapshot 和 combo CID 解释。

### 10.4 combo CID OOM

`GetComboCommandId()` 可能分配数组或 hash。`HeapTupleHeaderAdjustCmax()` 特意在修改 shared buffer 的 critical section 前调用。

如果 OOM：

- 当前语句 ERROR。
- 事务进入 abort。
- shared buffer 中 tuple header 尚未被半修改。

这是本节最重要的错误路径。

### 10.5 logical decoding 缺失 mapping

`ResolveCminCmaxDuringDecoding()` 如果没有可用 `tuplecid_data`，可能无法解析。源码中会尝试按 relation 和 snapshot 更新 logical mapping。

但如果 WAL 中没有记录所需 new cid，historic catalog visibility 可能无法正确解释。因此 `log_heap_new_cid()` 对 logical wal level 和 catalog tuple 是必要边界。

### 10.6 aborted subtransaction

当前事务判断不是只看 XID 是否在 transaction tree 中。`TransactionIdIsCurrentTransactionId()` 会跳过处于 abort 状态的 transaction state。

subabort 还会清理 child XID 状态。这让 savepoint 回滚后的 tuple 不会被后续命令错误看成有效 self-visible 修改。

## 11. 成本、资源与跨模块传播

### 11.1 common case 成本

最常见路径非常便宜：

- `GetCurrentCommandId(true)` 设置一个 bool。
- tuple header 写入一个 32 bit cid。
- CCI 递增一个 counter。
- snapshot `curcid` 更新。

heap visibility 中，当前事务分支需要：

- `TransactionIdIsCurrentTransactionId()`。
- 读取 cmin/cmax。
- 与 `curcid` 比较。

这比访问 CLOG 或 ProcArray 更本地。所以 self visibility common case 是低成本设计。

### 11.2 combo CID 成本

combo CID 是 rare path。成本包括：

- hash lookup。
- 数组扩展。
- `TopTransactionContext` 分配。
- `HEAP_COMBOCID` 标志处理。

成本随同一事务中不同 `{cmin,cmax}` 组合数量增长。一个长事务反复插入再删除大量 tuple，可能制造很多 combo mapping。

但多数 OLTP 事务很短。所以把成本放到 local side table 是合理的。

### 11.3 catalog invalidation 传播

CCI 不只影响 heap visibility。它还调用 `AtCCI_LocalCache()`。

很多 DDL 和 catalog 修改依赖命令边界让后续步骤看到刚创建的对象。例如创建 catalog row 后，后续命令需要通过 syscache 找到它。

所以 command counter 也是 catalog state publication boundary。不要把它理解成只服务 heap tuple。

### 11.4 snapshot 与 isolation 传播

READ COMMITTED 每条语句重新取 snapshot。RR 复用 snapshot。

但两者都会让 `curcid` 随 CCI 更新。所以隔离级别影响外部事务集合。

它不取消当前事务内部 command 顺序。这是 09 和 10 的连接点。

### 11.5 logical decoding 成本

logical wal level 下 catalog tuple 修改可能多写 `XLOG_HEAP2_NEW_CID`。这增加 WAL 量。

reorder buffer 还要保存 tuple CID mapping。长事务或大量 catalog churn 会放大：

- reorder buffer 内存。
- spill 文件。
- decoding 延迟。

这个成本不是普通 heap scan 的成本。它来自把 backend-local command id 语义跨进程、跨时间传给 decoder。

### 11.6 parallel 成本

parallel 启动需要序列化：

- transaction state。
- snapshot state。
- combo CID state。

如果 combo CID 很多，序列化空间和启动成本会增加。但更重要的是限制：

- worker 不允许产生新的写 command。

这是 correctness 优先的边界。

## 12. 观测与诊断入口

### 12.1 SQL 现象入口

前序命令可见：

```sql
BEGIN;
INSERT INTO cid_demo VALUES (10, 10);
SELECT * FROM cid_demo WHERE id = 10;
ROLLBACK;
```

解释：

- `INSERT` 的 cmin 小于下一条 `SELECT` snapshot 的 curcid。

同一命令不重读：

```sql
TRUNCATE cid_demo;
INSERT INTO cid_demo VALUES (1, 1);
UPDATE cid_demo SET v = v + 1 WHERE v < 5 RETURNING id, v;
```

解释：

- 新版本的 cmin 不小于当前 scan 的 curcid。
- 所以不会被同一条 UPDATE 再扫描到。

### 12.2 pageinspect 入口

如果有权限：

```sql
CREATE EXTENSION IF NOT EXISTS pageinspect;
SELECT lp, t_xmin, t_xmax, t_field3, t_infomask
FROM heap_page_items(get_raw_page('cid_demo', 0));
```

注意：

- `t_field3` 是 raw command id。
- 如果 `HEAP_COMBOCID` 设置，它不是普通 cmin 或 cmax。
- pageinspect 不是语义解释器。
- 最终仍要回到 `htup_details.h` 和 `combocid.c`。

### 12.3 gdb 断点入口

建议断点：

- `GetCurrentCommandId`
- `CommandCounterIncrement`
- `SnapshotSetCommandId`
- `HeapTupleSatisfiesMVCC`
- `HeapTupleHeaderAdjustCmax`
- `GetComboCommandId`
- `AtEOXact_ComboCid`

观察变量：

- `currentCommandId`
- `currentCommandIdUsed`
- `snapshot->curcid`
- tuple raw `xmin/xmax/t_cid`
- `iscombo`
- `usedComboCids`

调试步骤：

1. 在 `INSERT` 前看 `currentCommandId`。
2. 在 `heap_insert()` 看写入的 cmin。
3. 在 CCI 后看 `snapshot->curcid`。
4. 在下一条 SELECT 的 `HeapTupleSatisfiesMVCC()` 看 cmin 比较。

### 12.4 logical decoding 入口

如果调试 logical decoding：

- 在 `log_heap_new_cid()` 断点。
- 确认它只用于 logical wal level 和 catalog tuple。
- 在 `SnapBuildProcessNewCid()` 看 WAL record 被转换为 reorder buffer mapping。
- 在 `ResolveCminCmaxDuringDecoding()` 看 historic snapshot 如何解析。

不要期望普通 SQL 视图直接显示 reorder buffer 的 tuplecid hash。这类状态通常需要日志、断点或专门调试。

### 12.5 parallel 入口

如果分析 parallel query：

- 看 `InitializeParallelDSM()` 中序列化 snapshot 和 combo CID state。
- 看 worker 初始化中的 `RestoreComboCIDState()`。
- 看 `GetCurrentCommandId(true)` 对 parallel worker 的 ERROR。

这能解释为什么某些写操作不允许在 parallel worker 中发生。

## 13. 常见误区

误区一：

- “命令号推进只是某个 command id 自增 helper。”

当前源码中真实入口是：

- `CommandCounterIncrement()`。

误区二：

- “RR 固定 snapshot，所以不会看到自己的写入。”

更准确：

- RR 固定外部事务集合。
- `curcid` 仍随 CCI 推进。

误区三：

- “tuple header 的 `t_cid` 永远是 cmin。”

更准确：

- 它可能是 cmin、cmax 或 combo cid。
- 必须结合 infomask 和上下文解释。

误区四：

- “combo CID 是全局可解释的 ID。”

更准确：

- 它主要是 backend-local transaction state。
- parallel 和 logical decoding 需要额外序列化或 WAL mapping。

误区五：

- “当前事务的 tuple 可见性只看 XID 是否等于自己。”

更准确：

- 还要看 command id 和 subtransaction 状态。

误区六：

- “CCI 每条命令都递增。”

更准确：

- 只有 `currentCommandIdUsed` 为 true 时才递增。
- 只读命令的 CCI 可以是 no-op。

## 14. 课堂实验

实验一：前序命令可见。步骤：

1. 在 `GetCurrentCommandId()`、`CommandCounterIncrement()`、`SnapshotSetCommandId()` 打断点。
2. 执行 `BEGIN; INSERT ...; SELECT ...;`。
3. 记录 INSERT 使用的 cid。
4. 观察 CCI 后 `currentCommandId` 和 `snapshot->curcid`。
5. 在 SELECT 的 `HeapTupleSatisfiesMVCC()` 中验证 `cmin < curcid`。

实验二：同一 UPDATE 不重复更新。步骤：

1. 表中插入一行 `v = 1`。
2. 执行 `UPDATE ... SET v = v + 1 WHERE v < 5 RETURNING v`。
3. 在 `HeapTupleSatisfiesMVCC()` 断点观察新版本。
4. 解释为什么新版本不会被同一条 UPDATE 再次消费。

实验三：combo CID。步骤：

1. 开启事务。
2. 插入一行。
3. 下一条命令删除同一行。
4. 在 `HeapTupleHeaderAdjustCmax()` 断点。
5. 观察 `TransactionIdIsCurrentTransactionId(raw xmin)` 为真。
6. 进入 `GetComboCommandId()`，记录 `{cmin, cmax}` 和返回的 combo id。
7. 事务结束时观察 `AtEOXact_ComboCid()` 清空状态。

实验四：logical decoding 边界。步骤：

1. 在支持 logical wal level 的测试实例中修改 catalog 或运行 DDL。
2. 在 `log_heap_new_cid()` 断点。
3. 观察 `XLOG_HEAP2_NEW_CID` 记录内容。
4. 在 decoder 侧跟 `SnapBuildProcessNewCid()`。
5. 说明为什么 decoder 不能依赖原 backend 的 combo CID hash。

## 15. 讨论题

1. 为什么 `GetCurrentCommandId(false)` 不应该设置 `currentCommandIdUsed`？
2. 为什么 CCI 可以在只读命令后成为 no-op？
3. 为什么同一条 UPDATE 不会反复看到自己刚生成的新版本？
4. tuple header 为什么不直接保存两个 command id？
5. combo CID 为什么可以是 backend-local 状态？
6. parallel worker 为什么不能调用 `GetCurrentCommandId(true)` 修改数据？
7. logical decoding 为什么需要 `XLOG_HEAP2_NEW_CID`？
8. subtransaction abort 为什么会影响 self visibility 判断？

## 16. 本节小结

本节主问题是：

- 同一事务内部如何用 `CommandId` 和 combo CID 判断 self visibility。

核心链路是：

```text
GetCurrentCommandId(true)
  -> heap insert/update/delete 写 cmin/cmax
  -> CommandCounterIncrement()
  -> SnapshotSetCommandId()
  -> HeapTupleSatisfiesMVCC()
  -> cmin/cmax 与 snapshot->curcid 比较
  -> 必要时 HeapTupleHeaderAdjustCmax() 创建 combo CID
```

核心状态是：

- `currentCommandId` 表示事务内命令时间轴。
- `currentCommandIdUsed` 决定 CCI 是否需要推进。
- `snapshot->curcid` 是当前 snapshot 对本事务命令的可见边界。
- tuple header 的 raw `t_cid` 需要结合 `HEAP_COMBOCID` 解释。
- combo CID table 保存 `{cmin, cmax}` 映射。

正确性边界是：

- 外部事务可见性由 MVCC snapshot 和 XID 状态决定。
- 当前事务可见性由 XID 是否属于当前事务、subtransaction 状态、command id 顺序共同决定。
- RR 固定外部世界，但不冻结 self visibility。

异常和边界是：

- command id overflow 报错。
- parallel worker 不能创建新的写 command 状态。
- combo CID OOM 必须发生在 critical section 前。
- logical decoding 用 `XLOG_HEAP2_NEW_CID` 把 backend-local CID 语义带到 decoder。

可迁移规律：

- 当一个系统把全局版本和事务内局部时间轴分开时，读路径必须同时解释两套时间。
- PostgreSQL 用 XID 解决事务间顺序，用 CommandId 解决事务内顺序，用 combo CID 在不扩大 tuple header 的前提下保存少数双命令语义。
