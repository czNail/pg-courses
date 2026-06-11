# PostgreSQL UPDATE 版本链与 t_ctid 追踪

## 课程定位

前置知识：已经理解 heap tuple header 如何用 `xmin/xmax/t_ctid/infomask` 描述一个 tuple version，也理解普通 MVCC 判断和当前事务 command id 边界。

本节唯一主问题：

```text
UPDATE 为什么生成新 tuple version 而不是原地覆盖，t_ctid 如何把旧版本、新版本和并发更新冲突连接起来？
```

本节核心矛盾：

```text
UPDATE 希望像修改一行那样简单；
但 MVCC 必须让旧 snapshot 继续看到旧版本，让新 snapshot 看到新版本，同时让并发 updater、locker、index scan 和 VACUUM 都能沿版本关系做正确判断。
```

学完本节后，你应该能独立判断：

- 为什么 heap UPDATE 本质是 delete old version + insert new version。
- 为什么旧 tuple 的 `xmax` 和 `t_ctid` 都要更新。
- 为什么新 tuple 的 `xmin` 应等于更新事务。
- 为什么跟随 `t_ctid` 时必须验证后继 tuple 的 `xmin` 等于前驱 update xid。
- 为什么 `heap_get_latest_tid()` 不是简单读最后一个 `ctid`。
- 为什么并发 UPDATE 需要等待、重试或返回 `TM_Updated` / `TM_Deleted`。
- 为什么分区键 update、pruning 和 VACUUM 会让 t_ctid 链路出现边界。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

本节只讲普通 UPDATE chain。

HOT chain 的同页索引优化留到下一节。

## 1. 本节在总主线中的位置

前面几节已经能解释单个 tuple version 是否可见。

现在要把多个版本串起来。

一个 SQL UPDATE 逻辑上修改一行。

但 heapam 不能在原 tuple 上覆盖用户数据。

原因是旧 snapshot 可能仍然需要旧内容。

所以 UPDATE 产生新版本。

旧版本仍在 page 上。

新版本插入到某个 page 上。

旧版本通过 `t_ctid` 指向新版本。

旧版本的 `xmax` 记录更新事务。

新版本的 `xmin` 也记录更新事务。

这样后续系统可以证明：

```text
旧版本被哪个事务替换。
新版本是否真的是旧版本的后继。
并发 updater 应该等谁。
当前 snapshot 应该看到哪个版本。
```

本节主线是：

```text
heap_update()
  -> 锁定旧 tuple
  -> 判断更新资格
  -> 决定新 tuple 放置位置
  -> 设置旧 tuple 的 xmax/cmax/t_ctid
  -> 设置新 tuple 的 xmin/cmin/xmax/t_ctid
  -> 写 WAL
  -> 后续 heap_get_latest_tid() 或 tuple lock 沿 t_ctid 追踪
```

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
heap_update() 不覆盖旧 tuple，而是把旧 tuple 标记为被当前事务更新，并让旧 tuple 的 t_ctid 指向新 tuple；后续沿链追踪时必须用前驱 update xid 与后继 xmin 匹配，防止 VACUUM/pruning 后的 slot 重用把无关 tuple 误认为同一行后继。
```

这里的 tension 是：

```text
逻辑行需要连续身份；
物理 heap tuple version 又必须允许旧版本和新版本同时存在，并在 cleanup 后局部断裂。
```

PostgreSQL 的版本链不是永久对象图。

它只是 heap page 上由 tuple header 和 line pointer 共同形成的可验证链路。

它可以被 VACUUM 缩短。

它可以因为分区移动而终止。

它可以因为跨页 update 需要新索引入口。

因此跟随 `t_ctid` 的正确姿势不是：

```text
一直沿指针走到最后。
```

而是：

```text
每一步都检查 line pointer 状态和 xmin/xmax 关系；
如果验证失败，就停止。
```

## 3. 核心文件分工与阅读顺序

| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/access/htup_details.h` | 先读 `t_ctid` 注释、`HeapTupleHeaderGetUpdateXid()`、moved partitions 和 speculative token 边界。 |
| 2 | `src/backend/access/heap/heapam.c` | 主读 `heap_update()`、`heap_get_latest_tid()`、`heap_lock_tuple()` 中的 follow-updates 逻辑。 |
| 3 | `src/backend/access/heap/heapam_visibility.c` | 对照 `HeapTupleSatisfiesUpdate()` 如何决定 `TM_Result`。 |
| 4 | `src/include/access/tableam.h` | 看 `TM_Result` 和 `TM_FailureData` 对上层 executor 的语义。 |
| 5 | `src/include/access/heapam.h` | 对照 heap update / lock / latest tid 的公开声明。 |
| 6 | `src/backend/access/heap/pruneheap.c` | 看 pruning 为什么可能让链路中的 line pointer 被 redirect、dead 或 unused。 |
| 7 | `src/backend/access/heap/README.HOT` | 只读普通 update chain 与 HOT chain 的差别，下一节再深入 HOT。 |

