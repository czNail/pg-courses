# PostgreSQL SnapshotData 的区间、运行集合与命令边界

## 课程定位

前置知识：

你已经理解 XID 不是事务一开始就分配。你已经理解 commit 结果要通过 WAL 和 CLOG/pg_xact 建立持久边界。

你已经理解 SubXID overflow 会让子事务集合从完整枚举退化到 `pg_subtrans` fallback。你已经知道 heap tuple header 里的 `xmin` / `xmax` 只是事务 ID 引用，不是完整可见性答案。

本节唯一主问题：

```text
`SnapshotData` 为什么必须同时保存 `xmin`、`xmax`、`xip`、`subxip`、`suboverflowed` 和 `curcid`，而不能只是一个时间戳或一个事务列表？
```

本节核心矛盾：

一个 MVCC snapshot 必须给每个 tuple 一个稳定、一致、可重复的运行事务视角。但 tuple visibility hot path 不能在每个 tuple 上重新扫描 ProcArray、重新查询所有事务结果，或者保存无界事务历史。

学完本节后，你应该能独立判断：

- 为什么 snapshot 是区间加例外集合。
- 为什么 `xmin` 和 cleanup horizon 相关，但不是“所有旧 tuple 都能删”的同义词。
- 为什么 `xmax` 之后的事务对当前 snapshot 一律按仍在运行处理。
- 为什么 `xip` 保存的是 running top-level XID，而不是 committed 列表。
- 为什么 `subxip` 只是 fast path，`suboverflowed` 会改变判断路径。
- 为什么同一事务内部还需要 `curcid` 来切开命令边界。
- 为什么 `HeapTupleSatisfiesMVCC()` 既看 snapshot，又看 tuple hint bit 和 CLOG。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

本节不展开：

- READ COMMITTED 语句级生命周期。
- REPEATABLE READ 的事务级 snapshot。
- ActiveSnapshot / RegisteredSnapshot 的完整资源释放细节。
- VACUUM cleanup horizon 的完整计算。

这里先把一个 `SnapshotData` 本身为什么长成现在这样讲清楚。

## 1. 本节在总主线中的位置

前六节课回答了事务 ID 和事务结果从哪里来。第一节说明一个事务什么时候真正拥有 XID。

第三节说明 commit WAL 和 CLOG/pg_xact 状态之间的顺序。第六节说明 SubXID overflow 后为什么仍然能判断可见性，只是成本会退化。

现在要把这些状态合成一个读者视角。一个 heap tuple 上可能有 `xmin`。

一个 heap tuple 上可能有 `xmax`。这两个字段可能指向顶层事务。

也可能指向子事务。它们可能已经提交。

可能已经回滚。也可能在 snapshot 创建时仍在运行。

如果每次判断 tuple 都去全局事务系统问一次最新状态，结果会变快照外的实时读。如果只用事务提交时间戳，系统又无法表达“某些更老的事务还没结束”。

如果只保存完整 running 列表，子事务和 backend 数一多就会让 snapshot 大小失去上界。`SnapshotData` 的设计不是为了漂亮。

它是 PostgreSQL 在 MVCC correctness、ProcArray 扩展性、heap visibility 热路径和本事务自可见性之间做出的组合。本节只追一条主链路。

一个查询开始时获取 MVCC snapshot。`GetSnapshotData()` 从 ProcArray 复制运行事务集合。

`XidInMVCCSnapshot()` 把 tuple XID 对照这个集合。`HeapTupleSatisfiesMVCC()` 再把 snapshot 结果、当前事务命令边界和 CLOG/hint bit 合并成 tuple 可见性。

这条链路之后，下一节才能讨论 READ COMMITTED 为什么每条语句重取一次 snapshot。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
GetSnapshotData() 在一个时刻复制 ProcArray 中的 running top-level XID 和 cached SubXID；
SnapshotData 用 xmin/xmax 给出区间边界，用 xip/subxip 描述区间内仍 running 的例外；
suboverflowed 决定是否需要 pg_subtrans fallback；
curcid 再把同一事务内部的命令顺序切开。
```

本节的核心矛盾不是“snapshot 里字段很多”。

真正的矛盾是：查询需要一个创建后稳定的可见性世界，但 heap visibility hot path 又不能在每个 tuple 上重新扫描 ProcArray、保存完整历史或依赖提交时间戳。

因此 `SnapshotData` 选择了一个压缩结构：

```text
区间边界
  + running exception set
  + subtransaction fallback flag
  + current-command cutoff
```

后面的 runtime 现象、源码 walkthrough、成本模型都围绕这个压缩结构展开。

## 3. 核心文件分工与阅读顺序

| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/utils/snapshot.h` | 先看 `SnapshotData` 字段、snapshot 类型和 `curcid` 边界。 |
| 2 | `src/backend/storage/ipc/procarray.c` | 再读 `GetSnapshotData()` / `GetSnapshotDataReuse()` 如何复制 running XID 集合。 |
| 3 | `src/backend/utils/time/snapmgr.c` | 跟 `XidInMVCCSnapshot()` 如何消费 `xmin/xmax/xip/subxip/suboverflowed`。 |
| 4 | `src/backend/access/heap/heapam_visibility.c` | 看 `HeapTupleSatisfiesMVCC()` 如何把 snapshot、hint bit 和 CLOG 合并。 |
| 5 | `src/include/access/xact.h` | 对照隔离级别和事务 accessor 的公开边界。 |
| 6 | `src/backend/access/transam/xact.c` | 最后看 command id 推进与 `curcid` 更新来源。 |

## 4. 一个 runtime 现象先定锚

先看一个可以复现的现象。它不需要任何源码修改。

建一张普通表。一个会话打开事务并插入一行但不提交。

另一个会话启动一个查询。在第二个会话查询运行期间，第一个会话提交。

第二个会话的这一次查询仍然不会把那一行纳入自己的扫描结果。下一条语句才可能看到。

这个现象不是 READ COMMITTED 的全部。本节只借它说明 `SnapshotData` 的形状。

