# PostgreSQL CommandId 与同事务内可见性
## 课程定位
前置知识：
- 已理解事务级和语句级 snapshot 的差别。
- 已理解 tuple header 中既有 XID，也有 command id。
- 已理解同一事务自己的写入不能简单走普通 running XID 判断。
本节唯一主问题：
同一个事务内部的 INSERT、UPDATE、DELETE 为什么还需要 `cmin`、`cmax`、combo CID 和 command counter，才能判断当前命令能否看到自己刚写或刚删的 tuple？
核心矛盾：
MVCC snapshot 主要描述事务之间的可见性；
但同一事务内部多个 SQL command 也需要有先后顺序，且 tuple header 空间有限，不能同时无成本保存所有 command 边界。
一句话模型：
XID 决定“哪个事务写的”，CommandId 决定“这个事务中的哪个命令写的或删的”。
学完后应能判断：
- 为什么当前事务的 tuple 不能只看 `xmin == currentXid`。
- `curcid` 为什么在 snapshot 中。
- `CommandCounterIncrement()` 为什么只有 command id 被使用后才递增。
- 同一事务内 UPDATE 自己插入的行为什么需要 combo CID。
- combo CID 为什么是 backend-private 状态。
源码基线：
```text
/home/nail/postgres-lab
branch: feature/pg-pv-storage-design
commit: 2d5ed10b0bb1d1c16df7ff408eb65df4609006ae
```
## 1. 本节在总主线中的位置
前几节讲 snapshot 的事务边界。
事务边界不能解释所有可见性。
例如：
```sql
BEGIN;
INSERT INTO t VALUES (1);
SELECT * FROM t;
DELETE FROM t WHERE id = 1;
SELECT * FROM t;
COMMIT;
```
四条语句在同一个 XID 下执行。
如果只看 XID，INSERT 和 DELETE 都来自当前事务。
但第二条 SELECT 应该看到 INSERT。
第四条 SELECT 不应该看到已 DELETE 的行。
这需要 command id。
CommandId 是事务内部的时间轴。
它不是全局时间。
它不参与其他事务之间的 snapshot 排序。
本节只讲 self-visibility。
普通 `xmin/xmax` 与其他事务的 committed/running 判断仍由前几节机制处理。
## 2. 核心矛盾与一句话运行模型
同一事务内可见性有两个要求。
第一个要求：
前一个 SQL command 的写入，后一个 command 应该看见。
第二个要求：
当前 command 自己正在写入或删除的 tuple，不能在同一 command 的某些路径中被错误重复处理。
因此 snapshot 里需要 `curcid`。
`curcid` 表示：
当前 snapshot 可以看见本事务中 command id 小于它的效果。
tuple header 中的 `t_cid` 保存插入或删除 command id。
问题是 heap tuple header 只有一个 `t_cid` 字段。
同一事务内如果一个 tuple 同时需要记录 cmin 和 cmax，就需要 combo CID。
combo CID 把 `(cmin, cmax)` 映射到一个本地编号。
这个映射只在创建它的 backend 内有意义。
## 3. 核心文件分工与阅读顺序
| 顺序 | 文件 | 关注点 |
| --- | --- | --- |
| 1 | `src/backend/access/transam/xact.c` | `GetCurrentCommandId()`、`CommandCounterIncrement()` |
| 2 | `src/include/access/htup_details.h` | `t_cid`、`HEAP_COMBOCID`、getter/setter |
| 3 | `src/backend/utils/time/combocid.c` | combo CID 映射表 |
| 4 | `src/include/utils/combocid.h` | combo CID 对外函数 |
| 5 | `src/backend/utils/time/snapmgr.c` | `SnapshotSetCommandId()` |
| 6 | `src/backend/access/heap/heapam_visibility.c` | `snapshot->curcid` 如何参与可见性 |
不要先背 tuple header bit。
先看 `xact.c` 里 command id 何时递增。
再看 heap visibility 如何比较 `cmin/cmax` 和 `snapshot->curcid`。
最后看 combo CID 为什么是本地状态。
## 4. 关键数据结构与状态
`currentCommandId` 是当前事务本地 command id。
事务开始时初始化为 `FirstCommandId`。
`currentCommandIdUsed` 表示当前 command id 是否被需要。
如果一个 command 没有写入可见性相关状态，`CommandCounterIncrement()` 可以不递增。
这避免 command id 无意义增长。
`SnapshotData.curcid` 是当前 snapshot 的 command 边界。
`SnapshotSetCommandId()` 会更新当前 snapshot 的 `curcid`。
`HeapTupleHeaderData.t_choice.t_heap.t_cid` 存 raw command id。
它可能是 cmin。
也可能是 cmax。
也可能是 combo CID。
`HEAP_COMBOCID` infomask 标记说明 `t_cid` 需要通过 combo CID 映射解释。
`comboCids` 是 backend-private array。
`comboHash` 用于复用相同 `(cmin, cmax)` 对。
这个状态不能跨 backend 直接解释。
## 5. 主流程源码 walkthrough
INSERT 路径需要当前 command id。
调用 `GetCurrentCommandId(true)`。
`used=true` 会设置 `currentCommandIdUsed`。
tuple header 写入 cmin。
语句结束时 `CommandCounterIncrement()` 被调用。
如果 `currentCommandIdUsed` 为 true，`currentCommandId++`。
然后 `SnapshotSetCommandId(currentCommandId)` 更新 snapshot。
下一条语句看到前一条语句的 tuple。
DELETE 或 UPDATE 时需要 cmax。
如果 tuple 由当前事务先前 command 插入，又被当前事务当前 command 删除，就需要同时保留 cmin 和 cmax。
`HeapTupleHeaderAdjustCmax()` 判断是否需要 combo CID。
需要时调用 `GetComboCommandId(cmin, cmax)`。
tuple header 保存 combo CID，并设置 `HEAP_COMBOCID`。
visibility 判断读取 tuple。
如果 `xmin` 是当前事务，比较 cmin 与 `snapshot->curcid`。
如果 cmin >= curcid，说明插入发生在当前 snapshot 之后，不可见。
如果 `xmax` 是当前事务，比较 cmax 与 curcid。
如果 cmax >= curcid，删除发生在当前 snapshot 之后，旧版本仍可见。
如果 cmax < curcid，删除已经对当前 command 可见，tuple 不可见。
## 6. 生命周期 / ownership / cleanup
CommandId 生命周期绑定当前 transaction state。
事务开始时初始化。
每个 command 结束时可能递增。
事务结束时丢弃。
combo CID 生命周期也是 backend-local、transaction-local。
`combocid.c` 中的 hash 和 array 在当前事务中增长。
事务结束时 `AtEOXact_ComboCid()` 清理。
并行 worker 是特殊边界。
如果 worker 需要解释 combo CID，leader 必须序列化并恢复 combo CID state。
`SerializeComboCIDState()` 和 `RestoreComboCIDState()` 服务这个场景。
ERROR/abort 时，transaction cleanup 丢弃 command id 和 combo CID 状态。
tuple header 中已经写出的 combo CID 只有在当前事务上下文中有意义。
其他事务通常不会用 combo mapping 解释它，而是按事务结果判断整个 tuple 是否可见。
## 7. 正确性机制层次
第一层是 command boundary。
前一个 command 的写入对后一个 command 可见。
第二层是 current command 防重复。
同一 command 不能因为扫描路径再次看到自己刚写入或刚删除的 tuple 而重复处理。
第三层是 combo CID。
同一事务内同一 tuple 同时需要 cmin/cmax 时，不能丢掉其中一个。
第四层是 backend-private 解释。
combo CID 不写成全局含义，避免扩张 tuple header。
第五层是 snapshot curcid 同步。
如果 command id 递增但 snapshot 不更新，visibility 会错。
因此 `CommandCounterIncrement()` 调用 `SnapshotSetCommandId()`。
## 8. 错误路径 / 异常路径 / fallback
CommandId 会溢出。
`CommandCounterIncrement()` 发现到 `InvalidCommandId` 时会报错。
这保护 tuple header 中有限字段的语义。
parallel worker 不能随意把 `currentCommandIdUsed` 从 false 改 true。
源码在 `GetCurrentCommandId(true)` 中限制 parallel mode。
原因是 worker 无法把新的 command id 使用状态安全反馈给 leader。
combo CID lookup 如果在不合适的 backend 中解释，会触发 assertion 或得到无意义结果。
logical decoding / parallel restore combo CID state 时必须按协议恢复。
这不是普通 SQL 路径。
## 9. 成本、资源与跨模块传播
CommandId hot path 成本通常很低。
`currentCommandIdUsed` 让 no-op command counter increment 便宜。
combo CID 成本来自 hash lookup 和 array growth。
大量同事务内 update/delete 自己写入的 tuple，会增加 combo CID 状态。
长事务中很多 command 也会增加 command id 压力。
并行查询需要序列化 combo CID state。
这把本地状态传播到 worker。
成本传播路径：
```text
many commands / self-updates
  -> command id use
  -> combo CID creation
  -> backend-local memory
  -> parallel serialization if needed
```
## 10. 观测与诊断入口
SQL 不直接显示 command id。
可以用 `pageinspect` 查看 tuple header 的 raw command id 和 infomask。
`heap_page_items(get_raw_page(...))` 能看到 `t_field3` 等字段。
解释时要结合当前事务和源码。
gdb 入口：
`GetCurrentCommandId`。
`CommandCounterIncrement`。
`SnapshotSetCommandId`。
`HeapTupleHeaderAdjustCmax`。
`GetComboCommandId`。
`HeapTupleSatisfiesMVCC`。
关注：
`currentCommandId`、`currentCommandIdUsed`、`snapshot->curcid`、`HEAP_COMBOCID`。
## 11. 常见误区
- 以为一个事务内所有操作天然互相可见，不需要 command id。
- 把 command id 当成全局时间。
- 以为 `t_cid` 永远是 cmin。
- 以为 combo CID 可以被任意 backend 解释。
- 忽略 `SnapshotSetCommandId()`。
- 把 self-visibility 和 READ COMMITTED 的新 snapshot 混淆。
- 以为 `CommandCounterIncrement()` 每次都递增。
## 12. 课堂实验
实验 1：观察同事务内可见性。
```sql
BEGIN;
CREATE TABLE cid_demo(id int);
INSERT INTO cid_demo VALUES (1);
SELECT * FROM cid_demo;
DELETE FROM cid_demo WHERE id = 1;
SELECT * FROM cid_demo;
ROLLBACK;
```
解释两次 SELECT 为什么不同。
实验 2：用 pageinspect 看 tuple header。
在事务中插入、更新、删除同一行，用 `heap_page_items()` 观察 infomask 和 raw command id。
实验 3：断点跟踪。
断 `GetCurrentCommandId`、`CommandCounterIncrement`、`HeapTupleHeaderAdjustCmax`、`GetComboCommandId`。
执行同事务内 INSERT 后 UPDATE/DELETE。
## 13. 源码细读补充：command id 如何进入 tuple header
读 `xact.c:GetCurrentCommandId()`。
传入 `used=true` 时，函数会把 `currentCommandIdUsed` 标记为 true。
这说明 caller 接下来可能把当前 command id 写入 tuple header。
读 `xact.c:CommandCounterIncrement()`。
它先看 `currentCommandIdUsed`。
如果当前 command 没有使用 command id，就不递增。
这避免只读语句或不需要 CID 的操作消耗 command id。
如果需要递增，它检查 overflow。
然后更新 `currentCommandId`。
然后把 `currentCommandIdUsed` 置回 false。
最后调用 `SnapshotSetCommandId()`。
这个顺序很关键。
如果只递增本地变量，不更新 snapshot，heap visibility 仍会按旧 `curcid` 判断。
读 `htup_details.h`。
`HeapTupleHeaderSetCmin()` 写插入 command id。
`HeapTupleHeaderSetCmax()` 写删除 command id 或 combo id。
`HEAP_COMBOCID` 告诉 getter 需要去 combo CID 映射表解码。
读 `combocid.c`。
`HeapTupleHeaderAdjustCmax()` 判断是否需要 combo CID。
如果 tuple 是当前事务插入，并且又被当前事务删除/更新，就需要同时保存 cmin/cmax。
`GetComboCommandId()` 用 hash 查 `(cmin,cmax)`。
已有则复用。
没有则扩展 array。
这说明 combo CID 是压缩编码，不是新的全局语义。
## 14. Runtime case：同一事务先插入再删除
执行：
```text
BEGIN;
INSERT row;
SELECT row;
DELETE row;
SELECT row;
```
第一次 SELECT：
tuple `xmin` 是当前事务。
`cmin < snapshot->curcid`。
插入已经对当前 command 可见。
`xmax` 无效。
因此行可见。
DELETE 后第二次 SELECT：
tuple `xmax` 是当前事务。
删除 command id 小于当前 snapshot `curcid`。
因此删除对当前 command 可见。
行不可见。
如果没有 command id，这两个 SELECT 无法区分。
它们都只会看到同一个 XID。
所以 command id 是 SQL command 顺序，不是 MVCC 附属字段。
## 15. Runtime case：同一事务 UPDATE 自己插入的行
UPDATE 不是简单地原地覆盖。
它通常生成新 tuple version，并让旧 tuple 的 `xmax` 指向 updater。
如果旧 tuple 也是当前事务刚插入的，就同时需要 cmin 和 cmax。
heap tuple header 只有一个 command id slot。
combo CID 解决这个冲突。
旧 tuple 里保存 combo id。
本 backend 可通过 combo table 解出 cmin/cmax。
新 tuple 保存新的 cmin。
其它事务不需要解释 combo CID 来看到中间状态。
如果顶层事务 abort，两个版本都不可见。
如果 commit，后续事务看到最终版本链状态。
这就是 combo CID 可以 backend-private 的原因。
## 16. 诊断矩阵：同事务可见性异常怎么查
现象：
同一事务后续 SELECT 没看到刚 INSERT 的行。
先看是否真的跨 command。
如果在同一 SQL command 内，可能还没经历 command counter increment。
现象：
UPDATE 自己刚 INSERT 的行后，pageinspect 看到 `HEAP_COMBOCID`。
这是正常现象。
需要用当前 backend combo table 才能解码。
现象：
并行 worker 报 command id 相关限制。
看是否 worker 试图首次使用 command id。
parallel 模式下 command id 使用状态不能随意从 worker 传播回 leader。
现象：
超长事务报 command id overflow。
说明事务内使用 command id 的 command 太多。
这是语义上限，不是简单增大内存可解。
现象：
logical decoding 或 parallel worker 无法解释 combo CID。
看 combo CID state 是否已序列化并恢复。
## 17. 与 snapshot 隔离级别的关系
READ COMMITTED 每条语句取新 snapshot。
REPEATABLE READ 复用事务级 snapshot。
但两者都需要 `curcid`。
原因是 command id 是同一事务内部的顺序。
隔离级别解决事务之间的读视图。
command id 解决同一事务内部的读视图。
这两个维度相互独立。
例如 REPEATABLE READ 下，后续语句仍应看见本事务前面语句插入的行。
如果事务级 snapshot 完全冻结而不更新 `curcid`，自写入就会不可见。
因此 `SnapshotSetCommandId()` 是隔离级别之下的本地推进。
## 18. 逐函数阅读任务
任务 1：读 `GetCurrentCommandId()`。
观察 `used` 参数。
解释为什么读取 command id 也可能改变 `currentCommandIdUsed`。
任务 2：读 `CommandCounterIncrement()`。
标出 no-op fast path。
标出 overflow 检查。
标出 `SnapshotSetCommandId()`。
任务 3：读 `HeapTupleHeaderSetCmin()` 和 `HeapTupleHeaderSetCmax()`。
说明 raw command id 如何进入 tuple header。
任务 4：读 `HeapTupleHeaderAdjustCmax()`。
记录它何时创建 combo CID。
任务 5：读 `combocid.c:GetComboCommandId()`。
画出 hash lookup 和 array append。
解释为什么可以复用已有 combo CID。
任务 6：读 `HeapTupleSatisfiesMVCC()` 当前事务分支。
标出 `snapshot->curcid` 与 cmin/cmax 的比较。
任务 7：读 combo CID 序列化函数。
解释并行 worker 为什么需要恢复 combo state。
## 19. 案例推演：三条命令如何改变可见性
命令 1：
`INSERT id=1`。
tuple header 写入当前 XID 和 cmin=0。
`currentCommandIdUsed=true`。
命令结束：
`CommandCounterIncrement()` 把 command id 推到 1。
snapshot curcid 更新为 1。
命令 2：
`SELECT id=1`。
tuple 是当前事务插入。
cmin=0 < curcid=1。
可见。
命令 3：
`DELETE id=1`。
tuple xmax 是当前事务。
cmax=1。
命令结束后 curcid=2。
后续 SELECT 看到 cmax=1 < curcid=2。
删除已对当前 command 可见。
行不可见。
这个推演中 XID 从未变化。
可见性变化完全来自 command id。
## 20. 案例推演：combo CID 何时出现
命令 1 插入一行。
命令 2 更新这行。
旧 tuple 需要保留插入 command id。
旧 tuple 也需要记录删除/update command id。
header 只有一个 `t_cid`。
系统创建 combo CID。
combo CID 映射到 `(cmin, cmax)`。
当前 backend 能解码。
其他 backend 不需要在事务未提交时把它解释成正常可见行。
提交后，外部事务根据 XID 和版本链看最终结果。
combo CID 是局部压缩。
不是持久语义扩展。
## 21. 诊断案例：pageinspect 看到奇怪 command id
pageinspect 看到 raw command id。
如果 infomask 有 `HEAP_COMBOCID`，raw 值不是直接 cmin 或 cmax。
没有当前 backend 的 combo table，很难把它解释成人类想要的 pair。
这不是 pageinspect 错。
这是 combo CID 的设计边界。
诊断同事务可见性最好配合 gdb。
断 `GetComboCommandId()`。
打印 `comboCids`。
再看 tuple header。
不要离线拿 raw command id 做完整语义推断。
## 22. 反例与边界：CommandId 不能替代什么
CommandId 不能替代 XID。
它只在一个事务内部有顺序意义。
另一个事务看到你的 tuple 时，不会用你的 command id 判断提交结果。
它会看 XID、snapshot、CLOG 和 hint bit。
CommandId 不能替代 lock。
同一事务内部顺序清楚，不代表并发事务不会冲突。
UPDATE/DELETE 的并发等待仍要通过 tuple lock、transactionid lock 和 MultiXact 等机制处理。
CommandId 不能替代 snapshot。
当前事务自己的写入需要 command id。
其他事务的写入仍由 snapshot 读视图决定。
CommandId 不能跨 backend 离线解释 combo CID。
combo CID 的 raw number 只在生成它的 backend combo table 中有映射。
这就是为什么 pageinspect 看到 raw cid 后不能直接还原所有语义。
CommandId 不能无限增长。
事务中真正使用 command id 的命令过多会触发 overflow 防线。
这是保护 tuple header 和可见性语义。
边界记忆：
XID 是事务身份。
CommandId 是事务内命令顺序。
Combo CID 是一个 backend-local 压缩映射。
Snapshot curcid 是当前读视图的 self-command 边界。
## 23. 生产诊断提纲
怀疑同事务可见性问题时：
第一步确认是否同一事务。
第二步确认是否跨 command。
第三步确认是否当前 command 内部重复扫描。
第四步看 `snapshot->curcid`。
第五步看 tuple header `cmin/cmax` 或 combo cid。
第六步看是否 parallel worker 或 logical decoding 需要 combo state。
第七步再看隔离级别。
隔离级别不是 self-visibility 的首要解释。
如果看到 command id overflow：
统计单个事务内写入/修改命令数量。
检查批处理是否把太多逻辑命令放进一个事务。
如果看到 combo CID 数量很大：
检查同一事务内对自己刚写 tuple 的 update/delete 模式。
如果看到 pageinspect raw cid 异常：
先看 infomask 是否 `HEAP_COMBOCID`。
## 24. 深入展开：为什么 command id 不是 statement number
CommandId 和 SQL statement number 很像，但不能简单等同。
一个 SQL command 内部可能有多个执行阶段。
有些路径会主动做 `CommandCounterIncrement()`，让后续内部操作能看到前面步骤。
例如 DDL、rewrite、触发器、内部命令可能需要推进 command counter。
所以 command id 是事务内部可见性边界，不是用户输入的第几条语句。
`currentCommandIdUsed` 进一步说明：
如果当前 command 没有把 command id 写入需要可见性判断的状态，递增没有意义。
PostgreSQL 避免无意义递增。
这不仅省成本。
也降低 command id overflow 风险。
读源码时要跟实际调用，而不是按 SQL 文本数数。
## 25. 深入展开：combo CID 与 tuple header 空间设计
旧版本 PostgreSQL 曾经有更直接的表示。
当前 heap tuple header 把 cmin/cmax 叠放在一个字段里。
这样节省每个 tuple 的空间。
代价是当前事务内同时需要 cmin/cmax 时要本地映射。
这个 trade-off 很 PostgreSQL：
让所有 tuple 永久变大，成本太高。
让少数复杂 self-update 场景走 combo CID，成本更可控。
combo CID 的 hash 和 array 是 backend-local。
这也意味着它不是 WAL redo 必须长期解释的业务语义。
它服务当前事务内部判断。
事务结束后，外部世界主要看 XID 结果和版本链。
## 26. 深入展开：parallel worker 的 command id 边界
parallel worker 共享 leader 的事务上下文。
但 worker 不能随意产生新的 command id 使用状态。
如果 worker 调用 `GetCurrentCommandId(true)` 并首次标记 used，会破坏 leader 对 command counter 的控制。
因此源码在 parallel mode 下限制这种行为。
并行执行需要 leader 把必要状态传给 worker。
combo CID state 也可以被序列化恢复。
这说明 command id 虽然是 backend-local，但并行查询让“本地”边界变复杂。
诊断 parallel visibility 问题时，要看：
worker 是否只读。
是否需要 combo CID。
leader 是否序列化了相关状态。
是否触发了禁止在 worker 中修改 command id used 的路径。
## 27. 深入展开：logical decoding 对 combo CID 的特殊性
logical decoding 读取历史 change。
它可能需要解释当前事务内的 tuple command id 状态。
combo CID 不是普通持久字段。
因此 decoding 要在合适上下文中恢复 combo CID state。
`combocid.c` 提供序列化/恢复函数。
这不是普通查询会接触的路径。
但它说明一个原则：
当 backend-private 状态要跨执行主体传播时，必须显式序列化。
不能假设 raw tuple header 足够。
这也给调试 pageinspect 一个提醒：
离开生成 backend，只看 raw combo cid，语义是不完整的。
## 28. 小型练习：判断 self tuple
设当前事务 XID=200，snapshot curcid=3。
tuple A：`xmin=200, cmin=2, xmax invalid`。
A 可见。
tuple B：`xmin=200, cmin=3, xmax invalid`。
B 对当前 snapshot 不可见。
tuple C：`xmin=200, cmin=1, xmax=200, cmax=2`。
C 已被当前事务较早 command 删除，不可见。
tuple D：`xmin=200, cmin=1, xmax=200, cmax=4`。
D 的删除在当前 snapshot 之后，旧版本仍可见。
如果 C/D 使用 combo CID，先解码出 cmin/cmax 再比较。
练习时刻意不要查 CLOG。
当前事务分支先由 command id 决定。
## 29. 源码路径分解：INSERT、UPDATE、DELETE 如何使用 CID
INSERT：
heap insert 需要记录 tuple 的创建 command。
它调用 `GetCurrentCommandId(true)`。
tuple header 记录 cmin。
该 command 结束时 command counter 前进。
后续 command 能看见该 tuple。
UPDATE：
旧 tuple 记录 xmax 和 cmax。
新 tuple 记录 xmin 和 cmin。
如果旧 tuple 也是当前事务创建，旧 tuple 需要同时保留 cmin/cmax。
这触发 combo CID。
DELETE：
tuple 记录 xmax 和 cmax。
当前事务后续 command 根据 cmax 与 curcid 判断删除是否已经生效。
SELECT：
普通 SELECT 不写 CID。
但它消费 `snapshot->curcid`。
如果读取当前事务 tuple，会比较 cmin/cmax。
CommandCounterIncrement：
只有 `currentCommandIdUsed` 为 true 时才递增。
这表示系统只在 command id 参与可见性时推进边界。
## 30. 源码路径分解：combo CID 创建与恢复
创建路径：
当前事务先创建 tuple。
后续 command 删除或更新这个 tuple。
`HeapTupleHeaderAdjustCmax()` 发现需要同时表达 cmin/cmax。
它调用 `GetComboCommandId()`。
`GetComboCommandId()` 先查 hash。
如果 `(cmin,cmax)` 已存在，复用 combocid。
否则追加到 `comboCids` array。
tuple header 写 combo id。
读取路径：
`HeapTupleHeaderGetCmin()` 或 `GetCmax()` 看到 `HEAP_COMBOCID`。
通过 combo table 找真实 cmin/cmax。
并行/解码路径：
如果另一个执行主体需要解释 combo cid，必须恢复 combo CID state。
这就是 `SerializeComboCIDState()` 和 `RestoreComboCIDState()` 的意义。
## 31. 反例：为什么不能把 cmin/cmax 做成全局可解释
如果每个 tuple 都保存完整 cmin/cmax，header 会变大。
绝大多数 tuple 不需要同时表达两者。
如果 combo cid 做成全局表，事务结束后的生命周期、WAL、crash recovery、共享内存管理都会复杂化。
PostgreSQL 选择：
少数需要 pair 的情况用 backend-local mapping。
这让普通 tuple 保持紧凑。
代价是调试时 raw combo cid 不自解释。
这是空间和可诊断性的 trade-off。
## 32. 讨论题
1. 为什么 snapshot 需要 `curcid`？
2. 如果 `CommandCounterIncrement()` 不更新 snapshot，会出现什么错误？
3. tuple header 为什么不直接存 cmin 和 cmax 两个字段？
4. combo CID 为什么不能有全局含义？
5. parallel worker 为什么不能随便标记 command id used？
6. CommandId overflow 为什么必须报错？
7. 同事务自更新和普通并发更新在 visibility 上有什么不同？
8. `HeapTupleSatisfiesSelf` 和 MVCC snapshot 在 self-write 上有什么差异？
## 25. 本节小结
XID 解决事务之间的可见性。
CommandId 解决同一事务内部的命令顺序。
`currentCommandId` 和 `snapshot->curcid` 共同定义当前 command 能看到哪些 self-write。
tuple header 只有一个 `t_cid`，所以同一 tuple 需要同时表达 cmin/cmax 时使用 combo CID。
combo CID 是 backend-private、transaction-local 状态。
它用空间节省换来本地解释复杂度。
