# PostgreSQL heap tuple header 中的 MVCC 信息

## 课程定位

前置知识：已经理解 `SnapshotData` 如何描述一个读者视角，也已经理解 CLOG/pg_xact 记录事务提交结果，`xmin` horizon 决定旧版本 cleanup 边界。

本节唯一主问题：

```text
heap tuple header 为什么不能只存一个“当前行是否有效”的布尔值，而必须用 xmin、xmax、t_ctid、infomask、hint bit 和 command id 共同描述一个 tuple version？
```

本节核心矛盾：

```text
执行器希望每个 tuple 的可见性判断足够快；
但一个 tuple version 的语义同时取决于插入事务、删除或锁定事务、同事务命令顺序、版本链关系、事务状态缓存、MultiXact 和后续 VACUUM cleanup。
```

学完本节后，你应该能独立判断：

- 为什么 `xmin` 只回答“谁创建了这个版本”，不回答“是否可见”。
- 为什么 `xmax` 可能表示删除者、更新者、locker 或 MultiXact。
- 为什么 `t_ctid` 既可以指向自身，也可以指向更新后的版本。
- 为什么 `t_infomask` / `t_infomask2` 是 tuple header 语义的一部分，而不是可有可无的优化位。
- 为什么 command id 只对创建或修改该 tuple 的事务本身有完整语义。
- 为什么 hint bit 可以缓存事务结果，但不能改变事务结果。
- 为什么跟随 `t_ctid` 时必须校验后继 tuple 的 `xmin`。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

本节不展开完整 `HeapTupleSatisfiesMVCC()` 分支，也不展开 HOT pruning。

这里先把 heap tuple header 自身如何承载 MVCC 状态讲清楚。

## 1. 本节在总主线中的位置

前面几节已经把 MVCC 的读者视角讲完：snapshot 能回答某个 XID 在当前读者看来是不是 still-running。

但是 snapshot 不是 tuple。

现在要把视角落到 heap page 上的一条版本记录。

一条 heap tuple version 至少要回答这些问题：

```text
谁插入了我？
谁删除、更新或锁住了我？
这个删除是行删除，还是只是行锁？
我是版本链上的哪个节点？
我是否已经知道插入事务提交或回滚？
我是否已经知道删除事务提交或回滚？
我是否是 HOT chain 的 root 或 heap-only child？
如果插入和删除都来自当前事务，我该用哪个 command id 解释？
```

这些问题不能压缩成一个布尔值。

同一个 tuple version 对旧 snapshot、新 snapshot、创建它的事务、VACUUM 和索引扫描可能分别有不同答案。

因此 PostgreSQL 把 heap tuple header 设计成一个紧凑但语义密集的状态包。

本节的主线是：

```text
SQL INSERT / UPDATE / DELETE
  -> heapam.c 写入 tuple header 字段和 flags
  -> heapam_visibility.c 读取这些字段并结合 snapshot / CLOG
  -> VACUUM 和 HOT 继续解释 t_ctid、hint bit 和 cleanup flags
```

