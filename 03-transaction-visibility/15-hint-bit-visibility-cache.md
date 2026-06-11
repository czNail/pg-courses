# PostgreSQL hint bit 与可见性缓存

## 课程定位

前置知识：已经理解 heap tuple header 中 `HEAP_XMIN_COMMITTED`、`HEAP_XMIN_INVALID`、`HEAP_XMAX_COMMITTED`、`HEAP_XMAX_INVALID` 的位置，也理解 `HeapTupleSatisfiesMVCC()` 如何在 snapshot 和 CLOG 之间判断 tuple 可见性。

本节唯一主问题：

```text
PostgreSQL 为什么允许普通 backend 把事务提交或回滚结果写回 tuple header，hint bit 如何降低 CLOG 查询成本，又为什么它不能改变 MVCC 语义？
```

本节核心矛盾：

```text
每次可见性判断都查 CLOG 会让 heap scan、index scan 和 join input 付出重复共享状态成本；
但把事务结果缓存进 data page 又必须保证 crash safety、snapshot consistency 和 WAL 顺序不被破坏。
```

学完本节后，你应该能独立判断：

- hint bit 缓存的是什么，不缓存什么。
- 为什么 hint bit 可以缺失，但不能错误。
- 为什么设置 committed hint bit 要看事务 commit LSN 与 buffer LSN。
- 为什么普通 MVCC snapshot 不会为了设置 hint bit 去重新检查 snapshot 中仍 running 的事务。
- 为什么 batch visibility 要摊销 `BufferBeginSetHintBits()` / `BufferFinishSetHintBits()` 成本。
- 为什么 visibility map 不是 hint bit，但和可见性缓存同属“避免重复判断”的优化层。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

本节不展开 all-visible / all-frozen 的完整 VACUUM 规则。

这里只讲 hint bit 为什么能安全作为 tuple-level visibility cache。

## 1. 本节在总主线中的位置

上一节讲了普通 MVCC 判断。

判断一个 tuple 时，系统可能需要知道：

```text
xmin 是否提交？
xmin 是否回滚？
xmax 是否提交？
xmax 是否回滚？
```

如果每次遇到同一个 tuple 都去 CLOG/pg_xact 查事务状态，成本会重复出现。

一张老表被反复扫。

大多数 tuple 的插入事务早已提交。

删除事务要么无效，要么早已提交。

这些状态不会再改变。

所以 PostgreSQL 把部分事务结果缓存进 tuple header。

这个缓存就是 hint bit。

本节的主线是：

```text
第一次 visibility 检查发现事务结果
  -> 在满足安全条件时设置 hint bit
  -> 后续 visibility 检查先读 tuple header
  -> 少查 CLOG
  -> 但 snapshot 语义仍由 SnapshotData 决定
```

注意这里有一个边界。

hint bit 是缓存。

不是事务结果的来源。

如果 hint bit 不存在，系统仍然能查 CLOG 得到正确结果。

如果 hint bit 存在，系统也不能跳过 snapshot membership 的关键检查。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
visibility routine 在确认某个 xmin/xmax 的提交或回滚状态后，可以把这个不可逆事务结果写入 tuple header 的 hint bit；写入前要遵守 WAL flush 和 buffer 修改规则；以后相同 tuple 的可见性判断可以先消费 hint bit，但最终结果仍必须符合当前 snapshot。
```

这里的 tension 是：

```text
hot path 需要避免重复查 CLOG；
crash safety 要求 data page 不能先于对应 commit WAL 暗示一个不存在的提交。
```

这也是为什么 `SetHintBitsExt()` 不是简单：

```text
tuple->t_infomask |= HEAP_XMIN_COMMITTED
```

它要先确认：

```text
事务结果已经确定。
如果是 committed hint，commit LSN 与 buffer LSN 满足安全关系。
当前 backend 有权限修改这个 buffer 的 hint bit。
```

如果不满足，最保守的行为是：

```text
不设置 hint bit。
返回正确可见性结果。
等待以后再缓存。
```

## 3. 核心文件分工与阅读顺序

| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/access/heap/heapam_visibility.c` | 主读 `SetHintBitsExt()`、`SetHintBits()`、`HeapTupleSetHintBits()` 和 MVCC 分支中的 hint 写回点。 |
| 2 | `src/include/access/htup_details.h` | 对照 `HEAP_XMIN_COMMITTED`、`HEAP_XMIN_INVALID`、`HEAP_XMAX_COMMITTED`、`HEAP_XMAX_INVALID`。 |
| 3 | `src/backend/storage/buffer/bufmgr.c` | 看 `BufferSetHintBits16()`、`BufferBeginSetHintBits()`、`BufferFinishSetHintBits()` 的 buffer 边界。 |
| 4 | `src/backend/access/transam/transam.c` | 对照 `TransactionIdDidCommit()`、`TransactionIdDidAbort()` 等事务状态查询。 |
| 5 | `src/backend/access/transam/xact.c` | 对照 commit LSN 与事务提交记录。 |
| 6 | `src/backend/access/heap/visibilitymap.c` | 区分 tuple-level hint bit 与 page-level visibility map。 |
| 7 | `src/backend/access/heap/heapam.c` | 看 `HeapTupleSetHintBits()` 在 heap update/delete 后等待事务结束时的使用。 |

