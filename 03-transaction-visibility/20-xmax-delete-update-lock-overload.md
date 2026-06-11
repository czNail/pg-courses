# PostgreSQL xmax 的删除、更新与锁定多重含义

## 课程定位

前置知识：已经理解 heap tuple header 中 `xmin` / `xmax`、infomask、hint bit、`t_ctid`、普通 MVCC snapshot 和 cleanup horizon。

本节唯一主问题：

```text
为什么 tuple header 里的一个 xmax 字段，既能表示删除者、更新者，又能表示一个或多个行锁持有者？
```

本节围绕的核心矛盾：

```text
heap tuple header 空间极小，不能为“删除事务”“更新事务”“行锁集合”分别保存完整状态；
但可见性、并发更新、外键检查、SELECT FOR UPDATE 和 VACUUM 又必须准确区分这些语义。
```

学完后应能独立判断：

- 为什么 `xmax` 有值不等于 tuple 已删除。
- 为什么 `HEAP_XMAX_LOCK_ONLY` 是理解行锁的第一道分界。
- 为什么 `HEAP_XMAX_IS_MULTI` 会把 `xmax` 从 XID 解释成 MultiXactId。
- 为什么 `HEAP_KEYS_UPDATED` 决定 key-share 是否必须等待。
- 为什么 `HEAP_XMAX_EXCL_LOCK` 单独不能区分 `FOR UPDATE` 和 `FOR NO KEY UPDATE`。
- 为什么删除、更新和锁定最终都会走 `compute_new_xmax_infomask()`。
- 为什么锁 tuple 也需要 WAL。
- 为什么 VACUUM 不能把所有旧 `xmax` 都当成可清理垃圾。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

前面第 13-19 节规划的是 heap tuple visibility。

那组课回答：

```text
一个 tuple version 对当前 snapshot 是否可见？
```

从本节开始进入 row lock / MultiXact / update conflict 主题组。

这组课回答另一个问题：

```text
一个 tuple version 在被读、锁、更新、删除时，如何在同一个 tuple header 上表达并发占用关系？
```

`xmin` 的语义相对稳定。

它回答 tuple version 从哪里来。

`xmax` 更复杂。

它回答 tuple version 可能被谁“占用”或“终结”。

这种占用可以是：

- 纯行锁。
- 删除。
- 非 key UPDATE。
- key UPDATE。
- 多个事务共同锁住同一行。
- 一个 MultiXact 中同时保留 locker 和 updater。

所以本节不是讲完整行锁矩阵。

行锁模式和冲突矩阵放到第 21 节。

MultiXact 成员存储放到第 22 节。

UPDATE / DELETE 的等待、重查和 EPQ 放到第 23 节。

本节先解决最底层的语义入口：

```text
读到一个 tuple header 的 xmax 以后，为什么必须结合 infomask 才能知道它代表什么？
```

如果这个入口没建立好，后面看到 `TM_Updated`、`TM_Deleted`、`MultiXactIdWait()`、`FOR KEY SHARE` 或 `relminmxid` 都会混在一起。

本节的主线是：