下一节会进入普通 `SELECT` 的 MVCC 判定框架；本节只回答 tuple header 给 visibility routine 提供哪些原始材料，以及这些材料为什么必须组合解释。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
heap tuple header 把一个行版本的出生事务、可能的死亡或锁定事务、版本链后继、当前事务命令边界和可缓存事务状态放在同一个页内 header 中；visibility routine 再把这些字段与 snapshot、CLOG、MultiXact、command id 和 cleanup horizon 组合成最终语义。
```

这里的 tension 是：

```text
tuple visibility 是 heap scan、index scan、join input 的 hot path；
但 tuple header 必须保留足够信息，让并发事务、同事务多命令、行锁、UPDATE 版本链、HOT 和 VACUUM 都能在以后正确解释。
```

PostgreSQL 的折中是：

| 状态 | 放在哪里 | 解决什么问题 |
| --- | --- | --- |
| `t_xmin` | `HeapTupleFields` | 记录创建该版本的事务。 |
| `t_xmax` | `HeapTupleFields` | 记录删除者、更新者、locker 或 MultiXact。 |
| `t_field3.t_cid` | `HeapTupleFields` | 记录当前事务内部的 command id。 |
| `t_ctid` | `HeapTupleHeaderData` | 指向自身、后继版本或特殊状态。 |
| `t_infomask` | `HeapTupleHeaderData` | 记录事务 hint、锁语义、tuple 物理属性。 |
| `t_infomask2` | `HeapTupleHeaderData` | 记录属性数、HOT / heap-only / key-updated 状态。 |
| null bitmap / `t_hoff` | tuple header 后部 | 让执行器能定位用户列数据。 |

本节要建立的系统规律是：

```text
raw field 不是 MVCC 语义；
field + flag + snapshot + transaction status + command boundary + cleanup lifecycle 才是 tuple version 的语义。
```

## 3. 核心文件分工与阅读顺序

| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/access/htup_details.h` | 先读 `HeapTupleHeaderData`、`HeapTupleFields`、`t_ctid` 注释、`HEAP_XMIN_*` / `HEAP_XMAX_*` / HOT flags。 |
| 2 | `src/include/access/htup.h` | 对照 `HeapTupleData` 和 `HeapTupleHeader` 的外层结构。 |
| 3 | `src/backend/access/heap/heapam.c` | 看 `heap_insert()`、`heap_update()`、`heap_delete()` 如何写 header。 |
| 4 | `src/backend/access/heap/heapam_visibility.c` | 看 `HeapTupleSatisfiesMVCC()`、`HeapTupleSatisfiesUpdate()` 如何消费 header。 |
| 5 | `src/backend/utils/time/combocid.c` | 看 `HeapTupleHeaderGetCmin()`、`HeapTupleHeaderGetCmax()`、`HeapTupleHeaderAdjustCmax()` 的 command id 边界。 |
| 6 | `src/include/access/heapam.h` | 看 visibility / update / vacuum routine 的公开声明和结果类型。 |
| 7 | `src/backend/access/heap/README.HOT` | 最后读 HOT chain 对 `HEAP_HOT_UPDATED`、`HEAP_ONLY_TUPLE`、`t_ctid` 的约束。 |

推荐阅读顺序：

```text
先读 htup_details.h 的字段和 flags
  -> 再读 heapam.c 中插入、更新、删除如何设置这些字段
  -> 再读 heapam_visibility.c 中可见性如何组合判断
  -> 最后读 combocid.c 和 README.HOT 处理同事务与 HOT 版本链边界
```

不要从 `HeapTupleSatisfiesMVCC()` 的所有 if 分支开始。

那样很容易把主问题看成“函数分支很多”。

本节要先把 tuple header 的状态语言读懂。

## 4. 一个 runtime 现象先定锚

先看一个可以复现的现象。

同一行被更新以后，表里不是原地改写一个 tuple。

heap page 上会出现旧版本和新版本。

旧版本的 `xmax` 指向更新事务。

旧版本的 `t_ctid` 指向新版本。

新版本的 `xmin` 指向同一个更新事务。

如果安装了 `pageinspect`，可以用下面的方式观察。

```sql
CREATE EXTENSION IF NOT EXISTS pageinspect;

DROP TABLE IF EXISTS mvcc_header_demo;
CREATE TABLE mvcc_header_demo(id int primary key, payload text);

INSERT INTO mvcc_header_demo VALUES (1, 'old');
UPDATE mvcc_header_demo SET payload = 'new' WHERE id = 1;

SELECT lp,
       t_xmin,
       t_xmax,
       t_field3,
       t_ctid,
       t_infomask,
       t_infomask2
FROM heap_page_items(get_raw_page('mvcc_header_demo', 0))
ORDER BY lp;
```

你通常会看到至少两个 line pointer：一个旧版本，一个新版本。

旧版本可能已经对当前 snapshot 不可见，但仍保存在 heap page 上，因为其他 snapshot、索引入口和 cleanup horizon 仍可能需要它。

这个现象说明：tuple header 描述的是一个版本，不是逻辑行；它必须描述版本之间的关系，而不仅是“活着或死了”。

把这个现象压回源码，就是：

```text
heap_update()
  -> 在旧 tuple 上设置 xmax、cmax、t_ctid、HEAP_UPDATED / HOT flags
  -> 插入新 tuple，设置 xmin、cmin、xmax invalid、t_ctid self
  -> 后续 visibility routine 根据 snapshot 判断哪个版本可见
```