在第二个会话创建 snapshot 的时刻，第一个会话的 top-level XID 还在运行。`GetSnapshotData()` 会把这个 XID 放进 `snapshot->xip`。

以后即使第一个事务真实提交了，这个 snapshot 仍然把它当作“创建 snapshot 时 running”。`HeapTupleSatisfiesMVCC()` 遇到那行的 `xmin` 时会调用 `XidInMVCCSnapshot()`。

`XidInMVCCSnapshot()` 发现 `xmin` 在 `xip` 中。于是这行对该 snapshot 不可见。

关键点是：

snapshot 保存的是创建时的运行集合。它不是全局事务状态的实时视图。

这能解释“单条查询内部稳定”。但它还不能解释“为什么不是一个时间戳”。

因为 PostgreSQL 的 XID 分配顺序不是事务完成顺序。一个更小的 XID 可以在 snapshot 创建时还没提交。

一个更大的 XID 可以已经提交。单个时间点或单个最大已提交 ID 都无法表达这个交错。

因此 snapshot 需要区间。还需要区间里的例外集合。

还需要对本事务内命令边界的特殊处理。这就是本节主线。

## 5. `SnapshotData` 不是事务状态本身

`SnapshotData` 是 backend-local 的读视角。它不是 shared memory 中的全局事实。

它保存的是某一刻从全局状态复制出来的一份判断材料。`xip` 和 `subxip` 指针指向当前 backend 私有内存。

其他 backend 不能直接使用这份指针。snapshot 创建以后，它的运行集合不再跟随 ProcArray 实时变化。

这正是 MVCC snapshot 的意义。如果一个事务在 snapshot 创建后提交，旧 snapshot 不会把它从 `xip` 中删除。

如果一个事务在 snapshot 创建后分配了新 XID，新 XID 通常会因为 `xid >= xmax` 被旧 snapshot 当作 still-running。因此 snapshot 的字段不是“最新事务系统状态”。

它们是“这个读者在创建 snapshot 时承诺要使用的可见性边界”。普通 MVCC snapshot 中最核心的字段组合是：

`xmin` `xmax`

`xip` `subxip`

`suboverflowed` `takenDuringRecovery`

`curcid`此外还有生命周期字段：

`active_count` `regd_count`

`copied` `snapXactCompletionCount`

这些字段不参与 tuple 可见性本身。但它们决定 snapshot 能不能安全地被 executor、portal 或 resource owner 持有。

这点后续课程会展开。本节先记住一句话：

raw field 不是语义。`xmin + xmax + xip + subxip + suboverflowed + curcid + snapshot type + lifecycle` 才是语义。

## 6. 为什么不是一个时间戳

看起来最简单的 MVCC snapshot 是一个时间戳。例如“读取 T 时刻以前提交的所有版本”。

PostgreSQL 没有把普通 MVCC snapshot 建成这个形状。原因不是没有 timestamp。

原因是 tuple header 存的是 XID。heap visibility hot path 面对的是 `xmin` 和 `xmax`。

它需要快速判断这些 XID 在 snapshot 创建时是否 running。提交时间戳不是普通 tuple 可见性的主索引。

并且 PostgreSQL 的普通事务提交时间戳功能本身也是可选能力，不是 MVCC 判断的基础。更关键的是，时间戳无法直接表达并发事务交错。

设想三个事务。T1 分配 XID 100，长时间不提交。

T2 分配 XID 101，很快提交。T3 分配 XID 102，并创建 snapshot。

T3 的 snapshot 应该能看见 T2 的提交。但不能看见 T1 的未提交写入。

如果 snapshot 只保存一个“最大可见 XID = 101”，它会误以为 100 也可见。如果保存一个“当前时间戳”，还需要把 tuple XID 映射到 commit timestamp。

这会把 heap scan 的 hot path 推向事务结果查询。如果 T1 后来提交，旧 snapshot 仍然不能因此看到 T1。

所以 snapshot 不能只描述完成时间。它必须描述“创建 snapshot 时哪些 XID 仍在运行”。

这就是 `xip` 存在的根本原因。

## 7. 为什么不是一个完整事务列表

另一个简单方案是保存完整事务列表。把所有正在运行的 XID 全部塞进 snapshot。

包括顶层事务。包括所有子事务。

再让 visibility check 只做成员判断。PostgreSQL 的 common case 确实接近这个模型。

但它不能把这个模型无限扩展。顶层事务数量受 `max_connections`、prepared transaction、后台进程等影响。

子事务数量则可以被 savepoint、PL/pgSQL exception block 和复杂过程放大。每个 `PGPROC` 只缓存有限数量的 SubXID。

本源码基线中 `PGPROC_MAX_CACHED_SUBXIDS` 是 64。第 65 个以后不会继续完整公开到 `PGPROC->subxids.xids[]`。

这不是子事务失败。这是 shared memory fast path 的容量边界。

因此 snapshot 保存 `subxip`。同时还必须保存 `suboverflowed`。

`suboverflowed == false` 时，不在 `subxip` 里基本可以当作不在 running subxid fast path 中。`suboverflowed == true` 时，不在 `subxip` 里不能证明它不 running。

`XidInMVCCSnapshot()` 需要把候选 XID 通过 `SubTransGetTopmostTransaction()` 规范化到 top-level XID。然后再查 `xip`。

这就是 PostgreSQL 的取舍：

保存完整 top-level running 集合。尽量保存 SubXID fast path。

当 SubXID 信息不完整时，显式把不完整性写入 snapshot。后续 hot path 在必要时走 fallback。

完整事务列表看起来简单。但它会把 shared memory 和 snapshot 大小变成无界。

PostgreSQL 选择了有界 fast path 加 correctness fallback。

## 8. `xmin`：不是删除许可，而是范围下界

`SnapshotData.xmin` 在 `snapshot.h` 中表达一个下界。普通 MVCC 语义里，所有 XID `< xmin` 对这个 snapshot 来说不再 running。

这句话经常被误读成“这些 XID 的 tuple 都可见”。更准确的说法是：

`xmin` 之前的 XID 不需要再查 `xip` 来判断 running。它们仍然可能 committed。