推荐阅读顺序：

```text
先读 htup_details.h 对 t_ctid 的完整注释
  -> 读 heap_update() 如何设置旧/新版本
  -> 读 heap_get_latest_tid() 如何沿链验证
  -> 再读 heap_lock_tuple() 的 follow_updates 场景
  -> 最后读 pruning 边界
```

## 4. 一个 runtime 现象先定锚

用 `pageinspect` 观察一个普通 UPDATE。

```sql
CREATE EXTENSION IF NOT EXISTS pageinspect;

DROP TABLE IF EXISTS update_chain_demo;
CREATE TABLE update_chain_demo(id int primary key, payload text);

INSERT INTO update_chain_demo VALUES (1, 'v1');
UPDATE update_chain_demo SET payload = 'v2' WHERE id = 1;

SELECT xmin, xmax, ctid, * FROM update_chain_demo;

SELECT lp,
       lp_flags,
       t_xmin,
       t_xmax,
       t_ctid,
       t_infomask,
       t_infomask2
FROM heap_page_items(get_raw_page('update_chain_demo', 0))
ORDER BY lp;
```

你会看到 SQL 只返回当前可见版本。

但 heap page 上可能仍有旧版本。

旧版本的 `t_ctid` 指向后继。

旧版本的 `xmax` 通常是更新事务。

后继版本的 `xmin` 也通常是这个更新事务。

这就是版本链的最小证据：

```text
old.xmax == new.xmin
old.t_ctid == new.t_self
```

注意这不是永久保证。

VACUUM 或 pruning 之后，后继 slot 可能消失。

所以源码每次跟随时都要重新验证。

## 5. UPDATE 为什么不能原地覆盖

如果 UPDATE 原地覆盖 tuple data，会出现直接错误。

旧 snapshot 本来应该看到旧值。

覆盖以后旧值消失。

系统要么破坏 repeatable read。

要么必须把旧值复制到另一个 undo log，再让旧 snapshot 回放。

PostgreSQL heap 选择的是 append new version。

这让 heap page 本身保存版本历史。

旧版本继续服务旧 snapshot。

新版本服务新 snapshot。

VACUUM 在 horizon 允许时删除旧版本。

这个设计把 UPDATE 拆成：

```text
旧版本:
  标记被更新。

新版本:
  插入一条新 tuple。
```

和真正 DELETE 不同，UPDATE 旧版本还要指向新版本。

这让并发 updater 可以知道：

```text
这行已经被谁更新到哪里。
```

## 6. `heap_update()` 的关键状态

`heap_update()` 入口拿到：

```text
Relation relation
old tuple TID
new tuple
CommandId cid
Snapshot crosscheck
wait policy
TM_FailureData
```

它先读取旧 tuple 所在 page。

然后在 buffer 上加锁。

接着调用：

```text
HeapTupleSatisfiesUpdate(&oldtup, cid, buffer)
```

这个结果决定后续：

```text
TM_Ok:
  可以更新。

TM_BeingModified:
  等待其他事务并重试。

TM_Updated:
  旧 tuple 已被其他事务更新。

TM_Deleted:
  旧 tuple 已被删除。

TM_SelfModified:
  当前事务已经处理过它。

TM_Invisible:
  对当前修改语义不可见。
```

如果可以更新，`heap_update()` 会计算修改列。

这些列决定：

```text
是否需要更新所有索引。
是否可以 HOT update。
是否需要更强 tuple lock。
是否影响 replica identity。
```

本节只关注普通 chain。

HOT 决策留到下一节。

## 7. 旧版本如何被修改

UPDATE 成功时，旧版本不是直接删除。

它会记录：

```text
xmax = current transaction or MultiXact result
cmax = current command id
t_ctid = new tuple TID
HEAP_UPDATED
maybe HEAP_HOT_UPDATED
maybe HEAP_KEYS_UPDATED
```

`xmax` 告诉 visibility routine：

```text
这个旧版本被哪个事务更新或删除。
```

`t_ctid` 告诉追踪者：

```text
后继版本在哪里。
```

`HEAP_UPDATED` 说明这是一个被 UPDATE 替换的版本。

`HEAP_KEYS_UPDATED` 影响锁冲突和索引语义。