## 5. `HeapTupleHeaderData` 的边界

`HeapTupleHeaderData` 定义在 `src/include/access/htup_details.h`。

它不是完整 heap tuple。

它是 heap tuple 内部的 header。

外层 `HeapTupleData` 还保存：

```text
t_len
t_self
t_tableOid
t_data
```

`t_data` 指向 `HeapTupleHeaderData`。

真正落在 heap page item 里的核心字段包括：

| 字段 | 语义 |
| --- | --- |
| `t_choice.t_heap.t_xmin` | 插入该版本的事务 ID。 |
| `t_choice.t_heap.t_xmax` | 删除、更新、锁定该版本的事务 ID 或 MultiXactId。 |
| `t_choice.t_heap.t_field3.t_cid` | 当前事务内部 command id 或 combo cid。 |
| `t_ctid` | 当前版本自身 TID，或后继版本 TID，或特殊 token。 |
| `t_infomask2` | 属性数和 HOT / key update / heap-only 状态。 |
| `t_infomask` | null、varwidth、external、事务状态 hint、锁语义等 flag。 |
| `t_hoff` | 用户数据起始偏移。 |
| `t_bits` | null bitmap 起点。 |

这些字段全在一个 tuple header 里。

但它们的 owner 不一样。

`xmin` 和 `xmax` 是事务系统语义。

`t_ctid` 是版本链和 line pointer 语义。

`t_infomask` 同时混合物理布局、事务 hint 和锁语义。

`t_infomask2` 同时混合属性个数和 HOT 状态。

`t_field3` 在当前事务里可能解释成 `cmin`。

同一个字段在删除时又可能解释成 `cmax`。

如果同时需要 `cmin` 和 `cmax`，还可能通过 combo cid 间接表示。

所以读取 tuple header 时不能只看一个字段。

必须先问：

```text
我在什么 snapshot type 下读？
这个 xid 是否属于当前事务？
这个 xmax 是 update/delete，还是 lock-only？
这个 xmax 是普通 XID，还是 MultiXactId？
这个 t_ctid 是否可信地指向后继版本？
这个 tuple 是否属于 HOT chain？
这个 hint bit 是可靠缓存，还是需要回到 CLOG 查询？
```

## 6. `xmin`: 版本的出生事务

`t_xmin` 记录创建该 tuple version 的事务。

在普通插入路径里，新 tuple 的 `xmin` 来自当前事务 ID。

它回答的问题是：

```text
这个版本由哪个事务产生？
```

它不直接回答：

```text
这个版本对当前 snapshot 是否可见？
```

因为 `xmin` 的最终解释至少要经过这些步骤：

```text
如果 xmin 是当前事务:
  比较 cmin 和 snapshot->curcid。

如果 xmin 在 snapshot 的 running 集合中:
  对这个 snapshot 不可见。

如果 xmin 已提交:
  插入侧通过。

如果 xmin 已回滚:
  这个版本无效。
```

这就是为什么 `htup_details.h` 同时提供：

```text
HeapTupleHeaderGetRawXmin()
HeapTupleHeaderGetXmin()
HeapTupleHeaderXminCommitted()
HeapTupleHeaderXminInvalid()
HeapTupleHeaderXminFrozen()
```

`RawXmin` 是原始插入 XID。

`GetXmin` 会把 frozen tuple 解释成 `FrozenTransactionId`。

`XminCommitted` 和 `XminInvalid` 读取 hint bit。

这些 accessor 本身仍然不是完整可见性判断。

它们只是把 header 字段读成可被 visibility routine 消费的局部状态。

一个关键边界是：

```text
HEAP_XMIN_COMMITTED 不是事务提交的来源；
它只是事务提交结果已经被缓存到 tuple header 的证据。
```

真正的事务结果来自 CLOG/pg_xact。

hint bit 只是减少重复查询。

## 7. `xmax`: 版本的死亡、更新或锁定入口

`t_xmax` 更容易误读。

它看起来像“删除事务 ID”。

但在 PostgreSQL heap 中，它可能有多种含义：