也仍然可能 aborted。heap visibility 还要看 tuple hint bit 或 CLOG/pg_xact。

如果 `xmin` 是插入事务，并且该事务 committed，插入版本可能可见。如果 `xmin` 是插入事务，并且该事务 aborted，插入版本不可见。

如果 `xmax` 是删除事务，并且该事务 committed，旧版本通常不可见。如果 `xmax` 是删除事务，并且该事务 aborted，旧版本仍然可见。

所以 `SnapshotData.xmin` 不是最终 visibility answer。它是 running-set 搜索优化。

`XidInMVCCSnapshot()` 先做范围判断。如果候选 XID `< snapshot->xmin`，直接返回 false。

false 的意思是“按这个 snapshot 不在运行”。它不是“tuple 可见”。

最终可见性由 `HeapTupleSatisfiesMVCC()` 继续判断。`GetSnapshotData()` 计算 `xmin` 时从 `xmax` 初始化。

然后考虑当前 backend 自己的 XID。再扫描 ProcArray 中 relevant backend 的 top-level XID。

如果某个 running XID 更老，就把 `xmin` 拉低。这也是为什么长事务会拖住 cleanup horizon。

只要某个 backend 的 snapshot 需要旧 XID，系统就不能简单回收所有旧版本。但本节仍然要把两件事分开：

snapshot 的 `xmin` 是这个读者的可见性下界。VACUUM 的可移除边界还要结合全局 horizon、catalog/shared relation、replication slot、prepared xact 等因素。

不要把某个 backend 的 `backend_xmin` 直接等同于某个 tuple 可以删除。

## 9. `xmax`：为什么未来和并发晚到者都不可见

`SnapshotData.xmax` 是普通 MVCC snapshot 的上界。`GetSnapshotData()` 中它来自 `TransamVariables->latestCompletedXid + 1`。

源码注释说 `xmax` 永远是 latest completed xid 加一。对 `XidInMVCCSnapshot()` 来说，任何候选 XID `>= snapshot->xmax` 都按 still-running 返回 true。

这不是因为这些事务一定真实还在运行。而是因为它们在 snapshot 创建时不可能已经作为 completed 事务进入这个读视角。

一个事务可能在 snapshot 创建后才分配 XID。它自然不应该被旧 snapshot 看见。

一个事务可能在 snapshot 创建后很快提交。旧 snapshot 也不应该因为它提交了就突然看见它。

因此 `xmax` 把“snapshot 创建之后才进入可见性世界的事务”全部挡住。这能避免每次 visibility check 都去问“这个更大的 XID 是否后来提交了”。

它也解释了为什么 snapshot 不是 committed-list。PostgreSQL 不需要列出所有 committed XID。

它只需要：

低于 `xmin` 的 XID 不再 running。高于等于 `xmax` 的 XID 对我仍算 running。

中间区间用 running set 做例外判断。这个模型把大量旧 tuple 的判断变成范围检查。

把大量新事务的判断也变成范围检查。只有 `[xmin, xmax)` 区间内的候选 XID 需要数组搜索或 fallback。

这就是区间设计的性能意义。

## 10. `xip`：区间里的 running top-level 例外

`xip` 保存普通运行期的 running top-level XID。它不是所有事务。

它不是所有 committed 事务。它也不保存当前 backend 自己的 XID。

`GetSnapshotData()` 持有 `ProcArrayLock` shared。它扫描 `ProcGlobal->xids[]` 这样的 dense array。

遇到 `InvalidTransactionId` 就跳过。遇到自己也跳过。

遇到 `xid >= xmax` 也跳过，因为这种 XID 之后会被范围规则当作 running。遇到 lazy vacuum 或 logical decoding 特殊状态也有对应跳过或另行处理。

剩下的 top-level XID 被加入 `snapshot->xip`。这组 XID 的意义很窄：

它们在 snapshot 创建时仍在运行。并且它们落在 `[xmin, xmax)`。

后续 `XidInMVCCSnapshot()` 在非 overflow 情况下会搜索 `xip`。如果候选 XID 在 `xip`，返回 true。

这表示该 XID 对这个 snapshot 仍在运行。对插入者 `xmin` 来说，通常意味着插入版本不可见。

对删除者 `xmax` 来说，通常意味着删除尚未生效，旧版本仍可见。这也说明不要把 `xip` 当成“不可见 tuple 列表”。

同一个 running XID 出现在 `xip` 中，对 `xmin` 和 `xmax` 的影响方向不同。visibility 函数必须知道它正在解释的是插入事务还是删除事务。

## 11. `subxip` 与 `suboverflowed`：完整性标志比数组本身更重要

子事务让 snapshot 复杂起来。heap tuple header 里可能保存 SubXID。

例如某个 savepoint 内执行了 INSERT。tuple 的 `xmin` 就可能是子事务 XID。

如果 snapshot 只保存 top-level XID，遇到这个 SubXID 时就无法直接判断它是否属于某个 running top transaction。PostgreSQL 的 fast path 是把每个 backend 缓存的 running SubXID 复制到 `snapshot->subxip`。

如果所有 relevant backend 都没有 overflow，`subxip` 是完整的 running subtransaction fast path。`XidInMVCCSnapshot()` 会先搜索 `subxip`。

找到了就返回 true。没找到再搜索 `xip`。

如果某个 backend 的 SubXID cache overflow，`GetSnapshotData()` 设置 `snapshot->suboverflowed = true`。这时 `subxip` 不再是完整集合。

`XidInMVCCSnapshot()` 不再用“不在 subxip”来证明不 running。它会调用 `SubTransGetTopmostTransaction()`。

候选 XID 被沿 `pg_subtrans` parent chain 转成 top-level XID。然后再查 `xip`。

这个设计有一个重要后果：

`suboverflowed` 是 snapshot 语义的一部分。它不是附加诊断字段。

如果丢掉这个 bool，系统会把未缓存 SubXID 错判为不在运行。这会直接破坏 MVCC correctness。