推荐阅读顺序：

```text
先读 heapam_visibility.c 顶部关于 SetHintBitsExt() 的注释
  -> 看 MVCC 分支何时设置 xmin/xmax hint
  -> 看 commit LSN 检查为什么可能让写回放弃
  -> 再读 batch MVCC 如何摊销 hint bit 写回
  -> 最后对照 visibilitymap.c 区分页级缓存
```

## 4. 一个 runtime 现象先定锚

构造一张表。

插入很多行并提交。

第一次扫描表时，backend 可能需要为很多 tuple 查询插入事务状态。

后续扫描同一批 page 时，tuple header 上可能已经有 committed hint bit。

于是重复 CLOG 查询减少。

实验不保证每次都能在 SQL 层稳定看到相同指标。

因为缓存、checkpoint、shared buffer、异步提交、autovacuum 和 pageinspect 观察时机都会影响现象。

但可以用下面的方式建立直觉：

```sql
CREATE EXTENSION IF NOT EXISTS pageinspect;

DROP TABLE IF EXISTS hint_demo;
CREATE TABLE hint_demo(id int primary key, payload text);

INSERT INTO hint_demo
SELECT g, repeat('x', 100)
FROM generate_series(1, 10000) AS g;

CHECKPOINT;

SELECT lp,
       t_xmin,
       t_xmax,
       t_infomask
FROM heap_page_items(get_raw_page('hint_demo', 0))
ORDER BY lp
LIMIT 10;

SELECT count(*) FROM hint_demo;

SELECT lp,
       t_xmin,
       t_xmax,
       t_infomask
FROM heap_page_items(get_raw_page('hint_demo', 0))
ORDER BY lp
LIMIT 10;
```

观察目标不是记住某个 `t_infomask` 数字。

观察目标是：

```text
tuple header 可能在普通读取后发生 hint bit 变化。
这种变化不会让 SQL 结果从错变对。
它只让后续判断更便宜。
```

如果没有看到变化，也不说明机制不存在。

可能原因包括：

```text
hint bit 已经被先前访问设置。
page 还在 buffer 中。
事务状态已经通过别的路径缓存。
构建或 VACUUM 已经处理过。
```

## 5. hint bit 缓存的状态

本节关注四个常见 hint bit：

```text
HEAP_XMIN_COMMITTED
HEAP_XMIN_INVALID
HEAP_XMAX_COMMITTED
HEAP_XMAX_INVALID
```

它们缓存的是事务结果。

`HEAP_XMIN_COMMITTED` 表示插入事务已提交。

`HEAP_XMIN_INVALID` 表示插入事务无效或回滚。

`HEAP_XMAX_COMMITTED` 表示删除或更新事务已提交。

`HEAP_XMAX_INVALID` 表示 `xmax` 无效、回滚或只是无效删除效果。

它们不缓存：

```text
这个 tuple 对所有 snapshot 可见。
这个 tuple 可以被 VACUUM 删除。
这个 tuple 是当前逻辑行的最新版本。
这个 xid 不在某个旧 snapshot 的 running set 中。
```

这一区分很关键。

例如 `HEAP_XMIN_COMMITTED` 设置以后，旧 snapshot 仍可能看不到该 tuple。

因为 `HeapTupleSatisfiesMVCC()` 还会问：

```text
XidInMVCCSnapshot(raw xmin, snapshot)
```

如果这个 XID 在 snapshot 看来 still-running，就算 committed hint 已经设置，也要当作不可见。

所以 hint bit 只是减少事务结果查询。

它不是跳过 snapshot 规则的许可。

## 6. 为什么 hint bit 可以写回 data page

事务提交或回滚结果一旦确定，就不会反转。