```text
删除该 tuple 的事务。
更新该 tuple 的事务。
锁住该 tuple 的事务。
多个 locker 或 updater 组成的 MultiXactId。
无效值，表示没有有效 xmax。
```

真正语义来自 `xmax + infomask`。

常见组合包括：

| 组合 | 大致语义 |
| --- | --- |
| `HEAP_XMAX_INVALID` | `xmax` 无效或已回滚，tuple 没被有效删除。 |
| `HEAP_XMAX_COMMITTED` | `xmax` 已提交，可能删除或更新生效。 |
| `HEAP_XMAX_LOCK_ONLY` | `xmax` 只是 locker，不是删除者。 |
| `HEAP_XMAX_IS_MULTI` | `xmax` 是 MultiXactId，需要解析成员。 |
| `HEAP_XMAX_EXCL_LOCK` | exclusive tuple lock。 |
| `HEAP_XMAX_KEYSHR_LOCK` | key-share tuple lock。 |
| `HEAP_KEYS_UPDATED` | key columns 被更新，影响并发锁和索引语义。 |

所以 `xmax` 不能单独读。

普通 MVCC 判断遇到 `xmax` 时会先问：

```text
xmax 是否 invalid？
xmax 是否 lock-only？
xmax 是否 MultiXact？
xmax 是否当前事务？
xmax 在 snapshot 看来是否 still-running？
xmax 是否已提交？
```

这也是为什么 `HeapTupleHeaderGetUpdateXid()` 不能简单返回 raw xmax。

当 `HEAP_XMAX_IS_MULTI` 且不是 lock-only 时，它可能需要解析 MultiXact，找出真正的 updater XID。

这一步可能涉及 MultiXact 状态访问。

所以 header accessor 的注释明确提醒：

```text
只在绝对需要时调用。
```

这个成本边界会在 UPDATE 和 tuple lock 课程中继续出现。

## 8. `t_ctid`: 一个版本如何指向后继版本

`t_ctid` 是本节最容易被低估的字段。

它不是“当前 tuple 的永久 ID”。

它是版本链的一条边。

`htup_details.h` 的注释给出核心规则：

```text
新 tuple 存盘时，t_ctid 初始化为自身 TID。
如果 tuple 被 update，旧 tuple 的 t_ctid 改成指向 replacement version。
如果 partition key update 把 tuple 移到别的 partition，t_ctid 会存特殊值。
```

因此一个 tuple 是该 row 的最新版本，通常需要满足：

```text
xmax 无效
  或
t_ctid 指向自身
```

但这里还有重要边界。

跟随 `t_ctid` 不等于无条件信任后继 slot。

VACUUM 可能先清理后继 tuple。

line pointer 可能被标记 unused、dead、redirected。

slot 甚至可能后来被另一个无关 tuple 重用。

所以跟随 `t_ctid` 时必须验证：

```text
后继 slot 是 normal item。
后继 tuple 的 xmin 等于前驱 tuple 的 update xid。
否则认为链路已经结束。
```

这个规则在 `heap_get_latest_tid()` 和 `heap_hot_search_buffer()` 中都能看到。

它是版本链正确性的核心。

如果只根据 `t_ctid` 指针跳转，不做 `priorXmax == next xmin` 校验，就可能把一个无关 tuple 误认为同一逻辑行的后继版本。

这会破坏 UPDATE 冲突检测、index scan 可见性和 row locking。

`t_ctid` 还可能短暂表示 speculative insertion token。

这个状态用于 speculative insertion，不用于 update chain。

因此跟随 update chain 时不应该看到 speculative token。

这也是源码注释中特别强调的边界。

## 9. `t_infomask`: 不是附属位，而是语义位

`t_infomask` 混合了三类信息。

第一类是物理布局信息：

```text
HEAP_HASNULL
HEAP_HASVARWIDTH
HEAP_HASEXTERNAL
```

这些决定 executor 如何解码 tuple data。

第二类是事务结果 hint：

```text
HEAP_XMIN_COMMITTED
HEAP_XMIN_INVALID
HEAP_XMIN_FROZEN
HEAP_XMAX_COMMITTED
HEAP_XMAX_INVALID
```

