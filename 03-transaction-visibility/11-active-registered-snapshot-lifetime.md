# PostgreSQL active / registered snapshot 生命周期
## 课程定位
前置知识：
- 已理解 `SnapshotData` 字段语义。
- 已理解 READ COMMITTED 和事务级 snapshot 的创建策略。
- 已理解 snapshot 的 `xmin` 会影响 VACUUM horizon。
本节唯一主问题：
executor、portal、cursor 和函数调用为什么要 pin 住 snapshot，过早释放或长期持有 snapshot 分别会破坏什么可见性和回收边界？
核心矛盾：
snapshot 是读取 tuple 的正确性输入，必须在执行期间稳定；
但它一旦长期存在，又会通过 `MyProc->xmin` 阻止旧版本回收。
一句话模型：
active snapshot 保护当前调用栈正在用的 snapshot，registered snapshot 保护跨调用栈/跨语句仍需存在的 snapshot。
学完后应能判断：
- active snapshot stack 和 RegisteredSnapshots 的区别。
- `active_count` / `regd_count` 为什么都存在。
- ResourceOwner 在 snapshot cleanup 中负责什么。
- portal/cursor 为什么容易制造长期 snapshot。
- snapshot leak 为什么会表现为 VACUUM 无法回收。
源码基线：
```text
/home/nail/postgres-lab
branch: feature/pg-pv-storage-design
commit: 2d5ed10b0bb1d1c16df7ff408eb65df4609006ae
```
## 1. 本节在总主线中的位置
前面几节讲 snapshot 内容和隔离级别。
这节讲 snapshot 活多久。
一个 snapshot 即使字段正确，如果生命周期错了，也会破坏系统。
过早释放：
executor 还在读 tuple，snapshot 已经被覆盖或释放。
可见性判断会用错读视图。
过晚释放：
`MyProc->xmin` 继续保留旧边界。
VACUUM 不能回收已经无业务需要的旧版本。
PostgreSQL 用两个机制分开处理：
active snapshot stack。
RegisteredSnapshots heap。
它们都引用 `SnapshotData`，但含义不同。
## 2. 核心矛盾与一句话运行模型
active snapshot 是当前执行栈需要的 snapshot。
例如 executor 正在跑一个 SELECT。
函数调用内部需要临时 snapshot。
这些 snapshot 的生命周期通常是 lexical / stack-like。
registered snapshot 是更长期的引用。
例如 portal/cursor、事务级 snapshot、导出 snapshot。
这些引用不一定跟当前 C 调用栈同步。
因此需要 ResourceOwner。
运行模型：
```text
GetTransactionSnapshot()
  -> PushActiveSnapshot()
  -> executor uses visibility
  -> PopActiveSnapshot()
```
长期模型：
```text
snapshot = RegisterSnapshot(snapshot)
  -> ResourceOwner owns reference
  -> later UnregisterSnapshot(snapshot)
  -> ResourceOwner cleanup on ERROR
```
两者都影响 snapshot 是否可释放。
两者也影响 `SnapshotResetXmin()`。
## 3. 核心文件分工与阅读顺序
| 顺序 | 文件 | 关注点 |
| --- | --- | --- |
| 1 | `src/backend/utils/time/snapmgr.c` | active stack、registered heap、cleanup |
| 2 | `src/include/utils/snapshot.h` | `active_count`、`regd_count`、`ph_node` |
| 3 | `src/include/utils/snapmgr.h` | public API |
| 4 | `src/backend/utils/resowner/resowner.c` | ResourceOwner cleanup |
| 5 | `src/backend/tcop/pquery.c` | portal / query execution snapshot |
| 6 | `src/backend/executor/execMain.c` | executor active snapshot assumptions |
| 7 | `src/backend/storage/ipc/procarray.c` | xmin horizon 被 snapshot 生命周期影响 |
先读 `snapmgr.c` 文件开头注释。
它明确区分 active 和 registered。
再读 `PushActiveSnapshot()` / `PopActiveSnapshot()`。
最后读 `RegisterSnapshot()` / `UnregisterSnapshot()`。
不要把 refcount 当作内存引用计数。
它同时影响 global xmin。
## 4. 关键数据结构与状态
`ActiveSnapshot` 是链表栈。
每个元素表示一个 active_count。
`PushActiveSnapshot()` 会复制或引用 snapshot，并增加 `active_count`。
`PopActiveSnapshot()` 减少 `active_count`。
`RegisteredSnapshots` 是 pairing heap。
它按 `xmin` 排序。
最小 `xmin` 的 snapshot 决定当前 backend 需要保护的最老版本。
`regd_count` 是注册引用数。
`active_count` 是 active stack 引用数。
两者都为 0 时，复制 snapshot 才能释放。
`copied` 标志说明 snapshot 是否是动态分配副本。
静态 snapshot 不按同样方式释放。
`SnapshotResetXmin()` 会检查 RegisteredSnapshots。
如果没有 snapshot 需要保护，则清空 `MyProc->xmin`。
如果还有，则把最小 `xmin` 安装到 `MyProc->xmin`。
## 5. 主流程源码 walkthrough
普通 executor 路径：
上层获得 transaction snapshot。
调用 `PushActiveSnapshot(snapshot)`。
executor 开始执行。
heap scan 调用 visibility routine。
visibility routine 使用 active snapshot。
executor 结束后调用 `PopActiveSnapshot()`。
如果 snapshot 不再 registered，且 active_count 为 0，可以释放或降低 xmin。
portal/cursor 路径：
portal 可能需要跨多次 fetch 保持 snapshot。
它不能只依赖 active stack。
因此要 register snapshot。
每次执行时再 push active。
fetch 结束 pop active。
portal 关闭时 unregister。
事务级 snapshot 路径：
第一次 snapshot 被 copy。
作为 `FirstXactSnapshot` 注册。
多个语句重复使用。
事务结束释放 registered reference。
这个流程让 snapshot 同时满足：
当前执行不能提前释放。
长期持有有明确 owner。
ERROR 时有兜底 cleanup。
## 6. 生命周期 / ownership / cleanup
创建者：
`GetSnapshotData()` 或 snapshot manager。
持有者：
active stack、registered heap、portal、ResourceOwner、transaction state。
释放者：
`PopActiveSnapshot()`、`UnregisterSnapshot()`、`AtEOXact_Snapshot()`、ResourceOwner cleanup。
ERROR 兜底：
ResourceOwner 释放 registered snapshot。
active stack 通常由 executor / portal error cleanup 保证平衡。
长期失效：
snapshot 不会“自动变新”。
它只会被释放。
如果 caller 持有旧 snapshot，系统必须尊重它。
这就是长期 cursor 能拖住 horizon 的根本原因。
## 7. 正确性机制层次
第一层是 pointer stability。
executor 不能使用会被覆盖的静态 snapshot。
第二层是 visibility stability。
同一执行范围内 snapshot 字段不能改变。
第三层是 xmin protection。
只要 snapshot 还可能被使用，它的 `xmin` 必须保护旧版本。
第四层是 ResourceOwner cleanup。
ERROR 不能泄漏 registered snapshot。
第五层是 ordered xmin heap。
RegisteredSnapshots 按 xmin 排序，让 `SnapshotResetXmin()` 快速找到最小 horizon。
这不是优化细节。
它影响 VACUUM 是否可以回收。
## 8. 错误路径 / 异常路径 / fallback
忘记 pop active snapshot 会导致 end-of-transaction 检查失败或 snapshot 生命周期异常。
忘记 unregister snapshot 会拖住 `MyProc->xmin`。
ResourceOwner cleanup 可以兜底 registered snapshot，但不能替代正常生命周期设计。
如果 snapshot 被导出，源事务需要继续保护它。
如果 portal 被长期保留，snapshot 生命周期跟 portal 而不是单条语句绑定。
如果 function 内部临时 push snapshot 后 ERROR，cleanup 必须保证 active stack 状态正确。
这些错误常表现为：
旧版本不能回收。
`backend_xmin` 长期很老。
或者开发环境 assertion 失败。
## 9. 成本、资源与跨模块传播
active snapshot 操作本身成本较低。
registered snapshot 的成本来自 lifecycle 和 horizon。
RegisteredSnapshots heap 让最小 xmin 查找便宜。
但每个长期 registered snapshot 都可能阻止 VACUUM。
portal/cursor 可能让 READ COMMITTED 看起来像长期 snapshot。
事务级隔离级别会自然注册长期 snapshot。
导出 snapshot 会把成本传播到另一个会话。
成本传播路径：
```text
snapshot registered
  -> MyProc->xmin retained
  -> global horizon older
  -> VACUUM cannot remove tuple
  -> heap/index bloat grows
```
这条路径比 snapshot 本身的内存成本更重要。
## 10. 观测与诊断入口
SQL 入口：
`pg_stat_activity.backend_xmin`。
`pg_stat_activity.xact_start`。
`pg_stat_activity.state`。
`pg_cursors` 可观察 cursor。
VACUUM 入口：
`VACUUM VERBOSE`。
`pg_stat_progress_vacuum`。
`pg_stat_all_tables.n_dead_tup`。
gdb 入口：
`PushActiveSnapshot`。
`PopActiveSnapshot`。
`RegisterSnapshot`。
`UnregisterSnapshot`。
`SnapshotResetXmin`。
`AtEOXact_Snapshot`。
重点看 `active_count`、`regd_count`、`RegisteredSnapshots` 最小 xmin、`MyProc->xmin`。
## 11. 常见误区
- active snapshot 不是 registered snapshot。
- snapshot refcount 不是单纯内存 refcount。
- READ COMMITTED 不等于不会长期持有 snapshot。
- cursor/portal 可能延长 snapshot 生命周期。
- ResourceOwner cleanup 不是设计正常路径的借口。
- `backend_xmin` 老不一定是当前语句慢，可能是 snapshot 持有者老。
- 导出 snapshot 不是无成本调试工具。
## 12. 课堂实验
实验 1：观察 cursor 持有 snapshot。
在事务中声明 cursor，fetch 一部分后保持事务打开。
另一个会话更新/删除并 VACUUM。
观察 `backend_xmin` 和 dead tuple。
实验 2：断点看 active stack。
断 `PushActiveSnapshot` 和 `PopActiveSnapshot`。
执行普通 SELECT、函数调用、cursor fetch。
比较 push/pop 层级。
实验 3：观察 registered snapshot。
断 `RegisterSnapshot`、`UnregisterSnapshot`、`SnapshotResetXmin`。
执行 REPEATABLE READ 事务和 cursor。
观察 `regd_count` 与 `MyProc->xmin`。
## 13. 源码细读补充：两个 refcount 和一个 xmin heap
读 `snapmgr.c` 文件头注释。
它明确说 active snapshot 和 registered snapshot 不是同一类引用。
active snapshot 是栈。
RegisteredSnapshots 是 pairing heap。
为什么不用一个链表？
因为 cleanup 时需要快速找到最小 xmin。
最小 xmin 决定当前 backend 必须继续向 ProcArray 发布的 `MyProc->xmin`。
读 `PushActiveSnapshotWithLevel()`。
它记录 snapshot 和 subtransaction nesting level。
这让 subtransaction abort 时可以按层清理。
读 `PopActiveSnapshot()`。
它减少 active_count。
如果 active_count 和 regd_count 都为 0，就释放 copied snapshot。
然后调用 `SnapshotResetXmin()`。
读 `RegisterSnapshotOnOwner()`。
它复制 snapshot 或增加已有副本引用。
它把 snapshot 登记到 ResourceOwner。
这保证 ERROR/abort 有兜底释放。
读 `UnregisterSnapshotNoOwner()`。
它从 RegisteredSnapshots 中移除 snapshot。
如果引用归零，释放并重置 xmin。
读 `SnapshotResetXmin()`。
如果没有 registered snapshot，清空 `MyProc->xmin`。
如果还有，取 heap 顶部最小 xmin。
这就是 snapshot lifecycle 和 VACUUM horizon 的连接点。
## 14. Runtime case：普通 SELECT 的短生命周期
普通 READ COMMITTED SELECT：
获取 snapshot。
push active。
executor 使用。
pop active。
如果没有注册引用，snapshot 结束。
`MyProc->xmin` 可以被清理。
VACUUM 不会因为这个语句结束后的旧 snapshot 被长期阻塞。
这解释了为什么短语句通常不是 bloat 根因。
但如果 SELECT 本身跑很久，在执行期间 active snapshot 仍然有效。
VACUUM 必须尊重它。
所以“短生命周期”是语句结束后的结论，不是语句执行中的结论。
## 15. Runtime case：cursor 如何改变边界
cursor 需要分批 fetch。
如果每次 fetch 都重新取 snapshot，cursor 结果可能不稳定。
因此 portal 需要持有 snapshot。
这个 snapshot 可能跨多次 client round trip 存活。
这会让 `backend_xmin` 保持较老。
用户看到的是：
会话没有在跑重 SQL。
但 VACUUM 仍被它拖住。
源码解释是：
portal registered snapshot 仍然存在。
active snapshot 不一定一直在栈顶。
registered snapshot 足以保护 xmin。
诊断时要看 cursor/portal，而不是只看当前 query。
## 16. Runtime case：事务级 snapshot 和 ResourceOwner
REPEATABLE READ 第一次 snapshot 被 copy/register。
ResourceOwner 持有它。
后续语句使用同一 snapshot。
事务 abort 时，即使用户代码没有显式 unregister，ResourceOwner cleanup 也会释放。
这不是鼓励忽略 unregister。
正常路径仍应平衡引用。
ResourceOwner 是错误路径兜底。
如果扩展代码手写 snapshot 生命周期，必须理解这个区别。
否则可能出现：
释放过早。
引用泄漏。
或 horizon 长期不前。
## 17. 诊断矩阵：`backend_xmin` 很老
第一步，看 `pg_stat_activity`。
确认 pid、state、xact_start、query_start、backend_xmin。
第二步，看是否有 cursor。
`pg_cursors` 能给出 SQL 层线索。
第三步，看隔离级别。
REPEATABLE READ / SERIALIZABLE 更可能有事务级 registered snapshot。
第四步，看是否 exported snapshot。
导出 snapshot 可能让源事务保持 xmin。
第五步，看扩展或函数。
某些扩展可能注册 snapshot 后生命周期很长。
第六步，必要时 gdb。
断 `RegisterSnapshot`、`UnregisterSnapshot`、`SnapshotResetXmin`。
打印 RegisteredSnapshots heap 顶部 xmin。
第七步，不要误判。
idle in transaction 可能持锁、持 XID、持 snapshot。
要具体确认是哪种资源拖住系统。
## 18. 错误边界：过早释放与过晚释放
过早释放的风险是正确性。
executor 可能用到被覆盖的 snapshot。
tuple visibility 结果不可预测。
过晚释放的风险是资源和回收。
old snapshot 继续保护旧版本。
系统表现为 bloat、VACUUM 无效、index-only scan 机会下降。
这两类问题方向相反。
一个是用不到了还留着。
一个是还要用却没了。
snapshot manager 的设计就是同时防住两边。
active stack 防过早释放。
registered heap 和 ResourceOwner 防错误路径泄漏。
`SnapshotResetXmin()` 防过晚影响 horizon。
## 19. 逐函数阅读任务
任务 1：读 `PushActiveSnapshotWithLevel()`。
标出 snapshot copy 规则。
标出 `active_count++`。
标出 nesting level。
任务 2：读 `PopActiveSnapshot()`。
标出 `active_count--`。
标出释放条件。
标出 `SnapshotResetXmin()`。
任务 3：读 `RegisterSnapshotOnOwner()`。
标出 ResourceOwner 如何记录 snapshot。
解释为什么注册引用不是 active stack。
任务 4：读 `UnregisterSnapshotFromOwner()`。
标出 owner 解除引用。
标出 heap 移除。
任务 5：读 `xmin_cmp()`。
说明 RegisteredSnapshots 为什么按 xmin 排序。
任务 6：读 `AtEOXact_Snapshot()`。
标出事务结束 cleanup。
任务 7：找 portal 使用 snapshot 的位置。
说明 cursor/fetch 为什么可能跨调用持有。
## 20. 案例推演：snapshot leak 如何变成 bloat
扩展函数注册一个 snapshot。
函数返回前没有 unregister。
ResourceOwner 如果没接管或生命周期过长，snapshot 仍存在。
`regd_count` 不归零。
RegisteredSnapshots heap 仍有元素。
`SnapshotResetXmin()` 不能清空 `MyProc->xmin`。
ProcArray horizon 仍旧。
VACUUM 看到 old xmin。
dead tuple 不能回收。
用户看到表膨胀。
根因不是 snapshot 占用内存。
根因是 horizon 被保守保留。
## 21. 案例推演：active snapshot 嵌套
外层 executor push snapshot A。
函数内部执行 SQL，需要 snapshot B。
系统 push snapshot B。
内部 SQL 结束 pop B。
外层 executor 继续使用 A。
最后 pop A。
这种 stack 模型允许嵌套调用。
如果内部错误跳过 pop，cleanup 必须修复。
如果错误地 pop 外层 snapshot，外层 executor 后续 visibility 就失去正确输入。
这解释了为什么 active stack 是严格结构。
## 22. 诊断案例：`backend_xmin` 老但 query 为空
会话可能 idle。
但 registered snapshot 仍存在。
cursor 可能没关闭。
事务级 snapshot 可能还活着。
导出 snapshot source 可能仍保护 xmin。
因此 query 文本为空不代表没有 snapshot owner。
诊断要看：
`xact_start`。
`backend_xmin`。
`pg_cursors`。
是否有 exported snapshot。
必要时 gdb 查看 RegisteredSnapshots。
## 23. 反例与边界：snapshot 生命周期不是内存生命周期
反例一：
snapshot 对象很小。
但它的 xmin 可以阻止大量 tuple 回收。
所以内存大小不是风险尺度。
反例二：
释放 MemoryContext 不等于释放全局 horizon。
如果 snapshot registered reference 仍在，`MyProc->xmin` 仍可能保留。
必须按 snapshot manager 协议 unregister。
反例三：
active snapshot pop 不等于 registered snapshot 释放。
cursor 可以没有 active 执行栈，却仍持有 registered snapshot。
反例四：
ResourceOwner cleanup 不等于正常生命周期。
它是 ERROR 兜底。
正常路径仍应明确 unregister。
反例五：
READ COMMITTED 不等于短 snapshot。
长语句、cursor、function、portal 都可以让 snapshot 活得很久。
边界总结：
MemoryContext 管内存。
ResourceOwner 管错误路径引用。
active stack 管当前执行。
registered heap 管长期引用。
ProcArray xmin 管全局回收承诺。
这些层次重叠，但不能互相替代。
## 24. 生产诊断提纲
如果 `backend_xmin` 很老：
先找 pid。
看 `state` 和 `xact_start`。
查 cursor。
查是否 REPEATABLE READ / SERIALIZABLE。
查是否 exported snapshot。
查扩展或函数是否可能注册 snapshot。
如果能 gdb：
断 `RegisterSnapshot`。
断 `UnregisterSnapshot`。
断 `SnapshotResetXmin`。
打印 `RegisteredSnapshots`。
如果不能 gdb：
用应用日志和 `pg_stat_activity` 关联请求生命周期。
不要只 kill 最慢 query。
真正持有 snapshot 的可能是 idle 会话。
修复建议：
缩短事务。
关闭不用 cursor。
避免长时间持有 exported snapshot。
让应用分页不要依赖单个长 cursor。
## 25. 深入展开：active stack 为什么带 nesting level
subtransaction 会改变 cleanup 边界。
函数、触发器、SPI、异常块都可能在子事务层级中 push snapshot。
如果子事务 abort，需要清理该层级之后的 active snapshot。
因此 active snapshot entry 记录 level。
这不是为了调试好看。
它是 ERROR cleanup 的定位信息。
没有 level，系统很难在嵌套 subtransaction 中只清理该清理的 snapshot。
这和 ResourceOwner 层级类似。
不同层管理不同资源。
snapshot active stack 管 visibility input。
ResourceOwner 管 registered snapshot 和外部资源。
MemoryContext 管内存。
## 26. 深入展开：RegisteredSnapshots 为什么用 pairing heap
registered snapshot 可能有多个。
清理 `MyProc->xmin` 时，系统只需要最小 xmin。
如果用普通链表，每次 reset 都要扫描全部 registered snapshot。
pairing heap 让最小值访问更直接。
这说明 RegisteredSnapshots 不是普通引用列表。
它是 horizon 数据结构。
它的排序键就是 `xmin`。
所以 snapshot 生命周期直接参与 ProcArray horizon。
这也是为什么“只是注册一个 snapshot”不是轻量无害操作。
## 27. 深入展开：portal、holdable cursor 与 snapshot 复制
普通 cursor 在事务内使用 snapshot。
holdable cursor 需要在事务后继续可用。
这会迫使系统 materialize 结果或改变持有方式。
不同 cursor 类型对 snapshot 生命周期影响不同。
本课程不展开 portal 全实现，但要建立诊断意识：
如果应用使用 cursor 慢慢 fetch，不要只看单条 fetch 的耗时。
要看整个 portal 生命周期。
如果 cursor 导致 snapshot 长期注册，VACUUM horizon 会被拖住。
如果 holdable cursor 已经 materialized，后续可能不再需要同样的 MVCC snapshot。
具体要读 portal 代码确认。
## 28. 深入展开：扩展开发的 snapshot 规则
扩展或内核新代码如果调用 SPI 或手动访问 heap，需要遵守 snapshot 协议。
只在当前调用栈用：
push active，结束 pop。
跨调用保存：
copy/register，绑定 ResourceOwner，结束 unregister。
跨事务保存：
普通 snapshot 不适合。
要使用导出 snapshot 等明确协议。
错误模式：
保存 `GetTransactionSnapshot()` 返回指针。
在 callback 里继续用。
没有注册。
或者注册后漏 unregister。
前者是 correctness risk。
后者是 horizon leak。
代码 review 时看到 snapshot 指针跨函数保存，就应该追 owner。
## 29. 小型练习：画 snapshot 生命周期图
画三条线：
active_count。
regd_count。
MyProc->xmin。
场景一：
普通 SELECT。
push 后 active_count=1。
pop 后 active_count=0。
无 registered，xmin 清理。
场景二：
REPEATABLE READ。
第一次 snapshot regd_count=1。
每条语句 push/pop active。
事务结束 regd_count=0。
场景三：
cursor。
open 时 register。
每次 fetch push/pop。
close 时 unregister。
把这三张图画出来，比背 API 更容易记住。
## 30. 源码路径分解：snapshot 释放如何推进 `MyProc->xmin`
`PopActiveSnapshot()` 释放当前执行栈引用。
如果 snapshot 仍 registered，不能释放全局 horizon。
`UnregisterSnapshot()` 释放长期引用。
如果这是最后一个 registered snapshot，RegisteredSnapshots heap 会移除该元素。
`SnapshotResetXmin()` 被调用。
如果 heap 为空：
`MyProc->xmin = InvalidTransactionId`。
如果 heap 非空：
取最小 xmin。
`MyProc->xmin` 设置为该值。
ProcArray 之后计算 horizon 时会看到这个 xmin。
VACUUM / pruning 会尊重它。
因此 snapshot 释放不是“free 一个 C struct”。
它可能让全局 cleanup horizon 立刻前进。
这也是为什么释放 cursor 或提交长事务后，下一轮 VACUUM 可能突然能清很多 dead tuple。
## 31. 反例：为什么 active snapshot 不适合跨语句保存
active snapshot stack 是栈式结构。
它适合当前 executor 调用。
跨语句保存需要在语句结束后仍存在。
如果硬把 active snapshot 留在栈里，会破坏 push/pop 平衡。
如果只保存指针而不注册，可能指向已释放或被覆盖的 snapshot。
因此 portal/cursor 需要 registered snapshot 或 materialization。
这个设计让短执行路径便宜。
同时让长期持有有明确 owner。
扩展开发中，如果需要跨 callback 使用 snapshot，应优先问：
谁 unregister？
ERROR 时谁 cleanup？
它是否拖住 xmin？
## 32. 讨论题
1. 为什么 active snapshot 用 stack，而 registered snapshot 用 heap？
2. `active_count` 和 `regd_count` 分别保护什么？
3. 为什么 RegisteredSnapshots 要按 xmin 排序？
4. cursor 如何改变 READ COMMITTED 的 snapshot 生命周期？
5. ResourceOwner 能清理什么，不能证明什么？
6. `backend_xmin` 老时如何定位 snapshot owner？
7. 为什么 snapshot memory 很小，却能造成巨大 bloat？
8. 导出 snapshot 的正确性依赖哪个 owner？
## 26. 本节小结
active snapshot 保护当前执行栈。
registered snapshot 保护跨执行栈或跨语句的长期引用。
两者通过 `active_count` 和 `regd_count` 决定 snapshot 是否可释放。
RegisteredSnapshots 还决定 `MyProc->xmin`。
snapshot 生命周期错误会直接影响可见性正确性和 VACUUM 回收。
因此 snapshot manager 不是普通内存管理器，而是 MVCC horizon 管理的一部分。