这让 hint bit 成为安全缓存的基础。

如果事务已提交：

```text
以后不会变成 aborted。
```

如果事务已回滚：

```text
以后不会变成 committed。
```

因此把这个结果缓存到 tuple header，本质上是写入一个可重复推导出的事实。

如果缓存缺失，可以重新查询 CLOG。

如果缓存存在，可以少查询一次 CLOG。

但它仍然必须满足 crash safety。

最危险的情况是：

```text
data page 上先持久化了 HEAP_XMIN_COMMITTED；
对应事务的 commit WAL 还没有持久化；
系统崩溃恢复后事务被视为未提交；
data page 却暗示 tuple 已提交。
```

为避免这个问题，`SetHintBitsExt()` 对 committed hint bit 做 commit LSN 检查。

如果对应 commit LSN 还需要 flush，并且 buffer LSN 早于 commit LSN，就不设置 hint bit。

这样宁愿少一个缓存，也不让 data page 提前承诺提交事实。

## 7. `SetHintBitsExt()` walkthrough

`SetHintBitsExt()` 的输入包括：

```text
HeapTupleHeader tuple
Buffer buffer
uint16 infomask
TransactionId xid
SetHintBitsState *state
```

`infomask` 是要设置的 hint。

`xid` 是需要检查 commit LSN 的事务。

如果 `xid` 无效，说明不需要 commit LSN 检查。

典型 invalid hint 就属于这种情况。

流程可以压缩为：

```text
如果 batch state 已经 disabled:
  直接返回。

如果 xid 有效且 buffer 是 permanent:
  读取 xid 的 commit LSN。
  如果 commit LSN 还需要 flush 且 buffer LSN 早于 commit LSN:
    不设置 hint bit。

如果不是 batch 模式:
  BufferSetHintBits16() 修改 infomask。
  返回。

如果是 batch 模式且第一次需要写:
  BufferBeginSetHintBits()
  如果失败，state 变 disabled。

设置 tuple->t_infomask。
```

这个流程有三个细节。

第一，hint bit 写回可以失败。

失败不影响当前可见性返回值。

第二，batch 模式延迟开始。

只有真的要设置 hint bit 时才调用 `BufferBeginSetHintBits()`。

第三，batch 模式结束后要 `BufferFinishSetHintBits()`。

这让一个 page 上多条 tuple 的 hint bit 写回共享一次 buffer 处理成本。

## 8. MVCC 分支中的 hint bit 写回

在 `xmin` 分支中，如果 `xmin` 不在当前 snapshot 的 running set，函数才会查真实事务状态。

如果 `TransactionIdDidCommit(xmin)` 为 true：

```text
SetHintBitsExt(tuple, buffer, HEAP_XMIN_COMMITTED, xmin, state)
```

如果不是提交：

```text
SetHintBitsExt(tuple, buffer, HEAP_XMIN_INVALID, InvalidTransactionId, state)
return false
```

在 `xmax` 分支中，如果删除事务已回滚：

```text
SetHintBitsExt(tuple, buffer, HEAP_XMAX_INVALID, InvalidTransactionId, state)
return true
```

如果删除事务已提交：

```text
SetHintBitsExt(tuple, buffer, HEAP_XMAX_COMMITTED, xmax, state)
return false
```

注意这几个写回点都在已经知道事务结果之后。

函数不会把“不确定”写成 hint。

也不会为了 hint 而破坏 snapshot 语义。

如果 `XidInMVCCSnapshot(xid, snapshot)` 返回 true，MVCC 分支通常直接按 running 处理。

它不会继续查 CLOG 只为了设置 hint。

## 9. hint bit 与 visibility map 的边界

hint bit 是 tuple-level cache。

visibility map 是 page-level cache。

二者都和可见性有关。

但回答的问题不同。

hint bit 回答：

```text
某个 tuple header 中的 xmin/xmax 事务结果是否已经知道。
```

visibility map 回答：

```text
某个 heap page 上的所有 tuple 是否对所有事务可见。
某个 heap page 是否全冻结。
```

hint bit 设置可能发生在普通 SELECT 中。

visibility map 的 all-visible/all-frozen 主要由 VACUUM / pruning 等路径维护。

hint bit 不足以让 index-only scan 跳过 heap。

index-only scan 需要 visibility map 的 all-visible 位。

但 visibility map 的判断也离不开底层 tuple visibility 和 cleanup horizon。

所以可以这样理解：

```text
hint bit:
  缓存单个 tuple 的事务结果。

visibility map:
  缓存整个 page 的全局可见性结论。
```

