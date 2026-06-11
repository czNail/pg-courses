# PostgreSQL 同事务写入、删除与版本替换可见性

## 课程定位

前置知识：已经理解普通 `SNAPSHOT_MVCC` 如何用 `xmin/xmax/snapshot` 判断外部事务的 tuple 可见性，也已经理解 tuple header 中的 `cmin/cmax` 和 combo cid 只在当前事务语境下有完整语义。

本节唯一主问题：

```text
为什么当前事务自己 INSERT、UPDATE、DELETE 的 tuple version 不能只按 XID 判断，还必须结合 command id、combo cid、HeapTupleSatisfiesSelf() 和 HeapTupleSatisfiesUpdate() 这样的特殊规则？
```

本节核心矛盾：

```text
同一个事务需要看到自己已经完成的前序命令效果；
但不能让当前命令内部的中间修改破坏语句级一致性、重复更新规则、ON CONFLICT 语义和并发冲突检测。
```

学完本节后，你应该能独立判断：

- 为什么 `GetSnapshotData()` 不把本 backend 的 XID 放进 snapshot arrays。
- 为什么当前事务自己的 tuple 要先走 `TransactionIdIsCurrentTransactionId()`。
- 为什么 `snapshot->curcid` 能切开同一事务内不同 SQL 命令。
- 为什么插入后又删除同一 tuple 需要 combo cid。
- 为什么 `HeapTupleSatisfiesSelf()` 与普通 MVCC 不完全一样。
- 为什么 `heap_update()` / `heap_delete()` 使用 `HeapTupleSatisfiesUpdate()`，而不是普通 SELECT 的 MVCC 判断。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

本节不展开所有 tuple lock MultiXact 分支。

这里只讲“当前事务自己改过的 tuple”为什么需要独立规则。

## 1. 本节在总主线中的位置

第 14 节讲的是普通 SELECT。

普通 SELECT 的核心是：

```text
外部事务的效果是否属于当前 snapshot。
```

但当前事务自己的修改不是普通外部事务。

一个事务可以这样执行：

```sql
BEGIN;
INSERT INTO t VALUES (1);
SELECT * FROM t;
UPDATE t SET ... WHERE id = 1;
SELECT * FROM t;
DELETE FROM t WHERE id = 1;
SELECT * FROM t;
COMMIT;
```

这些语句都在同一个 XID 下。

如果只看 XID，就无法区分：

```text
这是当前事务第一条命令插入的 tuple。
这是当前事务当前命令刚插入的 tuple。
这是当前事务前序命令删除的 tuple。
这是当前事务当前命令正在更新的旧版本。
```

所以 PostgreSQL 在 tuple header 中存 command id。

在 snapshot 中存 `curcid`。

在必要时用 combo cid 同时表达 `cmin` 与 `cmax`。

本节主线是：

```text
当前事务遇到自己的 tuple
  -> 不能走普通 external xid membership
  -> 先判断 cmin/cmax 与 curcid
  -> SELECT 使用 MVCC 或 Self 规则
  -> UPDATE/DELETE 使用 HeapTupleSatisfiesUpdate()
  -> 返回 visible、self-modified、invisible、being-modified 等不同语义
```

这一步之后，才容易理解特殊 snapshot 和 UPDATE chain。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
当前事务的 XID 只说明“这是我做的修改”，不能说明“这是我哪条命令做的修改”；PostgreSQL 用 tuple header 的 cmin/cmax、snapshot->curcid 和 combo cid 切分同一事务内的命令顺序，并用 HeapTupleSatisfiesSelf()/Update() 给读自己、改自己和冲突检测不同答案。
```

这里的 tension 是：

```text
事务内读写需要 read-your-writes；
语句级 MVCC 又要求当前命令的中间结果不能随意进入同一命令的扫描结果。
```

如果只用 XID，会出现两类错误。

第一类是看不见自己已经完成的修改。

这会破坏用户直觉和 SQL 执行。

第二类是看见当前命令尚不该看见的修改。

这会导致同一语句重复处理同一行，或者让 UPDATE/DELETE 的 Halloween 问题更难控制。

PostgreSQL 的解决方式是：

```text
XID:
  区分哪个事务。

CID:
  区分同一事务内哪个命令。

combo CID:
  在一个 tuple 同时需要 cmin 和 cmax 时提供映射。

special visibility routine:
  根据操作目的返回不同语义。
