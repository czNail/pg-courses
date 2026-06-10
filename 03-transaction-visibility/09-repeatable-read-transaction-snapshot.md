# PostgreSQL REPEATABLE READ / SERIALIZABLE 事务级 snapshot
## 课程定位
前置知识：
- 已理解 READ COMMITTED 的语句级 snapshot。
- 已理解 `SnapshotData` 字段和 snapshot lifecycle。
- 已理解 `MyProc->xmin` 会影响 VACUUM 回收边界。
本节唯一主问题：
为什么 REPEATABLE READ 和 SERIALIZABLE 要复用事务中的第一个 snapshot，而不是像 READ COMMITTED 一样每条语句重新获取？
核心矛盾：
事务级一致读要求同一事务内多条语句看到同一个 MVCC 世界；
但复用 snapshot 会延长 `xmin` horizon，阻止旧版本回收，并要求 snapshot 生命周期跨语句安全保存。
一句话模型：
更高隔离级别用“固定第一个 snapshot”换取事务内读稳定性，同时接受更长的回收压力和额外冲突检测边界。
学完后应能判断：
- 为什么 REPEATABLE READ 下相邻 SELECT 不会看到其他事务后提交的行。
- `FirstXactSnapshot` 为什么必须 copy/register。
- 事务级 snapshot 如何拖住 VACUUM。
- SERIALIZABLE 在 snapshot 之外还需要 SSI。
- 导入/导出 snapshot 为什么必须保护 xmin owner。
源码基线：
```text
/home/nail/postgres-lab
branch: feature/pg-pv-storage-design
commit: 2d5ed10b0bb1d1c16df7ff408eb65df4609006ae
```
## 1. 本节在总主线中的位置
上一节的 READ COMMITTED 允许每条语句看到新的 committed 数据。
REPEATABLE READ 反过来：
事务第一次需要 snapshot 时获取一个读视图。
后续语句继续使用它。
这让事务内部读结果稳定。
但这不是简单缓存。
snapshot 必须跨语句有效。
`MyProc->xmin` 必须保护它。
ResourceOwner 和 snapshot manager 必须确保 cleanup。
SERIALIZABLE 也使用事务级 snapshot。
但仅靠事务级 snapshot 不能阻止 write skew。
因此 SERIALIZABLE 还需要 SSI/predicate lock。
本节只讲事务级 snapshot 本身。
SSI 在 33-38 节展开。
## 2. 核心矛盾与一句话运行模型
事务级 snapshot 的承诺是：
同一事务内所有普通 MVCC 读都基于第一个 snapshot。
这让“重复读”成立。
如果另一个事务在你第一次 SELECT 后 INSERT 并 COMMIT，你后续 SELECT 仍看不见。
它牺牲的是新鲜度和回收。
运行模型：
```text
first GetTransactionSnapshot()
  -> GetSnapshotData()
  -> CopySnapshot()
  -> FirstXactSnapshot->regd_count++
later GetTransactionSnapshot()
  -> return CurrentSnapshot / FirstXactSnapshot view
transaction end
  -> AtEOXact_Snapshot()
  -> release registered snapshot
```
这个模型有两个关键点。
第一，第一次 snapshot 必须被复制。
不能直接指向 `CurrentSnapshotData`。
因为静态对象后续可能被覆盖。
第二，snapshot 的 `xmin` 必须继续安装在 `MyProc->xmin`。
否则 VACUUM 可能回收这个事务后续还要看的旧版本。
## 3. 核心文件分工与阅读顺序
| 顺序 | 文件 | 关注点 |
| --- | --- | --- |
| 1 | `src/backend/utils/time/snapmgr.c` | `GetTransactionSnapshot()` 的 `IsolationUsesXactSnapshot()` 分支 |
| 2 | `src/include/access/xact.h` | 隔离级别宏 |
| 3 | `src/backend/storage/ipc/procarray.c` | `GetSnapshotData()` 和 xmin 安装 |
| 4 | `src/backend/utils/time/snapmgr.c` | `CopySnapshot()`、`RegisterSnapshot()`、`AtEOXact_Snapshot()` |
| 5 | `src/backend/access/heap/heapam_visibility.c` | 消费同一个 snapshot |
| 6 | `src/backend/storage/ipc/procarray.c` | horizon / `ComputeXidHorizons()` |
不要把 REPEATABLE READ 的行为只归因于 parser 或 GUC。
真正的差异落在 `GetTransactionSnapshot()`。
`IsolationUsesXactSnapshot()` 为 true 时，第一次 snapshot 被保存。
后续语句复用。
对于 SERIALIZABLE，还可能调用 `GetSerializableTransactionSnapshot()`。
这把 SSI 状态接到同一个 snapshot 生命周期上。
## 4. 关键数据结构与状态
`FirstSnapshotSet` 标记第一个 transaction snapshot 是否已建立。
`FirstXactSnapshot` 保存事务级 snapshot。
`CurrentSnapshot` 指向当前可用 snapshot。
在事务级隔离级别下，它通常关联同一份 copied snapshot。
`regd_count` 很重要。
事务级 snapshot 需要注册，防止它在语句结束后被释放。
`MyProc->xmin` 同样重要。
它让其它 backend 和 VACUUM 知道：
这个事务后续仍可能需要旧版本。
`TransactionXmin` 是当前 backend 记录的 snapshot xmin。
它参与一些本地判断。
SERIALIZABLE 还会关联 predicate lock / SSI 状态。
但 `SnapshotData` 本身仍然是 MVCC 读视图。
不要把 snapshot 和 SSI 冲突图混成一个对象。
## 5. 主流程源码 walkthrough
事务第一次执行 SELECT。
executor 请求 snapshot。
`GetTransactionSnapshot()` 发现 `FirstSnapshotSet` 为 false。
隔离级别使用 transaction snapshot。
如果是 SERIALIZABLE，可能走 `GetSerializableTransactionSnapshot()`。
否则走 `GetSnapshotData(&CurrentSnapshotData)`。
然后 `CopySnapshot()`。
复制对象赋给 `FirstXactSnapshot`。
增加 `regd_count`。
设置 `FirstSnapshotSet = true`。
后续语句再次请求 snapshot。
`FirstSnapshotSet` 已经为 true。
函数不重新扫描 ProcArray 来构建新读视图。
它返回事务级 snapshot。
另一个事务后续 commit 不会进入这个 snapshot 的可见集合。
事务结束时调用 snapshot end-of-xact cleanup。
registered snapshot 引用释放。
`MyProc->xmin` 清理。
VACUUM horizon 可以继续推进。
这解释了两个现象：
事务内读稳定。
事务持续越久，旧版本越难回收。
## 6. 生命周期 / ownership / cleanup
事务级 snapshot 的 owner 是当前 transaction。
创建点是第一次 `GetTransactionSnapshot()`。
持有点是 `FirstXactSnapshot` 和 registered snapshot 引用。
使用点可以跨多个语句。
释放点是 transaction end。
ERROR/abort 时也要释放。
ResourceOwner 负责注册引用 cleanup。
`AtEOXact_Snapshot()` 负责事务结束时重置 snapshot manager 状态。
如果是 exported snapshot，生命周期更复杂。
导出 snapshot 要把字段写到文件。
同时源事务必须继续保护 xmin。
导入 snapshot 要调用安装 xmin 的逻辑。
如果源事务已经结束或无法保护 xmin，导入不能安全成立。
这说明事务级 snapshot 不是一个可随便复制的 JSON。
它依赖 live backend 或有效 owner 保护 horizon。
## 7. 正确性机制层次
第一层是事务内读稳定。
所有语句使用同一个 snapshot。
第二层是 horizon 保护。
只要 snapshot 存活，`xmin` 就不能丢。
第三层是 command id 更新。
同一事务自己的写入仍要按 `curcid` 判断。
第四层是 conflict/error 语义。
REPEATABLE READ 下某些并发更新可能导致 serialization failure 或 update conflict 行为。
第五层是 SERIALIZABLE 的额外 SSI。
事务级 snapshot 能防止 non-repeatable read。
但不能单独防止所有 serializability anomaly。
因此 SERIALIZABLE 在 snapshot 之上追踪 rw-conflict。
本节的边界是：
snapshot 提供稳定读视图。
SSI 提供可串行化冲突检测。
## 8. 错误路径 / 异常路径 / fallback
如果事务级 snapshot 没有正确注册，语句结束后可能释放过早。
源码通过 copy/register 避免。
如果长期事务保持 snapshot，VACUUM 不能回收旧版本。
这不是错误，但会造成 bloat。
如果事务尝试导入过期 snapshot，会失败。
因为 xmin owner 不再保证旧版本仍存在。
如果 SERIALIZABLE 发现 dangerous structure，事务可能 abort。
这不是 snapshot 损坏。
这是更高层的可串行化规则。
如果并行 worker 使用 snapshot，需要序列化/反序列化 snapshot，并恢复 combo CID 等相关状态。
这说明 snapshot lifecycle 还要跨 backend 边界。
## 9. 成本、资源与跨模块传播
事务级 snapshot 降低重复获取 snapshot 的成本。
长事务内 1000 条 SELECT 不必每条都完整扫描 ProcArray 来建立新读视图。
但它延长 `xmin`。
旧 tuple version 不能回收。
VACUUM 可能报告 dead tuple not yet removable。
visibility map all-visible 推进可能推迟。
索引膨胀和 heap bloat 可能累积。
复制槽、prepared transaction、cursor 与事务级 snapshot 一样可能拖住 horizon。
区别在 owner 和语义。
诊断时不要只看隔离级别。
要看实际 snapshot 是否持有、事务是否长期打开、backend_xmin 是否老。
## 10. 观测与诊断入口
实验 SQL：
`SHOW transaction_isolation`。
`SELECT txid_current_snapshot()`。
`pg_stat_activity.backend_xmin`。
`pg_stat_activity.xact_start`。
`pg_stat_activity.state`。
VACUUM 侧：
`VACUUM VERBOSE`。
`pg_stat_all_tables.n_dead_tup`。
`pg_stat_progress_vacuum`。
gdb 入口：
`GetTransactionSnapshot`。
`CopySnapshot`。
`RegisterSnapshot`。
`AtEOXact_Snapshot`。
`SnapshotResetXmin`。
观察：
第一次 snapshot 是否被复制。
后续语句是否复用。
事务结束时是否释放。
## 11. 常见误区
- REPEATABLE READ 不是在每条语句重新取 snapshot。
- 事务级 snapshot 不是“锁住表”。
- `backend_xmin` 老不一定是当前 SQL 正在跑，也可能是 cursor 或 exported snapshot。
- SERIALIZABLE 不只是 REPEATABLE READ 加名字。
- 事务级 snapshot 防止 non-repeatable read，但不自动防止 write skew。
- 长事务 bloat 不是 VACUUM 不工作，而是 horizon 不允许回收。
- 导出 snapshot 不是复制一段文本就安全。
## 12. 课堂实验
实验 1：相邻 SELECT 不变。
会话 A：
```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT count(*) FROM t;
```
会话 B 插入并提交。
会话 A 再查，结果不变。
实验 2：对比 READ COMMITTED。
重复上一实验，把会话 A 改成 READ COMMITTED。
第二次 SELECT 可看到新行。
实验 3：观察 horizon。
会话 A 开 REPEATABLE READ 并保持不提交。
会话 B 大量 UPDATE/DELETE 后 VACUUM。
观察 `backend_xmin` 和 dead tuple 回收。
## 13. 源码细读补充：第一个 snapshot 如何变成事务级状态
读 `snapmgr.c:GetTransactionSnapshot()`。
进入事务级隔离级别分支时，第一次调用才真正构造 snapshot。
构造可以是普通 `GetSnapshotData()`。
SERIALIZABLE 下可能是 `GetSerializableTransactionSnapshot()`。
无论哪条路径，结果都必须被 copy。
原因是 `CurrentSnapshotData` 是静态缓冲。
如果不 copy，后续调用可能覆盖字段。
copy 后赋给 `FirstXactSnapshot`。
随后增加注册引用。
`FirstSnapshotSet` 变成 true。
后续语句请求 snapshot 时不再建立新读视图。
这不是因为 ProcArray 没变化。
恰恰相反，ProcArray 可能已经变化。
事务级隔离级别的语义就是忽略这些后续变化。
如果是 SERIALIZABLE，还会把 snapshot 与 SSI 状态连接。
这不改变 MVCC snapshot 字段的基本含义。
它只是增加冲突检测层。
## 14. Runtime case：稳定读和旧版本保留
会话 A：
```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT * FROM t WHERE id = 1;
```
会话 B：
```sql
UPDATE t SET v = v + 1 WHERE id = 1;
COMMIT;
```
会话 A 再读同一行。
它仍看到旧版本。
这要求旧版本不能被 VACUUM 回收。
旧版本的保留不是因为表锁。
也不是因为 UPDATE 没提交。
而是会话 A 的 snapshot `xmin` 仍被全局 horizon 尊重。
如果会话 A 长期不提交，大量类似旧版本会堆积。
这就是事务级 snapshot 的成本。
## 15. Runtime case：为什么 SERIALIZABLE 还会 abort
事务级 snapshot 可以让读稳定。
但两个事务都基于稳定旧视图做决策时，仍可能出现 write skew。
例如两个医生值班的经典例子。
两个事务都看到对方仍在值班。
各自更新自己为不值班。
两个事务的 snapshot 都稳定。
但最终结果不可串行化。
PostgreSQL 的 SERIALIZABLE 通过 SSI 检测 rw-conflict。
因此 SERIALIZABLE 的异常路径不是 snapshot 失效。
它是 snapshot isolation 之上的冲突图判断。
本节只要求记住边界：
REPEATABLE READ 解释稳定读。
SERIALIZABLE 解释可串行化冲突。
不要用 snapshot 字段硬解释 SSI abort。
## 16. 诊断矩阵：为什么 VACUUM 被一个高隔离级别事务拖住
现象：
`pg_stat_activity.backend_xmin` 很老。
同时 `xact_start` 很早。
事务隔离级别是 REPEATABLE READ 或 SERIALIZABLE。
解释：
第一次 snapshot 的 xmin 仍被事务级 snapshot 持有。
VACUUM 必须保留可能被它看到的旧版本。
现象：
会话 state 是 idle in transaction。
解释：
即使没有正在跑 SQL，事务级 snapshot 仍可能注册。
要看 `backend_xmin`。
现象：
提交该事务后 VACUUM 立刻能清理更多。
解释：
`AtEOXact_Snapshot()` 释放 registered snapshot。
`SnapshotResetXmin()` 清理 `MyProc->xmin`。
现象：
SERIALIZABLE 事务 abort。
解释：
先看 SSI conflict，而不是怀疑 snapshot 字段变化。
现象：
导入 snapshot 失败。
解释：
source xmin owner 可能不存在或不能再保护旧版本。
## 17. 事务级 snapshot 与导出 snapshot
导出 snapshot 是事务级 snapshot 生命周期的扩展。
导出时会把 snapshot 字段序列化到文件。
但文件不是全部正确性。
源事务仍需要保护 `xmin`。
导入方使用这份 snapshot。
如果源事务结束，旧版本可能已经被回收。
因此导入必须安装 imported xmin。
源码中有 `ProcArrayInstallImportedXmin()`。
它要验证 source virtual transaction 仍能保护该 xmin。
这解释了为什么导出 snapshot 要求事务保持打开。
也解释了为什么导出 snapshot 是运维上有成本的工具。
长时间持有会拖住 VACUUM。
## 18. 成本对照：少取 snapshot 不等于更便宜
事务级 snapshot 减少了 ProcArray 扫描次数。
这对大量语句的事务可能有利。
但它增加旧版本保留成本。
heap bloat 增加。
index bloat 增加。
VACUUM 工作重复。
查询计划可能因统计和 bloat 变差。
SERIALIZABLE 还要维护 predicate locks 和 rw-conflict。
因此成本不是单维度。
短事务、高频语句：
ProcArray 获取成本更显眼。
长事务、大量更新：
horizon 成本更显眼。
SERIALIZABLE 混合读写：
SSI 冲突检测成本更显眼。
诊断时要先定位主要成本在哪一层。
## 19. 逐函数阅读任务
任务 1：读 `GetTransactionSnapshot()` 的 first snapshot 分支。
标出 `FirstSnapshotSet` 变成 true 的位置。
标出 `FirstXactSnapshot` 被赋值的位置。
标出 `CopySnapshot()` 的调用。
解释为什么事务级 snapshot 不能直接引用静态 `CurrentSnapshotData`。
任务 2：读 `CopySnapshot()`。
看它如何复制 `xip` 和 `subxip`。
看它如何重置 `regd_count` 和 `active_count`。
解释复制后的 snapshot 为什么是一个独立生命周期对象。
任务 3：读 `AtEOXact_Snapshot()`。
看事务结束如何重置 `FirstSnapshotSet`。
看 registered snapshot 如何释放。
看 `MyProc->xmin` 清理发生在哪个阶段。
任务 4：读 serializable 分支。
找到 `GetSerializableTransactionSnapshot()`。
只标出它接入点。
不要把 SSI 展开。
记录它如何在事务级 snapshot 基础上增加冲突检测语义。
任务 5：读 exported snapshot 相关函数。
标出 snapshot 序列化字段。
标出 imported xmin 安装。
解释为什么导入 snapshot 要验证 source owner。
## 20. 案例推演：稳定读如何影响更新工作负载
事务 A 以 REPEATABLE READ 开始。
它读取表中 100 万行的一部分。
事务 B 更新其中大量行并提交。
事务 A 后续继续读。
它仍需要旧版本。
旧版本可能分散在大量 heap page 中。
VACUUM 不能清掉这些旧版本。
index 中指向旧版本的 TID 也可能暂时保留。
visibility map all-visible 位可能无法设置。
事务 A 提交后，horizon 才能推进。
这个案例说明：
事务级 snapshot 的成本不在 snapshot 对象大小。
成本在它保护的历史版本集合。
## 21. 案例推演：事务级 snapshot 与应用缓存
应用常见误解：
“我在事务开始时查一次配置，后面应该一直稳定。”
在 READ COMMITTED 下不保证。
在 REPEATABLE READ 下保证普通 MVCC 读稳定。
但如果应用同时执行写操作，还要考虑当前事务自写入和 command id。
如果应用依赖可串行化业务约束，还要考虑 SERIALIZABLE。
因此选择隔离级别时要问三个问题：
是否需要重复读稳定？
是否需要防 write skew？
是否能接受长事务 bloat？
这比简单说“隔离级别越高越安全”更准确。
## 22. 诊断案例：SERIALIZABLE abort 不是 snapshot 变化
现象：
SERIALIZABLE 事务读到稳定结果，但 commit 时失败。
错误通常提示 serialization failure。
不要怀疑 snapshot 中途变了。
snapshot 没变。
失败来自 SSI 检测到危险结构。
诊断路径：
看事务是否只读。
看是否 DEFERRABLE。
看是否存在并发写入同一谓词范围。
看 predicate lock。
本节只要求能把它从 snapshot 生命周期问题中分离出来。
真正 SSI 在后续章节讲。
## 23. 与后续课程的衔接
第 10 节会说明事务级 snapshot 的 `curcid` 仍会推进。
第 11 节会说明 `FirstXactSnapshot` 为什么注册到 ResourceOwner。
第 12 节会说明长事务 snapshot 如何变成 cleanup horizon。
第 33-38 节会说明 SERIALIZABLE 在 snapshot 之上如何追踪 rw-conflict。
因此本节的边界是：
REPEATABLE READ 固定读视图。
SERIALIZABLE 复用这个基础并增加冲突检测。
不要把二者混成一个机制。
## 24. 反例与边界：事务级 snapshot 不能解决什么
反例一：
两个事务都读取“当前还有至少一名值班医生”。
两个事务分别把自己设为不值班。
各自的 snapshot 都稳定。
各自都没有看到对方后来的修改。
最终状态却违反业务约束。
这说明稳定读不等于可串行化。
需要 SSI 追踪 rw-conflict。
反例二：
事务级 snapshot 不能保证外部副作用可重复。
例如调用 volatile function、访问外部服务、读取非 MVCC 对象。
课程里的 snapshot 只覆盖 PostgreSQL MVCC tuple visibility。
不要把它扩展成所有世界状态。
反例三：
事务级 snapshot 不能让你看到自己后续命令前不存在的 self-write。
本事务自己的写入还要由 command id 处理。
这就是下一节 `curcid` 的意义。
反例四：
事务级 snapshot 不阻止并发事务提交。
并发事务可以继续写入和提交。
只是当前事务的 MVCC 读不把这些后提交结果纳入视图。
反例五：
事务级 snapshot 不自动保护无限历史。
它保护的是需要满足当前 snapshot 的旧版本。
如果管理员终止事务，horizon 可以释放。
如果 vacuum_defer_cleanup_age 这类旧思路或 standby delay 参与，还要看具体版本和配置。
边界总结：
事务级 snapshot 负责读稳定。
lock 负责冲突等待。
SSI 负责 serializable anomaly。
VACUUM horizon 负责旧版本保留。
这四层不要混用。
## 25. 生产诊断提纲
看到 bloat：
先查 `backend_xmin`。
再查隔离级别和事务开始时间。
再查是否有 cursor 或 exported snapshot。
再查 replication slot 和 prepared xact。
如果最老 xmin 来自 REPEATABLE READ 事务，不要先调 autovacuum。
先和业务确认该事务为什么长期存在。
看到 serialization failure：
先确认是否 SERIALIZABLE。
再看并发读写关系。
不要假设 snapshot 变了。
看到事务内读不到新数据：
确认隔离级别。
如果是 REPEATABLE READ，这是预期。
如果是 READ COMMITTED，再看语句是否真的重新执行。
看到导入 snapshot 失败：
确认 source transaction 是否仍打开。
确认 source xmin 是否还能安装。
## 26. 深入展开：事务级 snapshot 与 `MyProc->xmin`
事务级 snapshot 的真正成本在 `MyProc->xmin`。
第一次 snapshot 的 `xmin` 会安装到 PGPROC。
只要事务级 snapshot 存活，这个 xmin 就不能随便前进。
VACUUM 和 pruning 会把它当作仍可能需要旧版本的证据。
这个证据来自读视图，而不是锁。
所以你可能看不到重锁。
也可能看不到当前 query 在跑。
但只要事务还开着，snapshot 仍在。
`backend_xmin` 就可能很老。
这解释了经典现象：
应用开启 REPEATABLE READ 事务。
查询完后长时间不提交。
数据库没有等待锁。
但表越来越膨胀。
根因是 snapshot horizon。
不是 lock wait。
不是 autovacuum 参数太小。
## 27. 深入展开：事务级 snapshot 的“复用”不是重新验证
后续语句复用 snapshot 时，不会重新扫描 ProcArray 来修正它。
这不是漏掉优化。
这是语义要求。
如果后续语句重新验证并纳入新提交事务，REPEATABLE READ 就退化成 READ COMMITTED。
因此后续 commit 对当前事务不可见。
但当前事务自己的写入仍可见。
这依赖 command id。
也就是说事务级 snapshot 固定的是“其他事务”的世界。
本事务自己的 command timeline 仍然推进。
这个边界容易被误解。
如果在 REPEATABLE READ 中：
```text
SELECT count(*) FROM t;
INSERT INTO t VALUES (...);
SELECT count(*) FROM t;
```
第二次 SELECT 可以看到自己插入。
这不是 snapshot 变了。
这是 self-visibility 规则。
## 28. 深入展开：导出 snapshot 的运维成本
`pg_export_snapshot()` 让另一个事务使用同一读视图。
这对一致性备份或多连接并行读取有用。
但导出 snapshot 要求源事务保持打开。
源事务保持打开意味着 xmin 继续被保护。
如果备份任务很慢，旧版本回收会被拖住。
如果多个 worker 导入同一 snapshot，所有读都一致。
但系统为此付出 horizon 成本。
导出 snapshot 的文件只是载体。
真正的正确性来自源事务仍活着并保护 xmin。
这就是为什么导入时要安装 imported xmin。
如果源事务结束，导入方不能凭文件要求已经可能被回收的历史继续存在。
## 29. 深入展开：事务级 snapshot 与只读事务
只读事务也可能拖住 `xmin`。
它不写 tuple。
不一定分配 XID。
但它有 snapshot。
snapshot 就足够影响 cleanup。
这点很容易被忽略。
运维上常见问题：
报表查询是只读的。
开发者认为它不会影响写入。
它确实不直接写。
但长时间 REPEATABLE READ 报表可以阻止旧版本回收。
如果报表还使用 SERIALIZABLE DEFERRABLE，行为又进入 safe snapshot 语义。
后续第 38 节会讲。
本节只强调：
read-only 不等于 free。
只读 snapshot 也有 horizon 成本。
## 30. 小型练习：同一时间线三种隔离级别
构造同一时间线：
T1 开始。
T1 第一次 SELECT。
T2 INSERT COMMIT。
T1 第二次 SELECT。
分别填表：
READ COMMITTED。
REPEATABLE READ。
SERIALIZABLE。
前两者在第二次 SELECT 的可见性上不同。
REPEATABLE READ 和 SERIALIZABLE 在基础 MVCC snapshot 上相同。
SERIALIZABLE 额外可能因为冲突在提交时失败。
这个练习能帮助你把“读视图稳定”和“可串行化检测”分开。
## 31. 源码路径分解：事务级 snapshot 的持有链
阶段一：第一次需要 snapshot。
`GetTransactionSnapshot()` 发现 `FirstSnapshotSet=false`。
阶段二：构造 snapshot。
调用 `GetSnapshotData()` 或 serializable 入口。
阶段三：复制 snapshot。
`CopySnapshot()` 复制字段和数组。
复制后的对象不再依赖静态 `CurrentSnapshotData`。
阶段四：注册引用。
`FirstXactSnapshot` 持有它。
`regd_count` 增加。
ResourceOwner 参与 cleanup。
阶段五：后续语句复用。
不重新建立读视图。
只更新 command id 相关边界。
阶段六：事务结束。
`AtEOXact_Snapshot()` 清理。
`MyProc->xmin` 释放。
这个持有链解释了为什么事务级 snapshot 能跨语句稳定。
也解释了为什么它会拖住 cleanup horizon。
## 32. 反例：长只读事务不是零成本
只读事务可能不分配 XID。
它可能不写 WAL。
它可能不持有行锁。
但只要它持有 snapshot，就可能发布 `xmin`。
旧版本要保留。
VACUUM 要等待。
这对报表系统很重要。
报表查询常被认为“只是读”。
但在高更新表上，长时间事务级读会造成明显 bloat。
治理方法不是简单提高 autovacuum。
而是缩短 snapshot 生命周期。
例如拆分查询。
使用 READ COMMITTED。
在副本上跑报表并接受 standby conflict 策略。
或用物化数据源。
## 33. 讨论题
1. 为什么事务级 snapshot 要 copy/register？
2. 事务级 snapshot 如何影响 VACUUM horizon？
3. REPEATABLE READ 为什么能避免 non-repeatable read？
4. 为什么 SERIALIZABLE 还需要 SSI？
5. 如果导入 snapshot 时源事务已结束，风险是什么？
6. `curcid` 在事务级 snapshot 中为什么仍要变化？
7. 诊断 bloat 时如何确认是不是事务级 snapshot 拖住？
8. 什么时候事务级 snapshot 反而减少 ProcArray 成本？
## 27. 本节小结
REPEATABLE READ 和 SERIALIZABLE 使用事务级 snapshot。
第一次 `GetTransactionSnapshot()` 建立读视图。
后续语句复用。
这提供事务内稳定结果。
代价是 `xmin` horizon 延长，可能阻止 VACUUM 回收旧版本。
snapshot manager 通过 copy、register、ResourceOwner 和 end-of-xact cleanup 维护生命周期。
SERIALIZABLE 在这个基础上还要 SSI 冲突检测。
理解这节，要把“稳定读视图”和“可串行化”分开。