也要注意它不是 per-tuple 或 per-backend 的精确标志。只要 snapshot 采集时发现 relevant overflow，它就是 true。

tuple header 不告诉你这个 XID 来自哪个 backend。所以 visibility check 只能在整个 snapshot 层面保守处理。

## 12. `curcid`：为什么同一事务内部还需要命令切口

到这里，snapshot 已经能描述其他事务了。但它还不能描述当前事务自己的写入。

`GetSnapshotData()` 不会把当前 backend 自己的 XID 放进 `xip`。`XidInMVCCSnapshot()` 的注释也说明，调用者通常会先检查 `TransactionIdIsCurrentTransactionId()`。

当前事务的可见性不是靠 `xip` 判断。它靠 command id。

`SnapshotData.curcid` 记录当前命令开始时的 command id 边界。`HeapTupleSatisfiesMVCC()` 遇到当前事务插入的 tuple 时，会比较 tuple 的 `cmin` 和 `snapshot->curcid`。

如果 `cmin >= curcid`，说明这个 tuple 是当前扫描开始后插入的。它对这个 snapshot 不可见。

如果当前事务删除了 tuple，也会用 `cmax` 和 `curcid` 判断删除发生在扫描前还是扫描后。这解释了一个常见现象：

同一个事务中，前一条命令插入的行，后一条命令能看到。但同一条语句内部，主查询通常看不到同一命令刚写入基表的新版本。

这不是隔离级别之间的差异。这是同一事务内部 command cutoff 的规则。

`CommandCounterIncrement()` 在命令之间推进 `currentCommandId`。如果当前 command id 曾被用来标记 tuple，它会加一。

然后调用 `SnapshotSetCommandId()` 把新的 command id 传播到静态 snapshot。`GetSnapshotDataReuse()` 复用 snapshot 内容时，也会更新 `snapshot->curcid`。

因此 `curcid` 是 snapshot 的必要字段。没有它，PostgreSQL 只能在同一事务里选择“永远看不到自己的写入”或“扫描中途看到自己刚写的行”。

这两个选择都不符合 SQL 执行需要。

## 13. `GetSnapshotData()` 主流程

现在把字段放回创建流程。`GetSnapshotData(snapshot)` 要求传入的 snapshot 通常是静态存储。

第一次使用时，它会给 `xip` 和 `subxip` 分配数组。分配发生在拿 `ProcArrayLock` 之前。

这样 out-of-memory 不会发生在持锁区中。之后函数拿 `ProcArrayLock` shared。

先尝试 `GetSnapshotDataReuse(snapshot)`。如果上一次构造 snapshot 后没有带 XID 的事务完成，旧的 running-set 内容仍然可用。

这时函数只需要重新安装 `MyProc->xmin`、更新 `RecentXmin`、更新 `curcid` 和生命周期计数。如果不能复用，就读取 `latestCompletedXid`。

计算 `xmax = latestCompletedXid + 1`。把 `xmin` 初始化为 `xmax`。

把自己的 XID纳入 `xmin` 计算，但不加入 `xip`。然后扫描 ProcArray。

每个 proc 只读取一次 top-level XID。没有 XID 的 backend 跳过。

自己的 backend 跳过。`xid >= xmax` 的 backend 跳过。

相关 backend 的 top-level XID 加入 `xip`。如果尚未发现 suboverflow，就检查该 backend 的 SubXID 状态。

没有 overflow 时复制 cached SubXID 到 `subxip`。复制前使用 read barrier，和 SubXID 发布时的 write barrier 配对。

发现 overflow 时设置本地 `suboverflowed`。recovery 分支不同。

在 hot standby 中，系统从 KnownAssignedXids 获取 XID。因为恢复过程中不总是知道哪些是 top-level、哪些是 subxact，普通 `xip` 会保持空，XID 主要放进 `subxip`。

`takenDuringRecovery` 让 `XidInMVCCSnapshot()` 使用不同解释。离开锁前，函数还读取 replication slot xmin 等信息。

释放锁后，更新 `GlobalVis*` 的近似边界。最后写回 snapshot 字段：

`xmin` `xmax`

`xcnt` `subxcnt`

`suboverflowed` `snapXactCompletionCount`

`curcid` `active_count`

`regd_count` `copied`

这条流程解释了两个边界。snapshot 内容在持 `ProcArrayLock` 时形成。

snapshot 消费时不继续持这个锁。PostgreSQL 用一次复制换来后续 heap scan 的无锁可见性判断。

## 14. `GetSnapshotDataReuse()` 为什么成立

`GetSnapshotDataReuse()` 是本基线源码里必须注意的优化。它依赖 `TransamVariables->xactCompletionCount`。

这个计数在带 XID 的事务完成时、持 `ProcArrayLock` exclusive 的路径中推进。`GetSnapshotDataReuse()` 在持 `ProcArrayLock` shared 时读取当前计数。

如果当前计数等于 snapshot 上次构造时记录的 `snapXactCompletionCount`，说明从上次构造到现在没有带 XID 的事务完成。在这个条件下，重新扫描 ProcArray得到的 running-set 内容应与旧 snapshot 一致。

所以可以复用 `xip` 和 `subxip`。但 reuse 不是简单返回旧对象。

函数仍然要处理当前 backend 的 snapshot 持有边界。如果 `MyProc->xmin` 无效，会重新安装 `snapshot->xmin`。

`RecentXmin` 也会更新。`snapshot->curcid` 会用 `GetCurrentCommandId(false)` 更新。

`active_count` 和 `regd_count` 会清零。`copied` 会设回 false。

这说明 snapshot 有两类内容。一类是 running-set 内容。

它可以在事务完成计数没变时复用。另一类是调用者当前命令和生命周期内容。

它每次都要重新设置。如果把 snapshot 当成不可变值对象，就会误解这个优化。

它更像一个可复用的 backend-local 工作区。

## 15. `XidInMVCCSnapshot()` 的判断顺序

`XidInMVCCSnapshot(xid, snapshot)` 不返回 tuple 可见性。它只回答这个 XID 按该 snapshot 是否仍在运行。

