# PostgreSQL HeapTupleSatisfiesMVCC 判定框架

## 课程定位

前置知识：已经理解 heap tuple header 中的 `xmin`、`xmax`、`t_ctid`、`t_infomask`、hint bit 和 command id，也已经理解 `SnapshotData` 用 `xmin/xmax/xip/subxip/curcid` 描述一个读者视角。

本节唯一主问题：

```text
普通 SELECT 如何把 tuple header、snapshot、CLOG 和当前事务命令边界合成一个“这个 tuple version 对我是否可见”的判断？
```

本节核心矛盾：

```text
SELECT 的 heap visibility 是高频 hot path，必须尽量靠 header flag 和 snapshot 快速返回；
但它又必须正确处理插入事务、删除事务、当前事务、MultiXact、行锁、异步提交和 hint bit 滞后。
```

学完本节后，你应该能独立判断：

- 为什么 `HeapTupleSatisfiesMVCC()` 先判断 `xmin`，再判断 `xmax`。
- 为什么 `xmin committed` 以后还要问 `XidInMVCCSnapshot()`。
- 为什么 `xmax invalid`、lock-only、running、aborted 和 committed 对可见性含义不同。
- 为什么当前事务要先比较 `cmin/cmax` 与 `snapshot->curcid`。
- 为什么普通 SELECT 不会为了设置 hint bit 去重新检查仍在 snapshot running set 中的事务。
- 为什么 batch visibility 能减少 per-tuple 函数调用和 hint bit 写回成本。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

本节不展开 `HeapTupleSatisfiesSelf()`、`HeapTupleSatisfiesDirty()`、HOT chain 搜索和 cleanup horizon。

这些会在后续课程分别展开。

## 1. 本节在总主线中的位置

上一节讲 tuple header 如何表达一个版本。

现在进入普通读路径。

一个 heap scan 或 index scan 最终会拿到一个 heap tuple。

它要回答一个最小问题：

```text
这个 tuple version 属不属于当前 MVCC snapshot？
```

这个问题看似是一个布尔判断。

但它背后要消化三类状态。

第一类是 tuple-local 状态。

```text
xmin
xmax
cmin/cmax
infomask
infomask2
```

第二类是 snapshot-local 状态。

```text
snapshot->xmin
snapshot->xmax
snapshot->xip
snapshot->subxip
snapshot->suboverflowed
snapshot->curcid
```

第三类是全局事务状态。

```text
CLOG/pg_xact commit status
SubTrans fallback
MultiXact membership
commit LSN flush boundary
```

`HeapTupleSatisfiesMVCC()` 把这些状态折叠成一个返回值。

它不负责生成 snapshot。

它不负责清理 dead tuple。

它也不负责沿 HOT chain 找 child。

它只判断：

```text
在给定 snapshot 下，这一个 heap tuple version 是否可见。
```

本节的线性主流程是：

```text
heap scan / index scan 得到 tuple
  -> HeapTupleSatisfiesVisibility()
  -> HeapTupleSatisfiesMVCC()
  -> xmin 分支判断出生是否可见
  -> xmax 分支判断死亡是否屏蔽
  -> 必要时设置 hint bit
  -> 返回 visible / invisible
```

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
HeapTupleSatisfiesMVCC() 先证明 tuple 的创建事务对当前 snapshot 可见，再证明它没有被当前 snapshot 应该看见的删除或更新事务杀死；过程中优先消费 hint bit 和 snapshot membership，只有必要时才查事务状态，并把安全的结果写回 hint bit。
```

这里的 tension 是：

```text
每个 tuple 都要快速判断；
但 MVCC correctness 要求判断结果与 snapshot 创建时的运行事务集合一致，而不是与当前最新事务状态一致。
```

所以它不会简单问：

```text
xmin 是否已经提交？
xmax 是否已经提交？
```

它真正问的是：

```text
xmin 的效果是否属于当前 snapshot？
xmax 的效果是否已经在当前 snapshot 中生效？
```

这两个问题方向相反。

`xmin` 如果不可见，整个 tuple 不可见。

`xmax` 如果可见，说明删除或更新已经生效，tuple 不可见。

因此主判断可以压缩成：

```text
出生没通过:
  invisible