```

## 3. 核心文件分工与阅读顺序

| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/access/heap/heapam_visibility.c` | 主读 `HeapTupleSatisfiesMVCC()` 当前事务分支、`HeapTupleSatisfiesSelf()`、`HeapTupleSatisfiesUpdate()`。 |
| 2 | `src/include/utils/snapshot.h` | 对照 `SNAPSHOT_MVCC` 与 `SNAPSHOT_SELF` 对当前命令的语义差异。 |
| 3 | `src/backend/utils/time/combocid.c` | 读 `HeapTupleHeaderGetCmin()`、`HeapTupleHeaderGetCmax()`、`HeapTupleHeaderAdjustCmax()`、combo cid 生命周期。 |
| 4 | `src/include/access/htup_details.h` | 对照 `HEAP_COMBOCID`、`t_field3.t_cid`、`HeapTupleHeaderSetCmin()`、`HeapTupleHeaderSetCmax()`。 |
| 5 | `src/backend/access/heap/heapam.c` | 看 `heap_update()`、`heap_delete()` 如何消费 `HeapTupleSatisfiesUpdate()` 的 `TM_Result`。 |
| 6 | `src/include/access/tableam.h` | 对照 `TM_Result`、`TM_FailureData` 等 table AM 对外语义。 |
| 7 | `src/backend/access/transam/xact.c` | 看 command id 推进和事务结束时 combo cid cleanup。 |

推荐阅读顺序：

```text
先读 snapshot.h 对 SNAPSHOT_MVCC / SNAPSHOT_SELF 的语义差异
  -> 读 HeapTupleSatisfiesMVCC() 中 current transaction 分支
  -> 读 HeapTupleSatisfiesSelf()
  -> 读 HeapTupleSatisfiesUpdate()
  -> 最后读 combocid.c 解释一个字段如何同时表达 cmin/cmax
```

## 4. 一个 runtime 现象先定锚

先看最简单的 read-your-writes。

```sql
DROP TABLE IF EXISTS self_vis_demo;
CREATE TABLE self_vis_demo(id int primary key, payload text);

BEGIN;
INSERT INTO self_vis_demo VALUES (1, 'a');
SELECT * FROM self_vis_demo WHERE id = 1;
COMMIT;
```

当前事务能看到自己刚插入的行。

但这个说法还不够精确。

更准确地说：

```text
后续命令可以看到前序命令插入的行。
```

同一条命令内部，规则不同。

再看一个例子：

```sql
BEGIN;
INSERT INTO self_vis_demo VALUES (2, 'b');
UPDATE self_vis_demo SET payload = payload || '_u' WHERE id = 2;
SELECT * FROM self_vis_demo WHERE id = 2;
ROLLBACK;
```

`UPDATE` 能找到前一条命令插入的行。

但它不能把当前命令自己刚刚生成的新版本又反复更新。

这就是 command id 边界。

`xmin` 都是当前事务。

只有 `cmin/cmax` 与 `curcid` 能区分先后。

如果你用 `pageinspect` 看 page，会看到同一个事务产生多个版本。

这些版本不能靠 XID 单独排序。

必须结合 command id。

## 5. `curcid`: snapshot 中的当前命令边界

`SnapshotData` 中有：

```text
CommandId curcid;
```

它的语义是：

```text
在当前事务中，CID < curcid 的修改对这个 snapshot 可见。
```

普通 MVCC snapshot 包含前序命令效果。

不包含当前命令效果。

因此在 `HeapTupleSatisfiesMVCC()` 中：

```text
if xmin is current transaction:
  if cmin >= snapshot->curcid:
    return false
```

这表示：

```text
当前命令开始后插入的 tuple，不属于当前命令开始时的 snapshot。
```

对删除或更新也是类似。

如果当前事务的 `xmax` 有效：

```text
if cmax >= snapshot->curcid:
  删除发生在当前命令之后
  旧版本仍可见

else:
  删除发生在当前命令之前
  旧版本不可见
```

这个逻辑让一个事务既能 read-your-writes，又能保持每条语句扫描时的稳定边界。

## 6. `cmin`、`cmax` 与 combo cid

tuple header 的 `t_field3.t_cid` 只有一个槽。

插入时，它可以存 `cmin`。

删除或更新时，它可以存 `cmax`。

如果一个 tuple 在同一事务内被插入又被删除，它同时需要：

```text
cmin:
  什么时候出生。

cmax:
  什么时候死亡。
```

一个字段放不下两个值。

PostgreSQL 使用 combo cid。

流程是：