```text
一个可见 tuple
  -> 被 SELECT FOR KEY SHARE 锁住
  -> xmax 写入 locker XID，设置 LOCK_ONLY
  -> 另一个事务尝试 UPDATE
  -> heap_update 判断是否 key update
  -> 必要时等待或把 xmax 扩展成 MultiXact
  -> visibility 代码按 infomask 解释同一个 xmax
```

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
xmax 是一个 overloaded slot；
它只保存一个 32-bit 标识，真正语义由 HEAP_XMAX_INVALID、HEAP_XMAX_LOCK_ONLY、HEAP_XMAX_IS_MULTI、HEAP_XMAX_*_LOCK、HEAP_KEYS_UPDATED、t_ctid、事务状态和 MultiXact 成员共同决定。
```

这个模型有三个边界。

第一，raw `xmax` 不是语义。

同一个字段可能是：

- `InvalidTransactionId`。
- 一个 locker 的 XID。
- 一个 deleter 的 XID。
- 一个 updater 的 XID。
- 一个 MultiXactId。

第二，`LOCK_ONLY` 不是“没有锁”。

它表示：

```text
这个 xmax 不终结 tuple version；
它只是把 tuple 标记为被锁住。
```

因此 SELECT 仍可能看到该 tuple。

VACUUM 也不能把它当成 deleted tuple。

第三，MultiXact 不是“多个 share lock”的同义词。

在当前源码中 MultiXact 可以包含：

- `ForKeyShare`。
- `ForShare`。
- `ForNoKeyUpdate`。
- `ForUpdate`。
- `NoKeyUpdate` updater。
- `Update` updater。

因此判断 MultiXact 必须进入成员数组。

只看 `HEAP_XMAX_IS_MULTI` 只能知道解释入口变了。

不能直接知道是否有删除、更新或冲突。

本节要带走的系统规律是：

```text
当一个存储字段被多种并发语义复用时，字段值本身只是一枚引用；
语义必须由 flag、生命周期、事务状态和调用方目标共同解释。
```

## 3. 核心文件分工与阅读顺序

| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/access/heap/README.tuplock` | tuple lock 的两层协议、四种锁强度、`xmax` 和 infomask 的语义说明。 |
| 2 | `src/include/access/htup_details.h` | `HEAP_XMAX_*`、`HEAP_KEYS_UPDATED`、tuple header accessor。 |
| 3 | `src/backend/access/heap/heapam.c` | `heap_lock_tuple()`、`heap_update()`、`heap_delete()`、`compute_new_xmax_infomask()`。 |
| 4 | `src/backend/access/heap/heapam_visibility.c` | `HeapTupleSatisfiesMVCC()`、`HeapTupleSatisfiesUpdate()` 如何消费 `xmax`。 |
| 5 | `src/include/access/multixact.h` | `MultiXactStatus`、`MultiXactMember`、lock-only 与 updater 状态。 |
| 6 | `src/backend/access/transam/multixact.c` | `MultiXactIdCreate()`、`MultiXactIdExpand()`、`GetMultiXactIdMembers()`。 |
| 7 | `src/backend/storage/lmgr/lmgr.c` | `LockTuple()`、`XactLockTableWait()` 提供等待和排队层。 |
| 8 | `src/backend/executor/nodeLockRows.c` | `SELECT FOR UPDATE/SHARE` 如何映射到 `LockTupleMode`。 |
| 9 | `src/backend/executor/nodeModifyTable.c` | `ExecUpdate()` / `ExecDelete()` 如何消费 `TM_Result`。 |
| 10 | `src/backend/access/heap/vacuumlazy.c` | VACUUM 后续如何处理含有 `xmax` / MultiXact 的 tuple。 |

推荐阅读顺序：

```text
先读 README.tuplock 的语义说明
  -> 对照 htup_details.h 中 infomask 位
  -> 读 heap_lock_tuple() 看纯锁如何写 xmax
  -> 读 heap_update() / heap_delete() 看更新删除如何写 xmax
  -> 读 compute_new_xmax_infomask() 看已有 xmax 如何合并新语义
  -> 最后读 visibility 和 executor 如何解释返回结果
```

不要从 `heapam.c` 顶到底顺序读。

`heapam.c` 同时包含 insert、scan、toast、rewrite、vacuum helper、WAL logging 和 tuple lock。

本节只追 `xmax` 的语义转换。

## 4. 一个 runtime 现象先定锚

先看一个最小现象。

Session A：

```sql
DROP TABLE IF EXISTS xmax_demo;
CREATE TABLE xmax_demo(id int primary key, payload text);
INSERT INTO xmax_demo VALUES (1, 'old');

BEGIN;
SELECT * FROM xmax_demo WHERE id = 1 FOR KEY SHARE;
-- 保持事务不提交。
```

Session B：

```sql
SELECT * FROM xmax_demo WHERE id = 1;

UPDATE xmax_demo
SET payload = 'new'
WHERE id = 1;
```

现象是：

- 普通 SELECT 可以看到这一行。
- 行上已经有锁。
- 如果 UPDATE 不修改 key，它可能不需要被 key-share 阻塞。
- 如果 UPDATE 修改 key，或者 DELETE 这行，就必须与 key-share 发生冲突。

如果安装了 `pageinspect`，还可以观察 tuple header：

```sql
CREATE EXTENSION IF NOT EXISTS pageinspect;
SELECT lp, t_xmin, t_xmax, t_ctid, t_infomask, t_infomask2
FROM heap_page_items(get_raw_page('xmax_demo', 0));
```

这里可能看到 `t_xmax` 变成 Session A 的 XID。

但这行并没有被删除。

它仍然能被普通 SELECT 看见。

真正的解释不在 `t_xmax` 一列里。

解释必须加入：