这些让 visibility hot path 避免重复查 CLOG。

第三类是锁和 MultiXact 语义：

```text
HEAP_XMAX_KEYSHR_LOCK
HEAP_XMAX_EXCL_LOCK
HEAP_XMAX_LOCK_ONLY
HEAP_XMAX_IS_MULTI
```

这些决定 `xmax` 到底是删除、更新还是锁。

一个重要规则是：

```text
infomask 的事务 hint 可以滞后；
但不能提前表达错误事实。
```

例如某个事务已提交，但 tuple header 还没设置 `HEAP_XMIN_COMMITTED`。

这只是成本问题。

下一次合适的 visibility 检查可以查 CLOG，并把 hint bit 写回。

但如果把未提交事务提前标记 committed，就会破坏 MVCC。

因此 hint bit 写回必须遵守 WAL flush interlock。

第 15 节会专门展开。

本节只记住：

```text
infomask 是 tuple header 语义的一部分；
不是调试时可以忽略的附属字段。
```

## 10. `t_infomask2`: 属性数、key update 与 HOT 状态

`t_infomask2` 的低位保存属性数。

高位保存 MVCC / HOT 相关 flag。

本节关注三个：

```text
HEAP_KEYS_UPDATED
HEAP_HOT_UPDATED
HEAP_ONLY_TUPLE
```

`HEAP_KEYS_UPDATED` 表示 key columns 被更新，或者 tuple 被删除。

它会影响 tuple lock 冲突模式。

它也会影响 `heap_update()` 选择 lock mode。

`HEAP_HOT_UPDATED` 表示该 tuple 被 HOT update。

旧版本的 `t_ctid` 指向同页后继。

索引扫描遇到这个 flag 时要沿 HOT chain 找可见版本。

`HEAP_ONLY_TUPLE` 表示这个 tuple 没有自己的普通索引项。

它必须通过 HOT root 的索引入口找到。

这些 flag 让 tuple header 同时参与两个层面的语义：

```text
MVCC visibility:
  哪个版本对当前 snapshot 可见。

index reachability:
  index entry 能不能找到链上的当前可见版本。
```

这也是 HOT 不能跨 page 的原因之一。

如果 HOT child 在别的 page，索引扫描就要把一次 index hit 变成跨页链路追踪。

那会破坏 HOT 作为 page-local 优化的边界。

## 11. command id 与 combo cid

`xmin` 和 `xmax` 是事务级别。

但一个事务内部可以执行多条命令。

普通 MVCC snapshot 的 `curcid` 要回答：

```text
当前命令应该看到本事务之前命令做的修改吗？
当前命令应该看到本命令刚刚做的修改吗？
```

tuple header 里的 `t_field3.t_cid` 就服务这个问题。

插入时，它可以表示 `cmin`。

删除或更新时，它可以表示 `cmax`。

如果同一事务插入并删除同一个 tuple，就可能同时需要 `cmin` 和 `cmax`。

一个字段放不下两个 command id。

PostgreSQL 用 combo cid 解决：

```text
t_field3.t_cid 存 combo cid。
HEAP_COMBOCID flag 表示需要通过 backend-local combo cid table 展开。
```

`combocid.c` 中的关键边界是：

```text
combo cid 只对创建这个 combo cid 的事务有意义。
```

其他事务不能把 raw command id 当作完整语义。

所以 `HeapTupleHeaderGetCmin()` 和 `HeapTupleHeaderGetCmax()` 都带有 Assert。

它们只应该在能够确认该 tuple 来自当前事务的上下文里调用。

这也解释了一个常见误区：

```text
pageinspect 看到的 t_field3 不是全局可解释的“时间戳”。
```

它只是当前 tuple header 上的 command id 存储槽。

离开创建事务以后，command id 主要服务当前事务自可见性和冲突返回信息。

## 12. 主流程源码 walkthrough

### 12.1 INSERT 写入一个自指版本

普通插入的核心状态变化是：

```text
新 tuple:
  xmin = current xid
  cmin = current command id
  xmax = invalid
  t_ctid = self
  infomask 表示 xmax invalid 等初始状态
```

这意味着：

```text
版本出生了。
还没有有效删除者。
它是当前版本链末端。
```