二者都是避免重复判断。

但缓存粒度和正确性边界不同。

## 10. 生命周期 / ownership / cleanup

hint bit 写在 heap page 上。

它不是 backend-local 状态。

一旦 page 被写回，hint bit 可能持久存在。

它的生命周期和 tuple header 一起走。

但设置 hint bit 的 backend 不拥有这个语义。

它只是把可推导事实缓存进去。

修改 hint bit 需要 buffer manager 参与。

普通单 tuple 写回使用：

```text
BufferSetHintBits16()
```

batch 写回使用：

```text
BufferBeginSetHintBits()
BufferFinishSetHintBits()
```

cleanup 时不需要单独清理 hint bit。

如果 tuple 被 VACUUM 移除，整个 tuple header 消失。

如果 tuple 被 freeze，`xmin` 解释可能改变为 frozen。

如果 page 被重写，hint bit 随 tuple header 复制或重建。

因此 hint bit 没有独立 owner。

它依附于 tuple header。

## 11. 正确性机制层次

第一层是不变事务结果。

提交或回滚一旦确定，hint bit 可以缓存。

第二层是 WAL flush interlock。

committed hint 不能让 data page 比 commit WAL 更早持久化提交事实。

第三层是 snapshot consistency。

hint bit 不覆盖 `XidInMVCCSnapshot()`。

第四层是 buffer 修改规则。

设置 hint bit 要通过 buffer manager 标记 page 状态。

第五层是 fallback。

如果不能安全设置 hint bit，就不设置。

查询仍返回正确结果。

这套机制保证：

```text
hint bit 可以缺失，可以延迟，可以由任意后续 backend 设置；
但不能让一个 snapshot 看到它不该看到的事务效果。
```

## 12. 错误路径 / 异常路径 / fallback

### 12.1 commit LSN 未 flush

如果提交事务的 commit LSN 还需要 flush，且当前 buffer LSN 早于 commit LSN，`SetHintBitsExt()` 放弃设置 committed hint。

fallback 是：

```text
本次 visibility 判断照常返回；
以后再由其它访问者尝试设置 hint。
```

### 12.2 batch 写回无法开始

`BufferBeginSetHintBits()` 可能失败。

batch state 会变成 disabled。

同一批 tuple 后续不再尝试写 hint。

这避免反复付出失败成本。

### 12.3 snapshot 中仍 running

如果 xid 在当前 snapshot 看来 running，函数不会为了 hint 去查最新事务状态。

fallback 是：

```text
按 snapshot running 规则返回。
hint bit 留给更晚的 snapshot 设置。
```

### 12.4 非 permanent buffer

对非永久 buffer，commit LSN 检查边界不同。

临时或非持久关系不承担同样的 crash recovery 约束。

但课程主线仍然是：

```text
不能用 hint bit 改变事务语义。
```

## 13. 成本、资源与跨模块传播

hint bit 的收益是减少重复 CLOG 查询。

代价是可能 dirty heap page。

这个代价不是零。

一次只读查询也可能因为设置 hint bit 而产生脏页。

这会影响：

```text
checkpoint 写回量
buffer dirty rate
page checksum
wal_log_hints 场景
缓存命中后的后续扫描成本
```

系统接受这个折中，是因为长期看可见性判断是极高频路径。

把已知事务结果放在 tuple header，能让后续扫描更便宜。

跨模块传播包括：

```text
heapam_visibility.c:
  决定何时设置 hint。

transam:
  提供事务状态和 commit LSN。

bufmgr:
  管理 hint bit 写回和 dirty page。

checkpoint:
  负责之后把 dirty page 写回。

VACUUM / pruning:
  受益于更便宜的事务状态判断，但不把 hint bit 当 cleanup horizon。

visibility map:
  在更高粒度缓存 page-level 可见性。
```

## 14. 观测与诊断入口

SQL 观测：

```sql
CREATE EXTENSION IF NOT EXISTS pageinspect;

SELECT lp,
       t_xmin,
       t_xmax,
       t_infomask
FROM heap_page_items(get_raw_page('hint_demo', 0))
ORDER BY lp
LIMIT 20;
```

配合：

```sql
SELECT count(*) FROM hint_demo;
CHECKPOINT;
```

可以观察访问前后 `t_infomask` 是否变化。

源码断点：

```text
break SetHintBitsExt
break HeapTupleSatisfiesMVCC
break HeapTupleSatisfiesMVCCBatch
break BufferBeginSetHintBits
break BufferFinishSetHintBits
```

