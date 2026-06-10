# PostgreSQL Heap update、new version 与 old tuple xmax
## 课程定位
本节主题：Heap update、new version 与 old tuple xmax。
上一节已经把 heap page、line pointer、tuple header、`xmin/xmax`、`t_ctid` 和 HOT bit 的物理位置讲清楚。
本节沿着 `UPDATE` 的真实执行路径继续往下走。
前置知识：已理解 buffer pin/content lock、page LSN、MVCC snapshot、tuple header、line pointer 和 WAL-before-data。
本节唯一主问题：
`UPDATE` 为什么必须生成一个新的 heap tuple version，同时修改旧版本的 `xmax` 和 `t_ctid`，而不是原地覆盖旧 tuple？
本节围绕的核心矛盾：
SQL 层希望 `UPDATE` 像“改一行”。
MVCC 层要求旧 snapshot 继续看见旧版本。
索引层希望 TID 指向能继续解释。
并发控制层又要把 tuple lock、updater、deleter 和 MultiXact 塞进有限的 tuple header。
崩溃恢复还要求这些状态能通过 WAL redo 重建。
PostgreSQL 的答案不是“更新一行”，而是同时完成两件事：
创建一个新 tuple version。
把旧 tuple version 标记为被当前更新事务取代。
学完本节，你应该能独立判断：
- `old tuple xmax` 表示什么，什么时候不是普通事务 ID。
- `new tuple xmin` 为什么是当前更新事务 ID。
- `old tuple t_ctid` 为什么指向新版本，而 `new tuple t_ctid` 通常指向自己。
- `HEAP_UPDATED`、`HEAP_HOT_UPDATED`、`HEAP_ONLY_TUPLE` 分别在谁身上。
- `xmax` 有值为什么可能只是 tuple lock，而不是删除或更新。
- `combo cid` 为什么会出现在同一事务内的更新路径。
- `LockTupleNoKeyExclusive` 和 `LockTupleExclusive` 如何影响 `HEAP_KEYS_UPDATED`。
- 多个 locker 为什么要进入 MultiXact。
- HOT 和非 HOT update 的分界点在哪里。
- WAL update record 如何同时覆盖旧版本头部变化和新版本插入。
- redo 为什么要重放旧 tuple `xmax`、旧 `t_ctid` 和新 tuple `xmin`。
- 哪些并发边界必须等待、重试或回滚到 caller。
- 哪些状态能用 `pageinspect`、`pg_stat_user_tables`、`pg_stat_wal` 和 `gdb` 观察。
本节不把 `UPDATE` 扩展成完整 executor 课程。
我们只跟 heap access method 内部的状态变化。
executor、index AM、logical decoding、vacuum 和 pruning 只讲与本节主链路相交的边界。
## 源码基线
源码仓库：
```text
/home/nail/postgres-lab
```
基线：
```text
branch: master
commit: bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8
commit date: 2026-06-09
```
本节重点阅读：
```text
src/backend/access/heap/heapam.c
src/backend/access/heap/heapam_visibility.c
src/backend/access/heap/heapam_xlog.c
src/backend/access/heap/README.HOT
src/backend/access/heap/README.tuplock
src/include/access/htup_details.h
src/include/access/heapam_xlog.h
```
辅助核对：
```text
src/backend/access/heap/hio.c
src/backend/access/heap/pruneheap.c
contrib/pageinspect/heapfuncs.c
```
行号来自：
```text
git -C /home/nail/postgres-lab show bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8:<file> | nl -ba
```
## 1. 先给结论
`UPDATE` 在 heap 中不是 in-place overwrite。
它是一次版本追加加一次旧版本头部改写。
旧版本仍占据原来的 line pointer。
新版本被插入到某个 heap page 的新 line pointer。
如果新版本能放在同一页且没有修改 HOT-blocking index 相关列，可能形成 HOT chain。
如果不能满足 HOT 条件，新版本会有自己的 index entries。
无论 HOT 还是非 HOT，旧版本的 `t_ctid` 都会指向新版本的 TID。
无论 HOT 还是非 HOT，新版本的 `xmin` 都是当前更新事务 ID。
旧版本的 `xmax` 表示“谁取代、删除或锁住了我”。
在普通 update 中，旧版本 `xmax` 通常是当前事务 ID。
如果旧版本上还要保留其他 locker，旧版本 `xmax` 可能变成 MultiXactId。
如果只是 row lock，`xmax` 也可能有值，但 `HEAP_XMAX_LOCK_ONLY` 说明它不是 updater。
所以本节最重要的不变量是：
```text
UPDATE = insert new version + mark old version as updated by xmax + link old t_ctid to new t_self
```
再加一条诊断不变量：
```text
xmax raw field is not semantics; xmax + infomask + infomask2 + snapshot + transaction status is semantics
```
`heap_update()` 的核心状态变化发生在 `heapam.c:4012-4108`。
这段代码进入 critical section 后：
- 调用 `PageSetPrunable()` 给旧页和必要的新页留下 pruning hint。
- 根据 HOT 条件设置或清除 `HEAP_HOT_UPDATED` 和 `HEAP_ONLY_TUPLE`。
- 调用 `RelationPutHeapTuple()` 把新版本插入 page。
- 清理旧版本过时的 `xmax` 相关 bits。
- 写旧版本 `xmax`。
- 写旧版本 `cmax` 或 combo CID。
- 把旧版本 `t_ctid` 指向新版本。
- 清理 visibility map 的 all-visible bits。
- 写 `XLOG_HEAP_UPDATE` 或 `XLOG_HEAP_HOT_UPDATE`。
- 给涉及的 page 设置同一个 WAL record LSN。
这个顺序解释了很多运行现象。
你在 `pageinspect` 里会看到一行 update 之后有两个 tuple version。
旧版本的 `t_xmax` 不再为零或 invalid。
新版本的 `t_xmin` 等于更新事务。
旧版本的 `t_ctid` 指向新版本。
新版本的 `t_ctid` 指向自己。
在 HOT 场景下，旧版本带 `HEAP_HOT_UPDATED`。
新版本带 `HEAP_ONLY_TUPLE`。
在非 HOT 场景下，旧版本仍通过 `t_ctid` 指向新版本，但 index 层会为新版本建立新的 index entry。
## 2. 源码阅读顺序
不要从 `heapam.c` 顶部线性读。
本节推荐按状态转移读。
| 顺序 | 文件 | 读什么 | 目标 |
| --- | --- | --- | --- |
| 1 | `htup_details.h` | `HeapTupleFields`、`t_ctid` 注释、infomask bits、accessor | 先建立 header 语义 |
| 2 | `README.tuplock` | tuple lock 两层协议、MultiXact、`HEAP_KEYS_UPDATED` | 理解 `xmax` 为什么不只是 deleter |
| 3 | `heapam_visibility.c` | `HeapTupleSatisfiesUpdate()`、`HeapTupleSatisfiesMVCC()`、vacuum horizon | 理解 update 前判定和 update 后可见性 |
| 4 | `heapam.c` | `heap_update()` | 跟主状态变化 |
| 5 | `hio.c` | `RelationPutHeapTuple()` | 看新 tuple 的 `t_self` 和 `t_ctid` 如何落盘 |
| 6 | `README.HOT` | HOT chain、redirect line pointer、pruning | 理解 HOT/非 HOT 分界 |
| 7 | `heapam_xlog.h` | `xl_heap_update`、update flags、infobits | 理解 WAL record 表达什么 |
| 8 | `heapam_xlog.c` | `heap_xlog_update()` | 验证 redo 能重建同一组状态 |
| 9 | `heapam.c` | `compute_new_xmax_infomask()`、MultiXact wait helpers | 理解锁和并发边界 |
本节的阅读主轴是一个 tuple version 的时间线：
```text
old version is visible
  -> updater wants to replace it
  -> old xmax/infomask is interpreted
  -> conflicting locker/updater may be waited for
  -> new tuple gets xmin=current xid
  -> old tuple gets xmax=updater or MultiXact
  -> old t_ctid links to new t_self
  -> WAL records both sides
  -> readers and vacuum interpret both versions under different horizons
```
## 3. 关键状态不是单个字段
heap tuple header 的事务字段定义在 `htup_details.h:122-131`。
它把 `t_xmin`、`t_xmax` 和 `t_field3` 放在 `HeapTupleFields`。
`t_xmin` 是插入这个 tuple version 的事务。
`t_xmax` 是删除、更新或锁定这个 tuple version 的事务信息。
`t_field3` 在 `cmin`、`cmax` 和历史 `xvac` 之间复用。
`htup_details.h:73-84` 的注释说明了为什么需要 combo CID。
PostgreSQL 只在 tuple header 里放三个物理事务字段。
逻辑上却需要 `xmin`、`cmin`、`xmax`、`cmax` 和 `xvac`。
这成立的前提是 `cmin` 和 `cmax` 只对 originating transaction 的 command visibility 有意义。
如果同一事务内一条 tuple 被 insert 后又 update 或 delete，单个 command id 字段不够同时表达 `cmin` 和 `cmax`。
这时会用 combo CID。
combo CID 不是全局语义。
它依赖发起 backend 的本地状态。
所以 `heap_update()` 一开始禁止在 parallel mode 下执行。
`heapam.c:3256-3264` 给出的原因是 worker 可能需要 combo CID 做 visibility check，但没有广播机制。
这不是 executor 限制。
这是 tuple header 编码和 backend-local combo CID 的边界。
`t_ctid` 的注释在 `htup_details.h:86-112`。
新 tuple 存到磁盘时，`t_ctid` 初始化为自己的 TID。
如果 tuple 被 update，旧 tuple 的 `t_ctid` 改成替代版本的 TID。
如果 tuple 是最新版本，通常 `t_ctid` 指向自己。
如果 `xmax` valid 但 `t_ctid` 指向自己，这个 tuple 可能只是被锁住，也可能被 delete。
分区移动会使用特殊值。
speculative insertion 也可能临时把 token 放进 `t_ctid`。
所以 `t_ctid` 不是一个普通指针。
它是版本链候选边。
沿链时必须验证目标 slot 仍然是同一条链上的 tuple。
验证规则是：
目标 tuple 的 `xmin` 要等于引用 tuple 的 update `xmax`。
原因是 VACUUM 可能先移除新版本，再移除旧版本。
目标 line pointer 可能已经空了。
目标 line pointer 也可能被复用给无关 tuple。
`heap_get_latest_tid()` 在 `heapam.c:1782-1895` 就是这个规则的运行样本。
它每次追 `t_ctid` 后都用 `priorXmax` 和目标 tuple 的 `xmin` 匹配。
如果 offset 无效、line pointer 不是 normal、`xmin` 不匹配，链路停止。
## 4. `xmax` 的五种常见含义
本节不能把 `xmax` 翻译成“删除事务”。
`README.tuplock:4-13` 明确说 tuple lock 的第一层就存储在 tuple header 中。
单个 locker 的 XID 会写入 `xmax`。
infomask bits 区分这是锁还是删除/更新。
多个 locker 不能都塞进一个 `xmax` 字段。
于是 PostgreSQL 用 MultiXactId 替换单个 XID。
在 `UPDATE` 相关诊断里，`xmax` 至少可能有五种含义。
第一种是普通 updater。
旧 tuple 的 `xmax` 是更新事务 ID。
如果事务提交，旧 tuple 对新 snapshot 不可见。
如果事务 abort，旧 tuple 仍然可见。
第二种是普通 deleter。
`DELETE` 也写旧 tuple 的 `xmax`。
区别在于 delete 不把 `t_ctid` 指向新版本。
`heapam_xlog.c:350-354` 的 delete redo 会把 `t_ctid` 设回自身或分区移动特殊值。
第三种是纯 locker。
`HEAP_XMAX_LOCK_ONLY` 表示这个 `xmax` 只是锁信息。
它不应该让 tuple 从 MVCC 结果中消失。
`HeapTupleSatisfiesMVCC()` 在 `heapam_visibility.c:1031-1032` 遇到 lock-only 会返回 visible。
第四种是 MultiXactId。
`HEAP_XMAX_IS_MULTI` 说明 raw `xmax` 不是 TransactionId。
要找真正 updater，可能需要 `HeapTupleHeaderGetUpdateXid()`。
`htup_details.h:381-395` 提醒这个 accessor 可能涉及 multixact I/O。
所以热路径不能随便把 raw `xmax` 解析成 updater。
第五种是无效或已 abort 的残留值。
`HEAP_XMAX_INVALID` 表示当前 tuple 没有有效 `xmax` 语义。
旧事务 abort 后可能留下 raw 值。
后续 visibility check 会设置 hint bits。
因此，诊断时正确的读法是：
```text
raw t_xmax
+ HEAP_XMAX_INVALID
+ HEAP_XMAX_LOCK_ONLY
+ HEAP_XMAX_IS_MULTI
+ HEAP_KEYS_UPDATED
+ transaction status
+ snapshot
= visible / locked / updated / deleted / recently dead
```
## 5. `HeapTupleSatisfiesUpdate()` 是 update 前的语义闸门
`heap_update()` 在真正改 page 前，会调用 `HeapTupleSatisfiesUpdate()`。
调用点在 `heapam.c:3428-3432`。
这个函数定义在 `heapam_visibility.c:481-736`。
它返回的不是普通 true/false。
它返回 `TM_Result`。
这些状态决定 update 是否能继续。
`TM_Ok` 表示当前 command 可以更新这个 tuple。
`TM_Invisible` 表示这个 tuple 对当前 command 根本不存在。
`heap_update()` 遇到它会报错：
```text
attempted to update invisible tuple
```
`TM_SelfModified` 表示同一事务已经在当前 scan 后修改过。
`TM_Updated` 表示别的已提交事务更新了它。
`TM_Deleted` 表示别的已提交事务删除了它。
`TM_BeingModified` 表示正在被别的事务修改或冲突锁住。
这个函数是 `UPDATE` 的第一个重要边界。
它回答的问题不是“这个 tuple 对 SELECT 可见吗”。
它回答的是“当前 command 是否可以把这个 physical version 当作 update target”。
这比 MVCC visible 更细。
因为 update 还要区分自修改、并发更新、并发删除、lock-only 和 MultiXact。
`HeapTupleSatisfiesUpdate()` 的关键判断是：
如果 `xmin` 未提交、无效或晚于当前 command，不能更新。
如果 `xmax` invalid，可以更新。
如果 `xmax` committed 且不是 lock-only，用 `t_ctid` 是否指向自己区分 updated 和 deleted。
如果 `xmax` 是 MultiXact，要解析其中的 updater 或 locker。
如果 `xmax` 是当前事务，要用 `cmax` 和当前 command id 判断 self-modified。
同一个 physical tuple 的 `xmax` 有值，不等于这个 tuple 不能被 update。
如果这个 `xmax` 只是 lock-only，或者对应事务 abort，它仍然可以更新。
`heap_update()` 需要先把 header 中这些状态解释完，再决定是否写新版本。
## 6. 主流程起点：准备 old tuple 和 modified attrs
`heap_update()` 定义在 `heapam.c:3200-4178`。
入口参数包括 old TID、new tuple、command id、crosscheck snapshot、wait policy 和返回给 caller 的 `TM_FailureData`。
函数一开始取当前事务 ID：
```text
xid = GetCurrentTransactionId()
```
这就是后面新 tuple `xmin` 和旧 tuple update `xmax` 的来源。
`heap_update()` 先计算四组属性 bitmap。
`hot_attrs` 来自 `INDEX_ATTR_BITMAP_HOT_BLOCKING`。
它表示哪些列如果改变会阻止 HOT。
`sum_attrs` 表示 summarizing indexes 相关列。
BRIN 这类 summarizing index 不保存单个 tuple TID，可以允许 HOT chain，但可能仍需要更新 summary。
`key_attrs` 表示 tuple lock 层面的 key columns。
如果没有修改 key attrs，update 可以用 `LockTupleNoKeyExclusive`。
如果修改 key attrs，要用 `LockTupleExclusive`。
`id_attrs` 是 replica identity 相关列。
它影响 logical decoding 和 WAL 中旧 key/旧 tuple 是否需要记录。
这些 bitmap 必须在拿 buffer lock 前计算。
`heapam.c:3270-3285` 的注释说明了原因。
如果某些 catalog 的 update 在持 buffer lock 后再访问 relcache，可能死锁。
所以这里保留了一个 awkward 但重要的顺序：
先在 local memory 中拿到属性信息。
再进入 page 级状态修改。
随后 `heap_update()` 根据 old TID 读 buffer。
如果 page 看起来 all-visible，会先 pin visibility map page。
这是为了避免后面持有 buffer content lock 时发生 I/O。
拿到 old page exclusive content lock 后，代码读取 line pointer。
如果 line pointer 不是 `LP_NORMAL`，通常说明并发 pruning 已经改变了目标 slot。
`heapam.c:3317-3339` 专门解释了 syscache 场景下可能没有 pin 或 snapshot。
这种情况返回 `TM_Deleted`，并填充 `tmfd`。
普通 SQL UPDATE/MERGE 通常持有 snapshot，所以一般不会走这个分支。
然后代码构造 `oldtup`。
`oldtup.t_data` 指向 shared buffer page 里的旧 tuple header。
`oldtup.t_self` 是 old TID。
这个 `HeapTupleData` 不拥有 tuple bytes。
它只是对 buffer page 的临时视图。
后面的更新必须在正确的 buffer pin 和 content lock 保护下使用它。
## 7. 锁强度：key-intact update 的并发收益
`heap_update()` 在 `heapam.c:3386-3419` 根据修改列选择 tuple lock 强度。
如果没有修改 `key_attrs`，使用：
```text
LockTupleNoKeyExclusive
MultiXactStatusNoKeyUpdate
key_intact = true
```
如果修改了 key attrs，使用：
```text
LockTupleExclusive
MultiXactStatusUpdate
key_intact = false
```
这个选择不是性能微调。
它决定 update 是否与 `SELECT FOR KEY SHARE` 冲突。
`README.tuplock:49-61` 解释了四种 tuple lock：
`FOR UPDATE` 阻止所有修改。
`FOR NO KEY UPDATE` 阻止删除和 key 修改。
`FOR SHARE` 阻止所有修改。
`FOR KEY SHARE` 只阻止删除和 key 修改。
所以非 key update 可以和 key-share locker 并发。
这正是外键检查需要的并发性质。
外键检查通常只需要保证 referenced row 不消失或不改 key。
它不应该阻塞一个只改非 key 列的 update。
这条规则落到 tuple header 上，就是 `HEAP_KEYS_UPDATED`。
`README.tuplock:140-147` 说明 `HEAP_XMAX_EXCL_LOCK` 本身不能区分 `FOR UPDATE` 和 `FOR NO KEY UPDATE`。
这个差异由 `HEAP_KEYS_UPDATED` 表达。
因此 old tuple 的 `infomask2` 不是附属细节。
它参与了 tuple lock 冲突判断。
`compute_new_xmax_infomask()` 在 is-update 场景下，如果 mode 是 `LockTupleExclusive`，会设置 `HEAP_KEYS_UPDATED`。
如果 mode 是 `LockTupleNoKeyExclusive`，不会设置。
这就是 key-intact update 的状态落点。
## 8. 等待、重试和保留 locker
`heap_update()` 的等待逻辑从 label `l2` 开始。
它先调用 `HeapTupleSatisfiesUpdate()`。
如果返回 `TM_BeingModified` 且 caller 允许 wait，就要处理已有 locker 或 updater。
这里有一个重要协议：
等待前必须复制 tuple header 中的 `xmax` 和 `infomask`。
因为接下来会释放 buffer content lock。
释放后其他 backend 可能修改同一 tuple header。
等待结束重新拿锁后，必须检查 `xmax` 和关键 infomask bits 是否变化。
如果变化，回到 `l2` 重新解释 tuple。
如果旧 `xmax` 是 MultiXact，代码用 `DoesMultiXactIdConflict()` 判断是否冲突。
冲突时释放 buffer lock。
必要时调用 `heap_acquire_tuplock()` 进入 heavyweight lock manager 的排队。
然后调用 `MultiXactIdWait()` 等待冲突成员。
等待后重新拿 buffer lock，检查 tuple header 是否仍然是原来那个状态。
如果旧 `xmax` 是普通事务，代码可能调用 `XactLockTableWait()`。
但在等待前同样要用 heavyweight tuple lock 建立公平排队。
`README.tuplock:15-37` 解释了为什么只等 transaction id 不够。
如果所有 waiter 都在事务结束时同时醒来，会发生竞争和可能的 starvation。
heavyweight tuple lock 是第二层仲裁。
但 PostgreSQL 不把每个 row lock 都常驻 shared lock table。
只有需要等待时，最多每个 backend 持有或等待一个 tuple-level lock。
这就是两层协议的工程取舍。
如果已有 locker 是当前事务自己，或者只是 `FOR KEY SHARE` 且当前 update 不改 key，`heap_update()` 可以继续。
但它要保留 locker 信息。
这就是 `checked_lockers` 和 `locker_remains` 的意义。
后面会决定新 tuple 的 `xmax` 是否继承旧 tuple 上仍然存活的 key-share locks。
这里的可迁移规律是：
等待不是简单 sleep。
等待意味着释放 page lock、建立排队、睡在事务或 MultiXact 上、回来后重新验证物理 header。
## 9. `compute_new_xmax_infomask()`：把并发历史塞回 old tuple
真正写旧 tuple `xmax` 前，`heap_update()` 先调用 `compute_new_xmax_infomask()`。
调用点在 `heapam.c:3686-3691`。
参数包括旧 raw `xmax`、旧 `infomask`、旧 `infomask2`、当前事务 ID、lock mode 和 `is_update=true`。
这个函数定义在 `heapam.c:5271-5553`。
它的任务是：
在旧 tuple header 里表达“当前事务正在 update 这个 tuple”，同时不丢掉仍然需要保留的 locker 或已提交 updater 信息。
如果旧 `xmax` invalid，最简单。
当前 update 直接把 `new_xmax` 设为当前事务 ID。
如果 lock mode 是 `LockTupleExclusive`，设置 `HEAP_KEYS_UPDATED`。
如果 lock mode 是 `LockTupleNoKeyExclusive`，不设置。
如果旧 `xmax` 已经是 MultiXact，函数可能扩展这个 MultiXact。
如果 MultiXact 中所有 locker 已经结束，且没有需要保留的 committed updater，可以把它当 invalid 重新走简单路径。
如果 MultiXact 中还有需要保留的成员，就调用 `MultiXactIdExpand()`，把当前事务作为 updater 加进去。
然后用 `GetMultiXactIdHintBits()` 生成新的 infomask。
如果旧 `xmax` 是已经提交的 updater，函数会创建新的 MultiXact 来保留旧 updater 和当前 locker/updater 的组合。
这种情况在普通 heap update 主路径里较少见，但它说明 MultiXact 不只是多个 locker。
`README.tuplock:87-92` 明确说现代 MultiXact 可能包含 update 或 delete XID。
如果旧 `xmax` 是 still-in-progress 的普通 XID，函数创建 MultiXact 包含旧成员和当前事务。
旧成员的 status 来自旧 infomask 和 `HEAP_KEYS_UPDATED`。
当前成员 status 来自当前 lock mode 和 `is_update`。
如果旧事务刚刚结束，函数可能把旧状态改成 invalid，再跳回开头。
这处理了读取 old header 到检查 transaction state 之间的 race。
这个函数有副作用。
它可能创建 MultiXactId。
所以不能把它当纯计算函数。
它的输出是三件套：
```text
xmax_old_tuple
infomask_old_tuple
infomask2_old_tuple
```
后面 critical section 里会把这三件套写回旧 tuple。
## 10. 新 tuple 的 `xmin` 和可能继承的 `xmax`
旧 tuple 的 `xmax` 需要表达 updater。
新 tuple 的 `xmin` 则直接表达“谁插入了这个新版本”。
`heap_update()` 在 `heapam.c:3732-3742` 准备 new tuple header：
```text
clear HEAP_XACT_MASK and HEAP2_XACT_MASK
set xmin = current xid
set cmin = current cid
set HEAP_UPDATED
set xmax = xmax_new_tuple
```
`HEAP_UPDATED` 在新版本上。
它说明这个 tuple version 是由 UPDATE 产生的。
它不是“旧版本被更新”的标志。
旧版本是否 HOT-updated 用 `HEAP_HOT_UPDATED`。
新版本是否 heap-only 用 `HEAP_ONLY_TUPLE`。
大多数情况下，新 tuple 的 `xmax` 是 invalid。
代码在 `heapam.c:3700-3710` 如果旧 tuple `xmax` invalid、升级遗留 MultiXact、或者等待后没有 locker remain，就设置：
```text
xmax_new_tuple = InvalidTransactionId
infomask_new_tuple = HEAP_XMAX_INVALID
```
但也存在新 tuple 继承锁信息的情况。
如果旧 tuple 上还有 key-share lockers，且当前 update 不与它们冲突，新版本也必须保留这些 lock 语义。
否则外键检查可能以为新版本没有被保护。
这时新 tuple 的 raw `xmax` 可能不是 invalid。
如果旧 tuple `xmax` 是 MultiXact，调用 `GetMultiXactIdHintBits()` 生成新 tuple 的 lock bits。
如果是普通 key-share locker，设置 `HEAP_XMAX_KEYSHR_LOCK | HEAP_XMAX_LOCK_ONLY`。
这解释了另一个诊断陷阱：
新版本也可能有 `xmax`。
如果它带 lock-only bits，它仍然可能是 live tuple。
`xmax` 有值不是“旧版本”的专利。
## 11. combo CID：同一事务内的 command visibility
在准备 new tuple header 后，`heap_update()` 调用：
```text
HeapTupleHeaderAdjustCmax(oldtup.t_data, &cid, &iscombo)
```
调用点在 `heapam.c:3744-3748`。
这一步处理旧 tuple 的 `cmax`。
如果同一事务内既需要记住 old tuple 的 `cmin`，又需要记住 old tuple 的 `cmax`，单个 `t_field3` 不够。
于是 `cid` 会被替换成 combo CID。
后面写旧 tuple 时使用：
```text
HeapTupleHeaderSetCmax(oldtup.t_data, cid, iscombo)
```
调用点在 `heapam.c:4057`。
combo CID 的影响只在 originating backend 内完整可解释。
这也是 `heap_update()` 禁止 parallel mode 的原因。
逻辑解码还有额外边界。
如果 relation 能被 logical decoding 访问，`heap_update()` 在 WAL update 前调用：
```text
log_heap_new_cid(relation, &oldtup)
log_heap_new_cid(relation, heaptup)
```
调用点在 `heapam.c:4087-4095`。
原因是 logical decoding 需要正确解释 catalog 的 combo CID。
普通数据表的 MVCC 可见性可以依赖事务提交顺序和 tuple header。
但 decoding catalog 变化时，command boundary 和 combo CID 会影响 historic snapshot 的解释。
所以 combo CID 不只是“同事务里看不看得到”的局部小问题。
它会影响 WAL 里的可解码语义。
## 12. TOAST 或同页空间不足时的临时锁路径
`UPDATE` 的 happy path 是新 tuple 不需要 TOAST 且同页有足够空间。
但真实系统常常不是这样。
如果新 tuple 需要 TOAST，或者 old page 放不下新 tuple，`heap_update()` 必须释放 old page content lock 去做额外工作。
问题是：释放 page lock 后，其他 backend 可能抢先 update 同一个 old tuple。
PostgreSQL 的做法是先把 old tuple 临时标记成 locked。
这段逻辑在 `heapam.c:3778-3862`。
它调用 `compute_new_xmax_infomask()`，但这次 `is_update=false`。
得到的是 lock-only 的 `xmax_lock_old_tuple`。
随后进入 critical section：
- 清掉旧的 `xmax` bits。
- 清掉 `HEAP_KEYS_UPDATED`。
- 清掉 HOT-updated 标记。
- 写 lock-only `xmax` 和 infomask。
- 写 `cmax`。
- 暂时把 `oldtup.t_ctid` 设回自己。
- 必要时清 VM all-frozen bit。
- 标脏 buffer。
- 写 `XLOG_HEAP_LOCK`。
注释在 `heapam.c:3785-3794` 说得很直接。
为了满足“任何可能出现在 buffer 中的 XID 都要可 WAL 恢复”的规则，这个临时修改也要写 WAL。
如果在真正 update 之前 crash 或 ERROR，这个临时 `xmax` 会属于 aborted transaction。
其他 session 之后可以继续。
这条路径解释了一个边界：
`UPDATE` 有时会先产生一个 lock WAL record，再产生 update WAL record。
看到旧 tuple 曾经 `t_ctid` 指向自己，不一定说明没有 update 尝试。
可能只是 update 在释放 page lock 前的临时 lock 状态。
## 13. 选择新版本落点和死锁规避
如果新 tuple 同页放不下，`heap_update()` 调用 `RelationGetBufferForTuple()` 找新 page。
这不是简单找空间。
它涉及 old page 和 new page 两个 buffer lock。
`heapam.c:3888-3895` 说明必须按 relation 中较低 block number 先加锁。
否则两个 backend 同时把不同页上的 tuple update 到对方页附近时，可能互相等待。
实现上，`heap_update()` 在不持 old page content lock 的情况下调用 `RelationGetBufferForTuple()`。
让这个函数按统一规则拿锁。
如果新 tuple 仍然能放回 old page，代码会重新 pin VM page、重新拿 old page lock、重新计算 free space。
如果这期间 page 空间或 all-visible 状态变化，又要释放锁重试。
这就是 `heapam.c:3903-3938` 的 loop。
这段代码体现了 heap update 的真实复杂度：
新版本落点不是由旧 tuple 自己决定。
它受 page free space、TOAST、visibility map pin、buffer lock ordering 和 concurrent insert/update 影响。
如果最终 `newbuf != buffer`，说明新版本去了别的 page。
这会直接结束 HOT 可能性。
同时 old page 被 `PageSetFull(page)` 标记为可考虑 prune/defrag。
这个 flag 是 hint，不是严格事实。
## 14. HOT 与非 HOT 的连接点
HOT 不是 update 的另一套实现。
HOT 是 `heap_update()` 在新版本已经有落点后选择的一组标志和 index 维护策略。
关键判断在 `heapam.c:3972-3993`。
必须先满足：
```text
newbuf == buffer
```
也就是新旧 tuple version 在同一个 heap page。
然后还要满足：
```text
!bms_overlap(modified_attrs, hot_attrs)
```
也就是没有修改 HOT-blocking index 相关列。
满足时：
```text
use_hot_update = true
```
如果修改了 summarizing index 相关列，还会设置：
```text
summarized_update = true
```
真正写 bit 在 critical section 里。
`heapam.c:4029-4037`：
- 旧 tuple 设置 `HEAP_HOT_UPDATED`。
- 新 tuple 设置 `HEAP_ONLY_TUPLE`。
- caller 的 newtup copy 也设置 `HEAP_ONLY_TUPLE`。
如果不是 HOT，`heapam.c:4038-4044` 会清掉这些 bit。
这保证 abort/retry 或临时 lock path 不会留下过时 HOT 标志。
HOT 的 index 边界在函数返回前设置。
`heapam.c:4159-4167`：
如果 HOT 且不需要 summarizing index update，`*update_indexes = TU_None`。
如果 HOT 但 summarizing index 受影响，`*update_indexes = TU_Summarizing`。
如果非 HOT，`*update_indexes = TU_All`。
这就是 executor/index AM 后续工作的分叉。
`README.HOT:53-80` 给出 HOT 的核心收益。
index entry 仍指向 HOT chain root。
heap-only 新 tuple 没有自己的普通 index entry。
index scan 找到 root 后，沿 `HEAP_HOT_UPDATED` 和 `t_ctid` 继续找子版本。
`README.HOT:113-122` 解释为什么 HOT 不跨页。
跨页会破坏 page-local pruning 的目标。
也会让 index fetch 为了追链多读 heap page。
所以新版本离开 old page，就必须结束 HOT chain，并让新版本拥有自己的 index entries。
## 15. 真正的 page 修改顺序
到 `heapam.c:4012`，代码进入：
```text
START_CRIT_SECTION()
```
注释写着：
```text
NO EREPORT(ERROR) from here till changes are logged
```
这不是风格要求。
这是 crash safety 和 shared buffer consistency 的边界。
进入 critical section 前，代码已经完成可能分配内存的工作。
比如 replica identity 的 old key tuple 在 `heapam.c:4000-4010` 预先提取。
critical section 内部的状态变化要么完成并写 WAL，要么 PANIC 让 recovery 接管。
第一步是设置 pruning hint。
`PageSetPrunable(page, xid)` 表示如果当前事务提交，old tuple 将来会变 dead。
如果新版本在另一页，新页也设置 prunable。
如果事务最后 abort，后续 pruning 会发现没有可做的事并清理 hint。
第二步是 HOT flags。
旧 tuple 和新 tuple 的 HOT/heap-only bits 在真正插入新 tuple 前确定。
第三步是插入新 tuple。
调用：
```text
RelationPutHeapTuple(relation, newbuf, heaptup, false)
```
这个函数在 `hio.c:58-79` 调用 `PageAddItem()`，得到 offset number。
然后设置 `tuple->t_self` 为实际 `(block, offnum)`。
最后把 page 中实际 tuple header 的 `t_ctid` 设置为这个 `t_self`。
这说明新 tuple 的 self `t_ctid` 不是 caller 预先知道的。
它必须等 page add item 分配 line pointer 后才确定。
第四步是写旧 tuple。
代码清掉旧 tuple 过时的 `HEAP_XMAX_BITS` 和 `HEAP_MOVED`。
清掉旧 `HEAP_KEYS_UPDATED`。
然后写：
```text
HeapTupleHeaderSetXmax(oldtup.t_data, xmax_old_tuple)
oldtup.t_data->t_infomask |= infomask_old_tuple
oldtup.t_data->t_infomask2 |= infomask2_old_tuple
HeapTupleHeaderSetCmax(oldtup.t_data, cid, iscombo)
oldtup.t_data->t_ctid = heaptup->t_self
```
这就是本节主题的中心。
旧 tuple 的 `xmax` 和 `t_ctid` 是同一条 update 边的两半。
`xmax` 说明“哪个事务做了取代”。
`t_ctid` 说明“替代版本在哪里”。
只看一半都不够。
第五步是清 all-visible。
如果 old page 或 new page 在 visibility map 中被标记 all-visible，update 后必须清掉。
新旧 page 可能相同，也可能不同。
第六步是标脏 buffer 和写 WAL。
如果 relation 需要 WAL，调用 `log_heap_update()`。
返回的 `recptr` 会设置到涉及的 heap page LSN 上。
这是 WAL-before-data 的页面级落点。
## 16. WAL record 表达了哪些状态
`heapam_xlog.h:203-235` 定义 `xl_heap_update`。
核心字段是：
```text
old_xmax
old_offnum
old_infobits_set
flags
new_xmax
new_offnum
```
注意它不直接存完整旧 tuple header。
旧 tuple 已经在 page 上。
redo 只需要知道 old offset、old xmax、旧 infomask 中和 xmax 相关的 bits，以及新 tuple 的位置。
`old_infobits_set` 来自 `compute_infobits()`。
`heapam.c:2665-2682` 把这些 bits 压缩到 WAL：
- `XLHL_XMAX_IS_MULTI`
- `XLHL_XMAX_LOCK_ONLY`
- `XLHL_XMAX_EXCL_LOCK`
- `XLHL_XMAX_KEYSHR_LOCK`
- `XLHL_KEYS_UPDATED`
redo 中 `fix_infomask_from_infobits()` 做反向恢复。
它在 `heapam_xlog.c:259-284`。
新 tuple 的 WAL 表达不同。
`xl_heap_header` 只保存：
```text
t_infomask2
t_infomask
t_hoff
```
新 tuple data 会跟在 block data 中。
`heapam_xlog.h:146-159` 解释了为什么不把整个 fixed header 存进去。
`xmin` 可以从 WAL record 的 xid 得到。
`xmax` 有 `xlrec.new_xmax`。
`cmin` redo 时可用 `FirstCommandId`。
`t_ctid` redo 时设置成 new TID。
所以 WAL 记录的是重建所需的最小组合，而不是源码里每个字段的镜像。
`log_heap_update()` 在 `heapam.c:8799-8802` 根据新 tuple 是否 heap-only 选择 record 类型：
```text
XLOG_HEAP_HOT_UPDATE
XLOG_HEAP_UPDATE
```
同页 update 且不需要 logical tuple data 时，它还可能只记录新旧 tuple user data 的 prefix/suffix delta。
这在 `heapam.c:8804-8854`。
如果 logical decoding 需要完整新 tuple，或者 full-page image 条件要求保留数据，就不能只靠 delta。
flags 还记录是否清了 old/new all-visible。
如果更新的 tuple 对 logical decoding 需要 old key 或 old tuple，flags 还会带 `XLH_UPDATE_CONTAINS_OLD_KEY` 或 `XLH_UPDATE_CONTAINS_OLD_TUPLE`。
这说明 WAL update record 同时服务 crash recovery、standby replay 和 logical decoding。
它不是单纯的物理 diff。
## 17. redo 如何重建 old/new 两侧
redo 入口是 `heap_xlog_update()`。
它在 `heapam_xlog.c:693-950`。
record 类型 `XLOG_HEAP_UPDATE` 和 `XLOG_HEAP_HOT_UPDATE` 都走这里。
函数先构造 new TID：
```text
newtid = (newblk, xlrec->new_offnum)
```
如果 old page 和 new page 不同，WAL record 有第二个 block reference。
redo 注释在 `heapam_xlog.c:757-764` 提醒：
正常运行要按 page number 顺序加锁避免死锁。
redo 中没有并发 updater。
但 Hot Standby queries 可能读取。
所以不能在添加新 tuple 之前暴露不一致的 old page 状态。
redo 先处理 old tuple。
它定位 old offset。
要求 line pointer 是 normal。
然后：
- 清旧 `HEAP_XMAX_BITS` 和 `HEAP_MOVED`。
- 清 `HEAP_KEYS_UPDATED`。
- 根据 record 类型设置或清 `HEAP_HOT_UPDATED`。
- 用 `fix_infomask_from_infobits()` 恢复 old xmax 相关 bits。
- 写 `old_xmax`。
- 写 `cmax = FirstCommandId`。
- 把 old `t_ctid` 设置成 `newtid`。
- 设置 page prunable。
- 必要时清 all-visible。
- 设置 page LSN。
这正好对应前台 `heap_update()` 的旧版本改写。
redo 再处理 new tuple。
它从 WAL block data 读 `xl_heap_header` 和 tuple data。
如果 record 使用 prefix/suffix delta，就从 old tuple 复制前后缀。
然后设置：
```text
htup->t_infomask2 = xlhdr.t_infomask2
htup->t_infomask = xlhdr.t_infomask
htup->t_hoff = xlhdr.t_hoff
HeapTupleHeaderSetXmin(htup, XLogRecGetXid(record))
HeapTupleHeaderSetCmin(htup, FirstCommandId)
HeapTupleHeaderSetXmax(htup, xlrec->new_xmax)
htup->t_ctid = newtid
PageAddItem(...)
```
这说明 crash recovery 看到的 new tuple `xmin` 不是 WAL payload 里单独写的字段。
它来自 WAL record xid。
redo 把新 tuple `t_ctid` 设成 self。
这与 `RelationPutHeapTuple()` 的前台行为一致。
旧 tuple 的 `t_ctid` 指向 newtid。
这与前台 old header 改写一致。
如果 page 已经有足够新的 LSN，redo 可以跳过 page 修改。
但 visibility map 清理可能仍要处理。
`heap_xlog_update()` 在 old/new all-visible flags 上单独清 VM。
这解释了为什么 WAL record flags 需要显式记录 VM bit 变化。
## 18. MVCC reader 如何解释两个版本
`HeapTupleSatisfiesMVCC()` 定义在 `heapam_visibility.c:939-1095`。
它处理普通 MVCC snapshot。
更新提交前，一个旧 snapshot 可能仍然看见旧版本。
因为旧 tuple 的 `xmax` 对该 snapshot 来说可能仍在 snapshot 内或未提交。
更新提交后，一个新 snapshot 通常看不见旧版本。
它会看见新版本，因为新 tuple 的 `xmin` 已提交且不在 snapshot 内。
如果 snapshot 是更新前取得的，它通常看不见新版本。
因为新 tuple `xmin` 在 snapshot 中。
这就是 UPDATE 产生两个 tuple version 的根本原因。
同一条逻辑行在不同 snapshot 下会选择不同 physical version。
`HeapTupleSatisfiesMVCC()` 对 `xmax` 的核心逻辑是：
如果 `xmax` invalid，tuple visible。
如果 `xmax` lock-only，tuple visible。
如果 `xmax` 是 MultiXact，要取 updater XID。
如果 updater 还在当前 snapshot 中，旧 tuple 对该 snapshot 仍 visible。
如果 updater committed 且不在 snapshot 中，旧 tuple invisible。
如果 updater aborted，旧 tuple visible。
这不是“沿 `t_ctid` 找最新版本”的逻辑。
heap scan 会逐个检查 page 上的 tuple version。
index scan 命中某个 TID 后，还要按 HOT chain 规则可能继续追同页 heap-only tuple。
普通 MVCC snapshot 中，同一 update chain 最多一个版本可见。
但非 MVCC snapshot、dirty snapshot、vacuum/pruning 用的是不同语义。
所以不要把 `HeapTupleSatisfiesMVCC()` 的结果直接推广到所有 scan。
`HeapTupleSatisfiesVacuum()` 是另一个问题。
它在 `heapam_visibility.c:1100-1132`。
它问的是 tuple 是否还能被任何运行事务看见。
如果 old tuple 的 updater committed，但 `dead_after >= OldestXmin`，它返回 `HEAPTUPLE_RECENTLY_DEAD`。
这表示普通新 query 可能已经看不见旧版本。
但 VACUUM/pruning 仍不能移除它。
必须等 horizon 推进。
这是 UPDATE 版本链和 bloat 的核心边界。
## 19. HOT chain、pruning 和 line pointer 的后半段故事
UPDATE 当场只创建 chain。
它不负责完整清理 chain。
HOT 让 index entry 指向 root line pointer。
新 heap-only tuple 没有普通 index entry。
当 root tuple 对所有相关 snapshot 都死掉时，root line pointer 不能直接变 `LP_UNUSED`。
因为 index entry 仍指向它。
`README.HOT:82-111` 说明 root line pointer 会变成 redirect pointer。
redirect pointer 没有 tuple storage。
它把 root offset 转到仍然活着的 heap-only tuple offset。
中间 dead heap-only tuple 可以被剪掉。
当整条 HOT chain 没有非 dead 成员，root line pointer 可变 `LP_DEAD`。
之后普通 VACUUM 清理 index entries 后，line pointer 才能最终复用。
这解释了为什么 HOT update 当场只设置两个 bit：
旧版本 `HEAP_HOT_UPDATED`。
新版本 `HEAP_ONLY_TUPLE`。
真正的 `LP_REDIRECT`、`LP_DEAD`、tuple storage reclaim 是后续 pruning/vacuum 的工作。
`README.HOT:199-218` 还区分 pruning 和 defragmentation。
pruning 缩短 HOT chain，释放 line pointer。
defragmentation 把 page 中零散空洞集中成 `pd_lower` 到 `pd_upper` 之间的 free space。
UPDATE 失败同页放置时设置 `PD_PAGE_FULL` hint。
后续访问 page 时，如果能拿到 cleanup lock，可能触发 prune/defrag。
`README.HOT:228-239` 说明不能拿 cleanup lock 就推迟。
最坏后果通常是本来 HOT-safe 的 update 因空间未集中而只能变成非 HOT。
这也是 fillfactor 和 autovacuum 对 HOT 命中率有影响的原因。
## 20. 错误路径和异常边界
第一类错误是 parallel mode。
`heap_update()` 可能分配 combo CID。
由于没有机制把 combo CID 映射广播给其他 worker，所以在 parallel mode 下直接 ERROR。
第二类错误是 invisible tuple。
如果 `HeapTupleSatisfiesUpdate()` 返回 `TM_Invisible`，`heap_update()` 报错。
这通常意味着 caller 的可见性假设和 heap state 不一致。
第三类是并发 pruning 后 old TID 不再 normal。
对于普通 SQL UPDATE，snapshot 和 pin 通常让目标仍是 `LP_NORMAL`。
但 syscache 来源可能没有这样的保护。
`heap_update()` 在这种情况下返回 `TM_Deleted`，并填 `tmfd`。
第四类是 wait 后 tuple header 改变。
代码不会相信等待前的判断。
它检查 raw `xmax` 和关键 infomask bits。
变化就跳回 `l2`。
第五类是 VM pin 和 buffer lock 顺序。
如果 page 变成 all-visible 但当前没有 VM pin，必须释放 buffer lock，pin VM，再重新拿 lock 和重查 tuple。
这是为了避免持 buffer content lock 做可能 I/O 的操作。
第六类是 TOAST 或跨页 update 前的临时 lock。
如果临时 lock WAL 写完后事务 abort，后续 visibility check 会把 old tuple 的 `xmax` 当 aborted lock/updater 处理。
系统依靠事务状态和 hint bits 收尾。
第七类是 critical section。
从 `START_CRIT_SECTION()` 到 WAL record 写完，不允许普通 ERROR。
如果这里出问题，进程会走 PANIC 级别，让 crash recovery 用 WAL 恢复一致状态。
第八类是 logical decoding。
catalog update 如果需要 combo CID，必须在 WAL 中写 `NEW_CID` record。
否则 decoding historic snapshot 可能无法解释 command-level visibility。
这些边界共同说明：
UPDATE 的正确性不是一个机制保证的。
它是 tuple header、transaction status、buffer lock、tuple lock、MultiXact、visibility map、WAL、invalidation 和 pruning horizon 的组合。
## 21. ownership、cleanup 和 abort 后语义
`heap_update()` 持有的主要资源有：
- old buffer pin。
- old buffer exclusive content lock。
- 可能的新 buffer pin。
- 可能的新 buffer content lock。
- 可能的 visibility map buffer pin。
- 可能的 heavyweight tuple lock。
- local bitmapsets。
- 可能的 toasted heap tuple copy。
- 可能的 old replica identity tuple。
正常返回时，函数释放 buffer locks。
然后调用 `CacheInvalidateHeapTuple()`。
再 release buffer pins 和 VM pins。
如果持有 heavyweight tuple lock，调用 `UnlockTupleTuplock()`。
如果 toasted tuple 是 private copy，把 `t_self` 复制回 caller 的 `newtup`，再 `heap_freetuple()`。
最后释放 bitmapsets。
这里有一个细节：
catalog invalidation 要在 release buffer 之前做。
`heapam.c:4116-4124` 注释说 old tuple 在 buffer 中，必须在释放 buffer 前处理。
new tuple 可能在 local memory 中，但 old/new 两个版本最好一次传给 inval 层，减少重复 sinval message。
如果事务 abort，已经插入的新 tuple version 的 `xmin` 是 aborted xid。
普通 MVCC 不会看见它。
`HeapTupleSatisfiesVacuumHorizon()` 遇到 aborted `xmin` 会把它当 `HEAPTUPLE_DEAD`。
旧 tuple 的 `xmax` 如果是 aborted updater，后续 visibility check 会设置 `HEAP_XMAX_INVALID`。
旧 tuple 继续作为 live version。
如果中途只完成了临时 lock，`xmax` 也是 aborted lock/updater 语义。
后续访问会让其他事务继续。
这就是 UPDATE 能依靠事务 abort 回滚逻辑状态，而不是把 page bytes 物理回写到 update 前状态的原因。
物理页面可能留下 aborted tuple version 或 aborted xmax。
语义由 visibility rules 解释。
## 22. 成本模型：一次 UPDATE 放大了哪些资源
一次 heap UPDATE 至少写两处状态。
新 tuple bytes 占用 heap page 空间。
旧 tuple header 被改写。
如果非 HOT，还要为所有 indexes 插入新 index entries。
如果 HOT，普通 indexes 不插新 entry，但 heap page 上仍然多一个 tuple version。
如果 new tuple 变大，可能触发 TOAST。
如果同页空间不足，可能跨页。
如果跨页，可能增加 buffer miss、FSM 查询、额外 page dirty 和更多 WAL。
WAL 成本包括：
- update record 本身。
- 可能的 full-page image。
- 可能的 temporary lock record。
- 可能的 logical old key 或 old tuple。
- 可能的 `NEW_CID` record。
CPU 成本包括：
- modified attrs 计算。
- tuple visibility check。
- CLOG/pg_xact 查询和 hint bit 设置。
- MultiXact member 解析。
- HOT chain 追踪。
- index tuple 构造。
并发成本包括：
- old page exclusive content lock。
- new page content lock。
- tuple lock wait。
- transaction id wait。
- MultiXact wait。
- WAL insertion contention。
- visibility map page pin 和清理。
长期资源成本包括：
- dead tuple version 造成 table bloat。
- dead index entries 造成 index bloat。
- old snapshot 推迟 vacuum horizon。
- old MultiXact 推迟 `pg_multixact` truncate。
- replication slot 或 standby query 可能延迟 cleanup。
这解释了为什么 `UPDATE` workload 对 PostgreSQL 的压力经常比同等行数的 `INSERT` 更复杂。
它既写新数据，又产生旧版本清理债务。
HOT 只减少 index 放大。
它不消除 heap version 链。
## 23. 跨模块连接
executor 调用 table AM update。
heap AM 返回 `TM_Result` 和 `TU_UpdateIndexes`。
这决定 executor 是否重试 EvalPlanQual、报错、更新哪些 indexes。
非 HOT update 要更新所有 indexes。
HOT update 不更新普通 indexes，但可能更新 summarizing indexes。
index scan 命中 HOT root 后，要按 HOT chain 规则在同页追 `t_ctid`。
`xmin/xmax` 的真实语义依赖 pg_xact status、current transaction、subtransaction 和 snapshot。
多个 lockers 或 locker/updater 组合进入 `pg_multixact`。
freeze/vacuum 后续要维护 `relminmxid` 和 `datminmxid`。
前台 update 写 `XLOG_HEAP_UPDATE` 或 `XLOG_HEAP_HOT_UPDATE`。
startup process redo 旧 header 和新 tuple。
walwriter、checkpointer 和 buffer manager 共同维护 WAL-before-data。
UPDATE 清 all-visible。
VACUUM/pruning 后续根据 horizon 清理旧 version 和 HOT chain。
replica identity 决定 old key/old tuple 是否进入 WAL。
catalog update 可能需要 `NEW_CID`。
catalog heap update 必须把 old/new tuple 传给 invalidation 层。
它不属于单一模块。
## 24. 可观测状态：能看见、只能推断、几乎不可见
能直接看见的状态：
- SQL 层的 `ctid`、`xmin`、`xmax` system columns。
- `pageinspect` 里的 `t_xmin`、`t_xmax`、`t_ctid`、`t_infomask`、`t_infomask2`。
- `heap_tuple_infomask_flags()` 解码出来的 raw/combined flags。
- `pg_stat_user_tables.n_tup_upd`。
- `pg_stat_user_tables.n_tup_hot_upd`。
- `pg_stat_user_tables.n_dead_tup` 的估算值。
- `pg_stat_wal` 的 WAL 量变化。
- `EXPLAIN (ANALYZE, BUFFERS, WAL)` 的单 query WAL 估算。
- `pg_locks` 中事务等待和部分 tuple lock wait 的外部迹象。
只能推断的状态：
- 某个 `xmax` 当前是否会被解释为 updater、locker 或 aborted residual。
- 是否刚走过 temporary lock path。
- 某次非 HOT 是因为索引列改变，还是同页空间不足。
- page 是否因为没有 cleanup lock 而错过 pruning。
- MultiXact 中每个 member 的具体 status，除非借助源码断点或低层 inspection。
几乎不可见的状态：
- backend-local combo CID 映射。
- `compute_new_xmax_infomask()` 中间分支。
- critical section 内瞬时状态。
- buffer lock 释放和重拿之间的 race。
- redo 过程中对 Hot Standby query 暴露的一致性窗口。
所以做诊断时不要只看 `xmax`。
也不要只看 `n_tup_hot_upd`。
要把 pageinspect、统计计数、WAL 量、锁等待和源码断点合起来看。
## 25. 实验一：观察 old xmax、new xmin 和 ctid chain
实验目标：
看到一次普通 update 如何产生两个 tuple version。
前提：
需要 superuser 或有权限使用 `pageinspect`。
建议在测试库执行。
```sql
CREATE EXTENSION IF NOT EXISTS pageinspect;

DROP TABLE IF EXISTS upd_chain;
CREATE TABLE upd_chain (
    id int PRIMARY KEY,
    v int,
    pad text
) WITH (autovacuum_enabled = false, fillfactor = 100);

INSERT INTO upd_chain VALUES (1, 10, repeat('a', 100));
SELECT ctid, xmin, xmax, id, v FROM upd_chain;
```
记录初始 `ctid`。
然后执行：
```sql
UPDATE upd_chain SET v = 11 WHERE id = 1;
SELECT ctid, xmin, xmax, id, v FROM upd_chain;
```
再看 raw page：
```sql
SELECT
    lp,
    t_xmin,
    t_xmax,
    t_ctid,
    raw_flags,
    combined_flags
FROM heap_page_items(get_raw_page('upd_chain', 0)) h
CROSS JOIN LATERAL heap_tuple_infomask_flags(h.t_infomask, h.t_infomask2) f
ORDER BY lp;
```
你应该能看到至少两个 line pointer 上有 tuple。
旧 tuple 的 `t_xmax` 是 update 事务。
旧 tuple 的 `t_ctid` 指向新 tuple。
新 tuple 的 `t_xmin` 是 update 事务。
新 tuple 的 `t_ctid` 指向自己。
如果这次是 HOT，旧 tuple flags 中会有 HOT-updated 相关组合。
新 tuple flags 中会有 heap-only 相关组合。
如果没有 HOT，仍然会有 old `t_ctid -> new t_self`。
区别是 index 维护和 heap-only flag。
注意：
SQL system column `xmax` 的显示值不足以说明 lock/update 语义。
要结合 infomask flags。
也要考虑事务是否已经提交和当前 snapshot。
## 26. 实验二：用长事务固定旧 snapshot
实验目标：
证明旧 tuple version 不是“多余垃圾”。
它服务旧 snapshot。
Session A：
```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT ctid, xmin, xmax, id, v FROM upd_chain WHERE id = 1;
```
Session B：
```sql
UPDATE upd_chain SET v = v + 1 WHERE id = 1;
COMMIT;
```
Session A 再查：
```sql
SELECT ctid, xmin, xmax, id, v FROM upd_chain WHERE id = 1;
```
Session A 仍然应该看到旧版本的值。
这是因为 Session A 的 snapshot 早于 Session B 的 update commit。
Session C 或 Session B 新事务中查询，会看到新版本。
这时用 pageinspect 看 heap page，会看到版本链。
再尝试：
```sql
VACUUM upd_chain;
```
只要 Session A 还开着，旧版本通常不能被完全移除。
提交 Session A 后再 vacuum，旧版本才可能被 pruning/vacuum 清掉。
这个实验对应 `HeapTupleSatisfiesVacuum()` 的 horizon 逻辑。
普通查询可见性和可回收性不是同一个问题。
## 27. 实验三：HOT 与非 HOT 的分界
实验目标：
观察同页非索引列 update 和索引列 update 的差别。
```sql
DROP TABLE IF EXISTS hot_demo;
CREATE TABLE hot_demo (
    id int PRIMARY KEY,
    k int,
    v int,
    pad text
) WITH (autovacuum_enabled = false, fillfactor = 80);

CREATE INDEX hot_demo_k_idx ON hot_demo(k);
INSERT INTO hot_demo
SELECT g, g, 0, repeat('x', 50)
FROM generate_series(1, 100) g;

SELECT pg_stat_reset_single_table_counters('hot_demo'::regclass);
```
先更新非索引列：
```sql
UPDATE hot_demo SET v = v + 1 WHERE id BETWEEN 1 AND 20;
SELECT pg_stat_clear_snapshot();
SELECT n_tup_upd, n_tup_hot_upd
FROM pg_stat_user_tables
WHERE relname = 'hot_demo';
```
如果同页空间足够，`n_tup_hot_upd` 应该增长。
再更新索引列：
```sql
UPDATE hot_demo SET k = k + 1000 WHERE id BETWEEN 21 AND 40;
SELECT pg_stat_clear_snapshot();
SELECT n_tup_upd, n_tup_hot_upd
FROM pg_stat_user_tables
WHERE relname = 'hot_demo';
```
第二次 update 会修改 `k`。
`k` 被 btree index 使用。
这会阻止普通 HOT。
`n_tup_upd` 增长，但 `n_tup_hot_upd` 不应按同样数量增长。
可用 `EXPLAIN (ANALYZE, BUFFERS, WAL)` 比较两条 update 的 WAL 和 buffer 行为。
差异会受 full-page writes、checkpoint 位置、page 空间、版本和硬件影响。
不要把一次实验中的 WAL bytes 当成普遍常数。
要看趋势：
HOT 减少 index entry 写入。
非 HOT 增加 index 维护。
二者都会增加 heap tuple versions。
## 28. 实验四：tuple lock 和 MultiXact 边界
实验目标：
看到 `xmax` 可以是 lock-only 或 MultiXact，而不是 update/delete。
准备：
```sql
DROP TABLE IF EXISTS lock_demo;
CREATE TABLE lock_demo (
    id int PRIMARY KEY,
    v int
) WITH (autovacuum_enabled = false);

INSERT INTO lock_demo VALUES (1, 10);
```
Session A：
```sql
BEGIN;
SELECT * FROM lock_demo WHERE id = 1 FOR KEY SHARE;
```
Session B：
```sql
BEGIN;
SELECT * FROM lock_demo WHERE id = 1 FOR KEY SHARE;
```
Session C：
```sql
UPDATE lock_demo SET v = v + 1 WHERE id = 1;
```
这个 update 不修改 key。
它可以与 key-share lockers 兼容。
随后在另一个 session 查看 page：
```sql
SELECT
    lp,
    t_xmin,
    t_xmax,
    t_ctid,
    raw_flags,
    combined_flags
FROM heap_page_items(get_raw_page('lock_demo', 0)) h
CROSS JOIN LATERAL heap_tuple_infomask_flags(h.t_infomask, h.t_infomask2) f
ORDER BY lp;
```
你可能看到 `HEAP_XMAX_IS_MULTI` 和 `HEAP_XMAX_LOCK_ONLY` 相关 flags。
具体是否形成 MultiXact 取决于 timing、是否两个 lockers 都仍然存活、以及 update 观察到的 header 状态。
这个实验的重点不是固定一个输出。
重点是证明：
`xmax` 可以保存 lock-only 信息。
多个 locker 会把 raw `xmax` 变成 MultiXactId。
非 key update 会尽量保留 key-share lockers，而不是无条件等待。
源码断点建议：
```text
break heap_update
break compute_new_xmax_infomask
break GetMultiXactIdHintBits
break MultiXactIdWait
```
观察 `old_infomask`、`old_infomask2`、`mode`、`is_update` 和输出的 `result_xmax`。
## 29. 源码跟读练习
练习一：画出一次普通 HOT update 的状态表。
断点：
```text
heap_update
RelationPutHeapTuple
log_heap_update
heap_xlog_update
```
需要记录：
- old TID。
- new TID。
- old raw `xmax` before/after。
- old `infomask2` before/after。
- new `xmin`。
- new raw `xmax`。
- `use_hot_update`。
- `update_indexes`。
练习二：让 update 走非 HOT。
方法：
更新 indexed column，或者让新 tuple 放不回同一 page。
观察：
- `use_hot_update=false`。
- 旧 tuple 清 `HEAP_HOT_UPDATED`。
- 新 tuple 清 `HEAP_ONLY_TUPLE`。
- `update_indexes=TU_All`。
- WAL record type 是 `XLOG_HEAP_UPDATE`。
练习三：让 update 走 temporary lock path。
方法：
构造需要 TOAST 的新 tuple，或者让 same page 放不下。
断点：
```text
heap_update
compute_new_xmax_infomask
XLogInsert
```
观察：
- 第一次 `compute_new_xmax_infomask(..., is_update=false)`。
- `XLOG_HEAP_LOCK`。
- old tuple 临时 `t_ctid = t_self`。
- 后续真正 update 时 old `t_ctid = new t_self`。
练习四：验证 chain 追踪安全检查。
断点：
```text
heap_get_latest_tid
```
观察：
- `priorXmax` 如何从前一个 tuple 的 update xid 得到。
- 目标 tuple 的 `xmin` 如何与 `priorXmax` 比较。
- line pointer 非 normal 时如何停止。
这个练习对应 `htup_details.h:96-103` 的注释。
## 30. 常见误区
误区一：
`xmax` 有值表示 tuple 已删除。
正确说法：
`xmax` 有值只表示 header 中有删除、更新或锁相关信息。
必须结合 `HEAP_XMAX_LOCK_ONLY`、`HEAP_XMAX_IS_MULTI`、`HEAP_XMAX_INVALID`、`HEAP_KEYS_UPDATED` 和事务状态。
误区二：
`UPDATE` 修改原 tuple 的列值。
正确说法：
heap update 插入新 tuple version。
旧 version 只改 header 和 `t_ctid`。
用户列值通常不会在旧 tuple 上被覆盖。
误区三：
HOT update 不产生新 heap tuple。
正确说法：
HOT 仍然产生新 heap tuple version。
它只是避免普通 index entry 为新 heap-only tuple 重复插入。
误区四：
`t_ctid` 可以直接当链表指针追到最新版本。
正确说法：
追链必须验证 line pointer 状态和目标 tuple `xmin == prior update xmax`。
VACUUM 可能清理或复用目标 slot。
误区五：
`ctid` system column 总能代表逻辑行身份。
正确说法：
`ctid` 是 physical tuple version 位置。
UPDATE 后逻辑行的新版本有新 `ctid`。
HOT root/index TID 与可见 heap-only tuple 之间还可能有 redirect。
误区六：
`n_tup_hot_upd` 高表示没有 bloat。
正确说法：
HOT 减少 index 放大。
heap 版本链仍然存在。
如果 horizon 不推进或 pruning 失败，heap bloat 仍会增长。
## 31. 讨论题
1. 为什么 `UPDATE` 不能只把旧 tuple 的用户列值改掉？
2. 如果 old tuple 的 `xmax` 是 MultiXact，为什么 `HeapTupleHeaderGetUpdateXid()` 可能触发 I/O？
3. 为什么 non-key update 可以和 `FOR KEY SHARE` 并发？
4. 为什么 `heap_update()` 在 parallel mode 下直接 ERROR？
5. 为什么临时 lock 也必须写 WAL？
6. 为什么 HOT chain 不能跨 heap page？
7. 为什么沿 `t_ctid` 追链时要验证 `new.xmin == old.update_xmax`？
8. 为什么 old tuple 已经对当前 query 不可见，也未必能被 VACUUM 立即移除？
## 32. 本节小结
本节的核心链路是：
```text
heap_update()
  -> interpret old tuple visibility and locks
  -> wait/retry if conflicting updater or locker exists
  -> compute old xmax/infomask
  -> prepare new tuple xmin=current xid
  -> handle combo cid
  -> choose same page or new page
  -> decide HOT or non-HOT
  -> insert new tuple version
  -> set old xmax and old t_ctid
  -> clear VM bits
  -> WAL log update
  -> return index maintenance decision
```
核心状态是四组字段。
第一组是 old tuple：
`xmax`、`infomask`、`infomask2`、`cmax`、`t_ctid`。
第二组是 new tuple：
`xmin`、`cmin`、`xmax`、`infomask`、`infomask2`、`t_ctid`。
第三组是 page 状态：
line pointer、free space、`pd_prune_xid`、all-visible flag、page LSN。
第四组是外部状态：
transaction status、snapshot、MultiXact members、visibility map、WAL、index entries。
ownership 上，heap update 临时持有 buffer pins、content locks、VM pins、可能的 tuple lock 和 local tuple copies。
cleanup 上，正常路径释放所有 buffer 和 tuple locks，catalog update 还触发 cache invalidation。
abort 上，new tuple 的 aborted `xmin` 和 old tuple 的 aborted `xmax` 由 visibility rules 收尾。
系统不需要物理撤销 page bytes。
错误路径上，parallel mode、invisible tuple、concurrent pruning、wait/retry、VM pin recheck、temporary lock 和 critical section 是最重要边界。
观测上，`pageinspect` 能看 raw tuple header，`pg_stat_user_tables` 能看 HOT 计数，`EXPLAIN WAL` 和 `pg_stat_wal` 能看 WAL 粗粒度变化。
但 combo CID、MultiXact member status、critical section 中间态和 buffer lock race 需要源码断点或只能推断。
本节可迁移的系统规律是：
当一个系统同时要求旧读者继续读旧状态、新写者发布新状态、索引引用保持可解释、并发锁不爆 shared memory、崩溃恢复可重放时，单个字段不会承载语义。
语义会分散到 version record、link field、status bits、transaction table、lock manager 和 WAL record 之间。
诊断时要跟状态随时间推进，而不是解释某个 raw field 的静态值。