出生通过且死亡无效:
  visible

出生通过且死亡在 snapshot 看来尚未发生:
  visible

出生通过且死亡在 snapshot 看来已经发生:
  invisible
```

所有复杂分支都在维护这四句话。

## 3. 核心文件分工与阅读顺序

| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/access/heap/heapam_visibility.c` | 主读 `HeapTupleSatisfiesMVCC()`、`HeapTupleSatisfiesVisibility()`、`HeapTupleSatisfiesMVCCBatch()`。 |
| 2 | `src/include/utils/snapshot.h` | 对照 `SNAPSHOT_MVCC` 的语义和 `curcid` 规则。 |
| 3 | `src/backend/utils/time/snapmgr.c` | 读 `XidInMVCCSnapshot()`，理解 snapshot running set membership。 |
| 4 | `src/include/access/htup_details.h` | 对照 `HEAP_XMIN_*`、`HEAP_XMAX_*`、`HEAP_XMAX_IS_MULTI`、`HEAP_XMAX_LOCK_ONLY`。 |
| 5 | `src/backend/utils/time/combocid.c` | 看 `HeapTupleHeaderGetCmin()` / `GetCmax()` 为什么只对当前事务有意义。 |
| 6 | `src/include/access/heapam.h` | 看 visibility routine 的声明和 batch input/output 结构。 |
| 7 | `src/backend/access/transam/transam.c` / `src/backend/access/transam/xact.c` | 对照 `TransactionIdDidCommit()` 与提交状态来源。 |

推荐阅读顺序：

```text
先读 snapshot.h 的 SNAPSHOT_MVCC 注释
  -> 读 HeapTupleSatisfiesMVCC() 顶部关于 hint bit 的注释
  -> 只跟 xmin 半边
  -> 再跟 xmax 半边
  -> 最后回到 XidInMVCCSnapshot() 看 running membership
```

不要第一次就试图记住每一个分支。

先把两个问题分开：

```text
这个版本是否已经出生？
这个版本是否已经死亡？
```

## 4. 一个 runtime 现象先定锚

Session A：

```sql
DROP TABLE IF EXISTS mvcc_visible_demo;
CREATE TABLE mvcc_visible_demo(id int primary key, payload text);

BEGIN;
INSERT INTO mvcc_visible_demo VALUES (1, 'from A');
-- 保持事务打开，不提交。
```

Session B：

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT * FROM mvcc_visible_demo WHERE id = 1;
```

此时 Session B 看不到这行。

因为 tuple 的 `xmin` 属于 Session A。

Session B 的 snapshot 把 Session A 的 XID 记录为 running。

`HeapTupleSatisfiesMVCC()` 在 `xmin` 分支会返回 invisible。

现在 Session A：

```sql
COMMIT;
```

Session B 在同一个 repeatable read 事务中再次查询：

```sql
SELECT * FROM mvcc_visible_demo WHERE id = 1;
```

仍然看不到。

原因不是 CLOG 还不知道 commit。

原因是 Session B 的 snapshot 在创建时已经把 Session A 的 XID 当作 running。

于是 `XidInMVCCSnapshot(xmin, snapshot)` 仍然为 true。

如果 Session B 结束事务，再开始新事务：

```sql
COMMIT;
SELECT * FROM mvcc_visible_demo WHERE id = 1;
```

新 snapshot 才能看到这行。

这个现象说明：

```text
HeapTupleSatisfiesMVCC() 判断的不是“事务现在是否提交”；
而是“事务效果是否属于这个 snapshot”。
```

## 5. 关键状态：`SNAPSHOT_MVCC`

`snapshot.h` 对 `SNAPSHOT_MVCC` 的语义很明确。

它能看到：

```text
snapshot 创建时已经提交的事务效果。
当前事务以前命令的效果。
```

它看不到：

```text
snapshot 中显示为 in-progress 的事务。
snapshot 创建后才开始的事务。
当前命令刚刚做的修改。
```

这三条直接映射到 `HeapTupleSatisfiesMVCC()`。

当前事务以前命令：

```text
TransactionIdIsCurrentTransactionId(xmin)
  -> HeapTupleHeaderGetCmin(tuple) < snapshot->curcid
