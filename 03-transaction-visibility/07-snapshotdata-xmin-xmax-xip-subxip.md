# PostgreSQL SnapshotData 字段语义
## 课程定位
前置知识：
- 已理解 XID 分配、ProcArray、CLOG/pg_xact 和子事务父链。
- 已理解普通 MVCC 读不是读“最新行”，而是读某个时间点下可见的 tuple version。
- 已理解 tuple header 的 `xmin` / `xmax` 指向事务身份，不直接等于可见性结论。
本节唯一主问题：
`SnapshotData` 为什么只靠 `xmin`、`xmax`、`xip`、`subxip`、`suboverflowed`、`takenDuringRecovery` 和 `curcid` 这一组字段，就能描述一个 MVCC 读视图？
核心矛盾：
读取 tuple 时希望用一份本地 snapshot 快速判断可见性；
但事务运行集合在全局共享内存中不断变化，子事务可能 overflow，recovery 中 running XID 来源也不是普通 ProcArray。
PostgreSQL 的选择是：
snapshot 创建时把“当时仍运行的事务集合”压缩成一个区间加两个数组。
后续 tuple visibility 不再重新扫描全局 ProcArray，而是用这份 frozen view 判断。
学完后应能判断：
- `xmin` 不是“最老已提交事务”，而是本 snapshot 下仍可能运行的下界。
- `xmax` 不是“最大已提交事务”，而是本 snapshot 下新事务不可见的上界。
- `xip` 只列出 `[xmin, xmax)` 中仍在运行的 top-level XID。
- `subxip` 只在未 overflow 或 recovery 特殊场景下能完整回答子事务问题。
- `curcid` 只回答本事务内 command 可见性，不是全局 MVCC 时间。
- `takenDuringRecovery` 改变 `xip/subxip` 的解释来源。
源码基线：
```text
/home/nail/postgres-lab
branch: feature/pg-pv-storage-design
commit: 2d5ed10b0bb1d1c16df7ff408eb65df4609006ae
```
## 1. 本节在总主线中的位置
前 6 节回答了事务身份和事务结果。
现在进入 snapshot。
事务系统能告诉你：
某个 XID 是否仍在运行。
某个 XID 是否已经 committed。
某个 subxid 属于哪个 parent。
但普通 SELECT 不能在每个 tuple 上重新扫描 ProcArray。
那会把 heap scan 变成共享锁热点。
因此 SQL 语句开始时要把全局运行集合冻结成本地对象。
这个本地对象就是 `SnapshotData`。
本节不讲 READ COMMITTED 与 REPEATABLE READ 何时复用 snapshot。
那是下一节和第 9 节。
本节也不展开 tuple visibility 的所有分支。
第 14 节会专门讲 `HeapTupleSatisfiesMVCC()`。
本节只建立字段语义。
一条主线是：
```text
ProcArray running state
  -> GetSnapshotData()
  -> SnapshotData fields
  -> XidInMVCCSnapshot()
  -> HeapTupleSatisfiesMVCC()
```
如果字段语义理解错，后面的隔离级别、VACUUM horizon、index-only scan 诊断都会错。
## 2. 核心矛盾与一句话运行模型
一句话运行模型：
`SnapshotData` 把不断变化的全局 running XID 集合，冻结成 “区间 + 例外集合 + 子事务 fallback 标志 + 本事务命令边界”。
全局运行集合是动态的。
一个 backend 正在分配 XID。
另一个 backend 正在 commit。
第三个 backend 正在从 ProcArray 清除 XID。
如果每次 tuple visibility 都去问“此刻它是否 running”，同一条 SQL 可能在扫描过程中得到不一致答案。
snapshot 的核心就是把“此刻”固定下来。
`xmin` 是区间下界。
任何 XID 小于 `xmin`，对这个 snapshot 来说不再是 running。
它要么已经完成，要么太老而不能再出现在 running 集合。
`xmax` 是区间上界。
任何 XID 大于等于 `xmax`，对这个 snapshot 来说都当作未来事务。
即使它稍后 commit，也不能被当前 snapshot 看见。
`xip` 是区间内的 running top-level XID 列表。
它回答：
这个 XID 在 snapshot 创建时是否还没有结束。
`subxip` 是区间内 running subxid 的 fast path。
它回答：
这个 tuple header 里的子事务 XID 是否仍属于某个 running top-level transaction。
`suboverflowed` 是准确性边界。
它不是“snapshot 坏了”。
它表示 `subxip` 不完整，必须用 `pg_subtrans` 追溯 parent。
`curcid` 是 self-visibility 边界。
它让同一事务内部不同 command 之间仍能保持 SQL 语义。
## 3. 核心文件分工与阅读顺序
推荐阅读顺序：
| 顺序 | 文件 | 关注点 |
| --- | --- | --- |
| 1 | `src/include/utils/snapshot.h` | `SnapshotType`、`SnapshotData` 字段定义 |
| 2 | `src/backend/storage/ipc/procarray.c` | `GetSnapshotData()` 如何填字段 |
| 3 | `src/backend/utils/time/snapmgr.c` | snapshot 生命周期、复制、注册和 `XidInMVCCSnapshot()` |
| 4 | `src/backend/access/heap/heapam_visibility.c` | `HeapTupleSatisfiesMVCC()` 如何消费 snapshot |
| 5 | `src/backend/access/transam/subtrans.c` | overflow 后 parent chain fallback |
| 6 | `src/include/storage/proc.h` | `PGPROC.xid`、`xmin`、subxid cache |
不要从 `HeapTupleSatisfiesMVCC()` 直接硬读。
那会混在 xmin/xmax、CLOG、hint bit、command id 和 tuple lock 分支里。
先把 snapshot 字段当成接口 contract 读清楚。
`snapshot.h` 是入口。
`SnapshotData` 的字段旁边有注释。
这些注释很短，但密度很高。
`procarray.c` 是生成侧。
它决定哪些 running XID 被复制进 snapshot。
`snapmgr.c` 是生命周期侧。
它决定哪些 snapshot 可以被复用、注册、导出、释放。
`heapam_visibility.c` 是消费侧。
它不重新解释 ProcArray。
它只问 `XidInMVCCSnapshot()` 和事务结果状态。
## 4. 关键数据结构与状态
`SnapshotData.snapshot_type` 指明规则族。
本节关注 `SNAPSHOT_MVCC`。
特殊 snapshot 另有课程。
`xmin` 是最低仍可能运行 XID。
更准确说：
小于 `xmin` 的普通 XID 不需要再查 snapshot running set。
它可能 committed，也可能 aborted。
具体结果要问 CLOG 或 hint bit。
它不可能被当前 snapshot 当作 in-progress。
`xmax` 是 upper bound。
大于等于 `xmax` 的 XID 对当前 snapshot 都视为 in-progress / future。
它们不该对当前 MVCC 读可见。
这个规则防止扫描过程中后启动或后分配的事务突然变可见。
`xip` 是 top-level XID 数组。
只有 `[xmin, xmax)` 内的 XID 才需要在这里查。
因此 `XidInMVCCSnapshot()` 先做 range check。
低于 `xmin` 直接 false。
高于等于 `xmax` 直接 true。
中间才查数组。
`xcnt` 是 `xip` 元素数。
它不是全系统事务数。
它只是这个 snapshot 看到的运行事务数量。
`subxip` 是子事务 XID 数组。
`subxcnt` 是数量。
当 `suboverflowed` 为 false 时，`subxip` 可以帮助快速判断 running subxid。
当 `suboverflowed` 为 true 时，不在 `subxip` 不能证明它不 running。
必须追 parent。
`takenDuringRecovery` 表示 snapshot 来自 recovery shaped running set。
Hot standby 里没有普通 backend ProcArray 的全部写事务状态。
系统维护 KnownAssignedXids。
这会影响 `xip/subxip` 的来源和解释。
`curcid` 是 command id 边界。
`HeapTupleSatisfiesMVCC()` 用它判断本事务内同一 XID 的写入是否早于当前 command。
这解释了为什么 snapshot 不是纯事务 ID 集合。
## 5. 主流程源码 walkthrough
主流程从 `GetSnapshotData()` 开始。
调用方传入一个 `SnapshotData` 存储对象。
`GetSnapshotData()` 持有 ProcArrayLock 读取运行事务状态。
它填出 `xmin`、`xmax`、`xip`、`subxip` 和 overflow 标记。
然后 `snapmgr.c` 负责把这个 snapshot 放进当前语句或事务生命周期。
普通 tuple 判断进入 `HeapTupleSatisfiesMVCC()`。
对于 tuple `xmin`：
如果是当前事务 XID，转入 command id 分支。
如果 hint bit 已经说明 committed 或 invalid，减少 CLOG 查询。
否则可能先问 `XidInMVCCSnapshot(xmin, snapshot)`。
如果 `xmin` 在 snapshot 中 running，tuple 对当前 snapshot 不可见。
如果不 running，再问事务是否 committed。
对于 tuple `xmax`：
如果没有有效 `xmax`，tuple 没有被删除或锁定。
如果 `xmax` 在 snapshot 中 running，删除对当前 snapshot 不可见。
如果 `xmax` committed，tuple 可能已经死亡。
`XidInMVCCSnapshot()` 的运行模型是三步：
```text
if xid < snapshot->xmin: not running
if xid >= snapshot->xmax: running/future
else search xip/subxip or subtrans fallback
```
这个函数是 snapshot 字段语义的集中体现。
注意：
`XidInMVCCSnapshot()` 不回答 committed。
它只回答“在这个 snapshot 中是否视为 in-progress”。
事务最终结果仍由 CLOG/pg_xact 和 hint bit 判定。
所以 snapshot 和 CLOG 是两层。
snapshot 定义“读视图的时间边界”。
CLOG 定义“事务最终命运”。
## 6. 生命周期 / ownership / cleanup
`SnapshotData` 可以是静态对象、复制对象、active snapshot 或 registered snapshot。
`snapmgr.c` 中有 `CurrentSnapshotData`、`SecondarySnapshotData`、`CatalogSnapshotData`。
这些静态对象减少重复分配。
但调用者如果需要跨函数、跨 executor 节点或 portal 持有 snapshot，必须复制或注册。
`active_count` 表示 active snapshot stack 上的引用。
`regd_count` 表示 RegisteredSnapshots 的引用。
两者都为 0 时，复制出来的 snapshot 可以释放。
`PushActiveSnapshot()` 会把 snapshot 放入 active stack。
`PopActiveSnapshot()` 释放这一层引用。
`RegisterSnapshot()` 会把 snapshot 注册到 ResourceOwner。
`UnregisterSnapshot()` 解除注册。
ERROR 时 ResourceOwner 负责清理注册引用。
active stack 则由上层 executor / portal 调用协议维护。
`SnapshotResetXmin()` 是重要 cleanup。
当没有 active 或 registered snapshot 需要保护 xmin 时，backend 可以清空 `MyProc->xmin`。
否则 VACUUM horizon 会被长期 snapshot 拖住。
这也是 snapshot 生命周期不是内存生命周期的原因。
它还影响全局回收边界。
## 7. 正确性机制层次
第一层正确性是 snapshot 一致性。
同一 SQL 语句使用同一个 snapshot，就不会在扫描中途看到后来的 commit。
第二层正确性是区间规则。
`xmin/xmax` 让大多数 XID 不需要查数组，也避免未来事务被误看见。
第三层正确性是 running set。
`xip/subxip` 记录创建 snapshot 时仍运行的事务。
第四层正确性是 overflow fallback。
`suboverflowed` 不允许系统把“不在 subxip”误判为“不 running”。
第五层正确性是 self-command visibility。
`curcid` 保证同一事务内部也遵守 SQL command 顺序。
第六层正确性是 recovery 区分。
`takenDuringRecovery` 防止把 hot standby 的 KnownAssignedXids 规则误用成普通 backend ProcArray 规则。
这些层次共同形成一个判断：
snapshot 不是时间戳。
snapshot 是一个可验证的事务集合边界。
它的每个字段都服务一个具体分支。
## 8. 错误路径 / 异常路径 / fallback
常见 fallback 是 subxid overflow。
如果 `suboverflowed` 为 true，`XidInMVCCSnapshot()` 需要调用 `SubTransGetTopmostTransaction()`。
这会从 `pg_subtrans` 追溯 parent。
正确性保持，成本上升。
另一个 fallback 是 snapshot copy。
如果调用者要长期持有 snapshot，不能直接拿静态 `CurrentSnapshotData`。
必须 `CopySnapshot()` 或注册。
否则后续 `GetSnapshotData()` 可能覆盖内容。
ERROR cleanup 依赖 ResourceOwner。
registered snapshot 如果没有 unregister，事务 abort 时会清理。
active snapshot stack 若被调用协议破坏，通常会触发 assertion 或 end-of-xact 检查。
导出 snapshot 还有额外边界。
导出到文件时要序列化字段。
导入时要重新安装 xmin。
如果源事务不再能保护 xmin，导入会失败。
这说明 snapshot 不是纯数据文件。
它必须和某个仍存在的 xmin owner 关联。
## 9. 成本、资源与跨模块传播
获取 snapshot 的成本来自 ProcArray 扫描。
backend 数越多，ProcArray 越大，`GetSnapshotData()` 越贵。
running XID 越多，`xip` 越长。
子事务越多，`subxip` 越长或 overflow。
overflow 会把成本转给 `pg_subtrans`。
长期 snapshot 会把成本传播给 VACUUM。
因为 `MyProc->xmin` 不能提前清掉。
old tuple version 不能被回收。
visibility map all-visible 也可能推迟。
READ COMMITTED 每条语句取 snapshot，增加 ProcArray 读取频率。
REPEATABLE READ 复用 snapshot，减少获取频率，但延长 xmin 持有时间。
cursor、portal、logical decoding、复制槽和 prepared transaction 都可能形成 horizon 传播。
本节只讲普通 snapshot。
后续会拆开各自语义。
## 10. 观测与诊断入口
SQL 可见入口：
`pg_stat_activity.backend_xid` 和 `backend_xmin`。
`backend_xmin` 能提示某个 backend 正在保护旧版本。
它不是 snapshot 全部字段。
`txid_current_snapshot()` 可显示类似 `xmin:xmax:xip_list` 的用户层表示。
它不展示 `subxip` 完整细节。
`pg_locks` 可看到 virtualxid 和 transactionid lock。
这些不是 snapshot 字段，但能帮助定位谁在运行。
`pg_stat_slru` 的 `subtransaction` 可间接提示 overflow fallback 压力。
`VACUUM VERBOSE` 可提示旧版本无法回收。
gdb 入口：
`GetSnapshotData`、`GetTransactionSnapshot`、`XidInMVCCSnapshot`、`HeapTupleSatisfiesMVCC`。
观察字段：
`snapshot->xmin`、`xmax`、`xcnt`、`subxcnt`、`suboverflowed`、`curcid`。
不要期待 SQL 直接显示所有字段。
很多判断只能通过断点或源码插桩确认。
## 11. 常见误区
- 把 `xmin` 当成最老已提交事务。
- 把 `xmax` 当成当前最大 XID。
- 以为不在 `xip` 就一定 committed。
- 忽略 `[xmin, xmax)` 范围判断。
- 以为 `suboverflowed` 代表 snapshot 不可靠。
- 把 `curcid` 当成全局事务时间。
- 把 snapshot 生命周期当成普通内存引用。
- 用 `txid_current_snapshot()` 直接推断所有内部字段。
## 12. 课堂实验
实验 1：观察用户层 snapshot。
```sql
BEGIN;
SELECT txid_current_snapshot();
SELECT txid_snapshot_xmin(txid_current_snapshot());
SELECT txid_snapshot_xmax(txid_current_snapshot());
COMMIT;
```
开另一个会话保持写事务，再观察 `xip` 是否变化。
实验 2：断点观察字段。
在 debug 环境断 `GetSnapshotData` 和 `XidInMVCCSnapshot`。
执行一次普通 SELECT。
打印 `snapshot->xmin`、`xmax`、`xcnt`、`subxcnt`、`suboverflowed`。
实验 3：观察 snapshot 对 VACUUM 的影响。
会话 A 开长事务并 SELECT。
会话 B UPDATE/DELETE 大量行后 VACUUM。
观察 `pg_stat_activity.backend_xmin` 和 VACUUM 是否不能回收全部 dead tuple。
## 13. 源码细读补充：从 ProcArray 到 tuple 判断
第一步，读 `snapshot.h`。
不要先看字段名，先看 `SnapshotType`。
`SNAPSHOT_MVCC` 的语义是普通 MVCC 读。
它不同于 `SNAPSHOT_SELF`。
它不同于 `SNAPSHOT_ANY`。
它不同于 `SNAPSHOT_DIRTY`。
本节所有 `xmin/xmax/xip` 讨论都默认 `SNAPSHOT_MVCC`。
第二步，读 `SnapshotData` 字段注释。
`xmin` 注释告诉你：
小于它的 XID 都不在 snapshot running set。
这不是 committed 断言。
这是 in-progress 断言的下界。
`xmax` 注释告诉你：
大于等于它的 XID 都视为 in-progress。
这不是说这些事务现在一定存在。
它是 snapshot 对未来 XID 的屏障。
第三步，读 `procarray.c:GetSnapshotData()`。
这里会扫描 `PGPROC`。
它读取每个 backend 发布的 top-level xid。
它读取 subxid cache。
它维护 `xmin`。
它填充 `xip`。
它在 subxid cache overflow 时设置 `suboverflowed`。
第四步，读 `snapmgr.c:XidInMVCCSnapshot()`。
这是消费 snapshot 字段的最短路径。
先做 range check。
再判断 recovery 与普通 snapshot。
再查 `subxip` 或 `xip`。
最后在 overflow 情况下追 parent。
第五步，回到 `heapam_visibility.c`。
`HeapTupleSatisfiesMVCC()` 对 `xmin` 和 `xmax` 分别调用事务状态和 snapshot 判断。
它不会重新扫描 ProcArray。
这说明 snapshot 是 visibility 的输入，而不是附属缓存。
## 14. 字段判定矩阵
看到 `xid < snapshot->xmin`：
这个 XID 对本 snapshot 不 running。
下一步通常要问 CLOG 或 hint bit。
不要直接说它 committed。
看到 `xid >= snapshot->xmax`：
这个 XID 对本 snapshot 是未来事务。
tuple 的插入者是这种 XID，则 tuple 不可见。
tuple 的删除者是这种 XID，则删除对当前 snapshot 不可见。
看到 `xmin <= xid < xmax` 且 xid 在 `xip`：
这个 top-level XID 在 snapshot 创建时 running。
它的插入对当前 snapshot 不可见。
它的删除对当前 snapshot 不生效。
看到 `xmin <= xid < xmax` 且 xid 不在 `xip`：
如果它是 top-level XID，说明不 running。
仍要问 commit/abort。
如果它可能是 subxid，要看 `suboverflowed`。
看到 `suboverflowed = false`：
`subxip` 是子事务 fast path。
不在 `subxip` 的 subxid 不是 snapshot 中的 running subxid。
看到 `suboverflowed = true`：
不能用“不在 subxip”下结论。
必须追 `pg_subtrans` 找 top-level XID。
看到 `takenDuringRecovery = true`：
snapshot 来源不是普通 backend ProcArray。
要把 KnownAssignedXids 纳入 mental model。
看到 `curcid`：
不要把它和全局 XID 比较。
只用它判断本事务内 command 顺序。
## 15. Runtime case：为什么同一条扫描稳定
场景：
会话 A 开始一次长 SELECT。
会话 B 在 SELECT 扫描过程中 INSERT 并 COMMIT。
READ COMMITTED 下，会话 A 下一条语句可能看到新行。
但当前这条 SELECT 不应中途看到。
原因是会话 A 的 executor 使用同一个 active snapshot。
这份 snapshot 的 `xmax` 在语句开始时已经确定。
会话 B 后来分配或提交的 XID 大概率大于等于这个 `xmax`。
因此它被当作 future。
如果会话 B 的 XID 已经在会话 A snapshot 创建前分配但未提交，它会在 `xip` 中。
即使会话 B 扫描中途 commit，它仍对当前 snapshot 不可见。
这解释了“statement-level consistency”。
不是因为 heap scan 锁住了表。
不是因为 SELECT 等待所有并发写入。
而是 snapshot 字段把读视图固定。
## 16. Runtime case：为什么 `txid_current_snapshot()` 不能替代内核诊断
`txid_current_snapshot()` 能展示用户可见的 xmin、xmax 和 xip 列表。
它对理解事务级边界有帮助。
但它不展示所有内部细节。
它不直接展示 `subxip` 完整情况。
它不展示 `active_count`。
它不展示 `regd_count`。
它不展示 RegisteredSnapshots heap。
它不展示 snapshot 是不是来自 recovery。
它也不告诉你某个 heap tuple 的 hint bit 是否绕过了 CLOG 查询。
因此诊断时不能只拿它当 truth。
正确做法是：
用 SQL 确认大方向。
用 `pg_stat_activity.backend_xmin` 查 horizon。
用 gdb 在 `GetSnapshotData()` 和 `XidInMVCCSnapshot()` 验证字段。
用 `pageinspect` 或断点看 tuple header。
把四者连起来。
## 17. 诊断顺序：一行为什么不可见
第一步，看 tuple header。
确认 `xmin`、`xmax`、infomask、hint bit。
第二步，看当前 snapshot。
确认 `xmin/xmax/xip/suboverflowed/curcid`。
第三步，对插入者 `xmin` 做判断。
它是当前事务吗？
它小于 snapshot xmin 吗？
它大于等于 snapshot xmax 吗？
它在 xip 中吗？
它是 subxid 吗？
第四步，问事务结果。
如果不 running，查 CLOG 或 hint bit。
第五步，对删除者 `xmax` 做同样判断。
第六步，如果是当前事务，进入 command id 分支。
第七步，如果是 row lock / MultiXact，不要套用普通 delete 语义。
后续课程会展开。
这个顺序能避免两个常见错误：
把 snapshot 判断和 commit 判断混成一步。
把 `xmax` 一律理解为删除者。
## 18. 版本与实现边界
`SnapshotData` 的核心语义稳定。
但生成 snapshot 的具体优化会变化。
例如 ProcArray 扫描复用、completion count、KnownAssignedXids 管理都可能演化。
课程中要区分：
稳定语义是 snapshot 固定读视图。
当前实现是 `GetSnapshotData()` 填字段。
稳定语义是 `suboverflowed` 触发 parent fallback。
当前实现是通过 `pg_subtrans` SLRU 查 parent。
稳定语义是 `curcid` 管 self-command visibility。
当前实现是 `SnapshotSetCommandId()` 更新 snapshot。
诊断时要以本地源码为准。
不要把某篇旧博客里的字段顺序或函数名当成当前 truth。
## 19. 逐函数阅读任务
任务 1：`snapshot.h` 字段标注。
把 `SnapshotData` 每个字段分成四类。
第一类是事务区间字段：`xmin`、`xmax`。
第二类是数组字段：`xip`、`subxip`、`xcnt`、`subxcnt`。
第三类是解释标志：`suboverflowed`、`takenDuringRecovery`。
第四类是生命周期和本地语义：`curcid`、`active_count`、`regd_count`。
标注时不要复制结构体。
只写每个字段回答哪个可见性问题。
任务 2：`GetSnapshotData()` 生成字段。
在 `procarray.c` 中找到函数入口。
记录它什么时候持有 ProcArrayLock。
记录它什么时候设置 `xmin`。
记录它如何处理每个 `PGPROC` 的 xid。
记录它如何复制 subxids。
记录它如何处理 overflow。
记录它如何让 `MyProc->xmin` 与 snapshot 对齐。
任务 3：`XidInMVCCSnapshot()` 消费字段。
画出 range check。
再画出非 recovery 的 top-level XID 判断。
再画出 suboverflowed fallback。
最后把每个 return true/false 写成一句话语义。
不要写“返回 true”。
要写“在这个 snapshot 中视为 running”。
任务 4：`HeapTupleSatisfiesMVCC()` 接入点。
标出对 `xmin` 的判断。
标出对 `xmax` 的判断。
标出当前事务分支。
标出 hint bit 分支。
标出调用 `XidInMVCCSnapshot()` 的位置。
最后回答：
snapshot 判断和 CLOG 判断谁先谁后，为什么。
任务 5：recovery snapshot 对照。
在 `procarray.c` 搜 KnownAssignedXids。
把 hot standby running set 和普通 ProcArray running set 分开画。
解释 `takenDuringRecovery` 为什么不能省。
## 20. 案例推演：给定字段如何判定
案例 1：
`snapshot xmin=100, xmax=120, xip={105,110}`。
tuple `xmin=90`。
结论：
不在 snapshot running set。
下一步查事务结果。
如果 committed，则插入可见。
如果 aborted，则插入不可见。
案例 2：
同一 snapshot，tuple `xmin=105`。
结论：
插入者在 `xip`。
它在 snapshot 中 running。
插入不可见。
即使它在扫描后半段 commit，也不改变当前 snapshot。
案例 3：
同一 snapshot，tuple `xmin=121`。
结论：
大于等于 `xmax`。
它是 future transaction。
插入不可见。
案例 4：
tuple `xmax=110`。
删除者在 snapshot running set。
删除对当前 snapshot 不生效。
如果插入可见，则旧版本仍可见。
案例 5：
tuple `xmax=90` 且删除事务 committed。
删除早于 snapshot。
旧版本不可见。
案例 6：
tuple XID 是 subxid，`suboverflowed=false`，不在 `subxip`。
可以按不 running 继续查最终结果。
案例 7：
tuple XID 是 subxid，`suboverflowed=true`。
必须追 parent。
不允许因为不在 `subxip` 就判定不 running。
案例 8：
tuple 是当前事务插入，cmin 大于等于 `curcid`。
它来自当前 snapshot 之后的 command。
当前 command 不应看见。
## 21. 与后续课程的衔接
第 8 节会问：
READ COMMITTED 什么时候重新构造这些字段。
第 9 节会问：
REPEATABLE READ 什么时候复用这些字段。
第 10 节会问：
`curcid` 如何把同一事务内部 command 纳入判断。
第 11 节会问：
`active_count/regd_count` 如何决定字段对象活多久。
第 12 节会问：
`xmin` 如何变成 VACUUM cleanup horizon。
第 14 节会把这些字段带入 `HeapTupleSatisfiesMVCC()` 的完整分支。
所以本节最重要的产出不是背字段。
而是形成一个判断流程：
先判断 XID 区间。
再判断 running set。
再处理子事务 fallback。
再问最终事务结果。
最后处理当前事务 command id。
## 22. 深入展开：`xmin/xmax` 的双重误读
`xmin` 最容易被误读成“最老可见事务”。
这个说法不够精确。
对 snapshot 来说，`xmin` 是 running set 的低水位。
它告诉 visibility code：
小于这个 XID 的事务，不需要再问“它是否还在本 snapshot 中运行”。
它没有告诉你这个事务 committed。
如果 CLOG 说 aborted，tuple 仍不可见。
如果 hint bit 说 invalid，tuple 也不可见。
所以 `xmin` 只排除 in-progress，不证明 success。
`xmax` 也容易被误读成“当前 nextXid”。
它更像 snapshot 的 future fence。
大于等于 `xmax` 的 XID，对本 snapshot 来说都是未来。
未来事务的插入不可见。
未来事务的删除不生效。
这个 fence 是 READ COMMITTED 单语句稳定性的关键。
如果没有 `xmax`，扫描中途新分配 XID 的事务可能被错误纳入。
`xmin/xmax` 合起来把无限 XID 空间切成三段：
低段不 running。
中段需要查数组或 fallback。
高段视为 future/running。
`xip/subxip` 只负责中段。
这能显著减少查找成本。
这也解释了为什么 snapshot 不是单纯保存 running XID list。
没有区间边界，数组缺失就无法解释。
## 23. 深入展开：`xip` 与 `subxip` 的不同承诺
`xip` 是 top-level running XID 列表。
普通事务的 tuple header 大多数时候记录 top-level XID。
查 `xip` 就能回答它在 snapshot 中是否 running。
`subxip` 是优化。
它允许直接判断某个 subxid 是否 running。
但系统不保证 shared memory 中永远能保存完整 subxid 集合。
因此 `suboverflowed` 是 `subxip` 的 contract 边界。
如果没有 overflow：
`subxip` 不包含某 subxid，通常可以认为它不在 snapshot 的 running subxid 集合中。
如果 overflow：
`subxip` 不完整。
不在数组里不能说明 anything decisive。
必须追 parent。
这体现了 PostgreSQL 的常见设计：
fast path 保存有限集合。
overflow 后回到更慢但正确的持久映射。
把这个模式记住，后面看 MultiXact、predicate lock、relcache invalidation 都会遇到类似结构。
`takenDuringRecovery` 让这个判断更复杂。
recovery 中没有普通写事务 backend 发布自己的 `PGPROC.xid`。
standby 需要从 WAL replay 中维护 KnownAssignedXids。
因此 snapshot 的 running set 来源变了。
字段名称一样，生成机制不同。
诊断 hot standby 可见性时必须把这一层纳入。
## 24. 深入展开：snapshot 字段与 CLOG 的先后关系
普通 MVCC 判断经常遵循这个思路：
先判断事务是否在 snapshot 中 running。
再判断事务最终是否 committed。
这个顺序不是随意的。
`heapam_visibility.c` 文件开头注释解释了 race。
如果只查 CLOG，可能看到某个事务已经 committed。
但另一个并发 snapshot 在更早时刻仍会把它视作 running。
为了保持一致，visibility code 需要遵守 snapshot 的 running 判断。
这就是为什么 `TransactionIdIsInProgress()` / `XidInMVCCSnapshot()` 和 `TransactionIdDidCommit()` 不能任意交换。
在 snapshot 语境下：
“现在已经提交”不等于“对这个 snapshot 可见”。
可见性要问：
它在 snapshot 创建时是否 running？
如果 running，当前 snapshot 不看见它的插入。
如果不 running，再问它最终是否 committed。
这个顺序也解释了 hint bit 的保守设置。
hint bit 可以缓存 commit/abort 事实。
但它不能改变 snapshot 的时间边界。
一个 committed hint bit 只能说明事务成功。
不能让已经在 snapshot `xip` 中的事务突然可见。
## 25. 深入展开：`curcid` 为什么也在 `SnapshotData`
`SnapshotData` 看起来是事务间读视图。
但它还带 `curcid`。
原因是 visibility routine 的输入需要同时处理两类问题：
其他事务是否 running。
当前事务内哪个 command 的效果可见。
如果把 `curcid` 放在别处，heap visibility 每次处理 current transaction tuple 时仍要取本地事务状态。
把它放进 snapshot，可以让 visibility routine 用统一输入判断。
`SnapshotSetCommandId()` 负责同步。
`CommandCounterIncrement()` 递增 command id 后，会更新当前 snapshot。
这就是第 10 节要展开的内容。
本节先记住：
`curcid` 不参与 `xip` 判断。
它只在 tuple 的 `xmin/xmax` 属于当前事务时生效。
它和 `xmin/xmax` 字段同名相近，但语义完全不同。
一个是事务 ID 区间。
一个是命令 ID 边界。
## 26. 小型练习：手写 `XidInMVCCSnapshot()` 判定表
拿一张纸写四列：
输入 XID。
范围判断。
数组判断。
返回语义。
对下列输入逐个填写：
`xid=90, xmin=100, xmax=120`。
`xid=100, xip` 中存在。
`xid=100, xip` 中不存在。
`xid=119, suboverflowed=false, subxip` 中不存在。
`xid=119, suboverflowed=true`。
`xid=120`。
填表时不要写 committed / aborted。
只能写 running for this snapshot 或 not running for this snapshot。
然后再加一列：
下一步是否需要 CLOG。
这个练习能强迫你把 snapshot 判断和事务结果判断分开。
## 27. 讨论题
1. 为什么 snapshot 需要同时有 `xmin` 和 `xmax`，只保存 running XID 数组不够吗？
2. `xid < xmin` 时为什么不能直接说 committed？
3. `xid >= xmax` 为什么要当作 running/future？
4. `suboverflowed` 为什么是正确性 fallback，而不是错误状态？
5. `curcid` 为什么放在 snapshot 中，而不只放在全局 transaction state？
6. hot standby snapshot 为什么需要 `takenDuringRecovery` 这种标志？
7. 长期 registered snapshot 为什么会影响 VACUUM？
8. 诊断“为什么旧版本不回收”时，snapshot 字段和 CLOG 状态分别回答什么问题？
## 23. 本节小结
`SnapshotData` 是 MVCC 读视图的压缩表示。
`xmin/xmax` 给出事务 ID 区间边界。
`xip/subxip` 给出区间内仍运行的事务集合。
`suboverflowed` 保护子事务集合不完整时的正确性。
`takenDuringRecovery` 区分 recovery snapshot。
`curcid` 处理同事务内 command 可见性。
它不直接说明事务是否 committed。
它只说明在这个读视图中哪些事务仍应被视为进行中或未来。
最终可见性由 snapshot、CLOG/hint bit、tuple header 和 command id 共同决定。