```text
HeapTupleHeaderAdjustCmax()
  -> 如果 tuple 的 xmin 属于当前事务
  -> 读取 cmin
  -> 生成或复用 (cmin, cmax) 到 combo cid 的映射
  -> 把 tuple header 中的 cid 替换为 combo cid
  -> 设置 HEAP_COMBOCID
```

之后读取时：

```text
HeapTupleHeaderGetCmin()
  -> 如果 HEAP_COMBOCID，展开真实 cmin。

HeapTupleHeaderGetCmax()
  -> 如果 HEAP_COMBOCID，展开真实 cmax。
```

这个映射是 backend-local、transaction-local。

事务结束后会清理。

因此 combo cid 不是全局可解释状态。

它只服务当前事务自己的可见性判断和冲突结果。

## 7. `HeapTupleSatisfiesSelf()`

`SNAPSHOT_SELF` 的语义和普通 MVCC 不同。

它能看到：

```text
所有当前时刻已提交事务的效果。
当前事务以前命令的效果。
当前事务当前命令的效果。
```

它看不到：

```text
其他仍在进行中的事务效果。
```

所以 `HeapTupleSatisfiesSelf()` 更适合系统内部需要“把自己刚做的修改也看成可见”的场景。

它不是普通 SELECT 的默认规则。

普通 `SNAPSHOT_MVCC` 明确不包含当前命令的修改。

区别可以压缩成：

```text
SNAPSHOT_MVCC:
  看当前事务之前命令。
  不看当前命令。

SNAPSHOT_SELF:
  看当前事务之前命令。
  也看当前命令。
```

这就是为什么源码中有多个 visibility routine。

不是因为普通 MVCC 不够强。

而是不同内部操作需要不同可见性承诺。

## 8. `HeapTupleSatisfiesUpdate()`

UPDATE 和 DELETE 不能只问：

```text
这个 tuple 对我的 snapshot 是否可见？
```

它们还要问：

```text
我能不能修改它？
如果不能，是因为我自己已经改过，还是别人正在改，还是别人已经更新或删除？
```

所以 `heap_update()`、`heap_delete()` 使用：

```text
HeapTupleSatisfiesUpdate(tuple, cid, buffer)
```

它返回的是 `TM_Result` 语义，而不是简单 bool。

典型结果包括：

```text
TM_Ok
TM_Invisible
TM_SelfModified
TM_Updated
TM_Deleted
TM_BeingModified
```

这让调用者能决定：

```text
直接修改。
报错。
等待并重试。
返回并发更新冲突。
填充 TM_FailureData。
```

当前事务分支里，`HeapTupleSatisfiesUpdate()` 会用 `curcid` 判断：

```text
这个 tuple 是当前命令插入的吗？
它是否已经被当前事务前序命令更新或删除？
```

这比普通 SELECT 的 bool 更细。

## 9. 主流程源码 walkthrough

### 9.1 普通 SELECT 读自己前序命令

流程：

```text
SELECT 扫描 tuple
  -> HeapTupleSatisfiesVisibility()
  -> HeapTupleSatisfiesMVCC()
  -> xmin 是 current transaction
  -> cmin < snapshot->curcid
  -> 出生侧通过
  -> xmax invalid
  -> visible
```

这就是 read-your-writes。

它不是因为当前事务被放进 snapshot arrays。

实际上 `GetSnapshotData()` 不把本 backend 的 XID 放进去。

它靠显式 current transaction 分支处理。

### 9.2 普通 SELECT 不读当前命令插入

如果扫描遇到当前命令刚插入的 tuple：

```text
xmin 是 current transaction
cmin >= snapshot->curcid
return false
```

这避免一个语句在扫描过程中反复看到自己的新行。

### 9.3 UPDATE 修改自己前序命令插入的行

`heap_update()` 读目标 tuple。

调用：

```text
HeapTupleSatisfiesUpdate(&oldtup, cid, buffer)
```

如果 tuple 是当前事务前序命令插入，且未被当前事务前序命令删除：

```text
TM_Ok
```

然后 `heap_update()` 生成新版本。

旧版本的 `xmax/cmax/t_ctid` 被更新。

新版本的 `xmin/cmin` 属于当前事务当前命令。

### 9.4 UPDATE 遇到当前事务已经修改过的 tuple

如果 tuple 已经被当前事务当前或前序命令更新/删除，`HeapTupleSatisfiesUpdate()` 可能返回：

```text
TM_SelfModified
```

调用者通过 `TM_FailureData` 获得必要信息。

这避免同一语句或同一事务在不合适的语义下重复修改同一物理版本。

