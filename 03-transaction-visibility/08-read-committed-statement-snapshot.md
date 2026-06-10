# PostgreSQL READ COMMITTED 语句级 snapshot
## 课程定位
前置知识：
- 已理解 `SnapshotData` 字段语义。
- 已理解 `GetSnapshotData()` 从 ProcArray 构造 running XID 集合。
- 已理解 tuple visibility 通过 snapshot 和 CLOG 共同判断。
本节唯一主问题：
为什么 READ COMMITTED 要让每条语句重新获取 snapshot，而不是让整个事务复用一个读视图？
核心矛盾：
READ COMMITTED 想让每条语句看到语句开始前已经提交的结果；
但一个事务内部又要保持 command 顺序、自身写入可见性、cursor/portal 生命周期和 executor snapshot 引用安全。
一句话模型：
READ COMMITTED 的事务生命周期可以很长，但 MVCC 读视图生命周期默认是语句级。
学完后应能判断：
- 同一个事务内两个 SELECT 为什么可能看到不同结果。
- `GetTransactionSnapshot()` 在 READ COMMITTED 下为什么会反复调用 `GetSnapshotData()`。
- statement snapshot 和 active snapshot 不是同一个概念。
- 语句级 snapshot 仍会设置 `MyProc->xmin`，但释放更快。
- READ COMMITTED 更新冲突为什么还要重新检查 tuple。
源码基线：
```text
/home/nail/postgres-lab
branch: feature/pg-pv-storage-design
commit: 2d5ed10b0bb1d1c16df7ff408eb65df4609006ae
```
## 1. 本节在总主线中的位置
上一节解释了 snapshot 字段。
本节解释什么时候创建和复用 snapshot。
READ COMMITTED 是 PostgreSQL 默认隔离级别。
默认并不意味着最简单。
它的读视图按语句变化。
这带来一个非常常见的 runtime 现象：
```text
BEGIN;
SELECT count(*) FROM t;
-- 另一个事务 INSERT 并 COMMIT
SELECT count(*) FROM t;
COMMIT;
```
两个 SELECT 可以看到不同结果。
这不是 anomaly。
这是 READ COMMITTED 的定义。
但是同一条 SELECT 内部不能扫到一半突然看到后来的 commit。
所以 snapshot 不是事务级，也不是 tuple 级。
它是语句级。
本节只讲普通 SELECT / DML 语句的 snapshot 获取。
不展开 SERIALIZABLE 的 SSI。
不展开 cursor holdable snapshot 的所有边界。
不展开 `EvalPlanQual` 的完整冲突重查。
## 2. 核心矛盾与一句话运行模型
READ COMMITTED 的承诺是：
每条语句看到语句开始时已经 committed 的数据。
它不承诺同一事务内所有语句看到同一个世界。
这降低了长期事务对旧版本回收的压力。
也减少了用户在默认隔离级别下遇到 serialization error 的概率。
代价是：
同一事务内相邻语句可能看到不同版本。
应用不能把第一条 SELECT 的结果当成后续语句的稳定读视图。
源码运行模型：
```text
StartTransactionCommand()
  -> statement/executor needs snapshot
  -> GetTransactionSnapshot()
     -> GetSnapshotData(&CurrentSnapshotData)
  -> PushActiveSnapshot()
  -> executor consumes snapshot
  -> PopActiveSnapshot()
  -> next statement repeats
```
这里有两个边界。
第一个边界是创建 snapshot。
READ COMMITTED 下 `IsolationUsesXactSnapshot()` 为 false。
因此 `GetTransactionSnapshot()` 不长期复用第一个 snapshot。
第二个边界是使用 snapshot。
executor 需要 active snapshot 才能安全调用 visibility routine。
active snapshot 的 lifetime 通常覆盖语句执行。
## 3. 核心文件分工与阅读顺序
| 顺序 | 文件 | 关注点 |
| --- | --- | --- |
| 1 | `src/backend/utils/time/snapmgr.c` | `GetTransactionSnapshot()` 如何区分隔离级别 |
| 2 | `src/include/utils/snapmgr.h` | snapshot manager 对外入口 |
| 3 | `src/include/access/xact.h` | `IsolationUsesXactSnapshot()` 相关宏 |
| 4 | `src/backend/storage/ipc/procarray.c` | `GetSnapshotData()` 成本和字段填充 |
| 5 | `src/backend/tcop/postgres.c` | query loop 与 transaction command 边界 |
| 6 | `src/backend/executor/execMain.c` | executor 如何要求 active snapshot |
| 7 | `src/backend/access/heap/heapam_visibility.c` | heap visibility 消费 snapshot |
重点先读 `snapmgr.c:GetTransactionSnapshot()`。
这里能看到：
第一次 snapshot 和后续 snapshot 的区别。
`IsolationUsesXactSnapshot()` 的分支。
`CurrentSnapshotData` 的复用和覆盖。
再读 `PushActiveSnapshot()` / `PopActiveSnapshot()`。
这能避免把“获取 snapshot”和“当前调用栈正在使用 snapshot”混淆。
最后回到 `HeapTupleSatisfiesMVCC()`。
它不关心隔离级别。
它只消费传入的 snapshot。
## 4. 关键数据结构与状态
`FirstSnapshotSet` 表示当前事务是否已经拿过第一个 transaction snapshot。
在 READ COMMITTED 下，它仍会被设置。
但后续语句不因为它而复用第一次 snapshot。
`CurrentSnapshotData` 是静态 snapshot 存储。
READ COMMITTED 下每次 `GetSnapshotData()` 会覆盖其中字段。
因此不能把它裸指针长期保存。
需要长期保存时要 copy/register。
`CurrentSnapshot` 指向当前 transaction snapshot。
在 READ COMMITTED 下，它更像“最近一次语句 snapshot”。
`SecondarySnapshot` 用于需要 latest snapshot 的场景。
本节不展开。
`MyProc->xmin` 会因为 snapshot 设置而更新。
它告诉全局：
当前 backend 仍可能需要看到不早于某个 XID 的旧版本。
READ COMMITTED 的优势是语句结束后通常能更快释放这个 xmin。
`curcid` 仍然会随 command counter 更新。
READ COMMITTED 每条语句取新 snapshot，不意味着忽略同事务内 command 可见性。
`SnapshotSetCommandId()` 会同步更新当前 snapshot 的 command id。
## 5. 主流程源码 walkthrough
一条 READ COMMITTED SELECT 进入 executor 前需要 snapshot。
调用 `GetTransactionSnapshot()`。
如果隔离级别不使用事务级 snapshot，函数走重新获取路径。
它调用 `GetSnapshotData(&CurrentSnapshotData)`。
`GetSnapshotData()` 扫 ProcArray，填 `xmin/xmax/xip/subxip`。
返回后 snapshot 进入 active stack。
executor 扫 heap page。
每个 tuple 调用 visibility routine。
visibility routine 使用这个 snapshot 判断 `xmin/xmax`。
语句结束时 active snapshot pop。
如果没有 registered snapshot，`SnapshotResetXmin()` 可能降低或清空 `MyProc->xmin`。
下一条语句再次调用 `GetTransactionSnapshot()`。
此时另一个事务已经 commit 的 XID 不再在 `xip` 中。
如果它小于新的 `xmax` 且 CLOG 显示 committed，tuple 就能变可见。
这解释了 READ COMMITTED 的现象。
同一条语句内部不会改变 snapshot。
相邻语句会改变 snapshot。
## 6. 生命周期 / ownership / cleanup
READ COMMITTED snapshot 通常由语句创建，由 executor active stack 持有。
`PushActiveSnapshot()` 增加 active_count。
`PopActiveSnapshot()` 减少 active_count。
如果引用归零，复制 snapshot 可释放。
如果调用者 register 了 snapshot，ResourceOwner 负责 abort cleanup。
普通语句结束后不应长期保留 snapshot。
否则会把 READ COMMITTED 的语义变成长 snapshot。
Portal/cursor 会改变这个生命周期。
cursor 可能需要让 snapshot 活得比一个 executor 调用更久。
因此 cursor/portal 是 snapshot leak 诊断中必须考虑的对象。
ERROR 路径下，ResourceOwner 会清理 registered snapshot。
active snapshot 的栈平衡由执行器和 portal 协议维护。
事务结束时 `AtEOXact_Snapshot()` 检查并清理状态。
## 7. 正确性机制层次
第一层是语句一致性。
同一语句内所有 tuple 使用同一 snapshot。
第二层是语句间可刷新。
下一条语句可重新获取 snapshot。
第三层是自身写入可见性。
即使新 snapshot 不包含自己的 XID，visibility 仍用 current transaction 和 `curcid` 特殊处理。
第四层是 snapshot lifecycle。
active/register 机制防止 executor 使用已经被覆盖或释放的 snapshot。
第五层是 xmin horizon。
语句 snapshot 设置 `MyProc->xmin`，防止并发 VACUUM 回收当前语句可能需要的版本。
READ COMMITTED 的核心不是“弱一致性”。
它是“语句级一致性”。
这个边界必须比 tuple scan 长，但通常不必比事务长。
## 8. 错误路径 / 异常路径 / fallback
如果没有 active snapshot 就进入需要 MVCC snapshot 的 executor 路径，通常会触发错误或 assertion。
如果调用者把 `CurrentSnapshotData` 裸指针长期保存，后续 `GetSnapshotData()` 覆盖会造成语义错误。
正确做法是 copy/register。
如果语句 ERROR，active snapshot 会随 executor unwind 清理。
registered snapshot 由 ResourceOwner 清理。
如果 cursor 长期持有 snapshot，VACUUM 回收会被拖延。
这不是 ERROR。
这是用户可见生命周期导致的 horizon 延长。
DML 冲突是另一个 fallback。
READ COMMITTED UPDATE 可能等待并发事务结束后重新检查 tuple 是否仍满足条件。
这不是重新使用旧 snapshot 就能解决的。
它需要 EvalPlanQual 等机制。
本节只说明边界，完整冲突在后续 row lock/update conflict 课程展开。
## 9. 成本、资源与跨模块传播
READ COMMITTED 的成本来自频繁获取 snapshot。
每条语句都可能扫描 ProcArray。
短事务、小语句、高并发 workload 下，这会形成 ProcArrayLock 压力。
好处是 xmin 持有时间短。
VACUUM 更容易推进。
long-running READ COMMITTED 事务如果每条语句都很短，通常不会像 REPEATABLE READ 那样长期固定第一个 xmin。
但 cursor、portal、长 SQL、open transaction idle with active snapshot 都可能例外。
成本传播路径：
```text
statement count
  -> GetSnapshotData frequency
  -> ProcArrayLock/cache pressure
  -> MyProc->xmin lifetime
  -> VACUUM cleanup horizon
```
不要只看事务持续时间。
还要看是否持有 snapshot。
`idle in transaction` 如果没有 active/registered snapshot，影响可能不同。
但它仍可能持有 locks 和 XID。
## 10. 观测与诊断入口
`txid_current_snapshot()` 可以在同一事务内连续调用。
READ COMMITTED 下相邻语句可能返回不同 snapshot。
`pg_stat_activity.backend_xmin` 可以看是否有 snapshot horizon。
`pg_stat_activity.state` 可以看是否 idle in transaction。
`pg_locks` 可看 transactionid/virtualxid lock。
`pg_stat_slru` 不直接显示 snapshot，但 subxid overflow fallback 会影响 `subtransaction`。
`log_autovacuum_min_duration` 和 VACUUM VERBOSE 可看回收是否受 old xmin 限制。
gdb 入口：
`GetTransactionSnapshot`。
`GetSnapshotData`。
`PushActiveSnapshot`。
`PopActiveSnapshot`。
`SnapshotResetXmin`。
诊断顺序：
先确认隔离级别。
再确认每条语句是否重新获取 snapshot。
再看是否有 cursor/portal/register snapshot 延长生命周期。
最后再讨论 VACUUM 延迟。
## 11. 常见误区
- READ COMMITTED 不是每个 tuple 一个 snapshot。
- READ COMMITTED 不是完全没有一致性；它有语句级一致性。
- 同一事务内看到不同结果不是 bug。
- `BEGIN` 本身不一定建立 snapshot。
- `txid_current_snapshot()` 是观测函数，也可能影响状态。
- active snapshot 和 registered snapshot 不是同一件事。
- idle in transaction 不总等于持有 old snapshot，但仍可能持有其他资源。
- READ COMMITTED UPDATE 冲突不能只靠重新取 snapshot 解释。
## 12. 课堂实验
实验 1：两个 SELECT 看到不同结果。
会话 A `BEGIN; SELECT count(*) FROM t;`。
会话 B `INSERT` 并 `COMMIT`。
会话 A 再 `SELECT count(*) FROM t;`。
在 READ COMMITTED 下第二次可看到新行。
实验 2：观察 snapshot 字符串变化。
在同一事务内多次执行 `SELECT txid_current_snapshot();`。
并发制造 commit 后比较结果。
实验 3：断点确认重新获取。
断 `GetTransactionSnapshot` 和 `GetSnapshotData`。
在 READ COMMITTED 事务里执行多条 SELECT。
观察每条语句是否进入 `GetSnapshotData()`。
## 13. 源码细读补充：`GetTransactionSnapshot()` 的 READ COMMITTED 分支
先定位 `snapmgr.c:GetTransactionSnapshot()`。
第一件事是检查事务是否已经开始。
第二件事是判断是否要使用事务级 snapshot。
READ COMMITTED 下 `IsolationUsesXactSnapshot()` 为 false。
因此普通路径不会长期复用第一次 snapshot。
函数会调用 `GetSnapshotData(&CurrentSnapshotData)`。
这会覆盖 `CurrentSnapshotData`。
所以 READ COMMITTED 的 caller 不能长期保存这个静态对象。
如果需要跨语句保存，必须 copy 或 register。
`GetLatestSnapshot()` 不是 READ COMMITTED 的替代品。
它有自己的 catalog/最新视图用途。
`GetCatalogSnapshot()` 也不是普通用户表 snapshot。
不要把 snapshot manager 的多个入口混为一个。
接着看 executor。
executor 通常假设运行时有 active snapshot。
它不关心隔离级别怎么来的。
它只拿 `estate->es_snapshot` 或 active snapshot 做 visibility。
这就是分层：
snapmgr 决定何时取视图。
executor 决定视图覆盖执行期。
heapam 决定 tuple 是否可见。
## 14. 语句级 snapshot 的边界案例
案例一：两条 SELECT 结果不同。
这正是 READ COMMITTED 的定义。
第二条 SELECT 在开始时重新获取 snapshot。
它可以看到第一条 SELECT 之后提交的事务。
案例二：一条 SELECT 内部不变。
即使扫描很久，active snapshot 不变。
后提交事务不会突然进入当前执行。
案例三：INSERT 后 SELECT 能看到自己写入。
这不是因为新 snapshot 包含自己的 XID。
这是 current transaction special case 加 command id。
案例四：UPDATE 等待并发事务后重新检查。
READ COMMITTED DML 不是只靠语句开始 snapshot。
并发 UPDATE/DELETE 冲突可能进入 wait 和 EvalPlanQual。
这属于行锁与 update conflict 主线。
案例五：cursor。
cursor 可能持有 portal snapshot。
这会让 READ COMMITTED 下某些读取表现得比普通语句更长寿。
诊断 cursor 时要看 portal 生命周期。
案例六：函数内部执行 SQL。
函数调用可能 push 自己的 active snapshot。
嵌套 SQL 的 snapshot 生命周期要按调用栈理解。
不要只看最外层事务。
## 15. READ COMMITTED 与 `backend_xmin`
一条短 SELECT 会设置 `MyProc->xmin`。
语句结束后，如果没有其他 snapshot，`SnapshotResetXmin()` 可以清理。
因此 READ COMMITTED 通常不会长期拖住 VACUUM。
但是以下情况会改变结论：
长时间运行的单条 SELECT。
长 cursor。
holdable portal。
函数或扩展注册 snapshot 后未释放。
显式长事务中存在 active snapshot。
这些场景下 `backend_xmin` 仍可能很老。
诊断时要区分：
事务打开很久。
snapshot 持有很久。
XID 持有很久。
lock 持有很久。
四者可能重叠，但不是同义词。
## 16. 诊断矩阵：同一事务内为什么看到变化
现象：
READ COMMITTED 事务内第二次 SELECT 看到新行。
源码解释：
第二条语句重新 `GetSnapshotData()`。
新提交事务不在新 snapshot 的 running set。
如果 CLOG 显示 committed，则可见。
现象：
同一 SELECT 没看到中途提交。
源码解释：
active snapshot 在 executor 期间不变。
`xmax/xip` 固定。
现象：
UPDATE 结果和 SELECT 预想不同。
源码解释：
DML 可能等待并重查可见性。
需要看 `heap_lock_tuple`、EvalPlanQual 和 row lock 课程。
现象：
VACUUM 被 READ COMMITTED 会话拖住。
源码解释：
可能是长语句或 cursor，不是普通短语句。
查 active/registered snapshot。
现象：
`txid_current_snapshot()` 调用后状态变化。
源码解释：
观测函数本身可能要求 snapshot 或 XID。
不要把观测完全当被动操作。
## 17. 成本细化：为什么默认级别仍可能成为热点
READ COMMITTED 每条语句取 snapshot。
高 QPS 短语句会频繁进入 `GetSnapshotData()`。
`GetSnapshotData()` 需要读取 ProcArray。
ProcArray 数据结构为了 cache locality 做了优化。
但 backend 数增加仍会放大成本。
提交路径同时要从 ProcArray 清除 XID。
大量短事务会让 snapshot 获取和 commit clear 竞争相关锁/cache line。
子事务 overflow 还会增加 subxid 处理。
因此默认隔离级别不是“没有 MVCC 成本”。
它只是把读视图生命周期压短。
如果 profile 里看到 `GetSnapshotData()`，需要看：
backend 数。
每秒语句数。
事务持续时间。
subxid overflow。
ProcArrayLock wait。
是否有 snapshot reuse 优化命中。
## 18. 与事务级 snapshot 的对照
READ COMMITTED 优点：
看到更新鲜的提交。
减少长期 xmin。
降低 bloat 风险。
缺点：
同一事务内多次查询结果可能变化。
每条语句都可能付 snapshot 获取成本。
REPEATABLE READ 优点：
事务内读稳定。
减少反复获取 snapshot。
缺点：
长期拖住 xmin。
可能出现 serialization/update conflict。
选择隔离级别时，不要只问“一致性强不强”。
要问：
业务是否需要事务内稳定读？
能否接受旧版本保留？
语句频率和事务长度如何？
## 19. 逐函数阅读任务
任务 1：读 `GetTransactionSnapshot()`。
标出 READ COMMITTED 分支。
标出 `IsolationUsesXactSnapshot()` 为 false 时的路径。
标出每次调用 `GetSnapshotData()` 的位置。
标出返回的 snapshot 存在哪里。
解释为什么这个路径不能长期裸持有 `CurrentSnapshotData`。
任务 2：读 `PushActiveSnapshot()`。
找出它何时 copy snapshot。
找出它如何增加 active_count。
找出 active stack 的层级字段。
解释 executor 为什么不直接依赖全局 CurrentSnapshot。
任务 3：读 `PopActiveSnapshot()`。
记录它何时减少 active_count。
记录它何时释放 snapshot。
记录它何时调用 `SnapshotResetXmin()`。
解释语句结束如何释放 horizon。
任务 4：读 query execution 入口。
在 `pquery.c` 或 executor 初始化路径中找到 snapshot 如何进入 query descriptor。
把 tcop、portal、executor、heapam 四层画成调用链。
任务 5：读 UPDATE 冲突相关入口。
只需要标出 READ COMMITTED DML 可能等待并重查。
不要在本节展开 EPQ。
把它作为后续 row lock 课程入口。
## 20. 案例推演：READ COMMITTED 的四个时间点
时间点 1：
事务开始。
还不一定有 XID。
也不一定有 snapshot。
时间点 2：
第一条 SELECT 开始。
获取 snapshot S1。
S1 的 `xmax` 固定。
S1 的 `xip` 固定。
时间点 3：
并发事务提交。
S1 不变化。
当前 SELECT 不看见它。
时间点 4：
第二条 SELECT 开始。
获取 snapshot S2。
并发事务不再 running。
如果 committed，则可见。
这个推演解释了：
同一事务内结果变化。
同一语句内结果稳定。
如果应用需要事务内结果稳定，就不能依赖 READ COMMITTED。
如果应用需要尽快看到新提交，READ COMMITTED 正好满足。
## 21. 诊断案例：为什么默认隔离级别也会拖住 VACUUM
情况 1：
一条 SELECT 执行 30 分钟。
它是 READ COMMITTED。
但 active snapshot 覆盖整个执行期。
VACUUM 必须等待它。
情况 2：
客户端打开 cursor，慢慢 fetch。
snapshot 可能跟 portal 生命周期绑定。
即使每次 fetch 很短，整体 horizon 仍可能老。
情况 3：
函数或扩展注册 snapshot 后未及时释放。
READ COMMITTED 语义不自动救你。
情况 4：
事务 idle in transaction。
需要确认是否仍有 snapshot。
不能只看 state。
诊断路径：
查 `backend_xmin`。
查 cursor。
查事务开始时间。
必要时断 `RegisterSnapshot`。
不要直接把所有问题归咎于 autovacuum。
## 22. 与后续课程的衔接
第 9 节讲事务级 snapshot。
它和 READ COMMITTED 的最大差别是复用第一个 snapshot。
第 10 节讲 command id。
它解释为什么 READ COMMITTED 内自己的写入仍能按 command 顺序可见。
第 11 节讲 active/registered snapshot。
它解释为什么语句级 snapshot 也可能因为 portal 变长。
第 12 节讲 cleanup horizon。
它解释 READ COMMITTED 长语句如何影响 VACUUM。
第 23 节之后的 update conflict 会解释 DML 等待和 EPQ。
本节不要提前把这些都讲完。
只要记住 READ COMMITTED 的主轴：
每条语句一个新读视图。
## 23. 深入展开：READ COMMITTED 的“新鲜度”边界
READ COMMITTED 的名字容易让人误以为每次读取 tuple 时都会看到最新 committed 状态。
这不对。
它的边界是 statement start。
一条语句开始时建立 snapshot。
这条语句内部稳定。
下一条语句重新开始。
因此 READ COMMITTED 的新鲜度是语句级，不是 tuple 级。
这点对长查询很重要。
一个查询跑 10 分钟。
第 1 分钟后别的事务提交大量数据。
这个查询仍然使用第 0 分钟的 snapshot。
下一条查询才看见。
这也是为什么长 SELECT 可以拖住 VACUUM。
它虽然是 READ COMMITTED，但 active snapshot 仍持续 10 分钟。
如果应用用 READ COMMITTED 做分页，每页一个新 SQL。
不同页可能来自不同 snapshot。
这可能导致重复或漏读。
如果业务需要稳定分页，应使用 cursor、事务级 snapshot 或显式排序/锚点策略。
这不是 PostgreSQL bug。
这是隔离级别 contract。
## 24. 深入展开：READ COMMITTED DML 与 SELECT 的差别
普通 SELECT 的 snapshot 边界相对简单。
语句开始取 snapshot，执行期使用。
DML 更复杂。
UPDATE/DELETE 先按 snapshot 找候选 tuple。
如果遇到并发更新，可能等待对方事务结束。
等待后，READ COMMITTED 不能继续盲目使用旧 tuple 状态。
它需要重查 tuple 是否仍可更新。
这会进入 `HeapTupleSatisfiesUpdate()`、tuple lock、EvalPlanQual 等路径。
所以 READ COMMITTED 下：
SELECT 的“每语句 snapshot”解释读结果。
UPDATE 的最终行为还要叠加冲突等待和重查。
例如：
事务 A 更新某行但未提交。
事务 B 在 READ COMMITTED 下 UPDATE 同一行。
B 等待 A。
A commit 后，B 重新检查新版本是否满足 WHERE。
如果满足，B 更新新版本。
如果不满足，B 跳过。
这个行为不能只用 `GetTransactionSnapshot()` 解释。
它是 snapshot 与 row lock/update conflict 的组合。
后续第 23 节会专门讲。
## 25. 深入展开：为什么 `BEGIN` 不等于 snapshot 开始
用户常以为：
`BEGIN` 的瞬间就固定了读视图。
在 READ COMMITTED 下不是。
事务 block 开始，只是事务状态开始。
snapshot 通常在语句需要时才获取。
这解释了：
```text
BEGIN;
-- 等待一分钟
SELECT ...
```
SELECT 的 snapshot 是 SELECT 开始时的状态，而不是 BEGIN 时的状态。
这也解释了 `pg_current_xact_id_if_assigned()` 和 snapshot 观测的差别。
事务可以有 transaction state。
可以有 virtual transaction id。
可以没有真实 XID。
也可以尚未获取 MVCC snapshot。
这几个状态不能混用。
诊断时要分别问：
事务什么时候开始？
XID 什么时候分配？
snapshot 什么时候建立？
语句什么时候执行？
## 26. 深入展开：`CurrentSnapshotData` 的复用风险
`snapmgr.c` 使用静态 `CurrentSnapshotData`。
这是性能上的选择。
READ COMMITTED 每条语句可能重新填它。
如果某个代码路径保存它的指针，并在下一次 `GetSnapshotData()` 后继续使用，就会读到被覆盖的字段。
因此长期持有必须 `CopySnapshot()` 或 `RegisterSnapshot()`。
这个规则对扩展开发很关键。
错误示例：
函数缓存 `GetTransactionSnapshot()` 返回值到全局变量。
下一条语句覆盖 CurrentSnapshotData。
函数后续用旧指针，以为还是旧 snapshot。
正确做法：
如果只在当前调用栈使用，push active。
如果跨调用保存，copy/register 并绑定 ResourceOwner。
如果跨事务保存，基本不应该保存普通 snapshot；应使用导出 snapshot 协议，并理解 xmin 成本。
## 27. 小型练习：画两会话时间线
画四行：
会话 A 事务状态。
会话 A snapshot。
会话 B XID/CLOG。
会话 A 查询结果。
时间点：
A `BEGIN`。
A `SELECT count(*)`。
B `INSERT`。
B `COMMIT`。
A `SELECT count(*)`。
在 READ COMMITTED 下，A 有两个 snapshot。
第一个 snapshot 把 B 视为不存在或 future。
第二个 snapshot 看到 B committed。
把这张图再改成 REPEATABLE READ。
你会发现差别只在 A 是否复用第一个 snapshot。
## 28. 源码路径分解：一条 SELECT 的 snapshot 生命周期
阶段一：事务命令开始。
`StartTransactionCommand()` 确保当前 backend 处于可执行语句的事务状态。
这一步不等于已经建立 MVCC snapshot。
阶段二：executor 需要 snapshot。
上层调用 `GetTransactionSnapshot()`。
READ COMMITTED 下进入重新获取路径。
阶段三：ProcArray 读取。
`GetSnapshotData()` 扫描 running XID。
填 `xmin/xmax/xip/subxip`。
同时可能更新 `MyProc->xmin`。
阶段四：snapshot 进入 executor。
query descriptor 或 active snapshot stack 持有它。
heap scan 不再自己获取 snapshot。
阶段五：visibility 消费。
`HeapTupleSatisfiesMVCC()` 对每个 tuple 使用同一个 snapshot。
阶段六：语句结束。
active snapshot pop。
如果没有 registered snapshot，`SnapshotResetXmin()` 可能清理 `MyProc->xmin`。
阶段七：下一条语句。
重复阶段二到六。
如果并发事务已提交，它可能进入新 snapshot 的可见世界。
这个分解能解释：
READ COMMITTED 的稳定范围是 executor 执行期。
不是事务 block。
也不是单个 tuple。
## 29. 反例：把 READ COMMITTED 当事务级缓存
应用代码常这样写：
先 SELECT 一批 id。
中间执行很多业务逻辑。
再 SELECT 同一条件。
如果隔离级别是 READ COMMITTED，第二次 SELECT 不是第一次的复核。
它是新的读视图。
中间任何已提交事务都可能改变结果。
如果应用希望“同一事务中我看到的集合不变”，READ COMMITTED 不提供这个保证。
可选方案包括：
使用 REPEATABLE READ。
显式锁定需要保护的行。
把第一次结果物化到临时表。
使用业务版本号或时间戳作为锚点。
选择哪种方案取决于目标：
稳定读。
防并发修改。
防 write skew。
还是只需要分页一致性。
## 30. 讨论题
1. READ COMMITTED 为什么选择语句级 snapshot，而不是事务级 snapshot？
2. 同一条 SELECT 为什么不能在扫描中途刷新 snapshot？
3. `CurrentSnapshotData` 为什么不能长期裸用？
4. 语句级 snapshot 如何影响 VACUUM horizon？
5. READ COMMITTED 下 cursor 为什么可能改变 snapshot 生命周期？
6. DML 等待并发事务后为什么需要重查条件？
7. `backend_xmin` 能说明什么，不能说明什么？
8. 如何区分 ProcArray 获取 snapshot 成本和 heap visibility 成本？
## 24. 本节小结
READ COMMITTED 的核心是语句级读视图。
每条语句通常重新调用 `GetSnapshotData()`。
同一语句内部使用稳定 snapshot。
相邻语句可以看到新的 committed 结果。
active snapshot 保护执行期使用。
registered snapshot 和 portal/cursor 可能延长生命周期。
这种设计降低长期 snapshot 对 VACUUM 的压力，但增加频繁获取 snapshot 的成本。
理解它，要同时看隔离级别、snapshot manager、ProcArray 和 executor active snapshot。