对其他事务来说，它是否可见还要看：

```text
插入事务是否提交。
插入事务是否在 snapshot running set 中。
插入事务是否回滚。
```

对当前事务来说，它还要看：

```text
cmin < snapshot->curcid ?
```

这就是为什么 insert 后当前事务通常能在后续语句看到自己插入的行，但不一定在同一命令内部所有上下文都看到。

### 12.2 UPDATE 把一个版本分裂成旧版本和新版本

`heap_update()` 的核心不是“覆盖一行”。

它是生成一个新 tuple version。

旧版本会被改成：

```text
xmax = updating transaction
cmax = current command id
t_ctid = new tuple TID
HEAP_UPDATED / maybe HEAP_HOT_UPDATED
```

新版本会被写成：

```text
xmin = updating transaction
cmin = current command id
xmax = invalid 或继承必要 locker 信息
t_ctid = self
maybe HEAP_ONLY_TUPLE
```

这个设计让旧 snapshot 仍能看到旧版本。

新 snapshot 能看到新版本。

并发 updater 能通过旧 tuple 的 `xmax` 和 `t_ctid` 找到冲突或后继。

VACUUM 能在 cleanup horizon 允许后再移除旧版本。

### 12.3 DELETE、LOCK 与 cleanup 继续复用这些字段

DELETE 不生成新版本，而是在目标 tuple 上写入 deleting transaction、`cmax` 和死亡语义。

行锁也可能写 `xmax`，但要靠 `HEAP_XMAX_LOCK_ONLY`、`HEAP_LOCK_MASK` 和 `HEAP_XMAX_IS_MULTI` 区分它只是 locker，还是 updater/deleter。

VACUUM 和 pruning 以后还会继续解释同一组 header 字段。

它们不能只看 `xmax committed`，还要看 cleanup horizon。

跟随 `t_ctid` 时也必须准备好遇到 `LP_UNUSED`、`LP_DEAD`、`LP_REDIRECT` 或 normal item 但 `xmin` 不匹配的边界。

## 13. 生命周期 / ownership / cleanup

tuple header 的 owner 是 heap page。

它不是 backend-local 内存。

它不会跟随某个 executor node 的生命周期释放。

它在 shared buffer 中被读取、锁定、修改、写回。

修改 tuple header 时通常需要：

```text
pin heap buffer
lock heap buffer
必要时 pin visibility map buffer
进入 critical section
修改 page 和 tuple header
写 WAL 或满足 hint bit 写回规则
mark buffer dirty
unlock / release buffer
```

不同字段的 cleanup 生命周期不同。

`xmin` 通常不会被删除。

freeze 可能把它解释成 frozen。

`xmax` 可以被 hint bit 标记 invalid。

`t_ctid` 可能因为 update chain 和 pruning 变得不再可跟随。

HOT root 的 line pointer 可能从 normal 变成 redirect。

heap-only child 的 line pointer 可能被 prune 掉。

combo cid 生命周期更短。

combo cid table 是当前 backend、当前事务内部状态。

事务结束时会在 `TopTransactionContext` 释放。

因此 tuple header 中的 combo cid 值不能跨事务当作全局含义解释。

hint bit 生命周期则介于两者之间。

它写在 tuple header 上，可能持久化到磁盘。

但它只是事务结果缓存。

它不拥有事务结果本身。

如果 hint bit 不存在，系统仍然可以查 CLOG 得到正确答案。

## 14. 正确性机制层次

tuple header 正确性来自多层机制共同约束。

第一层是事务状态。

`xmin` 和 `xmax` 指向 XID 或 MultiXact。

最终提交/回滚来自 CLOG/pg_xact 或 MultiXact 状态。

第二层是 snapshot membership。

即使 CLOG 显示某事务后来提交，旧 snapshot 仍可能把它当作 running。

因此 `HEAP_XMIN_COMMITTED` 也不能绕过 `XidInMVCCSnapshot()`。

第三层是 command id。

当前事务内部不能只看 XID。

同一个 XID 下不同命令之间有顺序。

第四层是 buffer/page 保护。

修改 header 要在正确 buffer lock 下进行。

读 header 时也要遵守调用者约定。