如果是 HOT update，旧版本还会有 `HEAP_HOT_UPDATED`。

这些状态一起表达：

```text
旧版本不是无缘无故死亡；
它被某个更新事务替换成了一个后继版本。
```

## 8. 新版本如何被写入

新版本是一个普通 heap tuple。

它的核心状态是：

```text
xmin = current transaction
cmin = current command id
xmax = invalid 或继承必要锁状态
t_ctid = self
```

如果这是 HOT child，还会设置：

```text
HEAP_ONLY_TUPLE
```

如果不是 HOT，索引会获得新 tuple 的索引项。

新版本的 `xmin` 与旧版本的 update xid 匹配。

这不是偶然。

它是后续链路校验的依据。

跟随者看到旧 tuple 的 `xmax`。

再跳到旧 tuple 的 `t_ctid` 指向的 tuple。

如果后继 tuple 的 `xmin` 不等于旧 tuple 的 update xid，就不能相信这条链。

## 9. `heap_get_latest_tid()` walkthrough

`heap_get_latest_tid()` 的注释很清楚。

它拿到一个 TID。

然后尝试沿 `t_ctid` 链找到对 scan snapshot 可见的最新版本。

核心循环是：

```text
ctid = input tid
priorXmax = InvalidTransactionId

for each ctid:
  read buffer
  lock buffer share
  check offset number
  check line pointer normal
  form HeapTupleData
  if priorXmax valid:
    require priorXmax == current tuple xmin
  check tuple visibility
  if visible:
    update result tid
  if t_ctid cannot continue:
    break
  priorXmax = HeapTupleHeaderGetUpdateXid(current)
  ctid = current->t_ctid
```

这里有三个关键点。

第一，函数不假设链永远完整。

offset 越界、line pointer 非 normal 都会停止。

第二，函数不把 `t_ctid` 当作绝对可信指针。

它用 `priorXmax == current xmin` 校验后继。

第三，它按 snapshot 判断可见候选。

所以“latest” 是：

```text
对该 scan snapshot 可见的最新版本。
```

如果用 `SnapshotDirty`，可以找更接近物理最新、甚至未提交的版本。

## 10. 并发 UPDATE 如何使用 chain

两个事务同时更新同一行时，第二个事务可能遇到：

```text
旧 tuple 的 xmax 指向第一个事务。
旧 tuple 的 t_ctid 指向第一个事务创建的新版本。
```

此时它不能简单覆盖。

它要根据等待策略和事务结果决定：

```text
等待第一个事务。
如果第一个事务回滚，重试旧版本。
如果第一个事务提交，沿 t_ctid 找后继或返回 TM_Updated。
如果后继不可用，按删除或更新冲突处理。
```

`heap_lock_tuple()` 中的 `follow_updates` 参数也和这条链有关。

当调用者需要锁住更新后的后继版本时，它要沿 chain 继续。

但同样必须验证：

```text
后继 slot 仍存在。
后继 xmin 匹配前驱 update xid。
```

这保证并发控制不会被 VACUUM 后的 slot 重用欺骗。

## 11. `t_ctid` 链路的边界

`t_ctid` 有几种停止条件。

第一，`xmax` invalid。

说明没有有效后继。

第二，tuple only locked。

行被锁住不等于被更新。

第三，`t_ctid` 指向自身。

说明当前是链尾。

第四，partition movement。

分区键 update 可能把 tuple 移到别的 partition。

此时 `t_ctid` 是特殊标记。

第五，line pointer 不可用。

后继可能已经被 pruning。

第六，后继 `xmin` 不匹配。

说明 slot 可能被重用。

这些边界都不是可选防御。

它们是 MVCC 与物理 cleanup 共存的必要条件。

## 12. 主流程源码 walkthrough

### 12.1 找到旧 tuple

```text
heap_update()
  -> ReadBuffer(old block)
  -> LockBuffer(exclusive)
  -> PageGetItemId(old offset)
  -> form oldtup
```

这里持有 buffer pin 和 exclusive lock。

旧 tuple header 可以被安全检查和修改。

### 12.2 判断更新资格

```text
result = HeapTupleSatisfiesUpdate(&oldtup, cid, buffer)
```

`TM_Ok` 才能继续。

其他结果进入 wait、retry 或 failure path。

### 12.3 构造新 tuple

```text
set newtup tableOid
determine modified_attrs
choose lock mode
choose HOT or non-HOT
find target page
```

如果同页且索引列未改变，可能 HOT。

否则普通 update 需要新索引项。

### 12.4 修改旧 tuple 和插入新 tuple