```

snapshot 中 running 的外部事务：

```text
XidInMVCCSnapshot(xid, snapshot)
```

snapshot 创建后才开始的事务：

```text
xid >= snapshot->xmax
  -> XidInMVCCSnapshot() 返回 true
```

已经提交但在 snapshot 看来还 running 的事务：

```text
hint bit 可能显示 committed。
但 XidInMVCCSnapshot() 仍然要把它当作 running。
```

最后这一点最重要。

可见性不是全局实时状态。

可见性是 snapshot-consistent 状态。

## 6. `xmin` 半边：先证明版本出生

`HeapTupleSatisfiesMVCC()` 首先处理 `xmin`。

如果插入事务没有通过，后面不用看 `xmax`。

因为一个未出生的版本不可能可见。

`xmin` 半边大致是：

```text
if xmin not hinted committed:
  if xmin invalid:
    return false

  if moved tuple cleanup says invisible:
    return false

  if xmin is current transaction:
    compare cmin with snapshot->curcid
    then继续处理同事务 xmax

  else if xmin in MVCC snapshot:
    return false

  else if xmin committed in CLOG:
    set HEAP_XMIN_COMMITTED hint
    continue

  else:
    set HEAP_XMIN_INVALID hint
    return false

else:
  if xmin not frozen and xmin in MVCC snapshot:
    return false
```

这个分支有几个关键点。

第一，hint bit 是 fast path。

如果已经知道 `HEAP_XMIN_INVALID`，可以立即返回 false。

第二，当前事务要先处理。

因为 `GetSnapshotData()` 不会把本 backend 的 XID 存进 snapshot arrays。

所以不能指望 `XidInMVCCSnapshot()` 告诉你“这是我的 XID”。

第三，即使 `HEAP_XMIN_COMMITTED` 已经设置，也不能直接通过。

如果这个 committed XID 仍在当前 snapshot 的 running set 里，就要当作不可见。

这解释了 repeatable read 的现象。

第四，只有当 snapshot 不认为它 running，才会查真实事务状态。

如果事务在 snapshot 看来仍 running，即使它现在已经提交，也不能因为最新状态改变本次判断。

## 7. 当前事务插入的 tuple

当前事务路径是 MVCC 中最容易绕的部分。

当 `xmin` 是当前事务时，系统比较：

```text
HeapTupleHeaderGetCmin(tuple) >= snapshot->curcid
```

如果成立，说明该 tuple 是当前命令开始后插入的。

普通 MVCC snapshot 不看当前命令的修改。

所以返回 false。

如果 `cmin < curcid`，说明它来自当前事务之前命令。

出生侧通过。

接下来还要看它是否被当前事务删除或更新。

这时 `xmax` 也可能属于当前事务。

如果删除 command id 大于等于 `snapshot->curcid`，删除发生在当前命令之后。

当前 snapshot 仍应看到它。

如果删除 command id 小于 `snapshot->curcid`，删除发生在当前命令之前。

当前 snapshot 不应看到它。

这就是 command id 的作用：

```text
同一个 XID 内部，靠 CID 划分语句顺序。
```

如果没有 command id，当前事务会无法稳定回答：

```text
我应该看到自己刚插入的 tuple 吗？
我应该看到自己刚删除的 tuple 吗？
UPDATE 同一行后旧版本是否还对本语句可见？
```

第 16 节会专门展开这部分。

## 8. `xmax` 半边：再证明版本没有死亡

一旦 `xmin` 通过，tuple 已经出生。

接下来 `xmax` 决定它是否被删除或更新屏蔽。

最简单情况：

```text
HEAP_XMAX_INVALID:
  visible