- `t_infomask` 中是否有 `HEAP_XMAX_LOCK_ONLY`。
- 是否有 `HEAP_XMAX_IS_MULTI`。
- 是否有 `HEAP_XMAX_KEYSHR_LOCK` / `SHR_LOCK` / `EXCL_LOCK`。
- `t_infomask2` 中是否有 `HEAP_KEYS_UPDATED`。
- `xmax` 指向的事务是否仍在运行。
- 当前操作要申请哪种 `LockTupleMode`。

这个实验的关键不是 pageinspect 数值。

不同版本和不同执行时机可能设置不同 hint bit。

关键 runtime truth 是：

```text
xmax 非空只说明 tuple header 上存在“另一个事务相关的占用记录”；
是否表示 tuple 死亡，要看 infomask 和事务状态。
```

## 5. `xmax` 的三类主语义

### 5.1 无效或无占用

最简单状态：

```text
HEAP_XMAX_INVALID
```

这表示 header 里的 raw `xmax` 不应被当成有效事务引用。

常见来源：

- 新插入 tuple 没有被锁、更新或删除。
- 旧 locker 已结束，hint bit 把锁信息标成无效。
- VACUUM / freeze 把无意义的 `xmax` 清掉。

这个状态对 visibility 的意义是：

```text
tuple 还没有被有效 xmax 终结。
```

但它不表示 tuple 一定对当前 snapshot 可见。

还要看 `xmin` 是否可见。

### 5.2 纯行锁

纯行锁的典型组合：

```text
xmax = locker XID 或 MultiXactId
HEAP_XMAX_LOCK_ONLY
某个 HEAP_XMAX_*_LOCK bit
可能有 HEAP_XMAX_IS_MULTI
```

纯行锁不改变 tuple version 的存在性。

它只给并发写入者增加等待或冲突判断。

普通 SELECT 不会因为行被 `FOR SHARE` 锁住就看不到它。

这就是很多初读者犯错的地方：

```text
把 xmax 当作 delete xid；
看到 xmax 有值就认为 tuple 已死。
```

这在 PostgreSQL 里是错的。

### 5.3 更新或删除

更新和删除都会在旧版本上写 `xmax`。

区别在于 `t_ctid` 和 key bit。

DELETE：

```text
旧 tuple 的 xmax = deleter XID
旧 tuple 的 t_ctid 通常仍指向自己
HEAP_KEYS_UPDATED 语义上等价于 key gone
```

UPDATE：

```text
旧 tuple 的 xmax = updater XID
旧 tuple 的 t_ctid 指向新 tuple version
如果 key 变化，设置 HEAP_KEYS_UPDATED
如果 key 不变，可能是 no-key update
```

所以 `xmax` 也不能单独区分 UPDATE 和 DELETE。

可见性代码需要结合：

- `t_ctid` 是否指向自己。
- `HEAP_KEYS_UPDATED`。
- updater 事务状态。
- 调用者是 SELECT、UPDATE、DELETE、LOCK 还是 VACUUM。

## 6. 状态边界：字段组合才是语义

可以把 tuple header 里相关状态压缩成下表。

| 组合 | 初步解释 | 还必须检查 |
| --- | --- | --- |
| `HEAP_XMAX_INVALID` | 没有有效 xmax | `xmin` 可见性仍要判断。 |
| `xmax` + `LOCK_ONLY` | 纯行锁 | locker 是否运行、当前锁模式是否冲突。 |
| `xmax` + no `LOCK_ONLY` | 更新或删除 | XID 是否提交、`t_ctid` 是否指向新版本。 |
| `IS_MULTI` | `xmax` 是 MultiXactId | 成员 status、成员 XID 状态、是否有 updater。 |
| `EXCL_LOCK` + no `KEYS_UPDATED` | no-key exclusive 类语义 | 不能直接当 `FOR UPDATE`。 |
| `EXCL_LOCK` + `KEYS_UPDATED` | key-changing exclusive 类语义 | key-share 会冲突。 |

这里最重要的是：

```text
HEAP_XMAX_EXCL_LOCK 只是强锁位；
HEAP_KEYS_UPDATED 才把 key-changing 语义补齐。
```

`README.tuplock` 明确说明：

`HEAP_XMAX_EXCL_LOCK` 不区分 `FOR UPDATE` 和 `FOR NO KEY UPDATE`。

当前源码通过 `HEAP_KEYS_UPDATED` 区分。

因此外键相关的 key-share 逻辑不能只看 exclusive bit。

它要知道主键或唯一键是否可能消失或变化。