第一步是范围判断。如果 `xid < snapshot->xmin`，返回 false。

如果 `xid >= snapshot->xmax`，返回 true。这两个判断过滤掉大量 tuple。

只有落在 `[xmin, xmax)` 内的 XID 才继续。普通运行期且未 overflow：

先搜 `subxip`。再搜 `xip`。

都没有则返回 false。普通运行期且 overflow：

先用 `SubTransGetTopmostTransaction(xid)` 把候选 XID 转成 top-level XID。如果转换后小于 `xmin`，返回 false。

然后只搜 `xip`。recovery snapshot 的结构不同。

`xip` 通常为空。XID 主要保存在 `subxip`。

overflow 时同样需要先走 `pg_subtrans` 转换。最后搜索 `subxip`。

这段代码最重要的不是数组搜索函数。而是判断顺序。

先用区间减少工作。再根据 `takenDuringRecovery` 选择解释方式。

再根据 `suboverflowed` 决定是否需要 `pg_subtrans` fallback。最后才搜索数组。

## 16. `HeapTupleSatisfiesMVCC()` 如何消费 snapshot

`HeapTupleSatisfiesMVCC()` 是本节从 snapshot 回到 tuple 的核心入口。它的第一条重要约束是 snapshot 必须 active 或 registered。

源码里有断言检查 `regd_count > 0 || active_count > 0`。这不是 visibility 逻辑本身需要 refcount。

而是防止调用者使用可能被释放或失效的 snapshot。然后它处理插入事务 `xmin`。

如果 tuple 已有 `HEAP_XMIN_INVALID`，不可见。如果插入事务是当前事务，就用 `cmin` 和 `snapshot->curcid` 判断。

如果插入事务按 snapshot 仍在运行，返回不可见。如果不在运行，再查它是否 committed。

committed 时可以设置 committed hint bit。否则设置 invalid hint bit 并返回不可见。

如果 `xmin` 已有 committed hint bit，仍然不能直接认为可见。函数还会调用 `XidInMVCCSnapshot()`。

因为一个事务可能已经真实提交，并被某个访问者设置了 hint bit。但对旧 snapshot 来说，它在创建 snapshot 时仍然 running。

这种情况下仍要把它当作 running。处理删除或更新事务 `xmax` 时方向相反。

如果 `xmax` 无效或只是锁，不影响可见。如果删除事务是当前事务，用 `cmax` 和 `curcid` 判断删除是否发生在扫描开始前。

如果删除事务按 snapshot 仍在运行，旧版本仍可见。如果删除事务 committed，旧版本不可见。

如果删除事务 aborted，旧版本可见。这解释了为什么 `XidInMVCCSnapshot()` 不能叫 `TupleVisibleInSnapshot()`。

同一个 running 判断要根据它落在 `xmin` 还是 `xmax` 上翻译成不同结果。

## 17. 可复现 SQL 现象一：区间和 running set

下面实验展示 snapshot 不是最新全局状态。它要求两个会话。

会话 A：

```sql
DROP TABLE IF EXISTS mvcc07;
CREATE TABLE mvcc07(id int primary key, note text);
INSERT INTO mvcc07 VALUES (1, 'old');
BEGIN;
INSERT INTO mvcc07 VALUES (2, 'from A, not committed yet');
SELECT pg_backend_pid();
-- 保持事务打开，不要提交。
```

会话 B：

```sql
BEGIN;
SELECT pg_current_snapshot() AS s1;
SELECT count(*) FROM mvcc07, pg_sleep(5);
SELECT pg_current_snapshot() AS s2;
COMMIT;
```

在 B 的 `pg_sleep(5)` 期间让会话 A 执行：

```sql
COMMIT;
```

B 的这一次 `SELECT count(*)` 不会因为 A 在扫描期间提交而突然多出一行。如果 B 随后再执行一条新的查询，在 READ COMMITTED 下通常能看到第二行。

本节只解释第一条查询内部为什么稳定。B 创建 snapshot 时，A 的 XID 在运行。

这个 XID 落进 `xip` 或被 `xmax` 范围挡住。`HeapTupleSatisfiesMVCC()` 遇到 A 插入的 tuple `xmin` 时，按这个 snapshot 把它视为 still-running。

A 后来提交只会改变真实事务状态。不会修改 B 已经拿到的 snapshot。

所以单条查询内部稳定来自 snapshot 的复制语义。不是来自阻塞提交。

不是来自表锁。也不是来自每行重新读取最新 CLOG。

## 18. 可复现 SQL 现象二：同一事务内的 command cutoff

下面实验展示 `curcid`。仍然只需要普通 SQL。

```sql
DROP TABLE IF EXISTS mvcc07_cid;
CREATE TABLE mvcc07_cid(id int primary key);
BEGIN;
INSERT INTO mvcc07_cid VALUES (1);
SELECT count(*) FROM mvcc07_cid;
WITH ins AS (
  INSERT INTO mvcc07_cid VALUES (2)
  RETURNING id
)
SELECT
  (SELECT count(*) FROM mvcc07_cid) AS count_from_base_table,
  (SELECT count(*) FROM ins) AS count_from_returning;
COMMIT;
```

第一条 `SELECT count(*)` 能看到前一条命令插入的 `id = 1`。这是命令之间执行了 command counter 推进。

`currentCommandId` 增加后，新的 snapshot `curcid` 边界允许看到之前命令的写入。数据修改 CTE 这一条语句里，主查询对基表的读取仍使用同一语句的 snapshot/command boundary。

刚由同一命令写入的基表版本不会通过重新扫描基表变成可见。但 `RETURNING` 结果是数据修改节点显式返回的结果流。

它不是靠基表 visibility 重新读出来。这组现象能把两个概念分开：

前后两条命令之间，自写入通过 CCI 变得可见。同一条命令内部，`curcid` 阻止扫描中途看到自己刚写的基表版本。

源码对应点在 `HeapTupleSatisfiesMVCC()`。当前事务插入路径比较 `HeapTupleHeaderGetCmin(tuple)` 和 `snapshot->curcid`。

