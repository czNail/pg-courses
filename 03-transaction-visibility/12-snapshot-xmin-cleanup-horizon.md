# PostgreSQL xmin horizon 与 cleanup snapshot 语义
## 课程定位
前置知识：
- 已理解 snapshot 字段、生命周期和隔离级别。
- 已理解 VACUUM 只能回收所有可能读者都不再需要的 tuple version。
- 已理解 `MyProc->xmin` 是 backend 对全局发布的旧版本需求。
本节唯一主问题：
一个读 snapshot 的 `xmin` 如何限制 VACUUM 回收旧版本，这里的“仍可能被看见”到底是 MVCC 语义，还是 ProcArray 实现细节？
核心矛盾：
系统希望尽快回收 dead tuple、推进 visibility map 和降低 bloat；
但只要某个 snapshot 仍可能看见旧版本，回收就会破坏 MVCC 正确性。
一句话模型：
snapshot `xmin` 是当前读者向全局声明的“我仍可能需要这些旧版本”的下界，cleanup horizon 是所有这类声明和复制/恢复边界取最保守结果。
学完后应能判断：
- 为什么长事务会拖住 VACUUM。
- `backend_xmin` 和 `xmin` horizon 的关系。
- cleanup horizon 不是简单取当前最小 running XID。
- replication slot、prepared transaction、hot standby 会影响 horizon。
- GlobalVisTest 和 snapshot xmin 不是同一个 API，但服务同一个回收正确性目标。
源码基线：
```text
/home/nail/postgres-lab
branch: feature/pg-pv-storage-design
commit: 2d5ed10b0bb1d1c16df7ff408eb65df4609006ae
```
## 1. 本节在总主线中的位置
第 7 到 11 节解释了 snapshot 如何表示读视图以及如何活着。
本节解释 snapshot 活着会阻止什么。
MVCC 的旧版本不能在“对当前语句不可见”时就删除。
它必须对所有可能读者都不可见。
一个 REPEATABLE READ 事务可能还要看很老的版本。
一个 cursor 可能还没 fetch 完。
一个 replication slot 可能要求保留解码需要的 catalog 行。
一个 standby query 可能持有 recovery snapshot。
这些边界共同形成 cleanup horizon。
本节关注普通 MVCC 语义上的 `xmin` 和 cleanup horizon。
具体 VACUUM heap/index cleanup 在第 26-32 节展开。
## 2. 核心矛盾与一句话运行模型
tuple version 的物理回收是不可逆的。
如果 VACUUM 删除了某个 still-needed tuple version，后续 snapshot 就无法满足 MVCC 读视图。
因此 PostgreSQL 必须保守。
`snapshot->xmin` 表示：
小于这个 XID 的事务不再是该 snapshot 的 running set 成员。
但对 cleanup 来说，更重要的是：
这个 snapshot 可能仍需要看到某些由更老事务创建、被后续事务删除的旧版本。
`MyProc->xmin` 把这个需求发布到 ProcArray。
VACUUM / pruning / global visibility 逻辑计算最老 non-removable horizon。
任何可能仍被某个 snapshot 需要的 tuple，都不能被彻底移除。
一句话：
visibility 判断问“我能否看见这个 tuple”。
cleanup horizon 问“是否还有任何人可能看见这个 tuple”。
## 3. 核心文件分工与阅读顺序
| 顺序 | 文件 | 关注点 |
| --- | --- | --- |
| 1 | `src/backend/utils/time/snapmgr.c` | snapshot 如何设置和清理 `MyProc->xmin` |
| 2 | `src/backend/storage/ipc/procarray.c` | `ComputeXidHorizons()` 和 GlobalVis state |
| 3 | `src/include/storage/proc.h` | `PGPROC.xmin` 字段语义 |
| 4 | `src/backend/access/heap/pruneheap.c` | pruning 如何问 global visibility |
| 5 | `src/backend/access/heap/vacuumlazy.c` | VACUUM heap scan 的 horizon 消费 |
| 6 | `src/backend/access/heap/heapam_visibility.c` | tuple visibility 与 removable 判断的差别 |
先读 `snapmgr.c:SnapshotResetXmin()`。
它说明 snapshot 生命周期如何反馈到 `MyProc->xmin`。
再读 `procarray.c` 中 GlobalVisState 注释。
这段注释解释为什么 cleanup 需要多个 horizon。
最后读 pruning/vacuum 使用点。
不要把 `snapshot->xmin` 和 `OldestXmin` 简单画等号。
它们是不同层面的边界。
## 4. 关键数据结构与状态
`SnapshotData.xmin` 是单个 snapshot 的下界。
`PGPROC.xmin` 是 backend 当前公开的最老 snapshot 需求。
如果 backend 没有 active/registered snapshot，`xmin` 可以是 invalid。
`TransactionXmin` 是本 backend 记录的当前事务 xmin。
`RecentXmin` 是较近似的全局边界缓存。
`GlobalVisState` 是 ProcArray 中用于判断 tuple 是否可移除的状态。
源码区分 shared、catalog、data、temp relation 的 horizon。
原因是不同 relation 的可见性和复制需求不同。
replication slot 有 `replication_slot_xmin` 和 `replication_slot_catalog_xmin`。
catalog xmin 影响系统表和 logical decoding。
prepared transaction 也有 PGPROC，并参与 ProcArray。
hot standby 使用 KnownAssignedXids 和 recovery horizon。
这些都可能让 cleanup horizon 比当前普通 backend snapshot 更老。
## 5. 主流程源码 walkthrough
一个 backend 获取 snapshot。
`GetSnapshotData()` 填 `snapshot->xmin`。
snapshot manager 把它安装到 `MyProc->xmin`。
另一个 backend 删除或更新 tuple。
旧版本变成 dead for some snapshots。
VACUUM 或 pruning 想清理旧版本。
它不能只看当前执行 VACUUM 的 snapshot。
它要问全局：
是否还有任何 snapshot / slot / prepared xact / recovery 状态可能需要这个版本。
ProcArray 计算 horizon。
heap pruning 或 vacuum 用 global visibility 判断。
如果删除事务 XID 仍不早于 horizon，tuple 只能保持 dead/recently dead 状态。
等旧 snapshot 释放。
`SnapshotResetXmin()` 清掉 `MyProc->xmin`。
下一轮 horizon 推进。
VACUUM 才能回收。
这就是“长事务拖住 VACUUM”的源码链路。
## 6. 生命周期 / ownership / cleanup
snapshot xmin 的 owner 是持有 snapshot 的 backend 或 resource。
创建点：
`GetSnapshotData()`。
发布点：
`MyProc->xmin`。
持有点：
active snapshot、registered snapshot、portal/cursor、transaction snapshot。
释放点：
`PopActiveSnapshot()`、`UnregisterSnapshot()`、transaction end、portal close。
cleanup：
`SnapshotResetXmin()` 重新计算或清空 `MyProc->xmin`。
复制槽的 owner 是 slot。
prepared transaction 的 owner 是 two-phase state。
standby query 的 owner 是 standby backend / recovery snapshot。
因此 cleanup horizon 不是一个 backend 的私有状态。
它是多个 owner 的合取边界。
## 7. 正确性机制层次
第一层是 individual visibility。
当前 snapshot 判断当前 tuple 是否可见。
第二层是 global removability。
所有可能 snapshot 都不需要，才能移除。
第三层是 relation kind。
catalog、shared、data、temp 的 horizon 不完全相同。
第四层是 replication semantics。
logical decoding 需要 catalog history。
replication slot 会保留更老的 xmin。
第五层是 conservative approximation。
为了避免每个 tuple 都精确扫描全部状态，系统维护可更新的 horizon。
保守会导致少回收。
不保守会导致错误回收。
PostgreSQL 选择保守。
## 8. 错误路径 / 异常路径 / fallback
如果 backend ERROR，ResourceOwner 和 transaction cleanup 会释放 snapshot。
如果 backend 卡住或应用长期 idle in transaction，系统不能自动假设 snapshot 不需要。
VACUUM 只能等待或由管理员处理连接。
如果 replication slot 不推进，catalog/data horizon 会被拖住。
这不是 VACUUM bug。
这是 slot 的语义承诺。
如果 prepared transaction 长期存在，它也会拖住 horizon。
如果 standby query 与 recovery conflict，系统可能取消 standby query 或延迟 recovery，取决于配置。
这些都是 cleanup horizon 的外部边界。
## 9. 成本、资源与跨模块传播
cleanup horizon 推迟的成本很大。
heap dead tuple 保留。
index entries 保留。
visibility map 不能设置 all-visible。
index-only scan 机会减少。
VACUUM 重复扫描同一批 recently dead tuple。
WAL 增加。
表和索引 bloat 增长。
成本传播路径：
```text
long snapshot / slot / prepared xact
  -> old xmin horizon
  -> dead tuple not removable
  -> heap/index bloat
  -> more IO and worse plans
```
ProcArray horizon 计算本身也有成本。
但通常更重要的是它限制回收带来的存储和查询成本。
## 10. 观测与诊断入口
`pg_stat_activity.backend_xmin` 是首要入口。
看哪个 backend 持有最老 xmin。
`pg_replication_slots.xmin` 和 `catalog_xmin` 看 slot 边界。
`pg_prepared_xacts` 看 prepared transaction。
`VACUUM VERBOSE` 可提示 removable / nonremovable 现象。
`pg_stat_all_tables.n_dead_tup` 看积压。
`pg_stat_progress_vacuum` 看当前 vacuum 阶段。
gdb 入口：
`SnapshotResetXmin`。
`ComputeXidHorizons`。
`GlobalVisTestFor`。
`HeapPagePruneOpt`。
`lazy_scan_heap`。
诊断顺序：
先找最老 horizon owner。
再判断 owner 类型。
再看业务是否需要它。
最后才调 VACUUM 参数。
## 11. 常见误区
- 对当前 snapshot 不可见，不等于可以回收。
- `backend_xmin` 老不是 VACUUM 自己的问题。
- `xmin` horizon 不是只由当前 running XID 决定。
- replication slot 可以拖住 catalog cleanup。
- prepared transaction 也会参与 horizon。
- temp relation horizon 语义不同于 shared/catalog/data。
- 保守不回收比错误回收更可接受。
- 调大 autovacuum 频率不能解决被 horizon 阻止的回收。
## 12. 课堂实验
实验 1：长事务阻止回收。
会话 A 开事务并 SELECT。
会话 B UPDATE/DELETE 大量行并 VACUUM。
观察 `backend_xmin`、`n_dead_tup` 和 VACUUM VERBOSE。
实验 2：释放 snapshot 后再 VACUUM。
会话 A COMMIT。
会话 B 再 VACUUM。
比较 dead tuple 回收情况。
实验 3：观察 slot horizon。
创建 logical replication slot 或查看已有 slot。
观察 `pg_replication_slots.xmin/catalog_xmin`。
讨论为什么 slot 可能让 catalog tuple 不能回收。
## 13. 源码细读补充：从 `SnapshotResetXmin()` 到 `ComputeXidHorizons()`
先读 `SnapshotResetXmin()`。
这个函数只关心当前 backend 的 snapshot 注册状态。
如果没有 registered snapshot，它可以把 `MyProc->xmin` 清成 invalid。
如果还有 registered snapshot，它取最小 snapshot xmin。
这一步是本 backend 对全局的声明。
再读 `procarray.c:ComputeXidHorizons()`。
这个函数把所有 backend、slot、prepared transaction、KnownAssignedXids 等边界综合起来。
它不是只看一个 snapshot。
它会计算 shared、catalog、data、temp 等不同 horizon。
再读 GlobalVisState 注释。
源码明确说明不同 relation kind 的 non-removable horizon 不同。
普通数据表不一定需要 catalog xmin。
catalog relation 必须考虑 logical decoding 的 catalog_xmin。
temp relation 只考虑当前 session。
这就是为什么 cleanup horizon 不是单个全局变量。
最后读 pruning 或 vacuum 使用点。
pruning 可能用较轻量的 global visibility test。
VACUUM heap scan 也会围绕 horizon 判断 dead/recently dead/removable。
## 14. Runtime case：长事务导致 dead tuple 积压
会话 A 开事务并建立 snapshot。
会话 B 删除大量行并提交。
对会话 B 的新 snapshot 来说，这些行已经不可见。
但会话 A 的旧 snapshot 可能仍能看见删除前版本。
VACUUM 不能移除。
`n_dead_tup` 可能增长。
VACUUM VERBOSE 可能显示有 dead rows not yet removable。
会话 A 提交后，`MyProc->xmin` 清理。
下一次 VACUUM 才能回收。
这不是 VACUUM “不积极”。
这是 MVCC 正确性。
## 15. Runtime case：replication slot 拖住 catalog
logical replication slot 需要解码历史。
解码 catalog change 时，需要旧 catalog tuple 仍可解释。
因此 slot 可能发布 `catalog_xmin`。
即使没有用户 backend 持有老 snapshot，catalog cleanup 仍可能被 slot 拖住。
诊断时如果只看 `pg_stat_activity.backend_xmin`，会漏掉 slot。
要看 `pg_replication_slots.xmin` 和 `catalog_xmin`。
如果 slot 长期不消费，bloat 可能持续增长。
解决不是调大 autovacuum。
要推进或删除不需要的 slot。
## 16. Runtime case：prepared transaction
prepared transaction 已经脱离普通 backend 会话。
但它仍可能持有 XID。
ProcArray 为 prepared transaction 保留 PGPROC 表示。
因此 horizon 计算必须考虑它。
如果业务忘记提交或回滚 prepared xact，它会像长事务一样拖住系统。
诊断入口是 `pg_prepared_xacts`。
这解释了为什么 2PC 运维问题常表现为 VACUUM/bloat 问题。
## 17. 诊断矩阵：为什么 VACUUM 不能清理
现象：
有老 `backend_xmin`。
优先定位持有该 xmin 的 backend。
看它是否长事务、cursor、导出 snapshot 或卡住 query。
现象：
没有老 backend_xmin，但 catalog bloat。
检查 replication slots 的 `catalog_xmin`。
现象：
没有活跃 backend，但仍有旧 XID。
检查 `pg_prepared_xacts`。
现象：
standby 或 recovery 冲突。
检查 hot standby 查询、recovery delay 和 conflict logs。
现象：
VACUUM 跑很多次仍不回收。
看日志中 removable 边界，不要只看 autovacuum 触发频率。
现象：
index-only scan 变少。
旧版本不能清理会影响 all-visible 位设置。
这会把 snapshot 问题传播到 planner/executor 现象。
## 18. 概念边界：visibility 与 removability
visibility 是单个 snapshot 的问题。
removability 是所有可能 snapshot 的问题。
一个 tuple 对我不可见，不代表对别人不可见。
一个 tuple 对所有当前普通 backend 不可见，也可能被 slot 或 prepared xact 需要。
一个 tuple 可以是 dead，但 not yet removable。
这就是 VACUUM 术语中 dead / recently dead 的实质。
如果把二者混淆，会得出错误结论：
“我查不到这行，为什么 VACUUM 不删？”
答案是：
你查不到，只说明你的 snapshot 不需要。
VACUUM 要证明没人需要。
## 19. 逐函数阅读任务
任务 1：读 `SnapshotResetXmin()`。
写下没有 registered snapshot 时如何处理 `MyProc->xmin`。
写下有 registered snapshot 时如何选择 xmin。
任务 2：读 `ComputeXidHorizons()`。
标出它扫描哪些 `PGPROC`。
标出 slot xmin 如何加入。
标出 KnownAssignedXids 如何加入。
任务 3：读 GlobalVisState 注释。
解释 shared、catalog、data、temp horizon 的差别。
任务 4：读 heap pruning 使用点。
找 `GlobalVisTestFor()`。
说明 relation kind 如何选择 horizon。
任务 5：读 VACUUM heap scan。
标出 dead/recently dead/removable 的判断入口。
任务 6：读 replication slot xmin API。
解释 slot 为什么不属于普通 backend snapshot。
任务 7：读 prepared transaction 相关 ProcArray 入口。
解释 prepared xact 为什么仍参与 horizon。
## 20. 案例推演：dead、recently dead、removable
事务 A 持有旧 snapshot。
事务 B 删除 tuple 并 commit。
对新 snapshot 来说，该 tuple dead。
对事务 A 的旧 snapshot 来说，旧版本可能仍可见。
因此它是 recently dead 或 not yet removable。
VACUUM 可以记录它。
但不能物理移除 line pointer 所需信息。
事务 A 结束。
horizon 推进。
下一轮 VACUUM 才能把它当 removable。
这个状态变化解释了为什么 VACUUM 结果依赖时间。
同一张表连续 VACUUM 两次，第二次可能清掉更多。
不是因为第一次漏了。
而是 horizon 变了。
## 21. 案例推演：horizon 的多个 owner
owner 1：普通 backend snapshot。
通过 `PGPROC.xmin` 发布。
owner 2：事务级 snapshot。
通常让 xmin 持续到 transaction end。
owner 3：cursor / portal。
可能让 snapshot 跨 fetch 存活。
owner 4：replication slot。
通过 slot xmin 或 catalog_xmin 发布。
owner 5：prepared transaction。
通过 two-phase 状态参与 ProcArray。
owner 6：hot standby query。
通过 recovery running set 和 conflict 机制影响。
cleanup horizon 必须取最保守边界。
只检查其中一个 owner 会误诊。
## 22. 诊断案例：调 autovacuum 为什么没用
现象：
autovacuum 很频繁。
dead tuple 仍增长。
如果 horizon 被 old snapshot 拖住，跑多少次都不能回收。
调大 cost limit 也不能越过 correctness。
正确诊断：
找 old `backend_xmin`。
找 replication slot。
找 prepared xact。
找 standby conflict。
确认 horizon 释放后再 VACUUM。
只有当 horizon 允许回收但 vacuum 跑不动时，才优先调 autovacuum 参数。
这能避免把语义边界问题误当调参问题。
## 23. 反例与边界：cleanup horizon 不能靠单一指标解释
反例一：
`backend_xmin` 都不老，但 replication slot 的 `catalog_xmin` 很老。
catalog bloat 仍可能增长。
只看 `pg_stat_activity` 会漏诊。
反例二：
没有普通 backend，但有 prepared transaction。
旧 XID 仍被 two-phase state 持有。
VACUUM 仍必须保守。
反例三：
当前查询看不到某行。
VACUUM 仍不能删。
因为 cleanup 要证明所有可能读者都不需要。
反例四：
autovacuum 很频繁。
dead tuple 仍不下降。
如果 horizon 不允许回收，频率不是根因。
反例五：
temp relation 的 horizon 与普通 relation 不同。
不要把 shared/catalog/data/temp 混成一个全局答案。
反例六：
standby query 冲突不是主库 snapshot 字段能单独解释。
要看 recovery、hot standby feedback 和 conflict 处理。
边界总结：
单个 snapshot 解释 visibility。
全局 horizon 解释 removability。
relation kind 决定使用哪个 horizon。
slot/prepared/recovery 决定额外保守边界。
## 24. 生产诊断提纲
问题：表膨胀。
步骤 1：确认 dead tuple 是否存在。
看 `pg_stat_all_tables.n_dead_tup` 和 VACUUM VERBOSE。
步骤 2：找 horizon owner。
查 `pg_stat_activity.backend_xmin`。
查 `pg_replication_slots`。
查 `pg_prepared_xacts`。
步骤 3：判断 relation kind。
普通表、catalog、shared relation、temp relation 的边界不同。
步骤 4：确认业务原因。
长事务是否必要？
slot 是否消费？
prepared xact 是否遗留？
standby query 是否拖延？
步骤 5：再考虑调参。
如果 horizon 未释放，调大 autovacuum 只能增加重复工作。
如果 horizon 已释放但 vacuum 跟不上，才看 scale factor、cost limit、IO。
步骤 6：验证。
释放 owner 后手动 VACUUM。
观察 dead tuple 是否下降。
观察 visibility map 和 index-only scan 是否恢复。
## 25. 深入展开：`OldestXmin`、`RecentXmin`、GlobalVis 的命名陷阱
不同源码路径里会出现多个相近名字。
它们都和 xmin horizon 有关，但不是同一个东西。
`snapshot->xmin` 是单个 snapshot 的字段。
`MyProc->xmin` 是 backend 对外发布的需求。
`TransactionXmin` 是当前 backend 的事务 xmin 状态。
`RecentXmin` 是本 backend 近期计算得到的边界。
VACUUM 代码历史上常见 `OldestXmin` 这类概念。
新代码里 GlobalVisState 把不同 relation kind 的 horizon 区分得更清楚。
读代码时不要只看变量名。
要问：
这个值服务哪个 relation kind？
它是否包含 replication slot？
它是否包含 catalog_xmin？
它是精确值还是保守近似？
它在什么锁保护下计算？
它允许变老还是只能前进？
这些问题比名字更可靠。
## 26. 深入展开：为什么 cleanup horizon 要区分 relation kind
普通用户表的数据 tuple 不需要被 logical decoding 当作 catalog metadata 解释。
系统 catalog tuple 可能需要被 decoding 用来解释后续 WAL。
shared relation 跨 database。
temp relation 只属于当前 session。
因此一个统一 horizon 要么过于保守，要么不够安全。
GlobalVisState 分成 shared、catalog、data、temp。
这样可以在正确性允许时少保留一些旧版本。
但对 catalog 必须更保守。
这解释了为什么 catalog bloat 和普通表 bloat 的根因可能不同。
普通表看 data horizon。
catalog 要看 catalog horizon 和 slot catalog_xmin。
## 27. 深入展开：HOT pruning 与 VACUUM 的 horizon 差异
heap page pruning 可以由普通访问触发。
VACUUM 是专门清理。
二者都不能破坏可见性。
pruning 常在 page 层做局部优化。
VACUUM 会做更系统的 heap scan 和 index cleanup。
二者都需要知道某个旧版本是否仍可能被看见。
但它们的入口、锁、成本和可做动作不同。
因此不要看到 pruning 没清，就断言 VACUUM 也不能清。
也不要看到 VACUUM 后仍有 line pointer，就断言 horizon 判断错。
有些 line pointer 状态还受 HOT chain、index TID、page layout 影响。
后续第 26-28 节会展开。
本节只提供共同的 MVCC horizon 基础。
## 28. 深入展开：hot standby feedback 的传播
standby 上长查询也可能需要旧版本。
如果启用 hot standby feedback，standby 会把 xmin 需求反馈给 primary。
primary 因此延迟清理。
这能减少 standby query conflict。
代价是 primary bloat。
如果不启用，primary 可以清理更积极。
standby 可能因为 recovery conflict 取消查询。
这是一组运维 trade-off：
保护 standby 长查询。
还是保护 primary 存储健康。
它和 snapshot xmin 是同一个正确性思想在复制环境中的传播。
诊断时要看：
standby 查询。
hot_standby_feedback。
replication slot。
primary 上的 bloat。
不要只在 primary 找长事务。
## 29. 小型练习：定位 horizon owner
给出一个 bloat 现场：
`backend_xmin` 最老是 500。
replication slot `catalog_xmin` 是 450。
prepared transaction XID 是 470。
用户表膨胀。
catalog 表膨胀。
问：
用户表 cleanup 可能被谁拖住？
catalog cleanup 可能被谁拖住？
如果释放 backend 500 后，catalog 仍不回收，下一步查什么？
如果删除 slot 后仍不回收，下一步查什么？
这个练习的目的：
不要只找一个最老数字。
要按 relation kind 和 owner 类型拆解。
## 30. 源码路径分解：VACUUM 为什么要等 horizon
删除发生：
事务 B 删除 tuple，并最终 commit。
tuple 旧版本对新 snapshot 不可见。
VACUUM 扫描 heap page。
它看到 tuple 的 `xmax`。
它需要判断删除事务是否足够老，且没有任何可能读者仍需要旧版本。
它查询或使用全局可见性边界。
如果 `xmax` 不早于 cleanup horizon，tuple 不能彻底移除。
它可能被标为 recently dead 或保留。
如果 `xmax` 早于 horizon，并且索引清理等条件允许，VACUUM 才能推进物理回收。
这个流程说明：
事务提交只让 tuple 对新读者不可见。
cleanup horizon 才决定它能否物理消失。
两者之间可能隔很久。
## 31. 反例：为什么“没有长查询”仍可能 bloat
没有长查询只是排除了一个 owner。
还可能有：
replication slot 不消费。
prepared transaction 遗留。
standby feedback 拖住。
cursor 在 idle session 中打开。
导出 snapshot 源事务未结束。
autovacuum 被禁用或跟不上。
索引 cleanup 被配置或成本限制推迟。
其中前五个属于 horizon / owner 问题。
后两个属于执行能力或配置问题。
诊断时要先分层。
如果 horizon 不允许，执行多少次 VACUUM 都没用。
如果 horizon 允许但 VACUUM 没跑或跑太慢，才是调度和资源问题。
## 32. 讨论题
1. 为什么 tuple 对当前 snapshot 不可见仍不能立即删除？
2. `snapshot->xmin` 和 `PGPROC.xmin` 有什么区别？
3. cleanup horizon 为什么要考虑 replication slot？
4. prepared transaction 为什么像 backend 一样影响 horizon？
5. GlobalVisState 为什么区分 relation kind？
6. 为什么 horizon 判断宁可保守？
7. 如何证明 bloat 是 old snapshot 导致，而不是 VACUUM 没跑？
8. `backend_xmin` 为空能说明什么，不能说明什么？
## 26. 本节小结
snapshot `xmin` 是读者仍可能需要旧版本的下界。
`MyProc->xmin` 把这个需求发布给全局。
cleanup horizon 是所有 snapshot、slot、prepared xact、recovery 状态共同形成的保守边界。
VACUUM 只有在所有可能读者都不需要某个 tuple version 后才能回收。
这就是长事务、cursor、replication slot 和 prepared transaction 会导致 bloat 的根本原因。
本节把 snapshot 从“读视图”推进到了“回收边界”。