## 7. 主流程一：`heap_lock_tuple()` 写入纯锁

`SELECT ... FOR KEY SHARE`、`FOR SHARE`、`FOR NO KEY UPDATE`、`FOR UPDATE` 最终通过 table AM 进入 tuple lock。

heap AM 的核心入口是：

```text
nodeLockRows.c
  -> table_tuple_lock()
     -> heap_lock_tuple()
```

`nodeLockRows.c` 先把 SQL rowmark 转成 `LockTupleMode`：

```text
ROW_MARK_KEYSHARE
  -> LockTupleKeyShare

ROW_MARK_SHARE
  -> LockTupleShare

ROW_MARK_NOKEYEXCLUSIVE
  -> LockTupleNoKeyExclusive

ROW_MARK_EXCLUSIVE
  -> LockTupleExclusive
```

进入 `heap_lock_tuple()` 后，流程不是直接写 header。

它先做：

```text
ReadBuffer()
  -> LockBuffer(... BUFFER_LOCK_EXCLUSIVE)
  -> HeapTupleSatisfiesUpdate()
  -> 根据 TM_Result 判断是否可锁、要等、要追链、要报错
```

如果 tuple 当前可锁，才进入写 `xmax` 的阶段。

写入前会调用：

```text
MultiXactIdSetOldestMember()
compute_new_xmax_infomask(...)
```

这里有一个关键点：

即使最后没有真的创建 MultiXact，也要先设置本事务的 oldest member 边界。

原因是另一个 backend 可能马上把当前 XID 加入某个 MultiXact。

系统必须保证 MultiXact SLRU 不会在当前事务还可能成为成员时被截断。

真正写 tuple header 的关键状态是：

```text
tuple->t_infomask 清掉旧 HEAP_XMAX_BITS
tuple->t_infomask 写入 new_infomask
tuple->t_infomask2 写入 new_infomask2
HeapTupleHeaderSetXmax(tuple, xid)
如果是 lock-only，则 t_ctid 指回自己
```

所以纯锁也会改变 tuple header。

它不是只在 lock table 里登记。

## 8. 主流程二：`heap_update()` / `heap_delete()` 终结旧版本

UPDATE 和 DELETE 的入口不同，但它们面对旧 tuple 的核心问题相同：

```text
旧 tuple 上已有 xmax；
当前事务要把自己也写进 xmax；
需要判断已有 xmax 是锁、更新、删除还是 MultiXact；
不能覆盖仍有意义的状态。
```

DELETE 主流程：

```text
ExecDelete()
  -> table_tuple_delete()
     -> heap_delete()
        -> HeapTupleSatisfiesUpdate()
        -> 等待冲突事务或 MultiXact
        -> compute_new_xmax_infomask(..., LockTupleExclusive, is_update=true)
        -> 写旧 tuple xmax
        -> WAL XLOG_HEAP_DELETE
```

UPDATE 主流程：

```text
ExecUpdate()
  -> table_tuple_update()
     -> heap_update()
        -> 决定 LockTupleNoKeyExclusive 或 LockTupleExclusive
        -> HeapTupleSatisfiesUpdate()
        -> 等待冲突事务或 MultiXact
        -> compute_new_xmax_infomask(..., is_update=true)
        -> 写旧 tuple xmax
        -> 插入新 tuple version
        -> WAL XLOG_HEAP_UPDATE
```

UPDATE 的锁强度取决于 key 是否变化。

如果 key 没变：

```text
LockTupleNoKeyExclusive
MultiXactStatusNoKeyUpdate
```

如果 key 变化：

```text
LockTupleExclusive
MultiXactStatusUpdate
HEAP_KEYS_UPDATED
```

DELETE 等价于 key gone。

它必须比普通 no-key update 更强。

这就是外键检查能用 `FOR KEY SHARE` 的原因：

```text
child insert 只需要阻止 referenced key 消失或改变；
它不需要阻止 parent row 的非 key 列更新。
```

## 9. 主流程三：`compute_new_xmax_infomask()` 的状态机

`compute_new_xmax_infomask()` 是本节最值得精读的函数。

它不是一个简单 setter。

它是一个小状态机。

输入包括：

- 旧 `xmax`。
- 旧 `infomask`。
- 旧 `infomask2`。
- 要加入的新 XID。
- 新操作的 `LockTupleMode`。
- 这次是否是 update/delete 语义。

输出包括：

- 新 `xmax`。
- 新 `infomask`。
- 新 `infomask2`。