### 9.5 插入后删除同一 tuple

当同一事务删除自己插入的 tuple：

```text
xmin = current xid
xmax = current xid
cmin 和 cmax 都有意义
```

此时需要 combo cid。

`HeapTupleHeaderAdjustCmax()` 在进入 critical section 修改 tuple header 前完成可能失败的内存分配。

这样真正写 page 时不因 OOM 打断关键区。

## 10. 生命周期 / ownership / cleanup

command id 是事务生命周期内的状态。

它随命令推进。

tuple header 中的 `t_field3` 会持久存在。

但它的完整解释可能只在当前事务中有意义。

combo cid table 是 backend-local。

它分配在事务上下文中。

事务结束时清理。

这意味着：

```text
tuple header 中的 HEAP_COMBOCID 不等于所有后续事务都能展开真实 cmin/cmax。
```

后续事务通常也不需要展开。

它们判断可见性依赖事务提交状态和 snapshot。

当前事务需要展开，是因为它必须区分自己内部的命令顺序。

`HeapTupleSatisfiesUpdate()` 不负责 cleanup tuple。

它只告诉调用者当前是否可以修改。

真正修改和失败清理由 `heap_update()`、`heap_delete()` 负责。

## 11. 正确性机制层次

第一层是当前事务识别。

`TransactionIdIsCurrentTransactionId()` 处理顶层事务和子事务归属。

第二层是 command id 边界。

`curcid` 把当前事务内部命令切开。

第三层是 combo cid。

同一 tuple 同时有 `cmin` 和 `cmax` 时，保证二者可恢复。

第四层是 operation-specific routine。

SELECT、self visibility、UPDATE/DELETE 需要不同返回语义。

第五层是调用者处理。

`heap_update()` 根据 `TM_Result` 决定 wait、retry、报错或生成新版本。

这些层共同维护：

```text
同一事务内读写既要符合 SQL 可见性，又不能破坏物理版本链和并发更新规则。
```

## 12. 错误路径 / 异常路径 / fallback

### 12.1 combo cid 分配可能失败

`HeapTupleHeaderAdjustCmax()` 可能需要分配 combo cid。

它必须在 critical section 之前完成。

如果内存分配失败，错误发生在修改 shared buffer 之前。

这避免 page 上出现半修改状态。

### 12.2 并行模式禁止需要 combo cid 的更新

`heap_update()` 中可以看到并行模式限制。

原因是 combo cid 没有跨 worker 广播机制。

如果并行 worker 需要解释别的 worker 的 combo cid，就会失去语义。

所以这类修改在 parallel operation 中被拒绝。

### 12.3 当前事务多子事务

当前事务判断不只看 top-level xid。

子事务也属于当前事务。

如果子事务回滚，相关 tuple 需要按 abort 语义处理。

这就是为什么当前事务判断要通过事务系统 accessor，而不是裸比较一个 XID。

### 12.4 `TM_SelfModified`

UPDATE/DELETE 遇到自己已经修改过的 tuple 时，不应该简单当作 invisible。

调用者需要知道这是 self-modified。

这样才能给 SQL 层或 executor 正确语义。

## 13. 成本、资源与跨模块传播

command id 判断本身很便宜。

成本主要来自复杂语义带来的额外状态：

```text
tuple header 要保存 cid 或 combo cid。
当前事务要维护 combo cid table。
UPDATE/DELETE 要返回比 bool 更细的 TM_Result。
并行写路径要禁止无法广播的 combo cid 语义。
```

这些成本换来的是：

```text
当前事务 read-your-writes。
语句级快照稳定。
同一事务内 UPDATE/DELETE 不重复误处理。
错误路径能区分 self-modified 与 concurrent modified。
```

跨模块传播包括：

```text
snapmgr:
  提供 curcid。

xact.c:
  推进 command id。

combocid.c:
  保存 cmin/cmax 映射。

heapam_visibility.c:
  判断 self visibility 和 update eligibility。

heapam.c:
  按 TM_Result 执行更新、删除、等待或失败返回。

executor:
  依赖这些结果维持 SQL 语义。
```

## 14. 观测与诊断入口

SQL 观测：

```sql
SELECT xmin, xmax, ctid, * FROM self_vis_demo;
```

pageinspect：

```sql
CREATE EXTENSION IF NOT EXISTS pageinspect;

SELECT lp,
       t_xmin,
       t_xmax,
       t_field3,
       t_ctid,
       t_infomask
FROM heap_page_items(get_raw_page('self_vis_demo', 0))
ORDER BY lp;
```