```

这表示没有有效删除者或更新者。

另一个 fast path：

```text
HEAP_XMAX_LOCK_ONLY:
  visible
```

行锁不是删除。

即使 `xmax` 有值，只要它只是 locker，普通 SELECT 仍然能看到 tuple。

如果 `xmax` 是 MultiXact，就要解析真正的 update xid。

普通 locker 成员不应让 tuple 消失。

真正 updater 或 deleter 才会影响可见性。

如果 `xmax` 是当前事务，要看 `cmax` 与 `curcid`。

如果删除或更新发生在当前命令之后，当前 snapshot 仍看到旧版本。

如果发生在当前命令之前，旧版本不可见。

如果 `xmax` 是外部事务：

```text
如果 xmax 在当前 snapshot 中 running:
  删除在本 snapshot 看来尚未生效
  visible

如果 xmax 已回滚:
  删除无效
  visible

如果 xmax 已提交:
  删除生效
  invisible
```

注意方向：

`xmin running` 会让 tuple 不可见。

`xmax running` 会让 tuple 可见。

因为前者是“出生还没发生”。

后者是“死亡还没发生”。

## 9. `XidInMVCCSnapshot()` 的位置

`HeapTupleSatisfiesMVCC()` 不直接扫描 snapshot arrays。

它调用 `XidInMVCCSnapshot()`。

这个函数的核心规则是：

```text
xid < snapshot->xmin:
  not in progress

xid >= snapshot->xmax:
  in progress

xmin <= xid < xmax:
  查 subxip / xip
  overflow 时通过 pg_subtrans 找 top-level xid