第五层是 WAL 和 hint bit interlock。

把提交事实缓存进 data page 时，不能让 data page 暗示一个尚未持久化的提交。

第六层是 cleanup horizon。

即使 tuple 对当前 snapshot 不可见，也不能证明它可被物理移除。

这些层次共同说明：

```text
heap tuple header 是 MVCC 的本地证据；
不是 MVCC 的完整判决书。
```

## 15. 错误路径 / 异常路径 / fallback

### 15.1 `xmax` 是 MultiXact

多个 locker 可能同时存在，`xmax` 会被解释成 MultiXactId。

此时不能直接把 raw xmax 当成事务 ID，要结合 `HEAP_XMAX_IS_MULTI`、`HEAP_XMAX_LOCK_ONLY`、`HEAP_LOCK_MASK` 和 `HeapTupleGetUpdateXid()`。

MultiXact 解析可能带来额外 I/O 或状态访问，所以热路径会尽量先用 infomask 剪枝。

### 15.2 `t_ctid` 链路不可继续

后继 slot 可能已经被清理、重用，或者 `t_ctid` 可能表示 speculative insertion token / moved partition 特殊值。

正确 fallback 是停止跟随，而不是把它当成损坏。

普通 update chain 的关键校验是：

```text
priorXmax == next xmin
```

### 15.3 combo cid 只在本事务内可解释

如果不是 originating transaction，就不要用 `HeapTupleHeaderGetCmin()` / `GetCmax()` 解释别人的 command id。

其他事务只能根据 XID、snapshot、hint bit 和 CLOG 判断可见性。

## 16. 成本、资源与跨模块传播

tuple header 位于每条 heap tuple 前面。

这让 visibility hot path 可以在访问 tuple data 前先读少量字段。

成本收益是明显的：

```text
大多数 tuple 只需要读 header flags。
常见 committed / invalid 状态可以通过 hint bit 快速判断。
只有不确定状态才查 CLOG、MultiXact 或 snapshot array。
```

但这个设计也把复杂性传播到了多个模块：

```text
heapam.c:
  写 header，维护 update/delete/lock 语义。

heapam_visibility.c:
  读 header，结合 snapshot 和事务状态判断可见性。

combocid.c:
  解释当前事务 command id。

pruneheap.c:
  根据 header 和 cleanup horizon 修剪 page。

vacuumlazy.c:
  根据 header 判断 dead/recently-dead/freezable。

heapam_indexscan.c:
  根据 HOT flags 和 t_ctid 找可见 chain member。

visibilitymap.c:
  根据 page-level 全可见状态提供 index-only scan 优化。
```

因此 tuple header 的每一个 flag 都要谨慎修改。

它可能不只影响当前 heap operation。

它还可能影响索引扫描、VACUUM、logical decoding、recovery 和并发锁。

## 17. 观测与诊断入口

最直接的观测工具是 `pageinspect`。

常用字段：

```sql
SELECT lp,
       lp_flags,
       t_xmin,
       t_xmax,
       t_field3,
       t_ctid,
       t_infomask,
       t_infomask2
FROM heap_page_items(get_raw_page('mvcc_header_demo', 0));
```

观察时要注意：数字 XID 要结合事务状态解释，`t_infomask` / `t_infomask2` 要按 bit 解码，`t_field3` 不是全局时间戳，可能是 command id，也可能是 combo cid。

源码诊断入口：

```text
break heap_update
break HeapTupleSatisfiesMVCC
break HeapTupleHeaderAdjustCmax
break heap_hot_search_buffer
```

SQL 侧还可以配合：

```sql
SELECT txid_current();
SELECT xmin, xmax, ctid, * FROM mvcc_header_demo;
```

系统列 `xmin`、`xmax`、`ctid` 能帮助建立直觉，但它们不是完整 header，也不能替代源码和 `pageinspect`。

## 18. 常见误区