它的主要分支可以这样读：

```text
旧 xmax invalid
  -> 当前事务直接成为 xmax
  -> 如果只是锁，设置 LOCK_ONLY 和锁强度
  -> 如果是 key update，设置 HEAP_KEYS_UPDATED

旧 xmax 是 MultiXact
  -> 如果 old multi 已经无意义，退化成 invalid 后重算
  -> 否则 MultiXactIdExpand()
  -> 用成员推导新 hint bits

旧 xmax 是 committed updater
  -> 不能丢掉 updater
  -> 创建新 MultiXact 保存旧 updater 和当前成员

旧 xmax 是 in-progress XID
  -> 根据旧 infomask 推导旧 status
  -> 如果同一个事务升级锁，保留最强模式
  -> 否则创建 MultiXact

旧 xmax 刚刚结束
  -> 重新当作 invalid 处理
```

这里的设计目标不是“把新锁写进去”这么简单。

它必须保证：

- 不能丢掉已经提交的 updater。
- 不能把 aborted locker 长期保留成有效冲突。
- 不能把同一个事务的锁升级写成自冲突。
- 不能原地修改已有 MultiXact 成员。
- 不能把 key update 和 no-key update 混淆。

特别注意这条规则：

```text
MultiXactIdExpand() 不修改旧 MultiXact；
它创建一个新的 MultiXactId。
```

这样等待旧 MultiXact 的 backend 不会看到成员集合在脚下变化。

## 10. 生命周期 / ownership / cleanup

`xmax` 的 owner 不是 tuple 所在 backend。

tuple header 存在磁盘页和 shared buffer 里。

它可以比写入它的 backend 活得更久。

不同 `xmax` 语义有不同生命周期。

纯行锁：

```text
创建者:
  heap_lock_tuple()

持有者:
  locker transaction 或 MultiXact 成员事务

释放:
  事务结束释放事务锁

tuple header cleanup:
  后续读者或 updater 可设置 hint bit
  VACUUM / freeze 可清理无意义 lock-only xmax
```

DELETE：

```text
创建者:
  heap_delete()

持有者:
  deleter XID 的事务结果

释放:
  事务结束后 XID 状态固定

tuple header cleanup:
  VACUUM 在 cleanup horizon 安全后移除旧 tuple
```

UPDATE：

```text
创建者:
  heap_update()

持有者:
  updater XID

额外状态:
  t_ctid 指向新 tuple version
  可能设置 HEAP_KEYS_UPDATED

cleanup:
  HOT pruning / VACUUM 按 visibility horizon 删除旧版本
```

MultiXact：

```text
创建者:
  MultiXactIdCreate() 或 MultiXactIdExpand()

持有状态:
  pg_multixact offsets SLRU
  pg_multixact members SLRU
  tuple header 的 xmax 引用 MultiXactId

cleanup:
  VACUUM freeze 或 pruning 消除 tuple 引用
  relminmxid / datminmxid 推进后才能截断 SLRU
```

这说明 `xmax` cleanup 不是单点动作。

纯锁可以因为事务结束而语义失效。

但 header 位不一定马上被清掉。

更新删除的 `xmax` 即使事务已结束，也可能仍然是版本链和可见性判断的一部分。

MultiXact 的存储还要等所有引用它的 relation 都推进 `relminmxid`。

## 11. 正确性机制层次

本节涉及至少五层正确性机制。

第一层：buffer content lock。

写 tuple header 时必须持有 buffer exclusive lock。

它保护页内字节不会被并发修改破坏。

它不提供事务语义。

第二层：事务锁。

`XactLockTableWait()` 等待某个 XID 结束。

它回答：

```text
这个事务是否还在运行？
```

它不回答 tuple 是否可见。

第三层：tuple lock tag。

`LockTuple()` 在 lock manager 里建立排队顺序。

它避免一群 waiter 同时被 XID 结束唤醒后抢同一行造成饥饿。

但 PostgreSQL 不能为每个被锁行长期保存 lock table 对象。

所以 lock table 只是等待期间的第二层协议。

真正的长期锁记录仍在 tuple header / MultiXact。

第四层：infomask。

infomask 把 `xmax` 解释成锁、更新、删除或 MultiXact。

它是 visibility hot path 的压缩语义。

第五层：WAL。

即使只是行锁，也可能写 WAL。

因为 row lock 可能把一个新的 XID 或 MultiXactId 写进数据页。