当前事务删除路径比较 `HeapTupleHeaderGetCmax(tuple)` 和 `snapshot->curcid`。

## 19. 观测入口：能看到什么，不能看到什么

SQL 层可以看到一部分 snapshot 边界。`pg_current_snapshot()` 返回外部化的 snapshot 表示。

`pg_snapshot_xmin()` 可以取出 xmin。`pg_snapshot_xmax()` 可以取出 xmax。

`pg_snapshot_xip()` 可以列出 snapshot 外部表示里的 in-progress XID。历史兼容函数 `txid_current_snapshot()` 也存在。

这些函数适合展示区间模型。它们不完整暴露内部 `SnapshotData`。

例如 `curcid` 不会作为 SQL snapshot 输出的一部分出现。`suboverflowed` 也不是普通 SQL 直接可见字段。

`takenDuringRecovery` 是内部解释标志。`active_count` / `regd_count` 是生命周期字段。

`pg_stat_activity.backend_xmin` 能观察 backend 暴露给全局 cleanup 的 xmin。它不是某个具体 query 的完整 `SnapshotData` dump。

`backend_xid` 能看到该 backend 当前顶层事务 XID。没有 XID 的事务不会因此出现在 tuple visibility 的 XID 世界里。

如果要观察 `XidInMVCCSnapshot()` 是否走 overflow fallback，通常需要断点、debug log、DTrace/perf 或临时计数器。如果要观察 `HeapTupleSatisfiesMVCC()` 的分支，需要 gdb、tracepoint 或构造 hint bit/CLOG 状态实验。

不要把 `pg_current_snapshot()` 输出等同于所有内部字段。它是用户态可读的投影。

不是 `SnapshotData` 原样序列化。

## 20. 错误路径与 fallback

第一个错误路径是 snapshot 数组分配失败。`GetSnapshotData()` 第一次使用静态 snapshot 时会为 `xip` 和 `subxip` 调用 `malloc`。

失败会报 out of memory。这发生在拿 `ProcArrayLock` 之前。

因此不会留下持锁状态。第二个 fallback 是 `GetSnapshotDataReuse()` 失败。

只要 `snapXactCompletionCount` 为 0，或者全局 completion count 改变，就必须重新扫描 ProcArray。这不是错误。

这是复用条件不成立。第三个 fallback 是 SubXID overflow。

`suboverflowed` 不会让 snapshot 无效。它让 `XidInMVCCSnapshot()` 在必要时走 `pg_subtrans`。

正确性保住了。成本转移到了 tuple visibility hot path。

第四个边界是 recovery snapshot。`takenDuringRecovery` 改变 `xip` 和 `subxip` 的解释。

如果课程或调试时把 recovery snapshot 当成普通 snapshot，会把 `xip` 为空误读为没有 running XID。第五个边界是 command id overflow。

`CommandCounterIncrement()` 如果把 command id 推到无效值，会报错。这属于事务内命令数量的极端边界。

普通 workload 很少碰到。但它说明 `curcid` 不是无限计数器。

第六个边界是使用未注册 snapshot。生产版本不一定靠断言保护你。

代码约定要求调用者在非平凡工作前 `RegisterSnapshot()` 或 `PushActiveSnapshot()`。这就是为什么 `HeapTupleSatisfiesMVCC()` 开头强调 active/registered。

## 21. 成本模型

snapshot 的成本有两个阶段。创建阶段主要随 backend 数和 SubXID cache 大小增长。

消费阶段主要随扫描 tuple 数、XID 分布和 overflow 状态增长。`GetSnapshotData()` 要扫描 ProcArray。

相关成本受 `arrayP->numProcs`、active XID 数、SubXID 数、是否 recovery、是否能复用影响。它持 `ProcArrayLock` shared。

高并发提交路径需要 exclusive 结束事务状态。因此 snapshot 获取和事务结束之间存在同步边界。

`GetSnapshotDataReuse()` 减少重复扫描。当没有带 XID 的事务完成时，旧 running set 可以复用。

这对高频短查询有意义。但它不能消除所有 snapshot 成本。

`curcid`、refcount、`MyProc->xmin` 仍要更新。`XidInMVCCSnapshot()` 的成本通常很低。

大量 tuple 会被 `xid < xmin` 或 `xid >= xmax` 范围判断过滤。落在区间内才搜索数组。

数组搜索使用 `pg_lfind32()` 这样的局部优化。但它仍然是每个候选 tuple 的工作。

SubXID overflow 会把部分判断推到 `pg_subtrans`。这可能引入 SLRU lookup、cache miss 和 parent chain 遍历。

这个成本常常由读者承担。不一定由制造大量 savepoint 的事务承担。

hint bit 也影响成本。如果 tuple 上没有 committed/invalid hint bit，visibility 需要查询 CLOG/pg_xact。

但如果按旧 snapshot 某个 XID still-running，`HeapTupleSatisfiesMVCC()` 故意不去更新 hint bit。这样避免对高流量共享状态做无意义查询。

代价是后续足够新的 snapshot 才能设置 hint bit。成本传播路径可以概括为：

backend 数增加，ProcArray scan 成本上升。长事务存在，`xmin` 下界被拉低，旧版本和 CLOG 压力延长。

SubXID overflow 出现，tuple visibility fallback 成本上升。tuple 数增加，`XidInMVCCSnapshot()` 的每行成本被放大。

hint bit 未设置，CLOG 查询和 buffer dirty 机会增加。

## 22. 跨模块边界

`SnapshotData` 连接多个模块。第一个边界是 ProcArray。

`GetSnapshotData()` 从 ProcArray 复制 running top XID 和 SubXID 状态。它依赖事务结束路径先写事务结果，再从 ProcArray 清除 running 身份。

否则 snapshot 可能把一个结果还不可解释的事务当成已结束。第二个边界是 CLOG/pg_xact。

snapshot 只回答 running 与否。不回答 committed 或 aborted。

当 `XidInMVCCSnapshot()` 返回 false，`HeapTupleSatisfiesMVCC()` 还要通过 hint bit 或 `TransactionIdDidCommit()` 解释结果。第三个边界是 `pg_subtrans`。