```

所以它回答的是：

```text
这个 xid 在这个 snapshot 看来是否仍在运行？
```

不是：

```text
这个 xid 现在是否还在运行？
```

这一区别决定了整节课的正确性。

当 `HeapTupleSatisfiesMVCC()` 看到一个 committed hint bit 时，仍可能调用 `XidInMVCCSnapshot()`。

因为事务真实已提交，不代表它在旧 snapshot 中可见。

反过来，当 `XidInMVCCSnapshot()` 返回 true 时，函数通常不会再查真实事务状态。

因为对这个 snapshot 来说，它就是 running。

即使现实中它刚刚提交，也不能改变当前 snapshot 结果。

## 10. hint bit 的使用边界

`HeapTupleSatisfiesMVCC()` 可以设置 hint bit。

但它很克制。

源码注释强调：

```text
如果插入或删除事务仍然在当前 snapshot 中显示为 running，不要为了设置 hint bit 去查真实事务状态。
```

原因有两个。

第一，查真实事务状态可能访问高竞争共享结构。

第二，查到了也不会改变本 snapshot 的可见性结果。

旧写法试图尽早设置 hint bit。

这会在事务仍 running 时反复查 ProcArray。

当前设计宁愿让 hint bit 稍后由更新的 snapshot 设置。

这样把 hot path 成本压低。

当可以安全设置 hint bit 时，函数使用：

```text
SetHintBitsExt()
```

它会遵守 commit LSN flush 边界。

如果事务 commit 记录还没安全落盘，不能让 data page 先持久化一个暗示 committed 的 hint bit。

这部分第 15 节展开。

本节只需要记住：

```text
hint bit 是结果缓存；
MVCC 判断仍以 snapshot 语义为准。
```

## 11. batch MVCC 判断

当前源码里还有 `HeapTupleSatisfiesMVCCBatch()`。

它对一批 tuple 执行 MVCC 判断。

核心收益是：

```text
避免每个 tuple 跨 translation unit 调用。
让编译器优化跨调用逻辑。
把 hint bit 写回的 BufferBeginSetHintBits / BufferFinishSetHintBits 成本摊到一批 tuple 上。
```

调用路径可以从 heap scan 中看到。

当一个 page 上有多条 tuple 要判断时，batch 判断能减少 per-tuple overhead。

但 batch 不改变语义。

每个 tuple 的最终判断仍然等价于：

```text
HeapTupleSatisfiesMVCC(tuple, snapshot, buffer, state)
```

这体现了 PostgreSQL 在 hot path 上常见的分层：

```text
先保持单 tuple 语义清晰；
再在调用层做批量化和状态摊销。
```

## 12. 主流程源码 walkthrough

### 12.1 heap scan 入口

普通 heap scan 会读 page。

对每个 line pointer 构造 `HeapTupleData`。

然后调用：

```text
HeapTupleSatisfiesVisibility(tuple, scan->rs_snapshot, buffer)
```

`HeapTupleSatisfiesVisibility()` 根据 snapshot type 分发。

普通 SELECT 使用 `SNAPSHOT_MVCC`。

于是进入：

```text
HeapTupleSatisfiesMVCC()
```

### 12.2 进入 `xmin` 分支

函数先读 tuple header。

如果 `xmin` 无效，直接返回 false。

如果 `xmin` 属于当前事务，走 command id 分支。

如果 `xmin` 在 snapshot running set，返回 false。

如果不在 running set，再查真实提交状态。

提交则设置 `HEAP_XMIN_COMMITTED`。

回滚则设置 `HEAP_XMIN_INVALID` 并返回 false。

### 12.3 `xmin` 通过后进入 `xmax` 分支

如果 `xmax` invalid，返回 true。

如果 `xmax` lock-only，返回 true。

如果 `xmax` 是 MultiXact，找真正 updater。

如果 deleting/updating transaction 在 snapshot running set，返回 true。

如果它回滚，返回 true，并可设置 invalid hint。

如果它提交，返回 false，并可设置 committed hint。

### 12.4 返回给调用者

如果可见，heap scan 把 tuple 交给 executor。

如果不可见，heap scan 跳过。

可见结果还可能触发 SSI predicate lock 相关检查。

但那不是本节主线。

本节只追 visibility boolean。

## 13. 生命周期 / ownership / cleanup

`HeapTupleSatisfiesMVCC()` 不拥有 snapshot。

它假设传入 snapshot 已经 active 或 registered。

源码中有 Assert：

```text
snapshot->regd_count > 0 || snapshot->active_count > 0
```

这说明 visibility routine 只消费 snapshot。

它不负责延长 snapshot 生命周期。

`HeapTupleSatisfiesMVCC()` 也不拥有 buffer。

调用者必须保证 buffer 至少以合适模式锁住。

函数可能设置 hint bit。

因此 buffer 可能被标记 dirty。

但 tuple 是否删除、是否 cleanup，不由这个函数决定。

它返回 false 只是说明：

```text
对当前 snapshot 不可见。
```

这不等于：

```text
可以被 VACUUM 移除。
```

cleanup 生命周期由 `HeapTupleSatisfiesVacuum()`、`GlobalVisState`、pruning 和 VACUUM 判断。

这个边界非常重要。

否则会把“当前查询看不到”误读成“全系统没人需要”。

## 14. 正确性机制层次

第一层是 tuple header。

它提供 `xmin/xmax/infomask/cid` 原始证据。

第二层是 snapshot。

它提供运行事务集合和 `curcid`。

第三层是事务状态。

它在 snapshot 不认为事务 running 时给出 commit/abort 结果。

第四层是 hint bit 写回规则。

它只缓存已知安全的事务结果。

第五层是 MultiXact 解析。

它区分 locker 与 updater。

第六层是调用者锁和生命周期。

visibility routine 需要在 buffer 和 snapshot 有效期间运行。

这几层共同保护一个不变量：

```text
普通 MVCC SELECT 返回的 tuple 集合，必须等价于 snapshot 创建时承诺的可见世界。
```

## 15. 错误路径 / 异常路径 / fallback

### 15.1 SubXID overflow

如果 snapshot 的 `suboverflowed` 为 true，`XidInMVCCSnapshot()` 不能只查 `subxip`。

它会通过 `pg_subtrans` 把 subxact xid 转成 top-level xid。

这让判断仍然正确。

代价是可能增加 I/O 或共享状态访问。

### 15.2 MultiXact

`xmax` 可能是 MultiXactId。

普通 SELECT 要区分：

```text
只有 locker:
  visible