如果数据页先落盘而 WAL 没有覆盖这个 ID，崩溃后 ID 可能被重用。

这会破坏可见性判断。

所以 `heap_lock_tuple()` 中有 `XLOG_HEAP_LOCK`。

## 12. 错误路径 / 异常路径 / fallback

### 12.1 等待后必须重查

等待一个 XID 或 MultiXact 结束后，不能直接继续写 header。

原因是等待期间别人可能改了 tuple。

`heap_lock_tuple()` 会重拿 buffer lock 后检查：

```text
xmax_infomask_changed(...)
raw xmax 是否仍等于 xwait
```

如果变了，就跳回 `l3` 重新跑 `HeapTupleSatisfiesUpdate()`。

这是一条典型并发源码规律：

```text
等待只关闭一个旧条件；
重新持锁后必须重读共享状态。
```

### 12.2 NOWAIT / SKIP LOCKED

SQL 层的 `NOWAIT` 和 `SKIP LOCKED` 最终表现为不同 `LockWaitPolicy`。

在 heap tuple lock 中：

- `LockWaitBlock` 正常等待。
- `LockWaitSkip` 获取不到就返回 `TM_WouldBlock`。
- `LockWaitError` 获取不到就报 `lock_not_available`。

这些策略不改变 `xmax` 的语义。

它们只改变遇到冲突时调用者怎么收尾。

### 12.3 pg_upgrade 旧 MultiXact

源码中有 `HEAP_LOCKED_UPGRADED()` 分支。

它处理旧版本升级上来的 share-locked tuple。

这类 tuple 可能带有当前实现不再创建的 bit pattern。

当前代码不能把所有旧形态都当成普通 MultiXact。

它会在安全时把它们当作不再运行的 lock-only 状态处理。

### 12.4 同事务锁升级

同一个事务可能已经持有弱锁，又尝试更强锁。

如果照常进入 lock manager 排队，可能和另一个等待者形成死锁。

`compute_new_xmax_infomask()` 对同 XID 升级做了优化：

```text
只保留最强锁模式；
重新按 invalid 入口计算；
避免把同一事务写成多个相互冲突成员。
```

### 12.5 committed updater 必须保留

如果旧 `xmax` 是已提交 updater，当前事务还想加锁。

系统不能清掉旧 updater。

因为旧版本已经被更新。

`compute_new_xmax_infomask()` 会创建 MultiXact，把旧 updater 和当前 locker 都保留下来。

这就是 MultiXact 可以包含 updater 的直接原因。

## 13. 成本、资源与跨模块传播

`xmax` 复用看似节省空间，但成本会扩散。

### 13.1 heap visibility hot path

每次 heap scan 都可能要解释 tuple header。

hot path 希望只看 infomask 和 snapshot。

如果 hint bit 足够，很多 CLOG 查询可以省掉。

如果需要进入 MultiXact 成员查询，成本会明显变高。

### 13.2 MultiXact SLRU

多个 locker 共享一行时，`xmax` 变成 MultiXactId。

这会访问：

- offsets SLRU。
- members SLRU。
- MultiXact local cache。
- WAL for MultiXact create。

行锁冲突多的 workload 可能把成本从 heap page 转移到 `pg_multixact`。

### 13.3 WAL 放大

纯锁也可能产生日志。

频繁 `SELECT FOR UPDATE`、外键检查或并发 update 会增加 WAL。

这不是因为锁本身要 crash recovery 成“仍然锁着”。

而是因为数据页记录了新的 XID/MultiXactId，必须防止崩溃后 ID 复用。

### 13.4 VACUUM 与 relminmxid

如果 tuple header 保留旧 MultiXact 引用，VACUUM 就要处理。

VACUUM 不只看 `relfrozenxid`。

它还要推进 `relminmxid`。

这会影响 autovacuum 触发、SLRU 截断和 wraparound 保护。

### 13.5 外键与 executor

外键检查依赖 key-share 类锁。

executor 的 `LockRows`、`ModifyTable`、RI trigger 都会把 SQL 语义转成 `LockTupleMode`。

因此 `xmax` 不是 access method 私有细节。

它连接：

- heap AM。
- executor。
- lock manager。
- RI trigger。
- VACUUM。
- WAL。
- MultiXact SLRU。

## 14. 观测与诊断入口

### 14.1 能直接看到的状态

可以直接看：