源码断点：

```text
break HeapTupleSatisfiesMVCC
break HeapTupleSatisfiesSelf
break HeapTupleSatisfiesUpdate
break HeapTupleHeaderAdjustCmax
break HeapTupleHeaderGetCmin
break HeapTupleHeaderGetCmax
```

关键变量：

```text
snapshot->curcid
cid
tuple->t_choice.t_heap.t_field3.t_cid
tuple->t_infomask & HEAP_COMBOCID
result
```

观察时要问：

```text
这个 tuple 是当前事务插入的吗？
它是当前命令插入的吗？
它是否也被当前事务删除或更新？
调用者需要 bool，还是 TM_Result？
```

## 15. 常见误区

误区一：

```text
同一个事务内所有修改都天然可见。
```

不对。

普通 MVCC snapshot 不看当前命令的修改。

误区二：

```text
当前事务的 tuple 不需要 snapshot。
```

不对。

仍然需要 `curcid`。

误区三：

```text
cmin/cmax 是全局时间戳。
```

不对。

它们只在事务内部定义顺序。

误区四：

```text
combo cid 可以被任何事务展开。
```

不对。

combo cid table 是当前 backend 的事务局部状态。

误区五：

```text
UPDATE 只需要判断 tuple 是否可见。
```

不对。

UPDATE 还要区分 self-modified、being-modified、updated、deleted 等结果。

## 16. 课堂实验

### 实验一：前序命令可见

```sql
DROP TABLE IF EXISTS self_vis_demo;
CREATE TABLE self_vis_demo(id int primary key, payload text);

BEGIN;
INSERT INTO self_vis_demo VALUES (1, 'a');
SELECT xmin, xmax, ctid, * FROM self_vis_demo;
COMMIT;
```

解释目标：

```text
后续 SELECT 能看到前序 INSERT。
```

### 实验二：插入后更新

```sql
BEGIN;
INSERT INTO self_vis_demo VALUES (2, 'b');
UPDATE self_vis_demo SET payload = 'b2' WHERE id = 2;
SELECT xmin, xmax, ctid, * FROM self_vis_demo WHERE id = 2;
ROLLBACK;
```

解释目标：

```text
同一事务内生成多个版本。
XID 相同不足以解释哪个版本当前可见。
```

### 实验三：插入后删除

```sql
BEGIN;
INSERT INTO self_vis_demo VALUES (3, 'c');
DELETE FROM self_vis_demo WHERE id = 3;
SELECT * FROM self_vis_demo WHERE id = 3;
ROLLBACK;
```

源码观察：

```text
break HeapTupleHeaderAdjustCmax
break HeapTupleSatisfiesUpdate
```

目标：

```text
理解为什么同一 tuple 可能同时需要 cmin 和 cmax。
```

### 实验四：比较 MVCC 与 SELF

这个实验适合源码断点。

在不同调用点观察：

```text
snapshot->snapshot_type
snapshot->curcid
```

对比：

```text
SNAPSHOT_MVCC:
  不看当前命令。

SNAPSHOT_SELF:
  看当前命令。
```

## 17. 讨论题

1. 为什么当前事务的 XID 不放进 snapshot arrays？

2. 如果没有 `curcid`，同一个事务内 UPDATE 自己刚插入的 tuple 会出现什么歧义？

3. 为什么 combo cid 必须在 critical section 前分配？

4. 为什么 `HeapTupleSatisfiesUpdate()` 不能只返回 bool？

5. 为什么 parallel operation 中需要防止产生无法传播的 combo cid？

6. `SNAPSHOT_SELF` 比 `SNAPSHOT_MVCC` 多看当前命令，这对系统维护路径有什么价值？

## 18. 本节小结

本节把 MVCC 从“事务之间”推进到“同一事务内部”。

核心结论是：

```text
XID 只能区分事务；
CID 才能区分同一事务内的命令。
```

普通 MVCC snapshot 通过 `curcid` 看前序命令，不看当前命令。

`HeapTupleSatisfiesSelf()` 提供包含当前命令的特殊视角。

`HeapTupleSatisfiesUpdate()` 为 UPDATE/DELETE 提供比 bool 更细的修改资格判断。

combo cid 解决同一 tuple 同时需要 `cmin` 和 `cmax` 的存储问题。

下一节会继续扩展 visibility routine：

```text
为什么系统维护、toast、锁冲突探测和 vacuum 判断不能都使用普通 MVCC snapshot。
```