真正 updater/deleter:
  继续按 update xid 判断
```

如果把所有 MultiXact 都当成 delete，会错误隐藏 tuple。

### 15.3 异步提交 hint bit

事务可能异步提交。

commit 已经完成，但 commit WAL 还没 flush 到满足 data page hint 安全的位置。

此时可以返回正确可见性。

但可能暂时不设置 hint bit。

### 15.4 moved tuple 兼容

源码仍保留 `HEAP_MOVED_OFF` / `HEAP_MOVED_IN` 支持。

这是旧版本 VACUUM FULL 兼容痕迹。

正常新逻辑不要把它当成主线。

### 15.5 当前命令边界

当前事务自己的 tuple 不能只靠 XID 判断。

如果 `cmin >= curcid`，普通 MVCC snapshot 不看这个 tuple。

这不是回滚。

只是当前命令不可见。

## 16. 成本、资源与跨模块传播

成本主要来自四个层面。

第一，per-tuple header 读取。

这是不可避免的 hot path 成本。

第二，snapshot array 查找。

`XidInMVCCSnapshot()` 会用区间剪枝减少数组扫描。

第三，事务状态查询。

只有 snapshot 不认为事务 running 时才查 CLOG。

第四，hint bit 写回。

写回可能 dirty page，也可能受 WAL flush 边界限制。

batch MVCC 能降低部分 per-tuple overhead。

但真正的大方向是：

```text
尽量让常见 tuple 在 header hint 和 snapshot range check 后快速返回。
```

跨模块传播包括：

```text
snapshot manager:
  提供 SnapshotData 和生命周期。

ProcArray:
  影响 snapshot running set。

CLOG/pg_xact:
  提供真实事务结果。

MultiXact:
  解释 xmax 中的多个 locker/updater。

buffer manager:
  承担 hint bit dirty page。

executor:
  消费 visible tuple。
```

## 17. 观测与诊断入口

SQL 现象：

```sql
SELECT xmin, xmax, ctid, * FROM mvcc_visible_demo;
```

可以看到 tuple header 的部分系统列。

`pageinspect` 能看到更多：

```sql
SELECT lp,
       t_xmin,
       t_xmax,
       t_field3,
       t_ctid,
       t_infomask,
       t_infomask2
FROM heap_page_items(get_raw_page('mvcc_visible_demo', 0))
ORDER BY lp;
```

源码断点：

```text
break HeapTupleSatisfiesVisibility
break HeapTupleSatisfiesMVCC
break XidInMVCCSnapshot
break TransactionIdDidCommit
break SetHintBitsExt
```

关键观察：

```text
p snapshot->xmin
p snapshot->xmax
p snapshot->curcid
p tuple->t_infomask
p tuple->t_choice.t_heap.t_xmin
p tuple->t_choice.t_heap.t_xmax
```

诊断时要避免一个陷阱。

不要把 gdb 里看到的“事务现在已提交”直接套到旧 snapshot。

应该先问：

```text
这个 xid 是否在当前 snapshot 看来 running？
```

## 18. 常见误区

误区一：

```text
HeapTupleSatisfiesMVCC() 就是查 xmin/xmax 是否提交。
```

不对。

它先按 snapshot 判断事务效果是否属于当前读者世界。

误区二：

```text
HEAP_XMIN_COMMITTED 设置后就永远对所有 snapshot 可见。
```

不对。

旧 snapshot 仍可能把该 XID 视为 running。

误区三：

```text
xmax running 说明 tuple 不可见。
```

不对。

删除事务 running 通常说明删除在当前 snapshot 看来还没生效，所以旧 tuple 仍可见。

误区四：

```text
普通 SELECT 会跟着 t_ctid 找新版本。
```

普通 heap scan 扫 page 上每个 tuple。

HOT chain 搜索是 index scan 入口的主题。

本节只讲单 tuple visibility。

误区五：

```text
返回 false 就可以 cleanup。
```

不对。

当前 snapshot 不可见不等于全局无人需要。

## 19. 课堂实验

### 实验一：旧 snapshot 不看新提交

Session A：

```sql
DROP TABLE IF EXISTS mvcc_visible_demo;
CREATE TABLE mvcc_visible_demo(id int primary key, payload text);

