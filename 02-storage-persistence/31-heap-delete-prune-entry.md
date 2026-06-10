# PostgreSQL Heap delete、tuple lock 与 page pruning 入口
## 课程定位
本节主题：Heap delete、tuple lock 与 page pruning 入口。
上一节已经把 heap page layout、line pointer、tuple header、`xmin`、`xmax`、`infomask` 和 `t_ctid` 的基本边界建立起来。
本节把这些字段放进一个真实运行链路：
一条 SQL `DELETE` 到底在 heap page 上改了什么，为什么它不立即释放 tuple 空间，以及后续 page pruning 从哪里进入。
前置知识：
- 已理解 heap page 是 slotted page。
- 已理解 buffer pin/content lock、WAL-before-data、page LSN 和 visibility map 的基本角色。
- 已理解 MVCC snapshot 不是并发互斥机制。
- 已知道 `xmax` 不能脱离 `infomask` 和事务状态解释。
本节唯一主问题：
`DELETE` 已经把 tuple 标记为删除后，为什么 heap page 不能马上复用这块空间，PostgreSQL 又如何用 tuple lock、MultiXact 和 pruning 入口把并发正确性与空间回收连接起来？
本节围绕的核心矛盾：
前台 DELETE 需要尽快完成，并且要支持大量行级锁。
读事务、索引 TID、HOT chain、logical decoding、standby replay 和事务回滚又要求旧 tuple version 在一段时间内仍然可解释。
如果 DELETE 立即把 tuple bytes 从 page 上拿走，旧 snapshot、索引入口和并发等待者会失去共同的状态锚点。
如果永远不拿走，heap page 会膨胀，UPDATE 会越来越难在同页放下新版本。
PostgreSQL 的做法是把这件事拆成两个阶段：
DELETE 在 tuple header 中记录删除者和锁语义。
pruning 在 visibility horizon 允许时，才改 line pointer 状态并修复 page fragmentation。
读完本节，你应该能独立判断：
- `heap_delete()` 何时只设置 `xmax`，何时可能写入 MultiXactId。
- `HEAP_XMAX_LOCK_ONLY`、`HEAP_XMAX_IS_MULTI`、`HEAP_KEYS_UPDATED` 对 DELETE、UPDATE 和 tuple lock 的区别是什么。
- 为什么 `SELECT FOR UPDATE`、`DELETE` 和外键检查都复用 tuple header 中的 `xmax` 空间。
- 为什么 tuple lock 不是为每个锁住的行都在 lock table 里放一个长期对象。
- 为什么 `DELETE` commit 后 tuple 仍可能是 `LP_NORMAL`。
- `pd_prune_xid` 是什么 hint，为什么不是“这个 page 必须马上 prune”的命令。
- `heap_page_prune_opt()` 为什么经常被调用，但必须很快返回。
- on-access pruning、VACUUM pruning 和 recovery replay 的边界有什么不同。
- visibility horizon 如何决定 `HEAPTUPLE_RECENTLY_DEAD` 何时变成 `HEAPTUPLE_DEAD`。
- 哪些状态能通过 `pageinspect`、`pg_locks`、`pg_stat_activity` 和 VACUUM 日志看到，哪些只能推断。
## 源码基线
源码仓库：
```text
/home/nail/postgres-lab
```
基线：
```text
branch: master
commit: bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8
```
本节重点阅读：
```text
src/backend/access/heap/heapam.c
src/backend/access/heap/heapam_visibility.c
src/backend/access/heap/pruneheap.c
src/backend/access/heap/README.tuplock
src/include/access/htup_details.h
```
辅助定位：
```text
src/include/storage/bufpage.h
src/backend/access/heap/heapam_handler.c
src/backend/access/heap/heapam_indexscan.c
src/backend/storage/ipc/procarray.c
```
行号来自：
```text
nl -ba <source-file>
```
本节使用的主要源码入口：
| 入口 | 文件 | 角色 |
| --- | --- | --- |
| `heap_delete()` | `heapam.c:2716` | DELETE 的 heap AM 主入口，写 `xmax`、`infomask`、WAL 和 pruning hint |
| `heap_lock_tuple()` | `heapam.c:4538` | `SELECT FOR UPDATE/SHARE` 等 tuple lock 的主入口 |
| `compute_new_xmax_infomask()` | `heapam.c:5289` | 把旧 `xmax` 状态、新锁/更新请求合并成新的 `xmax` 与 infomask |
| `HeapTupleSatisfiesUpdate()` | `heapam_visibility.c:510` | UPDATE/DELETE/lock 前判断 tuple 是否可修改 |
| `HeapTupleSatisfiesVacuumHorizon()` | `heapam_visibility.c:1147` | 判断 tuple 对 pruning/VACUUM 是否 dead、recently dead 或 live |
| `heap_page_prune_opt()` | `pruneheap.c:270` | 普通访问路径上的 opportunistic pruning 入口 |
| `heap_page_prune_and_freeze()` | `pruneheap.c:1090` | 规划并执行 pruning、freeze、VM 更新的核心函数 |
| `heap_prune_chain()` | `pruneheap.c:1482` | 判断 HOT chain 中哪些 line pointer redirect、dead、unused |
| `heap_page_prune_execute()` | `pruneheap.c:2064` | 真正改 line pointer 并 repair fragmentation |
| `PageSetPrunable()` | `bufpage.h:479` | 设置 `pd_prune_xid` hint |
本节不展开：
- lazy VACUUM 的完整 heap 扫描和 index cleanup。
- HOT update 的所有索引维护条件。
- freeze 的全部 relfrozenxid 和 relminmxid 推进规则。
- logical decoding 如何解析 delete WAL。
- standby snapshot conflict 的完整处理。
这些内容都与本节相邻，但本节只取它们和 `DELETE -> pruning` 主链路的接口边界。
## 1. 先给结论
`DELETE` 不是“从 heap page 删除一段 bytes”。
`DELETE` 首先是一次 tuple header 状态变更。
在最简单的场景下，`heap_delete()` 把当前事务 XID 写入 tuple 的 `t_xmax`。
同时清理旧的 `HEAP_XMAX_BITS`，设置新的 infomask 组合。
对 DELETE 来说，`compute_new_xmax_infomask()` 以 `is_update = true`、`mode = LockTupleExclusive` 调用。
如果旧 `xmax` 无效，新 `xmax` 通常就是当前事务 XID。
`HEAP_KEYS_UPDATED` 会被设置在 `t_infomask2` 中。
`HEAP_XMAX_LOCK_ONLY` 不会表示这次 DELETE。
因为 DELETE 不是单纯锁，它会在提交后让这个 tuple version 对后续 snapshot 不可见。
如果 tuple 上已有并发 locker，`xmax` 可能被替换成 MultiXactId。
这时 `HEAP_XMAX_IS_MULTI` 表示 `t_xmax` 不是单个 TransactionId，而是一个 MultiXactId。
MultiXact 成员里可能既有 locker，也有 updater/deleter。
所以看到 `HEAP_XMAX_IS_MULTI` 不能推断“只是共享锁”。
`README.tuplock:87-90` 明确说，现代 MultiXact 可能包含 update 或 delete XID。
`DELETE` 还会把 `t_ctid` 设回自身。
这是为了避免删除的 tuple 留下 forward chain link。
UPDATE 旧版本的 `t_ctid` 会指向新版本。
DELETE 没有新版本，所以删除后的 tuple version 应该是链尾。
然后 `heap_delete()` 设置 `pd_prune_xid`。
`PageSetPrunable(page, xid)` 不是回收空间。
它只是说：当这个 XID 落到可移除 horizon 之后，这个 page 可能值得 pruning。
如果事务最终 abort，后续 pruning 会发现 tuple 仍然 live 或 xmax invalid，然后清掉过时 hint。
在 commit 之前，tuple 不能被当作 dead。
在 commit 之后，也不一定马上能移除。
旧 snapshot 可能仍能看到被删前的版本。
索引仍可能持有指向这个 line pointer 的 TID。
HOT chain 可能需要 redirect root。
standby 可能需要用 WAL 里的 conflict horizon 处理查询冲突。
因此，DELETE 完成时，line pointer 通常仍是 `LP_NORMAL`。
tuple bytes 仍在 page 上。
它只是带着新的 `xmax` 和 infomask，等待某个访问路径或 VACUUM 在 horizon 允许后 prune。
完整运行模型可以压缩成一句话：
```text
DELETE 写 tuple header 和 WAL, tuple lock/MultiXact 解决并发等待和锁语义, pruning 之后才在 visibility horizon 允许时改 line pointer 并回收 page 内空间。
```
这个模型有三个层次。
第一层是 tuple header：
`xmin`、`xmax`、`infomask`、`infomask2`、`cmax`、`t_ctid`。
它回答“这个版本由谁插入、被谁删除或锁住、是否还有新版本”。
第二层是 line pointer：
`LP_NORMAL`、`LP_REDIRECT`、`LP_DEAD`、`LP_UNUSED`。
它回答“索引 TID 和 page 内 tuple bytes 之间的锚点还在不在”。
第三层是 horizon：
`OldestXmin`、`GlobalVisState`、replication slot xmin、backend xmin、hot standby feedback。
它回答“这个旧版本是否已经对所有相关观察者不可见”。
只理解第一层，会把 `xmax` 误读成空间回收。
只理解第二层，会把 `LP_DEAD` 误读成 tuple header 删除。
只理解第三层，会把 pruning 误读成 VACUUM 的独占职责。
本节要把三层串起来。
## 2. `xmax` 不是一个语义，`xmax + flags + horizon` 才是语义
`HeapTupleHeaderData` 定义在 `htup_details.h:153-181`。
本节关注这些字段：
```text
t_choice.t_heap.t_xmin
t_choice.t_heap.t_xmax
t_choice.t_heap.t_field3
t_ctid
t_infomask2
t_infomask
t_hoff
```
`t_xmax` 的注释是 deleting or locking xact ID。
这句话本身就说明它不是“删除事务 ID”。
它可能表示删除者。
它可能表示 updater。
它可能表示 tuple locker。
它还可能表示 MultiXactId，而不是 TransactionId。
`htup_details.h:194-210` 定义了和 `xmax` 相关的关键 bit：
```text
HEAP_XMAX_KEYSHR_LOCK
HEAP_XMAX_EXCL_LOCK
HEAP_XMAX_LOCK_ONLY
HEAP_XMAX_COMMITTED
HEAP_XMAX_INVALID
HEAP_XMAX_IS_MULTI
HEAP_UPDATED
```
`htup_details.h:293-296` 又把 `HEAP_KEYS_UPDATED`、`HEAP_HOT_UPDATED`、`HEAP_ONLY_TUPLE` 放在 `t_infomask2`。
因此判断一个 tuple 是否被 DELETE 删除，至少要问：
- `xmax` 是有效 XID 还是 MultiXactId？
- `HEAP_XMAX_INVALID` 是否已说明这个 `xmax` 无效？
- `HEAP_XMAX_LOCK_ONLY` 是否说明它只是锁？
- 如果是 MultiXact，里面是否有 updater/deleter 成员？
- 如果 updater/deleter 成员提交了，它的 XID 是否已经落在 removable horizon 之前？
- `t_ctid` 是否仍指向自己，还是指向后继版本？
- 当前调用者要的是 MVCC 可见性、UPDATE 冲突判断，还是 VACUUM/pruning 判断？
单独看 `xmax` 有值，最多只能说这个 tuple header 记录过某种删除、更新或锁事件。
它不能说明 tuple 已经不可见。
它不能说明空间可以回收。
它也不能说明 page pruning 一定会发生。
`HEAP_XMAX_IS_LOCKED_ONLY()` 在 `htup_details.h:229-234`。
它不仅检查 `HEAP_XMAX_LOCK_ONLY`。
为了兼容旧版本，它还把非 MultiXact 的 `HEAP_XMAX_EXCL_LOCK` 某些组合视为 only locked。
这说明 infomask 的语义有历史包袱。
课程里不能把 bit 名字直接当成完整语义。
`HeapTupleHeaderGetUpdateXid()` 在 `htup_details.h:387-395`。
如果 `xmax` 是 MultiXact 且不是 lock-only，它会解析 MultiXact，找到真正的 updater XID。
注释提醒这可能涉及 multixact I/O。
这也是为什么热路径里会先用 infomask 做快速判断。
能不解析 MultiXact，就不要解析。
`HEAP_KEYS_UPDATED` 在 DELETE 中很重要。
`README.tuplock:143-147` 说它表示 XMAX 对 tuple key 有破坏性影响，例如 `SELECT FOR UPDATE`、修改 key 的 UPDATE 或 DELETE。
DELETE 会让 key 消失，因此按最强语义处理。
这会影响外键检查、tuple lock 冲突和 HOT chain 后续判断。
`t_ctid` 对 DELETE 也有边界。
UPDATE 的旧版本会通过 `t_ctid` 指向新版本。
DELETE 没有后继版本。
`heap_delete()` 在 `heapam.c:3013-3014` 明确把 `t_ctid` 设成 `t_self`。
后续 `HeapTupleSatisfiesUpdate()` 会用 `t_ctid` 是否等于 `t_self` 区分 committed UPDATE 和 committed DELETE。
在 `heapam_visibility.c:623-630`，如果 `HEAP_XMAX_COMMITTED` 且不是 lock-only：
`t_ctid != t_self` 返回 `TM_Updated`。
否则返回 `TM_Deleted`。
这就是字段组合形成语义的例子。
## 3. tuple lock 的两级机制
`README.tuplock` 是理解 DELETE 的必要阅读材料。
它开头说，tuple locking 不能像表锁那样简单。
原因是一个事务可能锁住大量 tuple。
如果每一行都在 shared memory lock table 中放长期 lock object，lock table 容量和争用都会成为硬瓶颈。
PostgreSQL 使用两级机制。
第一级是 tuple header。
把 locker 的 XID 写进 tuple `xmax`，再用 infomask 区分锁强度和 delete/update。
这一级可以承载大量被锁 tuple。
因为状态分散在 heap page 上，而不是集中在 lock table 中。
第二级是 standard lock manager 的 tuple lock。
它只在需要等待时提供排队和公平性。
`README.tuplock:24-29` 把等待协议写成：
```text
LockTuple()
XactLockTableWait()
mark tuple as locked by me
UnlockTuple()
```
这不是说锁语义只在 lock table 中。
真正长期可见的行锁状态仍在 tuple header。
lock manager 负责“谁排在下一位”。
`README.tuplock:31-37` 强调 at most one tuple-level lock will be held or awaited per backend。
这就是可扩展性的关键。
如果没有第二级 lock manager，所有 waiters 会同时从 `XactLockTableWait()` 或 `MultiXactIdWait()` 被释放。
之后谁先抢到 buffer content lock、谁先写 tuple header，存在竞态。
一个持续到来的 shared locker 流可能让 exclusive locker 长期饥饿。
因此 tuple lock 的 lock table 部分主要是排队语义，不是行锁状态的主存储。
PostgreSQL 提供四种 tuple lock strength：
```text
LockTupleExclusive       SELECT FOR UPDATE, DELETE, key-changing UPDATE
LockTupleNoKeyExclusive  UPDATE without key change
LockTupleShare           SELECT FOR SHARE
LockTupleKeyShare        SELECT FOR KEY SHARE, RI check
```
DELETE 使用 `LockTupleExclusive` 语义。
这会和所有其它 tuple lock mode 冲突。
外键检查常用 KeyShare，因为它只需要防止被引用 key 消失。
不改 key 的 UPDATE 可以和 KeyShare 共存。
这就是 `HEAP_KEYS_UPDATED` 为什么放在 `t_infomask2` 里参与冲突判断。
当只有一个 locker 时，`xmax` 可以直接存它的 XID。
当需要同时记录多个 locker，或者要保留已有 locker 再加入 updater/deleter 时，就需要 MultiXact。
`README.tuplock:79-85` 说 tuple header 只有一个 XID 和少量 bit，MultiXact 扩展为 XID 数组加成员 flag。
成员 flag 记录每个成员的锁强度，并区分 pure locker 和 updater。
因此 MultiXact 是把“多个事务对一个 tuple 的锁/更新语义”压缩成一个 `xmax` 值。
它不是一个独立于 tuple header 的锁表替代品。
它是 tuple header 可存储空间不足时的外部扩展。
MultiXact 的长期成本在 `pg_multixact` SLRU。
`README.tuplock:101-113` 说明，如果 MultiXact 包含 update，就必须跨 crash 持久化。
未来读者需要判断 update commit 还是 abort。
VACUUM 负责在 freezing 时移除老 MultiXact。
所以大量行锁和外键检查可能不是只带来 lock wait，还可能带来 MultiXact ID 消耗和 SLRU 压力。
`heap_lock_tuple()` 是 tuple lock 的主入口。
它先读 buffer 并拿 exclusive content lock。
然后调用 `HeapTupleSatisfiesUpdate()` 判断 tuple 当前是否能被锁。
如果发现 `TM_BeingModified`、`TM_Updated` 或 `TM_Deleted`，它会复制 `xmax`、`infomask`、`infomask2`、`t_ctid`，释放 buffer lock，再等待对应事务或 MultiXact。
释放 buffer lock 前复制状态很重要。
等待不能持有 buffer content lock。
但等待回来以后，tuple 可能已经被其它事务改过。
所以 `heap_lock_tuple()` 会重新拿 buffer lock，并比较 `xmax` 和 infomask 是否变化。
如果变化，就 `goto l3` 从头判断。
这个 retry loop 是 tuple header 并发协议的核心。
`heap_delete()` 也有类似的 `l1` retry。
## 4. `heap_delete()` 主流程 walkthrough
从 SQL 层看，`DELETE FROM t WHERE id = 1` 似乎只是删除一行。
从 heap AM 看，它是一次带并发检查、visibility check、WAL、VM 处理、toast cleanup 和 cache invalidation 的状态转换。
`heap_delete()` 入口在 `heapam.c:2716`。
它接收 relation、目标 TID、command id、选项、crosscheck snapshot、是否等待和失败返回结构。
第一步是获取当前事务 XID。
`heapam.c:2722` 调用 `GetCurrentTransactionId()`。
这意味着 DELETE 必然分配 XID。
没有 XID，就没有可以写入 tuple header 的删除者身份。
第二步是定位 heap page 和 line pointer。
代码读取目标 block，拿到 page，再在 `heapam.c:2767` 获取 buffer exclusive lock。
随后通过 offset number 找到 `ItemId`，并把 page 上 tuple bytes 包装成 `HeapTupleData`。
这里的锁是 buffer content lock。
它保证当前 backend 修改 page bytes 时不会被其它 backend 同时修改。
它不保证事务级可见性。
事务级判断在下一步。
第三步是 `HeapTupleSatisfiesUpdate()`。
`heapam.c:2792` 调用它，得到 `TM_Result`。
这个结果不是普通 MVCC visible boolean。
它告诉 UPDATE/DELETE/lock 调用者，是否可以继续修改这个 tuple。
可能结果包括：
```text
TM_Invisible
TM_Ok
TM_SelfModified
TM_Updated
TM_Deleted
TM_BeingModified
```
`TM_Invisible` 在 DELETE 中是 ERROR。
`heapam.c:2794-2800` 报错 attempted to delete invisible tuple。
这通常表示执行路径试图删除当前 command 不该看到的 tuple。
`TM_BeingModified` 且允许等待时，DELETE 进入等待路径。
它复制当前 `xwait = raw xmax` 和 `infomask`。
如果 `HEAP_XMAX_IS_MULTI`，就用 `DoesMultiXactIdConflict()` 判断 MultiXact 是否和 DELETE 的 exclusive lock 冲突。
如果冲突，先释放 buffer lock。
如果当前事务不是 MultiXact 成员，先通过 `heap_acquire_tuplock()` 获取 heavyweight tuple lock，用于排队。
然后调用 `MultiXactIdWait()` 等待。
等待结束后重新拿 buffer lock。
如果 `xmax` 或 infomask 变了，或者 page all-visible 状态需要 VM pin，跳回 `l1`。
如果不是 MultiXact，而 `xwait` 不是当前事务，就释放 buffer lock，获取 tuple lock，调用 `XactLockTableWait()` 等待普通事务结束。
等待回来后同样重拿 buffer lock，并检查 `xmax` 和 infomask 是否仍是刚才看到的状态。
这个重新检查不能省。
因为等待期间目标 tuple 可能被另一个事务锁住、更新、删除或清理 hint bit。
第四步是确认等待后的状态是否可覆盖。
如果旧 `xmax` aborted，或者只是 lock-only，DELETE 可以覆盖它。
如果旧 `xmax` committed 且 `t_ctid` 指向别处，结果是 `TM_Updated`。
如果旧 `xmax` committed 且 `t_ctid` 仍指向自己，结果是 `TM_Deleted`。
这段逻辑在 `heapam.c:2901-2912`。
它再次证明 `xmax` 本身不能判断 DELETE。
第五步是 crosscheck snapshot。
`heapam.c:2927-2932` 在 `crosscheck != InvalidSnapshot` 时调用 `HeapTupleSatisfiesVisibility()`。
这是 referential integrity 等路径需要的额外校验。
目标 tuple 对当前更新逻辑可修改，不代表它对 crosscheck snapshot 仍满足条件。
第六步是失败返回。
如果结果不是 `TM_Ok`，`heap_delete()` 填充 `tmfd`。
它记录 `ctid`、更新者 XID 和可能的 `cmax`。
随后释放 buffer、tuple lock 和 VM buffer。
调用者据此决定报并发更新错误、重新 EvalPlanQual，或者返回 would block。
第七步是 serializable conflict 检查。
`heapam.c:2959` 调用 `CheckForSerializableConflictIn()`。
这一步在真正修改 tuple header 前完成。
注释说明，之后会连续持有 exclusive buffer content lock，直到 delete 对 scan 可见。
第八步是处理 combo CID。
DELETE 需要在 tuple header 中记录 `cmax`。
如果同一事务内对同一 tuple 既有 insert command id 又有 delete command id，可能需要 combo CID。
`HeapTupleHeaderAdjustCmax()` 在 `heapam.c:2962` 处理这个问题。
这也是为什么 `heap_delete()` 禁止 parallel mode。
`heapam.c:2744-2752` 说明 parallel worker 没有机制广播 combo CID。
第九步是在 critical section 前完成可能分配内存的工作。
`ExtractReplicaIdentity()` 在 `heapam.c:2964-2969` 执行。
如果 logical decoding 需要 old tuple 或 old key，不能在 critical section 内分配失败。
第十步是设置 MultiXact oldest member。
`heapam.c:2971-2979` 调用 `MultiXactIdSetOldestMember()`。
即使最后只使用普通 XID，也要做。
因为其它 backend 之后可能把当前事务 XID 纳入 MultiXact。
这条 per-backend 边界用于 MultiXact 截断安全。
第十一步是计算新 `xmax` 和 infomask。
`heapam.c:2981-2984` 调用：
```text
compute_new_xmax_infomask(old_xmax, old_infomask, old_infomask2,
                          xid, LockTupleExclusive, true,
                          &new_xmax, &new_infomask, &new_infomask2)
```
最后一个 `true` 表示这不是纯锁，而是 update/delete 类操作。
对于 DELETE，mode 是 `LockTupleExclusive`。
第十二步进入 critical section。
从这里到 WAL 记录完成，不能随意 ERROR。
`heapam.c:2986` 后开始真实修改 page。
第十三步设置 pruning hint。
`heapam.c:2995` 调用 `PageSetPrunable(page, xid)`。
注释很直接：
如果事务提交，tuple 迟早会 DEAD。
当 xid 低于 OldestXmin horizon 时，这个 page 是 pruning candidate。
如果事务 abort，后续 pruning 会 no-op 并清掉 hint。
第十四步处理 visibility map。
如果 page 原来 all-visible，DELETE 必须清除 page flag 和 VM bit。
因为这个 page 现在出现了一个对所有事务不再简单 all-visible 的 tuple 状态。
`heapam.c:2997-3003` 清 `PD_ALL_VISIBLE` 并调用 `visibilitymap_clear()`。
注意它在拿 buffer lock 前可能提前 pin VM page。
如果没有 pin 而 page 变成 all-visible，代码会 unlock、pin、relock，再重试。
这是避免持有 buffer lock 做 I/O。
第十五步写 tuple header。
`heapam.c:3005-3014` 是本节核心。
它做了这些事：
```text
clear HEAP_XMAX_BITS and HEAP_MOVED
clear HEAP_KEYS_UPDATED
OR in new_infomask and new_infomask2
clear HOT-updated bit
set t_xmax = new_xmax
set cmax
set t_ctid = t_self
```
这里没有移动 tuple bytes。
没有把 line pointer 改成 `LP_DEAD`。
没有压缩 page。
只有 tuple header 改变。
第十六步处理 partition move。
如果这是跨分区 UPDATE 的 delete half，设置 moved partitions 标记。
这影响之后可见性和报错解释。
普通 DELETE 不需要这个状态。
第十七步 `MarkBufferDirty()`。
然后按 WAL-before-data 生成 `XLOG_HEAP_DELETE`。
WAL 记录包含 offnum、new xmax、infobits、是否清 all-visible、是否 partition move、是否包含 replica identity old tuple/key。
`heapam.c:3093-3095` 插入 WAL 并设置 page LSN。
只有 WAL 先于 data page 持久化，crash recovery 才能重放这个 tuple header 状态。
第十八步退出 critical section 并释放 buffer content lock。
这之后 page bytes 已经是删除标记状态。
但 buffer pin 仍持有，因为后续可能需要读 tuple 内容做 toast delete 和 cache invalidation。
第十九步处理 external toast。
`heapam.c:3105-3118` 说明，如果 tuple 有 out-of-line attributes，需要删除 toast entries。
这发生在释放 content lock 后，但释放 buffer pin 前。
第二十步做 cache invalidation。
`heapam.c:3120-3125` 在释放 buffer 前调用 `CacheInvalidateHeapTuple()`。
系统表 tuple 的删除要让相关 cache 在命令边界失效。
第二十一步释放 buffer 和 tuple lock。
如果曾经为等待排队获取 heavyweight tuple lock，`heapam.c:3131-3134` 释放它。
最后统计 `pgstat_count_heap_delete()`。
这个 walkthrough 的关键点是：
`heap_delete()` 完成后，page 上留下的是一个新的 tuple header 状态，而不是空洞回收后的 page。
空间回收属于后续 pruning/VACUUM。
## 5. `compute_new_xmax_infomask()` 如何合并 DELETE 和已有锁
`compute_new_xmax_infomask()` 是本节最容易被低估的函数。
它不是简单地返回当前事务 XID。
它必须把旧 `xmax` 状态、旧 infomask、新请求和 MultiXact 规则合并成一个可写入 tuple header 的状态。
函数注释在 `heapam.c:5272-5288`。
它明确说可能有副作用，例如创建新的 MultiXactId。
大多数调用者之前已经调用 `HeapTupleSatisfiesUpdate()`。
但状态仍可能在竞态中变化。
所以函数内部仍要处理 race，并且有 `goto l5` 重新归约到简单 case。
第一种 case：
旧 infomask 有 `HEAP_XMAX_INVALID`。
说明没有需要保留的旧 locker/updater。
如果 `is_update = true`，也就是 UPDATE/DELETE 类操作，新 `xmax = add_to_xmax`。
如果 mode 是 `LockTupleExclusive`，设置 `HEAP_KEYS_UPDATED`。
这就是普通 DELETE 的简单路径。
注意这里没有设置 `HEAP_XMAX_LOCK_ONLY`。
第二种 case：
旧 `xmax` 是 MultiXact。
如果是 pg_upgrade 旧格式 share-locked tuple，函数把它当作不再运行并转成 invalid。
如果 MultiXact 不再 running，而且里面要么只是 locker，要么 updater abort，也可以丢掉旧 MultiXact，重新走 invalid case。
否则，需要通过 `MultiXactIdExpand()` 把当前事务加入现有 MultiXact。
随后用 `GetMultiXactIdHintBits()` 从成员状态推导新 infomask。
第三种 case：
旧 infomask 已有 `HEAP_XMAX_COMMITTED`，并且表示 committed update。
这时旧 updater 必须被保留。
因为这个 tuple version 的历史已经包含一个 committed update/delete 语义。
函数创建新的 MultiXact，把旧 updater 和新请求都放进去。
这类路径说明 MultiXact 不只是“多个 locker”。
第四种 case：
旧 `xmax` 是仍在进行中的普通 TransactionId。
如果旧状态是 lock-only，函数把旧 XID 按锁强度转成 MultiXact member。
如果旧状态不是 lock-only，旧 XID 是 updater。
再把当前请求也加入 MultiXact。
如果旧 XID 和当前事务相同，有优化：
保留更强的锁语义，把旧状态标记 invalid 后回到简单 case。
这避免同一事务重复把自己加入 MultiXact。
第五种 case：
旧 `xmax` 不 lock-only 且已经 committed。
函数也要保留旧 updater，创建 MultiXact。
第六种 case：
旧事务在取 infomask 时还 running，但检查时已经结束。
函数把旧 infomask 视为 invalid，回到简单 case。
这就是注释所说 harmless race。
这几个 case 共同说明：
DELETE 改 `xmax` 不是单字段覆盖。
它是一次状态归并。
如果可以安全丢弃旧 locker，就丢弃。
如果必须保留旧 updater 或 surviving locker，就创建或扩展 MultiXact。
如果信息过旧或不确定，保守地重新归约，不破坏可见性。
这也是为什么写 DELETE 相关 patch 时不能绕过 `compute_new_xmax_infomask()`。
看起来只是“写当前 xid 到 xmax”的修改，很容易破坏 lock upgrade、MultiXact、pg_upgrade compatibility 或 key-share 语义。
## 6. 为什么 DELETE 不立即释放空间
DELETE 之后不立即释放空间，不是实现懒惰。
这是 MVCC、索引稳定性和 crash safety 的共同结果。
第一，旧 snapshot 仍可能需要看到旧版本。
一个事务在 DELETE commit 之前取得 snapshot。
DELETE commit 之后，这个事务按 MVCC 仍然可能看到被删前的 tuple。
如果 DELETE 在 commit 时就移除 tuple bytes，旧 snapshot 的 heap scan 或 index fetch 无法解释这个 TID。
第二，事务可能 abort。
PostgreSQL 没有物理 undo log 把 page bytes 回滚到旧状态。
DELETE 写入 tuple header 后，如果事务 abort，后续可见性检查会看到 `xmax` aborted，并把 `HEAP_XMAX_INVALID` hint 设上。
原 tuple bytes 还在，所以 abort 的 DELETE 可以自然变成“只是一个无效 xmax”。
如果 DELETE 立即移除 bytes，abort 就需要复杂 undo。
第三，索引 TID 仍指向 line pointer。
普通 btree index entry 指向 heap TID。
这个 TID 是 `(block, offset)`。
offset 指向 line pointer，不是直接指向 tuple bytes。
在 VACUUM 删除 index entry 前，heap 端不能把可能被 index 引用的 line pointer 随便变成可重用的 `LP_UNUSED`。
否则 index fetch 可能命中新 tuple。
第四，HOT chain 需要 root TID。
HOT update 允许 index 仍指向 root TID，再在 heap page 内沿链找到新版本。
pruning 可以把 root line pointer 改成 `LP_REDIRECT`。
这保留了索引入口到 heap-only successor 的桥。
如果直接清掉 root，HOT chain 就断了。
第五，logical decoding 和 replica identity 需要可解释的变更信息。
`heap_delete()` 在 WAL 中记录 delete 的 offnum、xmax、infobits 和必要 old tuple/key。
这不是空间回收信息。
它是语义变更信息。
空间回收由后续 prune WAL 或 vacuum index cleanup 处理。
第六，standby replay 需要冲突边界。
prune/freeze WAL 可能携带 snapshot conflict horizon。
standby 上旧查询是否要被取消，取决于被移除 tuple 的 XID horizon。
如果前台 DELETE 立即移除，回放端难以在正确边界处理旧 snapshot。
第七，前台 DELETE 的 latency 不能承担全量 cleanup。
DELETE 可以命中很多行。
如果每行都试图清索引、压缩 page、等待没有其它 pin、更新 FSM、处理 all-visible 和 freeze，前台写路径会变成 VACUUM。
PostgreSQL 把它拆开：
前台只做必要的 tuple header 状态变更。
访问路径 opportunistically prune。
VACUUM 做更完整的 dead item 和 index cleanup。
因此，DELETE commit 之后 page 上可能出现三种直观现象：
- tuple 仍是 `LP_NORMAL`，但 `xmax` 是 committed deleter。
- page free space 没有立刻增加。
- `n_dead_tup` 或 VACUUM VERBOSE 可能显示 dead tuple，但 `pageinspect` 仍能看到 tuple header。
这些不是 bug。
它们是延迟回收模型的正常表现。
## 7. DELETE 后 visibility 如何判断
`HeapTupleSatisfiesUpdate()` 是 UPDATE/DELETE/lock 前的判断。
`HeapTupleSatisfiesMVCC()` 是普通 MVCC scan 判断。
`HeapTupleSatisfiesVacuumHorizon()` 是 VACUUM/pruning 判断。
三者使用同一套 tuple header 状态，但回答的问题不同。
`HeapTupleSatisfiesUpdate()` 的注释在 `heapam_visibility.c:482-509`。
它需要比 visible boolean 更细的结果。
例如，tuple 被 committed UPDATE 修改和被 committed DELETE 删除，对调用者都不是 `TM_Ok`。
但前者返回 `TM_Updated`，后者返回 `TM_Deleted`。
当插入者已经 committed 后，如果 `HEAP_XMAX_INVALID`，返回 `TM_Ok`。
如果 `HEAP_XMAX_COMMITTED` 且不是 lock-only：
`t_ctid != t_self` 是 `TM_Updated`。
`t_ctid == t_self` 是 `TM_Deleted`。
如果 `xmax` 是 MultiXact：
先区分 lock-only。
如果只是 lock-only 且 MultiXact 仍 running，返回 `TM_BeingModified`。
如果不是 lock-only，要解析 update XID。
如果 update member committed，再用 `t_ctid` 区分 updated/deleted。
如果 update member aborted，但还有 locker running，仍可能返回 `TM_BeingModified`。
普通 reader 的 `HeapTupleSatisfiesMVCC()` 则关心当前 snapshot 是否应该看到这个 tuple version。
它会用 `XidInMVCCSnapshot()` 判断 inserting XID 和 deleting/updating XID 是否在 snapshot 内。
对 reader 来说，committed delete 可能使 tuple 不可见。
对 old snapshot 来说，同一个 tuple version 仍可能可见。
VACUUM/pruning 的问题更强：
这个 tuple 是否对任何可能相关的 snapshot 都不可见？
这就是 `HeapTupleSatisfiesVacuumHorizon()` 的职责。
它在 `heapam_visibility.c:1147`。
它把 tuple 分成：
```text
HEAPTUPLE_LIVE
HEAPTUPLE_DEAD
HEAPTUPLE_RECENTLY_DEAD
HEAPTUPLE_INSERT_IN_PROGRESS
HEAPTUPLE_DELETE_IN_PROGRESS
```
如果 inserting transaction aborted，tuple 从未对其它事务可见，可以是 `HEAPTUPLE_DEAD`。
如果 inserter committed 且 `xmax` invalid，tuple live。
如果 `xmax` lock-only，tuple live。
如果 deleting/updating XID still in progress，tuple delete in progress。
如果 deleting/updating XID committed，函数通常返回 `HEAPTUPLE_RECENTLY_DEAD`，并把 `dead_after` 设成删除者或 updater XID。
`HeapTupleSatisfiesVacuum()` 再用 `OldestXmin` 比较。
`heapam_visibility.c:1121-1127`：
如果 `dead_after` precedes `OldestXmin`，`RECENTLY_DEAD` 变成 `DEAD`。
否则仍是 recently dead。
pruning 使用类似逻辑，但通过 `GlobalVisState`。
`heap_prune_satisfies_vacuum()` 在 `pruneheap.c:1400-1433`。
它先调用 `HeapTupleSatisfiesVacuumHorizon()`。
如果结果不是 `RECENTLY_DEAD`，直接返回。
如果是 `RECENTLY_DEAD`，VACUUM 路径可先用 `cutoffs->OldestXmin` 判断。
然后用 `GlobalVisTestIsRemovableXid()` 判断是否对当前 relation 的 visibility horizon 可移除。
这说明 horizon 不是一个固定全局常量。
不同 relation kind 可能使用不同 horizon。
catalog、data、shared、temp relation 的 conservative boundary 不同。
## 8. visibility horizon 与 `GlobalVisState`
`GlobalVisTestFor()` 在 `procarray.c:4106`。
它根据 relation 选择对应的 global visibility horizon state。
可能是 shared、catalog、data 或 temp horizon。
这反映了 PostgreSQL 的实际约束：
catalog tuple 可能被 logical decoding 的 catalog xmin 保护。
data tuple 可能被 replication slot 的 xmin 或 backend xmin 保护。
shared relation 需要跨 database 考虑。
temp relation 可以更激进。
`GlobalVisState` 有两个边界：
`maybe_needed` 和 `definitely_needed`。
`GlobalVisTestIsRemovableFullXid()` 在 `procarray.c:4226`。
如果 XID 早于 `maybe_needed`，它肯定对所有人可见或者不再需要，可以移除。
如果 XID 晚于等于 `definitely_needed`，它很可能仍被某个 snapshot 认为 running，不能移除。
如果 XID 落在两者之间，函数可以根据 `allow_update` 决定是否重新计算 horizons。
这是一种性能和准确性的折中。
精确 horizon 需要扫描 ProcArray、slot 等状态。
普通 page access 不能每次都付出这个成本。
因此 pruning 使用一个保守测试：
能确认 removable 才移除。
不确定就留下。
这解释了一个常见现象：
某个 tuple 从人的角度看“已经没有事务能看到”，但一次 scan 没有 prune 掉。
原因可能只是当前 `GlobalVisState` 还没更新，或者 page 没满足 pruning heuristic，或者 cleanup lock 没拿到。
这不是 correctness 问题。
它只影响空间回收时机。
长事务会压住 horizon。
`pg_stat_activity.backend_xmin` 可以帮助观察。
只要某个 backend 持有很老的 snapshot，许多 deleted tuple 只能是 `RECENTLY_DEAD`。
replication slot 也会压住 horizon。
`pg_replication_slots.xmin` 和 `catalog_xmin` 是线上 bloat 分析必须看的字段。
hot standby feedback 也可能让 primary 不能移除 standby 查询仍可能需要的 tuple。
MultiXact 还多一个 horizon。
老 MultiXact 需要 relminmxid/datminmxid 保护。
如果 workload 里大量外键检查或 `SELECT FOR KEY SHARE`，即使 tuple 没被 DELETE，也可能积累 MultiXact 消耗。
本节不展开 MultiXact wraparound，但要记住：
tuple lock 的可扩展性不是免费午餐。
它把长期行锁状态从 lock table 转移到了 tuple header 和 pg_multixact。
## 9. page pruning 的入口
page pruning 不是只有 VACUUM 会做。
普通 heap scan、bitmap heap scan 和 index heap fetch 都可能 opportunistically prune。
`heapam.c:638` 在 heap page mode scan 中调用 `heap_page_prune_opt()`。
`heapam_handler.c:2568` 在 bitmap heap scan 中调用。
`heapam_indexscan.c:260` 在 index scan 第一次 pin 某个 heap buffer 时调用。
这些入口有一个共同点：
它们已经 pin 了 heap buffer。
它们即将读取 tuple visibility。
如果 page 正好有可回收的 dead tuple，顺手 prune 可以降低后续 page 访问成本。
但它们不能变成阻塞式 VACUUM。
因此 `heap_page_prune_opt()` 设计为 opportunistic。
`pruneheap.c:245-250` 说，它只在 page 看起来值得 pruning 且能非阻塞拿到 cleanup lock 时执行。
它被经常调用，所以必须很快返回。
具体过滤顺序如下。
第一，recovery 模式直接返回。
`pruneheap.c:279-285`：
standby 不能本地写 WAL。
所以没有必要尝试清理 page。
primary 之后会发出清理 WAL，standby 通过 replay 应用。
第二，检查 `pd_prune_xid`。
`pruneheap.c:293-295`：
如果 `PageGetPruneXid(page)` 无效，直接返回。
这避免了每次 scan 都计算 horizon。
`pd_prune_xid` 是 page header 中的最早可能 prunable XID hint。
`PageSetPrunable()` 在 `bufpage.h:479-485`：
只有当前 XID 比已有 `pd_prune_xid` 更早，才更新。
也就是说它记录“这个 page 上已知最早可能需要 prune 的 XID”。
第三，用 `GlobalVisTestIsRemovableXid()` 判断这个 hint 是否已经过 horizon。
`pruneheap.c:301-304`：
如果 hint 还不可移除，直接返回。
这一步仍只是基于 hint 的快速判断。
实际每个 tuple 是否 dead，要等 `heap_page_prune_and_freeze()` 里逐个检查。
第四，检查 free space heuristic。
`pruneheap.c:307-323`：
如果 page 被标记 `PD_PAGE_FULL`，或者 free space 低于 relation fillfactor target 和 10% BLCKSZ 的较大者，才继续。
这说明即使有 old dead tuple，普通访问也不一定马上 prune。
如果 page 空间压力不大，没必要让 scan 承担 cleanup 成本。
第五，非阻塞尝试 cleanup lock。
`pruneheap.c:324-326` 调用 `ConditionalLockBufferForCleanup()`。
拿不到就返回。
cleanup lock 比普通 exclusive content lock 更强。
它要求没有其它 backend 正在 pin 使用这个 buffer。
这保证 pruning 可以移动 tuple bytes、改变 line pointer，而不会破坏其它持 pin 访问者看到的 page 内容。
普通访问路径不等待 cleanup lock。
第六，拿到 lock 后重新检查 heuristic。
刚才读 free space 时没有 buffer lock，可能不准确。
`pruneheap.c:328-333` 在 lock 下重查。
如果仍值得 prune，设置 `PruneFreezeParams`，reason 是 `PRUNE_ON_ACCESS`。
第七，调用 `heap_page_prune_and_freeze()`。
on-access 路径传 `HEAP_PAGE_PRUNE_ALLOW_FAST_PATH`。
如果 relation scan 被认为 read-only，还可能传 `HEAP_PAGE_PRUNE_SET_VM`。
但它不会传 `HEAP_PAGE_PRUNE_MARK_UNUSED_NOW`。
`pruneheap.c:349-354` 说，on-access pruning 当前不能安全判断 relation 是否没有 indexes。
因此它不能把可能被 index 引用的 dead root 直接变成 `LP_UNUSED`。
这就是为什么 on-access prune 后仍可能留下 `LP_DEAD`。
第八，释放 buffer lock。
`pruneheap.c:381-388` 明确说，此处不更新 FSM。
避免把 unrelated UPDATE/INSERT 创建出的 free space 暴露给其它页面选择逻辑。
这些空间更希望被同 page 的 UPDATE 复用。
这个入口的 mental model：
```text
pd_prune_xid hint -> horizon quick check -> free-space pressure -> cleanup lock -> exact tuple visibility -> line pointer state changes
```
前面几步都是为了避免在热 scan 路径上付出过多成本。
## 10. `heap_page_prune_and_freeze()` 做什么
`heap_page_prune_and_freeze()` 是 pruning 的核心。
它不是一进来就改 page。
它先 plan，再进入 critical section 执行。
`pruneheap.c:1105-1108` 初始化 `PruneState`。
`PruneState` 保存 relation、buffer、page、block、vistest、cutoffs、visibility map bits、计划修改数组、统计结果和 conflict horizon。
`pruneheap.c:1110-1118` 先修复一种 VM/page hint 不一致：
VM bit set 但 page `PD_ALL_VISIBLE` clear。
在 pruning/freezing 前要保证这两个状态有一致起点。
`pruneheap.c:1120-1134` 有 fast path。
如果 page 已经 all-frozen，或者不尝试 freeze 且 page 已经 all-visible，可以跳过完整 pruning/freezing。
这符合“高频入口要快”的要求。
`pruneheap.c:1136-1142` 调用 `prune_freeze_plan()`。
它遍历 line pointer 和 tuple visibility。
注意这里只是准备计划。
具体要 redirect、dead、unused 的 offset 放进数组。
`pruneheap.c:1164-1166` 用计划数组判断是否真的要 prune：
```text
nredirected > 0 || ndead > 0 || nunused > 0
```
`pruneheap.c:1168-1174` 判断是否只需要更新 hint：
`pd_prune_xid` 是否变化，或者 `PD_PAGE_FULL` 是否需要清。
这意味着一次 pruning 调用可能不回收任何 tuple，只是清理 stale hint。
`pruneheap.c:1213-1223` 计算 WAL record 的 snapshot conflict horizon。
如果要设置 VM，使用 newest live xid。
如果要 freeze，使用 freeze conflict xid。
如果要 prune，使用 latest xid removed。
最终取最保守的 newest horizon。
这对 standby query conflict 很重要。
`pruneheap.c:1229-1230` 进入 critical section。
如果只是 hint prune，不需要 WAL，可以 `MarkBufferDirtyHint()`。
如果真的 prune、freeze 或设置 VM，则会 `MarkBufferDirty()` 并记录 WAL。
`pruneheap.c:1262-1267` 调用 `heap_page_prune_execute()` 执行 line pointer 改动。
`pruneheap.c:1270-1271` 执行 freeze。
`pruneheap.c:1273-1291` 更新 VM 和 `PD_ALL_VISIBLE`。
`pruneheap.c:1299-1310` 写 `XLOG_HEAP2_PRUNE*` WAL。
这个 WAL 和 DELETE WAL 是不同阶段的 WAL。
DELETE WAL 记录逻辑删除标记。
prune WAL 记录物理 page cleanup。
这两个阶段可以相隔很久。
`heap_prune_chain()` 决定 HOT chain 的 fate。
注释在 `pruneheap.c:1451-1480` 很关键。
如果 HOT chain 开头有 DEAD tuple，root line pointer 会 redirect 到第一个非 DEAD tuple。
如果整条 chain 都 DEAD，root line pointer 标成 `LP_DEAD`。
中间可以安全移除的 heap-only tuple 标成 `LP_UNUSED`。
它不会在 plan 阶段直接改 page。
它只是把结果记录到 `redirected[]`、`nowdead[]`、`nowunused[]`。
`heap_page_prune_execute()` 才改 page。
它先处理 redirects。
然后处理 now-dead。
再处理 now-unused。
最后调用 `PageRepairFragmentation()`。
这会压缩 page 内 tuple bytes，更新 free space 边界。
所以真正的空间增加通常在 `heap_page_prune_execute()` 之后。
`LP_DEAD` 和 `LP_UNUSED` 的区别是本节边界之一。
`LP_DEAD` 表示这个 line pointer 没有 live tuple storage，但 index 可能仍指向它。
VACUUM 还需要据此清理 index entry。
`LP_UNUSED` 表示 line pointer 可以被新 tuple 复用。
on-access pruning 通常不敢把 index-referenced root 直接变 `LP_UNUSED`。
VACUUM 在确认 index cleanup 边界后，才能进一步处理。
## 11. DELETE、UPDATE 和 pruning hint 的连接
`PageSetPrunable()` 不只在 DELETE 中出现。
`heap_insert()` 在 `heapam.c:2072-2085` 也可能设置它。
注释说，如果插入事务最终 abort，tuple 会变成 DEAD。
如果没有其它 UPDATE/DELETE，这个 aborted tuple 否则可能要等到下次 VACUUM 才被 prune。
所以 insert 也会留下 page 访问时可检查的 hint。
`heap_update()` 在 `heapam.c:4025-4027` 会给 old page 设置 prunable。
如果 new tuple 放在不同 page，也给 new page 设置 prunable。
因为 old tuple 将来可能 dead。
new tuple 也可能因为事务 abort 或 chain 状态需要被考虑。
`heap_update()` 如果新版本放不进 old page，会在 `heapam.c:3997` 调用 `PageSetFull(page)`。
`PD_PAGE_FULL` 是另一个 pruning hint。
它表示曾经因为空间不够没能在此页放下新 tuple version。
下次访问时如果 horizon 允许，pruning 值得尝试。
`bufpage.h:205-211` 明确说 `PD_HAS_FREE_LINES` 和 `PD_PAGE_FULL` 都是 hint。
它们不是 WAL 记录的强事实。
redo 比较 page 时甚至会忽略这些 hint。
所以课程里不能把 `PD_PAGE_FULL` 解释成“page 当前一定满”。
它只是让未来访问更愿意进入 prune 路径。
DELETE 设置的是 `pd_prune_xid`。
UPDATE 也设置。
INSERT 也可能设置。
`pd_prune_xid` 的含义不是“有 delete”。
它是“page 上可能有一个 XID，在它过 horizon 后值得检查 pruning/freezing/all-visible”。
这解释了为什么有时 `pd_prune_xid` 存在但 pruning 没有真正删除任何 tuple。
hint 可能来自 aborted insert。
也可能来自 aborted delete。
也可能在 horizon 还没过时被看到。
也可能 exact tuple visibility 检查发现没有可 prune 对象。
## 12. 错误路径与并发边界
本节的错误路径不是附录。
DELETE 和 pruning 的正确性很大一部分来自 non-happy path。
边界一：parallel mode 禁止 DELETE tuple。
`heap_delete()` 在 `heapam.c:2744-2752` 报错。
原因是 DELETE 可能分配 combo CID。
其它 parallel worker 可能需要这个 combo CID 做 visibility check，但没有广播机制。
这不是 SQL 层限制，而是 tuple header command id 解释的边界。
边界二：`TM_Invisible` 是 ERROR。
如果目标 tuple 对当前 command 不存在，DELETE 不能继续。
否则会把不该修改的 tuple 写上 `xmax`。
边界三：等待期间必须释放 buffer content lock。
无论等普通 XID 还是 MultiXact，都不能持有 page content lock 睡眠。
否则其它 backend 不能推进 tuple 状态，容易形成严重阻塞。
释放前必须复制 `xmax` 和 infomask。
回来后必须比较。
边界四：VM pin 的 unlock/relock。
如果 page 是 all-visible，需要 pin VM page 才能清 VM bit。
如果拿 heap buffer lock 后才发现 page 变成 all-visible，而 VM page 尚未 pin，代码要先释放 heap lock，pin VM，再重拿 heap lock并重试。
这避免持 buffer lock 做可能阻塞的 I/O。
边界五：`xmax` 或 infomask 变化必须 restart。
等待的事务结束后，目标 tuple 可能被其它事务锁过。
如果不 restart，就可能基于旧状态覆盖别人刚写的 lock/update 信息。
`heap_delete()` 的 `goto l1` 和 `heap_lock_tuple()` 的 `goto l3` 是这种并发协议的显式形态。
边界六：critical section 内不能有可恢复 ERROR。
`heap_delete()` 在进入 critical section 前先计算 replica identity。
因为那可能分配内存。
真正修改 tuple header 和 WAL 记录之间如果发生错误，系统不能留下 page 已改但 WAL 未写的状态。
边界七：abort 不物理回滚 tuple header。
如果 DELETE 事务 abort，tuple header 里可能仍有这个 `xmax`。
后续 visibility check 会查 pg_xact，发现 aborted，然后设置 `HEAP_XMAX_INVALID` hint。
pruning 看到这个 tuple 会把它当 live，而不是删除。
边界八：on-access pruning 拿不到 cleanup lock 就返回。
普通 scan 不应该为了清理空间阻塞等待其它 pin。
这是 latency 和 space reclaim 的取舍。
VACUUM 可以更主动，但也要遵守 pin、cleanup lock、index cleanup 和 horizon 边界。
边界九：recovery 中不做本地 opportunistic pruning。
`heap_page_prune_opt()` 在 recovery 直接 return。
standby 通过 WAL replay 应用 primary 的 prune 结果。
这保证物理 page 变更和 conflict horizon 由 WAL 驱动。
边界十：`LP_DEAD` 不是最终释放。
如果 index 仍可能指向这个 line pointer，不能直接设 `LP_UNUSED`。
否则旧 index entry 可能指向后来复用这个 offset 的新 tuple。
这个边界是 line pointer 层面对索引稳定性的保护。
## 13. 成本、资源与跨模块传播
DELETE 本身的 page 修改很小。
成本来自它触发和延迟的状态。
CPU 成本：
每次可见性检查都要解释 `xmin`、`xmax`、infomask 和 snapshot。
hint bit 不完整时，还要查 pg_xact。
如果 `xmax` 是 MultiXact，可能要查 pg_multixact。
这会把单 tuple 判断从纯内存 bit test 变成 SLRU 访问。
contention 成本：
多个 backend 删除或锁同一 tuple，会在 tuple header、MultiXact、transactionid wait 和 heavyweight tuple lock 上交互。
buffer content lock 保护 page bytes。
lock manager 负责等待队列。
ProcArray 和 CLOG/SLRU 提供事务状态。
这些层次各自有不同 contention 点。
WAL 成本：
DELETE 写 `XLOG_HEAP_DELETE`。
tuple lock 也可能写 `XLOG_HEAP_LOCK`。
注释在 `heapam.c:5168-5178` 说明，即使只是锁 tuple，也要 WAL。
因为写入的 TransactionId 或 MultiXactId 可能 crash 后被重用，必须遵守 WAL-before-data。
pruning 后续还会写 `XLOG_HEAP2_PRUNE*`。
同一逻辑行的一次删除可能导致多阶段 WAL。
空间成本：
DELETE 立刻制造 dead tuple version。
如果 horizon 被长事务或 slot 压住，dead tuple 不能移除。
表膨胀、index bloat、visibility map 失效和 cache locality 下降会扩散到查询性能。
MultiXact 成本：
外键检查和 `SELECT FOR KEY SHARE` 可以大量创建或扩展 MultiXact。
大量 MultiXact 会增加 SLRU miss、wraparound 管理和 VACUUM freeze 压力。
`relminmxid` 和 `datminmxid` 可能成为运维关注点。
ProcArray/horizon 成本：
pruning 是否能移除 tuple 依赖 global horizon。
horizon 计算涉及 backend xmin、replication slots、prepared transactions、hot standby feedback 等。
为了避免频繁精确扫描，`GlobalVisState` 使用近似边界和按需更新。
这降低热路径成本，但也让空间回收时机更滞后。
FSM/VM 传播：
DELETE 清 all-visible。
pruning 可能设置 all-visible/all-frozen。
on-access pruning 不主动更新 FSM。
VACUUM 才会更系统地更新 FSM、VM 和统计。
所以一个 page 的 DELETE 后状态会通过 VM、FSM、pg_stat、autovacuum threshold 和 planner visibility estimates 间接传播。
跨模块连接可以这样记：
```text
heapam.c          前台 DELETE/UPDATE/lock 写 tuple header
heapam_visibility 解释 tuple header 和事务状态
pruneheap.c       在 horizon 允许时改 line pointer 和压缩 page
procarray.c       提供 removable horizon
multixact.c       承载多事务 tuple lock/update 成员
vacuumlazy.c      更完整地 prune、freeze、清 index、更新 FSM/VM
wal/redo          持久化 DELETE 和 PRUNE 的物理顺序
```
## 14. 观测与诊断入口
最直接的观测工具是 `pageinspect`。
它能读 raw heap page，显示 line pointer、`t_xmin`、`t_xmax`、`t_ctid`、`t_infomask`、`t_infomask2`。
它看到的是 page 当前物理状态。
它不等于某个 MVCC snapshot 的可见性结果。
建议用这种查询看 tuple header：
```sql
CREATE EXTENSION IF NOT EXISTS pageinspect;

SELECT lp,
       lp_flags,
       t_xmin,
       t_xmax,
       t_ctid,
       to_hex(t_infomask) AS infomask,
       to_hex(t_infomask2) AS infomask2
FROM heap_page_items(get_raw_page('demo_delete', 0))
ORDER BY lp;
```
`lp_flags` 可以看到 line pointer 是否仍是 normal、redirect、dead 或 unused。
`t_xmax` 可以看到 DELETE 或 lock 写入的 raw value。
`to_hex(t_infomask)` 和 `to_hex(t_infomask2)` 帮你对照 `htup_details.h`。
但 `pageinspect` 不能告诉你：
- `xmax` 对某个 snapshot 是否 committed visible。
- MultiXact 中每个成员是什么状态。
- 当前 global horizon 是否允许 prune。
- index 是否还引用某个 dead line pointer。
`pg_locks` 可以帮助看等待。
但不要期待它枚举所有被 tuple header 锁住的行。
长期 tuple lock 状态在 heap tuple header 中。
`pg_locks` 更多显示等待排队时的 transactionid、tuple 或 relation lock。
这和 `README.tuplock` 的两级模型一致。
常用等待诊断：
```sql
SELECT pid, wait_event_type, wait_event, state, backend_xmin, query
FROM pg_stat_activity
WHERE datname = current_database()
ORDER BY pid;

SELECT locktype, mode, granted, relation::regclass, page, tuple,
       transactionid, pid
FROM pg_locks
WHERE NOT granted OR locktype IN ('tuple', 'transactionid')
ORDER BY pid, locktype;
```
`backend_xmin` 是判断长事务压 horizon 的入口。
如果某个 idle in transaction backend 持有很老 `backend_xmin`，DELETE 后的 tuple 可能长期只能 `RECENTLY_DEAD`。
replication slot 入口：
```sql
SELECT slot_name, slot_type, active, xmin, catalog_xmin, restart_lsn
FROM pg_replication_slots;
```
如果 `xmin` 或 `catalog_xmin` 很老，VACUUM 和 pruning 的 removable horizon 会被压住。
`pg_stat_all_tables` 可以看近似 dead tuple：
```sql
SELECT relname, n_live_tup, n_dead_tup,
       vacuum_count, autovacuum_count,
       last_vacuum, last_autovacuum
FROM pg_stat_all_tables
WHERE relname = 'demo_delete';
```
这些统计是估计和累计，不是 page 级真相。
`n_dead_tup` 不能精确解释某个 page 为什么没有 prune。
VACUUM VERBOSE 可以提供更接近 cleanup 的日志：
```sql
VACUUM (VERBOSE, ANALYZE) demo_delete;
```
它能看到 scanned pages、removed tuples、dead item 等信息。
但它仍不会显示每个 tuple 的 infomask。
`EXPLAIN (ANALYZE, BUFFERS, WAL)` 可以看 DELETE 和后续 VACUUM/UPDATE 是否制造 WAL 和 buffer dirties。
但它不能直接告诉你 `xmax` 是普通 XID 还是 MultiXact。
源码断点建议放在 `heap_delete()`、`compute_new_xmax_infomask()`、`HeapTupleSatisfiesUpdate()`、`heap_page_prune_opt()`、`heap_prune_chain()` 和 `heap_page_prune_execute()`。
重点看 raw `xmax`、`infomask`、`infomask2`、`t_ctid`、`new_xmax`、`PageGetPruneXid(page)`、`prstate.latest_xid_removed` 和 planned line pointer changes。
观测时要区分三类事实。
能直接看到：
line pointer flag、raw `xmax`、infomask hex、wait event、backend xmin、slot xmin。
只能推断：
为什么这次 scan 没有拿到 cleanup lock、为什么 `GlobalVisState` 没更新、为什么 on-access pruning 没动某个 line pointer。
几乎不可见：
某次短暂 retry loop 中 `xmax` 是否刚好变化、某个 MultiXact 成员解析的 SLRU cache miss、hint bit FPI 是否由这个 visibility check 触发。
## 15. 课堂实验一：DELETE commit 后 tuple 为什么还在
目标：
观察 DELETE 只设置 tuple header，空间不会在 commit 后立刻消失。
准备：
```sql
CREATE EXTENSION IF NOT EXISTS pageinspect;
DROP TABLE IF EXISTS demo_delete;
CREATE TABLE demo_delete (id int primary key, payload text) WITH (autovacuum_enabled = off);
INSERT INTO demo_delete
SELECT g, repeat('x', 100)
FROM generate_series(1, 20) AS g;
CHECKPOINT;
```
记录初始 page：
```sql
SELECT lp, lp_flags, t_xmin, t_xmax, t_ctid,
       to_hex(t_infomask) AS infomask,
       to_hex(t_infomask2) AS infomask2
FROM heap_page_items(get_raw_page('demo_delete', 0))
ORDER BY lp;
```
会话 A：
```sql
BEGIN;
SELECT count(*) FROM demo_delete;
```
保持事务打开。
会话 B：
```sql
DELETE FROM demo_delete WHERE id <= 5;
COMMIT;
```
会话 C 观察 raw page：
```sql
SELECT lp, lp_flags, t_xmin, t_xmax, t_ctid,
       to_hex(t_infomask) AS infomask,
       to_hex(t_infomask2) AS infomask2
FROM heap_page_items(get_raw_page('demo_delete', 0))
ORDER BY lp
LIMIT 8;
```
预期：
前几行的 `t_xmax` 被设置。
`t_ctid` 通常指向自身。
`lp_flags` 大概率仍是 normal。
这说明 DELETE commit 没有立即把 line pointer 改成 dead/unused。
会话 A：
```sql
SELECT count(*) FROM demo_delete;
COMMIT;
```
会话 B 或 C：
```sql
VACUUM (VERBOSE) demo_delete;
SELECT lp, lp_flags, t_xmin, t_xmax, t_ctid,
       to_hex(t_infomask) AS infomask,
       to_hex(t_infomask2) AS infomask2
FROM heap_page_items(get_raw_page('demo_delete', 0))
ORDER BY lp
LIMIT 8;
```
对照源码解释：
DELETE 在 `heap_delete()` 中设置 `xmax`、`cmax`、`t_ctid` 和 `pd_prune_xid`。
会话 A 的 old snapshot 压住 horizon。
VACUUM 或后续 pruning 只有在 old snapshot 结束后才能把 recently dead 转成 dead。
如果 VACUUM 后 line pointer 仍不是 `LP_UNUSED`，检查是否还有 index cleanup 边界或 page 布局因素。
不要把一次实验中某个具体 `lp_flags` 变化推广成所有版本和所有 workload 的固定结果。
## 16. 课堂实验二：tuple lock、MultiXact 与 DELETE 等待
目标：
观察多个 tuple locker 如何让 `xmax` 变成 MultiXact，并让 DELETE 通过 tuple lock/MultiXact wait 排队。
准备：
```sql
DROP TABLE IF EXISTS demo_lock;
CREATE TABLE demo_lock (id int primary key, note text) WITH (autovacuum_enabled = off);
INSERT INTO demo_lock VALUES (1, 'one');
```
会话 A：
```sql
BEGIN;
SELECT * FROM demo_lock WHERE id = 1 FOR KEY SHARE;
```
会话 B：
```sql
BEGIN;
SELECT * FROM demo_lock WHERE id = 1 FOR KEY SHARE;
```
会话 C 观察 tuple header：
```sql
SELECT lp, lp_flags, t_xmin, t_xmax,
       to_hex(t_infomask) AS infomask,
       to_hex(t_infomask2) AS infomask2
FROM heap_page_items(get_raw_page('demo_lock', 0));
```
预期：
`t_xmax` 可能已经是 MultiXactId。
`infomask` 中可看到 `HEAP_XMAX_IS_MULTI` 对应 bit。
具体 hex 需要对照 `htup_details.h`。
会话 D：
```sql
BEGIN;
DELETE FROM demo_lock WHERE id = 1;
```
此时会话 D 应等待。
会话 C 观察等待：
```sql
SELECT pid, wait_event_type, wait_event, state, query
FROM pg_stat_activity
WHERE query LIKE '%demo_lock%'
ORDER BY pid;

SELECT locktype, mode, granted, relation::regclass, page, tuple,
       transactionid, pid
FROM pg_locks
WHERE pid IN (
  SELECT pid FROM pg_stat_activity WHERE query LIKE '%demo_lock%'
)
ORDER BY pid, locktype, granted;
```
预期：
能看到 DELETE backend 等待 lock 或 transactionid/MultiXact 相关事件。
不一定能在 `pg_locks` 中看到所有已经持有的行锁。
因为长期状态在 tuple header。
释放会话 A/B：
```sql
COMMIT;
```
会话 D 完成后 commit。
再次观察 page：
```sql
SELECT lp, lp_flags, t_xmin, t_xmax, t_ctid,
       to_hex(t_infomask) AS infomask,
       to_hex(t_infomask2) AS infomask2
FROM heap_page_items(get_raw_page('demo_lock', 0));
```
对照源码解释：
两个 KeyShare lockers 可能促使 `compute_new_xmax_infomask()` 使用 MultiXact。
DELETE 请求 `LockTupleExclusive`。
它在 `heap_delete()` 中用 `DoesMultiXactIdConflict()` 判断冲突。
等待时先通过 `heap_acquire_tuplock()` 建立排队优先级，再 `MultiXactIdWait()`。
等待回来必须 recheck `xmax` 和 infomask。
## 17. 课堂实验三：on-access pruning 的入口条件
目标：
观察 `pd_prune_xid`、page 空间压力、scan 入口和 VACUUM 的差异。
准备一个容易形成 HOT chain 的表：
```sql
DROP TABLE IF EXISTS demo_prune;
CREATE TABLE demo_prune (
  id int primary key,
  payload text,
  pad text
) WITH (fillfactor = 70, autovacuum_enabled = off);

INSERT INTO demo_prune
SELECT g, 'v1', repeat('x', 100)
FROM generate_series(1, 50) AS g;
```
制造非 key UPDATE：
```sql
UPDATE demo_prune
SET payload = 'v2'
WHERE id BETWEEN 1 AND 20;

UPDATE demo_prune
SET payload = 'v3'
WHERE id BETWEEN 1 AND 20;
```
观察 page：
```sql
SELECT lp, lp_flags, t_xmin, t_xmax, t_ctid,
       to_hex(t_infomask) AS infomask,
       to_hex(t_infomask2) AS infomask2
FROM heap_page_items(get_raw_page('demo_prune', 0))
ORDER BY lp;
```
触发普通 scan：
```sql
SELECT count(*) FROM demo_prune WHERE id BETWEEN 1 AND 50;
```
再观察 page。
如果没有明显变化，不要立刻判定 pruning 没工作。
可能原因：
- `pd_prune_xid` 还没过 horizon。
- page free space heuristic 没满足。
- scan 没拿到 cleanup lock。
- exact tuple visibility 仍是 recently dead。
- 这个 page 不是 id 1 到 20 所在 page。
强制 VACUUM：
```sql
VACUUM (VERBOSE) demo_prune;
```
再观察 page。
对照源码解释：
普通 scan 走 `heap_page_prune_opt()`。
它必须通过 hint、horizon、free-space pressure 和 cleanup lock。
VACUUM 直接调用 `heap_page_prune_and_freeze()`，且有更完整的 index cleanup 和 VM/FSM 处理。
二者都可能进入 `heap_prune_chain()`，但参数和权限边界不同。
## 18. 常见误区
误区一：
`xmax` 有值就是 tuple 被删除。
正确说法：
`xmax` 可能是 locker、updater、deleter 或 MultiXactId。
必须结合 infomask、infomask2、MultiXact 成员、事务提交状态和 snapshot/horizon。
误区二：
`DELETE` commit 后 page free space 应该马上增加。
正确说法：
DELETE commit 后旧版本通常仍占用 tuple bytes。
空间增加要等 pruning/VACUUM 在 horizon 允许时改 line pointer 并 repair fragmentation。
误区三：
`pg_locks` 能列出所有行锁。
正确说法：
大量 tuple lock 状态在 tuple header。
lock manager 主要在等待冲突时提供排队。
`pg_locks` 只反映 lock manager 层和 transactionid wait 等状态。
误区四：
MultiXact 一定只是多个 shared locker。
正确说法：
现代 MultiXact 可以包含 update/delete 成员。
是否 lock-only 要看 member status 和 `HEAP_XMAX_LOCK_ONLY`。
误区五：
`pd_prune_xid` 表示 page 上一定有可删除 tuple。
正确说法：
它是 hint。
事务可能 abort。
horizon 可能还没过。
exact visibility 可能发现没东西可 prune。
误区六：
on-access pruning 和 VACUUM 一样彻底。
正确说法：
on-access pruning 是 opportunistic。
它非阻塞尝试 cleanup lock，不更新 FSM，也不能安全地把所有 dead root 直接设 unused。
误区七：
`LP_DEAD` 等于空间已经可复用。
正确说法：
`LP_DEAD` 通常意味着 index 可能还指向这个 root TID。
只有 `LP_UNUSED` 才能立即复用 line pointer。
## 19. 讨论题
1. 为什么 DELETE 使用 `LockTupleExclusive` 语义，而不是只写一个“deleted” bit？
2. 如果 `xmax` 是 MultiXactId，为什么不能直接把它当成 locker 集合？
3. `HEAP_XMAX_LOCK_ONLY` 和 `HEAP_KEYS_UPDATED` 分别保护什么语义？
4. 为什么 `heap_delete()` 等待并发事务时要先释放 buffer content lock，再等待，回来后又要检查 `xmax` 是否变化？
5. 为什么 DELETE commit 后不能立即把 line pointer 设成 `LP_UNUSED`？
6. on-access pruning 为什么要先看 `pd_prune_xid`，再看 horizon，再看 free space heuristic？
7. 一个长事务如何同时影响 VACUUM、on-access pruning 和查询扫描成本？
8. `pg_locks` 看不到某些已锁 tuple 时，应该用什么 mental model 解释？
## 20. 本节小结
本节的核心链路是：
```text
heap_delete()
  -> HeapTupleSatisfiesUpdate()
  -> wait/retry through XactLockTableWait or MultiXactIdWait
  -> compute_new_xmax_infomask()
  -> write xmax/infomask/cmax/t_ctid
  -> PageSetPrunable()
  -> XLOG_HEAP_DELETE
  -> later heap_page_prune_opt() or VACUUM
  -> HeapTupleSatisfiesVacuumHorizon() + GlobalVisTest
  -> heap_prune_chain()
  -> heap_page_prune_execute()
```
DELETE 的核心状态在 tuple header。
它记录删除者、锁语义和 command id。
但 tuple header 不决定空间复用。
空间复用要等 line pointer 层发生变化。
line pointer 层又受 index TID、HOT chain 和 cleanup lock 约束。
visibility horizon 是 DELETE 和 pruning 之间的时间边界。
只要还有 snapshot、replication slot 或 standby feedback 可能需要旧版本，tuple 就只能 recently dead，不能被安全移除。
tuple lock 的核心取舍是：
大量行锁状态放在 tuple header 中，避免 lock table 按行膨胀。
等待公平性由 heavyweight tuple lock 临时提供。
多个 locker/updater/deleter 通过 MultiXact 扩展 `xmax` 的表达能力。
错误路径的核心规律是：
等待时释放 buffer lock，回来后 recheck。
critical section 内先改 page 再写 WAL 的风险由 WAL-before-data 和 PANIC 边界控制。
abort 通过事务状态和 hint bit 解释，而不是物理 undo。
观测上，`pageinspect` 能看到 raw header 和 line pointer。
`pg_locks` 能看到等待队列的一部分。
`pg_stat_activity.backend_xmin`、replication slots 和 VACUUM VERBOSE 能解释 horizon 和 cleanup 滞后。
没有一个视图能单独给出完整因果。
可迁移的系统规律：
当一个系统同时需要稳定引用、并发可见性和空间回收时，不要把“逻辑删除”和“物理回收”合成一个动作。
先留下可解释的版本状态。
再用全局安全边界和局部 cleanup lock 延迟回收。
这种两阶段模型会增加 hint、retry、WAL 和观测复杂性，但它把前台 latency、MVCC correctness 和 page space reuse 放在了可以独立调度的位置。