- `pg_locks` 中的 tuple lock wait。
- `pg_stat_activity.wait_event_type = 'Lock'`。
- `pg_stat_activity.wait_event` 中的 transactionid / tuple 相关等待。
- `pageinspect` 暴露的 `t_xmax`、`t_infomask`、`t_infomask2`。
- `pg_get_multixact_members()` 输出的 MultiXact 成员。
- `pg_get_multixact_stats()` 输出的 MultiXact 使用量。

### 14.2 只能推断的状态

很多语义不能直接从一个视图得出。

例如：

- `xmax` 是 locker 还是 updater。
- key 是否被更新。
- 当前等待是 tuple lock 排队还是 XID wait。
- MultiXact 中是否还有 live member。
- hint bit 是否已经刷新。

这些需要结合 tuple header、事务状态、SQL 行为和源码分支推断。

### 14.3 诊断问题的顺序

遇到“为什么这一行被阻塞”时，按这个顺序查：

```text
1. pg_stat_activity 看谁在等。
2. pg_locks 看 locktype 是 transactionid、tuple 还是 relation。
3. 确认 SQL 是 FOR UPDATE、UPDATE、DELETE、FK 检查还是触发器。
4. 用 pageinspect 看目标 tuple 的 xmax 和 infomask。
5. 如果是 MultiXact，查询 pg_get_multixact_members()。
6. 回到 heap_lock_tuple() 的冲突分支解释等待或跳过。
```

不要直接从 `xmax` 数字判断谁删了这一行。

## 15. 常见误区

误区一：

```text
xmax 有值，所以 tuple 已删除。
```

错误。

纯行锁也会写 `xmax`。

必须看 `HEAP_XMAX_LOCK_ONLY`。

误区二：

```text
MultiXact 只表示多个共享锁。
```

错误。

当前 MultiXact 可以包含 updater。

并且每个成员有自己的 `MultiXactStatus`。

误区三：

```text
FOR NO KEY UPDATE 和 FOR UPDATE 都是 exclusive，所以没区别。
```

错误。

两者都可能设置 exclusive 类锁位。

区别要结合 `HEAP_KEYS_UPDATED` 和 `MultiXactStatus`。

误区四：

```text
行锁都保存在 pg_locks，所以 tuple header 不重要。
```

错误。

lock manager 只提供等待排队层。

长期记录在 tuple header 或 MultiXact 里。

误区五：

```text
等待事务结束后就可以直接写 tuple。
```

错误。

等待后必须重新拿 buffer lock 并重查 `xmax`。

## 16. 课堂实验

### 实验一：观察纯行锁写入 `xmax`

目标：

```text
证明 xmax 有值不等于删除。
```

步骤：

```sql
DROP TABLE IF EXISTS xmax_lock_demo;
CREATE TABLE xmax_lock_demo(id int primary key, payload text);
INSERT INTO xmax_lock_demo VALUES (1, 'a');

-- Session A
BEGIN;
SELECT * FROM xmax_lock_demo WHERE id = 1 FOR SHARE;

-- Session B
SELECT * FROM xmax_lock_demo WHERE id = 1;
SELECT lp, t_xmin, t_xmax, t_ctid, t_infomask, t_infomask2
FROM heap_page_items(get_raw_page('xmax_lock_demo', 0));
```

观察重点：

- SELECT 仍能看到该行。
- `t_xmax` 可能已经有值。
- 解释必须依赖 infomask。

源码回读：

```text
nodeLockRows.c
  -> table_tuple_lock()
  -> heap_lock_tuple()
  -> compute_new_xmax_infomask()
```

### 实验二：比较 key update 和 no-key update

目标：

```text
理解 HEAP_KEYS_UPDATED 为什么存在。
```

步骤：

```sql
DROP TABLE IF EXISTS xmax_parent;
CREATE TABLE xmax_parent(id int primary key, payload text);
INSERT INTO xmax_parent VALUES (1, 'a');

-- Session A
BEGIN;
SELECT * FROM xmax_parent WHERE id = 1 FOR KEY SHARE;

-- Session B
UPDATE xmax_parent SET payload = 'b' WHERE id = 1;
UPDATE xmax_parent SET id = 2 WHERE id = 1;
```

观察重点：

- 非 key 列更新与 key-share 的冲突边界不同。
- key 更新必须保护引用完整性。
- `HEAP_KEYS_UPDATED` 是区分边界的一部分。

### 实验三：让多个 locker 形成 MultiXact

目标：

```text
看到单个 xmax 不够时，系统如何转向 MultiXact。
```

步骤：