在 critical section 中：

```text
旧 tuple:
  set xmax/cmax
  set t_ctid to new tuple TID
  set flags

新 tuple:
  set xmin/cmin
  set xmax invalid
  set t_ctid self
  insert into page
```

同时清理 visibility map 的 all-visible 状态。

因为 page 上已经有未全局可见的新版本。

### 12.5 WAL 与返回

`log_heap_update()` 记录 update WAL。

返回时，调用者知道：

```text
新 tuple TID。
是否需要更新索引。
失败时的 TM_FailureData。
```

`TM_FailureData` 可能包含：

```text
ctid
xmax
cmax
```

上层 executor 用这些信息处理并发更新。

## 13. 生命周期 / ownership / cleanup

UPDATE chain 的生命周期跨多个阶段。

创建阶段：

```text
heap_update() 创建旧 -> 新的链接。
```

可见性阶段：

```text
不同 snapshot 可能看到旧版本或新版本。
```

冲突阶段：

```text
并发 updater/locker 可能沿链等待或重试。
```

索引阶段：

```text
非 HOT update 为新版本建立索引项。
HOT update 复用 root 索引项。
```

cleanup 阶段：

```text
VACUUM/pruning 在 horizon 安全后移除旧版本或修剪链。
```

任何阶段都不能假设链永久完整。

`t_ctid` 是可验证线索。

不是不可破坏引用。

## 14. 正确性机制层次

第一层是旧版本保存。

原地覆盖会破坏旧 snapshot。

第二层是 `xmax`。

它记录替换旧版本的事务。

第三层是 `t_ctid`。

它提供后继位置。

第四层是后继校验。

`priorXmax == next xmin` 证明后继属于同一 update。

第五层是 buffer lock。

修改 header 和 page 必须受保护。

第六层是 WAL。

update chain 的创建要能 crash recovery。

第七层是 cleanup horizon。

旧版本只有在全局安全后才可移除。

## 15. 错误路径 / 异常路径 / fallback

### 15.1 并发事务正在修改

`HeapTupleSatisfiesUpdate()` 可能返回 `TM_BeingModified`。

`heap_update()` 根据 wait policy 等待或返回。

等待后必须重新检查 tuple。

因为事务状态和 header 都可能改变。

### 15.2 后继被 VACUUM 清理

跟随 `t_ctid` 时，如果后继 slot 不是 normal item，停止。

这不一定是错误。

可能只是 cleanup 已经安全推进。

### 15.3 后继 slot 被重用

如果后继 tuple 的 `xmin` 不等于前驱 update xid，停止。

否则可能误把无关 tuple 当作同一行的新版本。

### 15.4 分区移动

分区键 update 可能让新版本在另一个 partition。

原 tuple 的 `t_ctid` 会使用特殊标记。

普通同表链路追踪不能继续。

### 15.5 speculative insertion token

`t_ctid` 有时被 speculative insertion 临时占用。

这不是 UPDATE chain。

跟随 update chain 时不应把 token 当 TID。

## 16. 成本、资源与跨模块传播

UPDATE 的成本远高于原地修改。

它可能带来：

```text
新 heap tuple。
旧 tuple header 修改。
WAL 记录。
visibility map clear。
索引项新增。
旧版本未来 VACUUM 成本。
chain 追踪成本。
```

但它换来：

```text
旧 snapshot 可继续读旧版本。
新 snapshot 可读新版本。
并发 updater 能定位冲突。
VACUUM 能在安全后回收。
HOT 可以在合适时减少索引成本。
```

跨模块传播：

```text
heapam.c:
  创建和追踪 update chain。

heapam_visibility.c:
  判断旧/新版本对 snapshot 的可见性。

heapam_indexscan.c:
  HOT 情况沿同页链找可见版本。

pruneheap.c:
  修剪已死 chain member。

vacuumlazy.c:
  批量清理旧版本和索引项。

executor:
  根据 TM_Result 处理并发 UPDATE。
```

## 17. 观测与诊断入口

SQL 观察：

```sql
SELECT xmin, xmax, ctid, * FROM update_chain_demo;
```

pageinspect：

```sql
SELECT lp,
       lp_flags,
       t_xmin,
       t_xmax,
       t_ctid,
       t_infomask,
       t_infomask2
FROM heap_page_items(get_raw_page('update_chain_demo', 0))
ORDER BY lp;
```

源码断点：

```text
break heap_update
break HeapTupleSatisfiesUpdate
break heap_get_latest_tid
break heap_lock_tuple
break log_heap_update
```

关键变量：