当 `suboverflowed` 为 true，SubXID 需要映射到 top-level XID。`pg_subtrans` 只提供 parent chain。

它不提供事务结果。第四个边界是 command counter。

当前事务的可见性不能靠 ProcArray。它靠 `curcid`、`cmin`、`cmax` 和 combo CID 相关机制。

第五个边界是 VACUUM / GlobalVis。`GetSnapshotData()` 会维护 `RecentXmin` 和 `GlobalVis*` 近似边界。

这些边界影响 cleanup。但普通 MVCC tuple 可见性不能简化成 vacuum horizon 判断。

第六个边界是 recovery。Hot standby 使用 KnownAssignedXids。

它让 snapshot 字段布局和普通运行期不同。因此课程、断点和工具都必须先区分 `takenDuringRecovery`。

## 23. 常见误区

误区一：

把 `xmin` 读成“所有小于它的 tuple 都可见”。正确说法是小于它的 XID 按该 snapshot 不在运行。

最终可见性还要看 committed/aborted 以及它是 `xmin` 还是 `xmax`。误区二：

把 `xmax` 读成“最大可见事务”。更准确的是所有 `xid >= xmax` 对该 snapshot 视为 still-running。

它是上界，不是 commit 列表末尾。误区三：

把 `xip` 当成不可见行列表。`xip` 是 running top-level XID 列表。

对插入者和删除者的效果不同。误区四：

认为 `subxip` 为空说明没有 running SubXID。如果 `suboverflowed` 为 true，这个结论不成立。

如果 `takenDuringRecovery` 为 true，字段解释也不同。误区五：

把 `pg_current_snapshot()` 当成内部 `SnapshotData` 完整输出。它只是外部类型 `pg_snapshot` 的投影。

看不到 `curcid`、refcount、recovery 标志等内部状态。误区六：

把 hint bit 当成 snapshot 语义。hint bit 是 tuple header 上对事务结果的缓存。

旧 snapshot 仍然可以把一个已有 committed hint bit 的 XID 当作 still-running。误区七：

以为同一事务里所有自写入都自动可见。自写入要过 command cutoff。

`curcid` 决定当前扫描是否能看到该命令之前的写入。

## 24. 课堂实验

实验一：画出 snapshot 区间。打开两个会话。

在会话 A 中 `BEGIN` 并执行一次写入，保持不提交。在会话 B 中执行 `SELECT pg_current_snapshot();`。

再用 `pg_snapshot_xmin()`、`pg_snapshot_xmax()`、`pg_snapshot_xip()` 拆开输出。然后提交 A。

在 B 中再次执行 `SELECT pg_current_snapshot();`。对比两次输出。

把变化回到 `GetTransactionSnapshot()` 和 `GetSnapshotData()` 的调用边界解释。实验二：观察 long transaction 对 `backend_xmin` 的影响。

会话 A 执行 `BEGIN; SELECT pg_current_snapshot();` 后保持打开。会话 B 查询 `pg_stat_activity` 中 A 的 `backend_xmin`。

再制造大量 update/delete。观察 VACUUM 不能轻易移除仍可能被旧 snapshot 看到的版本。

解释时不要把 `backend_xmin` 当成唯一 cleanup 决策。只把它当成重要输入之一。

实验三：验证 command cutoff。使用前面 `mvcc07_cid` 的 SQL。

在 `HeapTupleSatisfiesMVCC()` 给当前事务插入分支打断点。观察 `HeapTupleHeaderGetCmin(tuple)` 与 `snapshot->curcid` 的比较。

再在 `CommandCounterIncrement()` 打断点。观察命令之间 `SnapshotSetCommandId()` 如何传播新的 command id。

实验四：观察 SubXID overflow 退化。用 PL/pgSQL exception block 或大量 savepoint 生成超过 64 个写入型子事务。

保持事务打开。另一个会话扫描这些 tuple。

在 `XidInMVCCSnapshot()` 和 `SubTransGetTopmostTransaction()` 打断点。对照 `snapshot->suboverflowed` 是否改变路径。

这个实验不要求把所有性能差异量化。重点是看 fallback 边界。

## 25. 讨论题

为什么一个更小的 XID 可能对当前 snapshot 不可见，而一个更大的 XID 可能已经对下一条语句可见？为什么 `GetSnapshotData()` 不把当前 backend 自己的 XID 放进 `xip`？

如果没有 `xmax`，系统要如何处理 snapshot 创建后才分配 XID 的事务？如果没有 `suboverflowed`，SubXID cache 满了以后哪类 tuple 会被错判？

为什么 `HeapTupleSatisfiesMVCC()` 在旧 snapshot 下不急着查询真实事务状态并设置 hint bit？`curcid` 解决的是隔离级别问题，还是同一事务内部命令边界问题？

`pg_current_snapshot()` 能证明哪些字段，不能证明哪些字段？为什么 snapshot 的 `xmin` 和 VACUUM 的可移除边界不能简单画等号？

## 26. 一次可见性判断的时间线复盘

把本节所有字段放进一条时间线。事务 A 分配 XID 100。

A 插入一行。tuple header 的 `xmin` 写成 100。

A 暂时不提交。事务 B 创建 snapshot。

`GetSnapshotData()` 计算 `xmax`。如果 latest completed 是 99，那么 `xmax` 可能是 100。

但如果 A 的 XID 已经分配且仍 running，它会被纳入 running 判断。实际情况取决于 A 分配和完成时序。

关键不是某个具体数字。关键是 snapshot 会把“创建时还没完成”的事实编码进区间或 `xip`。

B 开始扫描。`HeapTupleSatisfiesMVCC()` 看到 tuple 的 `xmin = 100`。

它先处理 hint bit。如果没有 committed hint bit，它会问 `XidInMVCCSnapshot(100, snapshot)`。

如果 100 按 B 的 snapshot still-running，B 直接认为插入者还在运行。这行不可见。

此时 A 提交。CLOG/pg_xact 已经可以回答 100 committed。