```sql
DROP TABLE IF EXISTS xmax_multi_demo;
CREATE TABLE xmax_multi_demo(id int primary key, payload text);
INSERT INTO xmax_multi_demo VALUES (1, 'a');

-- Session A
BEGIN;
SELECT * FROM xmax_multi_demo WHERE id = 1 FOR SHARE;

-- Session B
BEGIN;
SELECT * FROM xmax_multi_demo WHERE id = 1 FOR SHARE;

-- Session C
SELECT lp, t_xmax, t_infomask, t_infomask2
FROM heap_page_items(get_raw_page('xmax_multi_demo', 0));
```

如果 `t_xmax` 是 MultiXactId，可进一步：

```sql
SELECT * FROM pg_get_multixact_members('<t_xmax>'::xid);
```

不同版本中参数类型展示可能略有差异。

以本地 `pg_proc.dat` 和 `\df pg_get_multixact_members` 为准。

### 实验四：断点跟读

建议断点：

```text
heap_lock_tuple
compute_new_xmax_infomask
MultiXactIdCreate
MultiXactIdExpand
HeapTupleSatisfiesUpdate
```

观察变量：

```text
mode
is_update
old_infomask
old_infomask2
new_infomask
new_infomask2
xmax
new_xmax
```

要画出的状态图：

```text
invalid xmax
  -> single locker
  -> second locker
  -> MultiXact
  -> updater joins or waits
  -> VACUUM / freeze 清理旧引用
```

## 17. 讨论题

1. 为什么 PostgreSQL 不把 delete XID、update XID 和 lock XID 分成三个 tuple header 字段？

2. `HEAP_XMAX_LOCK_ONLY` 能否单独证明 tuple 对当前 snapshot 可见？为什么？

3. 为什么 `HEAP_XMAX_EXCL_LOCK` 还不足以表达 `FOR UPDATE`？

4. 一个 committed locker 和一个 committed updater 对 tuple header 的长期意义有什么不同？

5. 为什么 `MultiXactIdExpand()` 必须创建新 MultiXact，而不是原地追加成员？

6. 为什么行锁写 tuple header 后仍需要 WAL？

7. 等待 `XactLockTableWait()` 返回后，为什么必须重新检查 `xmax`？

8. 诊断行锁等待时，`pg_locks`、`pageinspect` 和 `pg_get_multixact_members()` 各自能看到什么，不能看到什么？

## 18. 本节小结

本节只回答一个问题：

```text
xmax 为什么有多重语义，以及 PostgreSQL 如何避免这些语义互相混淆？
```

核心链路是：

```text
SQL row lock / UPDATE / DELETE
  -> table AM
  -> heap_lock_tuple() / heap_update() / heap_delete()
  -> HeapTupleSatisfiesUpdate()
  -> compute_new_xmax_infomask()
  -> 写 xmax + infomask + infomask2
  -> WAL / wait / visibility / VACUUM 后续解释
```

核心状态是：

- raw `xmax`。
- `HEAP_XMAX_INVALID`。
- `HEAP_XMAX_LOCK_ONLY`。
- `HEAP_XMAX_IS_MULTI`。
- `HEAP_XMAX_KEYSHR_LOCK`。
- `HEAP_XMAX_SHR_LOCK`。
- `HEAP_XMAX_EXCL_LOCK`。
- `HEAP_KEYS_UPDATED`。
- `t_ctid`。
- XID / MultiXact 成员状态。

ownership 和 cleanup 的边界是：

```text
事务结束释放 lock manager 中的事务锁；
tuple header 里的 lock-only 痕迹可以稍后被 hint bit、VACUUM 或 freeze 清理；
updater / deleter xmax 必须保留到版本链和 cleanup horizon 安全；
MultiXact SLRU 只有在所有 relation 的 relminmxid / datminmxid 推进后才能截断。
```

错误路径的核心规律是：

```text
等待不是证明；
等待后重查共享状态才是并发正确性的边界。
```

可观测入口包括：

- `pg_stat_activity`。
- `pg_locks`。
- `pageinspect`。
- `pg_get_multixact_members()`。
- `pg_get_multixact_stats()`。
- gdb 断点。

但这些入口都不能单独给出完整语义。

必须回到源码中的状态组合。

本节的可迁移模型是：

```text
内核热路径常用一个小字段承载多种语义；
正确理解方式不是问“这个字段等于什么”，
而是问“这个字段在当前 flag、生命周期、事务状态和调用目标下被解释为什么”。
```