```text
oldtup.t_self
oldtup.t_data->t_ctid
oldtup.t_data->t_choice.t_heap.t_xmax
newtup->t_self
modified_attrs
use_hot_update
result
```

诊断并发 UPDATE 时，重点看：

```text
返回的是 TM_BeingModified、TM_Updated 还是 TM_Deleted。
tmfd->ctid 是否指向后继。
等待后是否重新检查。
```

## 18. 常见误区

误区一：

```text
UPDATE 是原地改 tuple data。
```

不对。

heap UPDATE 生成新版本。

误区二：

```text
t_ctid 永远指向最新版本。
```

不对。

它可能指向后继，也可能自指，也可能因为 cleanup 无法继续。

误区三：

```text
只要 t_ctid 指向某个 slot，就能相信它。
```

不对。

必须验证后继 `xmin`。

误区四：

```text
TM_Updated 和 TM_Deleted 只是错误码。
```

不对。

它们携带并发控制语义，上层会据此等待、重试或报错。

误区五：

```text
旧版本不可见后马上可以删除。
```

不对。

还要看 cleanup horizon。

## 19. 课堂实验

### 实验一：观察 update chain

```sql
CREATE EXTENSION IF NOT EXISTS pageinspect;

DROP TABLE IF EXISTS update_chain_demo;
CREATE TABLE update_chain_demo(id int primary key, payload text);

INSERT INTO update_chain_demo VALUES (1, 'v1');
UPDATE update_chain_demo SET payload = 'v2' WHERE id = 1;

SELECT xmin, xmax, ctid, * FROM update_chain_demo;

SELECT lp, lp_flags, t_xmin, t_xmax, t_ctid, t_infomask, t_infomask2
FROM heap_page_items(get_raw_page('update_chain_demo', 0))
ORDER BY lp;
```

解释目标：

```text
SQL 只返回当前可见版本。
heap page 上保留旧版本。
旧版本通过 t_ctid 指向新版本。
```

### 实验二：连续 UPDATE

```sql
UPDATE update_chain_demo SET payload = 'v3' WHERE id = 1;
UPDATE update_chain_demo SET payload = 'v4' WHERE id = 1;

SELECT lp, t_xmin, t_xmax, t_ctid, t_infomask, t_infomask2
FROM heap_page_items(get_raw_page('update_chain_demo', 0))
ORDER BY lp;
```

观察目标：

```text
链上可能出现多个版本。
每一步都需要 old.xmax 与 next.xmin 匹配。
```

### 实验三：并发 UPDATE

Session A：

```sql
BEGIN;
UPDATE update_chain_demo SET payload = 'from A' WHERE id = 1;
```

Session B：

```sql
UPDATE update_chain_demo SET payload = 'from B' WHERE id = 1;
```

源码断点：

```text
break HeapTupleSatisfiesUpdate
break heap_update
```

观察：

```text
第二个 updater 看到什么 TM_Result。
等待后是否沿 t_ctid 追踪。
```

### 实验四：`heap_get_latest_tid()`

在需要追踪 latest tid 的路径设置断点：

```text
break heap_get_latest_tid
```

观察：

```text
priorXmax
ctid
当前 tuple xmin
可见性判断结果
```

目标是确认它不是盲目沿链。

## 20. 讨论题

1. PostgreSQL 为什么选择 heap version chain，而不是原地覆盖加 undo log？

2. 如果不验证 `priorXmax == next xmin`，VACUUM 后 slot 重用会造成什么错误？

3. 为什么 UPDATE 旧版本要同时写 `xmax` 和 `t_ctid`？

4. 为什么 `heap_get_latest_tid()` 说 latest 是“按 snapshot 可见”的 latest？

5. 并发 UPDATE 等待后为什么必须重新检查 tuple header？

6. 分区移动为什么不能当作普通同表 `t_ctid` 链路继续追？

## 21. 本节小结

本节建立 UPDATE chain 的核心模型：

```text
UPDATE 不覆盖旧版本；
它让旧版本死亡，并创建一个新版本；
旧版本用 xmax 记录更新事务，用 t_ctid 指向后继；
后继用 xmin 证明自己确实由该更新事务创建。
```

`t_ctid` 是版本链线索。

不是永久可靠指针。

每次跟随都要校验 line pointer 和 `xmin/xmax` 关系。

并发 UPDATE、tuple lock、VACUUM 和 pruning 都围绕这条链协作。

下一节会进入 HOT：

```text
当 UPDATE 不改变索引列且新版本能放在同一页时，PostgreSQL 如何把版本链藏在 heap page 内部，让一个索引入口找到当前可见版本。
```