BEGIN;
INSERT INTO mvcc_visible_demo VALUES (1, 'from A');
```

Session B：

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT * FROM mvcc_visible_demo WHERE id = 1;
```

Session A：

```sql
COMMIT;
```

Session B：

```sql
SELECT * FROM mvcc_visible_demo WHERE id = 1;
COMMIT;
SELECT * FROM mvcc_visible_demo WHERE id = 1;
```

解释目标：

```text
同一个 committed xmin，在旧 snapshot 与新 snapshot 下结果不同。
```

### 实验二：xmax running 时旧版本仍可见

Session A：

```sql
UPDATE mvcc_visible_demo SET payload = 'new' WHERE id = 1;
BEGIN;
UPDATE mvcc_visible_demo SET payload = 'A updating' WHERE id = 1;
-- 不提交。
```

Session B：

```sql
SELECT * FROM mvcc_visible_demo WHERE id = 1;
```

观察目标：

```text
如果删除或更新事务在当前 snapshot 看来 running，旧版本仍可能可见。
```

### 实验三：断点跟踪 `xmin` 与 `xmax`

断点：

```text
break HeapTupleSatisfiesMVCC
break XidInMVCCSnapshot
```

执行：

```sql
SELECT * FROM mvcc_visible_demo WHERE id = 1;
```

观察：

```text
先进入 xmin 半边。
再进入 xmax 半边。
确认 running membership 如何改变返回值。
```

### 实验四：观察 hint bit 变化

```sql
CHECKPOINT;
SELECT count(*) FROM mvcc_visible_demo;
SELECT count(*) FROM mvcc_visible_demo;
```

配合 `pageinspect` 前后观察 `t_infomask`。

目标不是保证每次都看到变化。

目标是理解：

```text
第一次访问可能填充 hint bit；
第二次访问可能少查 CLOG；
可见性结果本身不依赖 hint bit 必然存在。
```

## 20. 讨论题

1. 为什么 `xmin committed` 以后仍要检查 `XidInMVCCSnapshot()`？

2. 为什么 `xmin running` 和 `xmax running` 对可见性的方向相反？

3. 如果当前事务没有 command id，`INSERT ... SELECT` 或 `UPDATE` 自己的行会出现什么歧义？

4. 为什么普通 MVCC 判断不能为了更新 hint bit 去读取“最新事务状态”？

5. batch MVCC 为什么只是性能优化，不改变单 tuple 语义？

6. `HeapTupleSatisfiesMVCC()` 返回 false 和 `HeapTupleSatisfiesVacuum()` 返回 dead 的区别是什么？

## 21. 本节小结

本节建立了普通 SELECT 的 heap visibility 主模型：

```text
先证明 tuple 已经在当前 snapshot 中出生；
再证明 tuple 没有被当前 snapshot 中已经生效的删除或更新杀死。
```

`xmin` 半边回答出生。

`xmax` 半边回答死亡。

`XidInMVCCSnapshot()` 保证判断绑定到 snapshot 创建时的运行事务集合。

hint bit 降低重复事务状态查询成本，但不改变 snapshot 语义。

command id 解决当前事务内部顺序。

下一节会专门讨论 hint bit：

```text
为什么 backend 可以把事务结果缓存进 tuple header，
以及这种写回为什么不能改变事务语义。
```