关键变量：

```text
infomask
xid
tuple->t_infomask
BufferGetLSNAtomic(buffer)
TransactionIdGetCommitLSN(xid)
```

诊断时要区分：

```text
SQL 结果错误:
  通常不是 hint bit 缺失导致。

扫描变慢或 CLOG 压力:
  可能和 hint bit 缺失、缓存冷、事务状态查询有关。
```

## 15. 常见误区

误区一：

```text
hint bit 是 MVCC 的事实来源。
```

不对。

事实来源是事务状态和 snapshot。

hint bit 是缓存。

误区二：

```text
只读 SELECT 永远不会 dirty page。
```

不对。

SELECT 可能设置 hint bit。

误区三：

```text
HEAP_XMIN_COMMITTED 一旦设置，所有 snapshot 都能看见该 tuple。
```

不对。

旧 snapshot 仍可能把该 XID 视为 running。

误区四：

```text
没有 hint bit 就表示事务还没提交。
```

不对。

没有 hint 只表示 header 还没缓存结果。

误区五：

```text
visibility map 和 hint bit 是同一个东西。
```

不对。

hint bit 是 tuple-level transaction status cache。

visibility map 是 page-level all-visible/all-frozen cache。

## 16. 课堂实验

### 实验一：观察读后 header 变化

```sql
CREATE EXTENSION IF NOT EXISTS pageinspect;

DROP TABLE IF EXISTS hint_demo;
CREATE TABLE hint_demo(id int primary key, payload text);

INSERT INTO hint_demo
SELECT g, repeat('x', 100)
FROM generate_series(1, 10000) AS g;

CHECKPOINT;

SELECT lp, t_xmin, t_xmax, t_infomask
FROM heap_page_items(get_raw_page('hint_demo', 0))
ORDER BY lp
LIMIT 20;

SELECT count(*) FROM hint_demo;

SELECT lp, t_xmin, t_xmax, t_infomask
FROM heap_page_items(get_raw_page('hint_demo', 0))
ORDER BY lp
LIMIT 20;
```

解释目标：

```text
如果 t_infomask 变化，说明普通访问可能写回 hint bit。
如果没变化，说明 hint 可能已经存在或观察时机不同。
两者都不改变 SQL 可见性语义。
```

### 实验二：删除后观察 xmax hint

```sql
DELETE FROM hint_demo WHERE id <= 100;
CHECKPOINT;

SELECT count(*) FROM hint_demo;

SELECT lp, t_xmin, t_xmax, t_infomask
FROM heap_page_items(get_raw_page('hint_demo', 0))
ORDER BY lp
LIMIT 120;
```

观察目标：

```text
xmax 相关 hint 可能被设置。
但 dead tuple 是否可 cleanup 仍要看 horizon。
```

### 实验三：源码断点

断点：

```text
break SetHintBitsExt
break HeapTupleSatisfiesMVCCBatch
```

执行：

```sql
SELECT count(*) FROM hint_demo;
```

观察：

```text
哪些 tuple 触发 SetHintBitsExt。
哪些路径因为 commit LSN 检查放弃。
batch state 何时进入 SHB_ENABLED 或 SHB_DISABLED。
```

## 17. 讨论题

1. 为什么 hint bit 缺失不会导致可见性错误？

2. 为什么 committed hint bit 需要 commit LSN flush interlock，而 invalid hint 通常不需要同样的 xid 检查？

3. 为什么普通 MVCC snapshot 不为了设置 hint bit 去查询 snapshot 中仍 running 的事务？

4. 只读查询 dirty page 是不是设计错误？收益和代价分别是什么？

5. visibility map 为什么不能替代 tuple-level hint bit？

6. 如果完全禁用 hint bit，系统正确性和性能会分别怎样变化？

## 18. 本节小结

hint bit 是 tuple header 上的事务结果缓存。

它让后续 visibility 判断少查 CLOG。

它可以由普通 backend 在读路径中写回。

但它必须服从三个边界：

```text
只缓存已经确定且不可逆的事务结果。
committed hint 不能越过 WAL flush 安全边界。
hint bit 不能覆盖 snapshot membership 和 command id 语义。
```

因此 hint bit 的正确理解是：

```text
它让已知事实更便宜；
它不创造新的事实。
```

下一节会继续处理当前事务自己的 tuple：

```text
同一个 XID 下，为什么还需要 command id、combo cid 和 HeapTupleSatisfiesSelf 这样的特殊规则。
```