但 B 的旧 snapshot 不因此改变。B 再次扫描同一行时，如果仍在同一 statement 或同一 registered snapshot 下，它仍按旧 snapshot 判断。

这就是 MVCC snapshot 的稳定性。之后事务 C 创建新的 snapshot。

100 已经不在 ProcArray running set 中。如果 100 小于 C 的 `xmin`，`XidInMVCCSnapshot()` 会直接返回 false。

`HeapTupleSatisfiesMVCC()` 再通过 hint bit 或 CLOG 确认 100 committed。这行对 C 可见。

同一个事务结果，在不同 snapshot 下得到不同 visibility。这不是矛盾。

这是 snapshot 语义。它表达的是“对这个读者而言，创建 snapshot 时世界长什么样”。

真实事务状态会继续前进。旧读者不会追着它前进。

如果 tuple 上后来被 C 设置了 committed hint bit，B 的旧 snapshot 仍然可能把 100 当成 still-running。`HeapTupleSatisfiesMVCC()` 里有专门分支处理这种情况。

hint bit 只是事务结果缓存。snapshot running 判断优先保留旧读者的稳定视角。

这条时间线也解释了为什么不能把 visibility debug 简化成“看 CLOG 当前状态”。你必须同时问：

tuple header 指向哪个 XID。这个 XID 对当前 snapshot 是否 still-running。

如果不 running，它在 CLOG 中 committed 还是 aborted。如果是当前事务，它的 command id 和 snapshot `curcid` 如何比较。

这四个问题少一个都会误判。

## 27. 为什么 snapshot 不记录 committed set

有人会提出另一种模型：

snapshot 创建时，把已经提交的事务都记录下来。之后 visibility 只问 tuple 的 XID 是否在 committed set 中。

这个模型在小型教学系统里很直观。但在 PostgreSQL 里不可行。

第一，committed set 会无限增长。集群运行时间越长，提交过的 XID 越多。

把它们复制进每个 snapshot，内存和 CPU 都无法接受。第二，tuple visibility 不只需要判断插入事务是否 committed。

删除事务 `xmax` committed 时，旧版本反而不可见。因此 committed set 仍然需要额外上下文。

第三，子事务状态会引入 `SUB_COMMITTED`、parent chain 和 top-level commit。单纯 committed set 无法表达“子事务自己的状态需要回到父事务解释”。

第四，当前事务自可见性不能用 committed set 解决。当前事务还没提交，但前一条命令写入的 tuple 对后一条命令可见。

同一条命令刚写入的 tuple 对基表扫描通常不可见。这只能靠 command id 边界解释。

第五，abort 和 crash 也必须能解释。如果 snapshot 只保存 committed set，那么不在集合里的 XID 可能是 running、aborted、future 或 unknown。

这些状态对 `xmin` 和 `xmax` 的处理不同。PostgreSQL 选择记录 running set。

因为活跃集合的规模受 backend 数和 SubXID cache/fallback 约束。旧的 committed 事务不需要逐个列出。

它们通过 `xmin` 范围和 CLOG/hint bit 被延迟解释。这个选择把 snapshot 大小从“历史长度”压缩到“当前并发宽度”。

这就是 MVCC 系统中非常典型的工程取舍。保存历史完整性很贵。

保存当前不稳定边界更划算。

## 28. 版本、workload 与推断边界

本节基于 `/home/highgo/postgres` 的当前分支。核心语义相对稳定。

例如普通 MVCC snapshot 由区间加 running set 构成。`XidInMVCCSnapshot()` 先做范围判断。

`HeapTupleSatisfiesMVCC()` 区分插入事务和删除事务。这些不是某个小 patch 的偶然结果。

但具体实现细节会随版本变化。`GetSnapshotDataReuse()`、dense ProcGlobal arrays、`GlobalVis*` 维护、batch visibility helper 都可能演进。

课程中提到的性能结论要带 workload 条件。backend 很少、事务很短时，ProcArray scan 可能不是瓶颈。

backend 很多、语句很短时，snapshot 获取可能很显眼。SubXID overflow 少见时，`pg_subtrans` fallback 不是主要成本。

PL/pgSQL exception block 很多时，它可能突然出现在读路径 profile 中。hint bit 已经热身过的表，CLOG 查询少。

冷数据、频繁 vacuum freeze、异步 commit 或 crash recovery 之后，hint bit 和事务状态查询的分布会不同。诊断时不要把一个实验结果直接推广到所有系统。

先确认 workload 的并发宽度。再确认长事务和 backend_xmin。

再确认 SubXID overflow 可能性。最后再把现象映射回本节的 snapshot 字段和 heap visibility 分支。

## 29. 本节小结

本节只回答了一个问题：

`SnapshotData` 为什么是区间、running set、SubXID 完整性标志和 command cutoff 的组合。答案来自 PostgreSQL 的运行约束。

XID 分配顺序不是提交顺序。事务完成状态会在 snapshot 创建后继续变化。

子事务集合不能无限保存在 shared memory 中。heap scan 不能每个 tuple 都重新扫描 ProcArray。

同一事务内部还必须区分不同命令写入的 tuple。因此普通 MVCC snapshot 采用：

`xmin` 过滤足够旧的 XID。`xmax` 挡住 snapshot 创建后才进入视野的 XID。

`xip` 保存区间内 running top-level XID。`subxip` 保存可枚举的 running SubXID。

`suboverflowed` 在信息不完整时触发 `pg_subtrans` fallback。`curcid` 切开当前事务自己的命令边界。

`XidInMVCCSnapshot()` 只回答 XID 是否按 snapshot still-running。`HeapTupleSatisfiesMVCC()` 才把这个答案翻译成 tuple 可见或不可见。

本节的可迁移规律是：

一个系统快照不是全局状态本身。它是为了后续 hot path 可重复判断而复制出来的最小充分状态。

字段的意义必须和创建时机、生命周期、fallback 和消费路径一起解释。下一节会把这个字段模型放进 READ COMMITTED。

同样的 `SnapshotData` 会在每条语句重新创建，但在单条语句内部被注册并固定。