- `xmin` 小不等于更早可见。XID 分配顺序不是提交顺序，snapshot 还要看 running set。
- `xmax` 非零不等于 tuple 已删除。它可能只是 locker、aborted updater 或 MultiXact。
- `t_ctid` 不是永久可靠指针。VACUUM 可能已经清理后继，必须校验 line pointer 和后继 `xmin`。
- hint bit 不是可见性语义来源。它只是事务结果缓存；没有 hint bit 时仍然能查 CLOG。
- command id 不是全局时间戳。combo cid 只对 originating transaction 有完整意义。
- HOT flags 不只是优化。`HEAP_HOT_UPDATED` 和 `HEAP_ONLY_TUPLE` 决定索引扫描能否通过 root 找到 child。

## 19. 课堂实验

### 实验一：观察 UPDATE 后的旧版本和新版本

```sql
CREATE EXTENSION IF NOT EXISTS pageinspect;

DROP TABLE IF EXISTS mvcc_header_demo;
CREATE TABLE mvcc_header_demo(id int primary key, payload text);

INSERT INTO mvcc_header_demo VALUES (1, 'old');
SELECT xmin, xmax, ctid, * FROM mvcc_header_demo;

UPDATE mvcc_header_demo SET payload = 'new' WHERE id = 1;
SELECT xmin, xmax, ctid, * FROM mvcc_header_demo;

SELECT lp,
       lp_flags,
       t_xmin,
       t_xmax,
       t_field3,
       t_ctid,
       t_infomask,
       t_infomask2
FROM heap_page_items(get_raw_page('mvcc_header_demo', 0))
ORDER BY lp;
```

观察目标：

```text
旧版本 t_ctid 是否指向新版本。
新版本 t_ctid 是否指向自身。
旧版本 xmax 与新版本 xmin 是否相关。
infomask2 是否出现 HOT 或非 HOT 相关变化。
```

### 实验二：观察 command id 边界

```sql
BEGIN;

CREATE TEMP TABLE cid_demo(id int);

INSERT INTO cid_demo VALUES (1);
SELECT xmin, xmax, ctid, * FROM cid_demo;

DELETE FROM cid_demo WHERE id = 1;
SELECT xmin, xmax, ctid, * FROM cid_demo;

ROLLBACK;
```

再用 `pageinspect` 观察 temp 表所在 page 时，重点看 `t_field3`。

目标是理解：

```text
同一个 tuple header 的 command id 字段必须通过当前事务上下文解释。
```

### 实验三：源码断点

建议断点：

```text
break heap_update
break HeapTupleSatisfiesMVCC
break HeapTupleHeaderAdjustCmax
break HeapTupleSetHintBits
```

观察顺序：先执行 INSERT，再执行 UPDATE；在 `heap_update` 中看 `oldtup` / `newtup`，在 `HeapTupleSatisfiesMVCC` 中看 `tuple->t_infomask` 分支。

## 20. 讨论题

1. 为什么 PostgreSQL 不把 tuple header 设计成“visible_from”和“visible_to”两个提交序号？

2. 如果 `xmax` 只能表示删除者，tuple locking 需要额外放在哪里？

3. 为什么 `t_ctid` 后继校验要比较前驱 update xid 与后继 `xmin`？

4. 如果 hint bit 永远不写回，正确性会不会破坏？成本会在哪里增加？

5. 为什么 combo cid 不能成为所有事务都能解释的持久语义？

6. HOT child 没有普通索引项，为什么它仍然能被 index scan 找到？

7. pageinspect 看到的 header 字段，哪些是稳定事实，哪些是需要上下文解释的局部证据？

## 21. 本节小结

本节只建立一个基础模型：

```text
heap tuple header 描述的是一个 tuple version；
它不是逻辑行，也不是完整可见性结果。
```

`xmin` 记录版本出生。

`xmax` 记录删除、更新、锁定或 MultiXact。

`t_ctid` 连接版本链。

`t_infomask` 和 `t_infomask2` 把事务 hint、锁语义、HOT 状态和物理布局压进 header。

`t_field3` 和 combo cid 解决同一事务内部命令顺序。

这些字段必须和 snapshot、CLOG、MultiXact、command id、buffer lock、WAL/hint bit interlock、cleanup horizon 一起解释。

下一节会在这个基础上进入 `HeapTupleSatisfiesMVCC()`。

那一节的主问题不再是“字段在哪里”。

而是：

```text
普通 SELECT 如何把这些字段折叠成对当前 snapshot 的一个布尔可见性结果。
```
