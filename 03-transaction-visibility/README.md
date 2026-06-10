# 事务与可见性

本目录约 6 个主题组，目标是解释一个 tuple version 为什么当前语句可见、可锁定、可更新或可回收。

课程安排：

1. Transaction / CLOG / Subtrans 与事务结果判定。
2. Snapshot 语义、生命周期与可见性边界。
3. Heap tuple visibility 与 tuple header 判定。
4. Row lock / MultiXact / update conflict。
5. VACUUM / pruning / freeze / visibility map 与回收边界。
6. Serializable / predicate lock 与可串行化冲突检测。

第 1 项 `Transaction / CLOG / Subtrans 与事务结果判定` 建议先拆成 6 个独立主题，后续生成课程时按“一个主问题一课”处理：

1. [XID 分配与事务身份边界](01-xid-assignment-transaction-identity.md)：为什么 PostgreSQL 要区分 virtual transaction、未分配 XID 的只读事务和真正写入 tuple header 的 XID，XID 什么时候才成为 MVCC 可见性判断的一部分？
2. [pg_xact / CLOG 状态机](02-clog-transaction-status-machine.md)：`pg_xact` 只记录 committed、aborted、sub-committed 等结果时，可见性代码如何把“事务仍在运行”和“事务已经结束”区分开？
3. [提交顺序、WAL 与 CLOG 持久化边界](03-commit-wal-clog-ordering.md)：为什么事务提交必须先满足 WAL 持久化规则，再让其它 backend 通过 CLOG 看到 committed 状态？
4. [abort 路径与未完成事务判定](04-abort-status-visibility-boundary.md)：事务 ERROR、显式 ROLLBACK 和 backend 崩溃时，tuple header 中已经写出的 XID 如何在可见性判断中被解释为 aborted 或仍需等待确认？
5. [Subtrans 父事务链](05-subtrans-parent-chain.md)：为什么子事务提交不能简单写成普通 committed，`pg_subtrans` 如何把 subxid 追溯到 top-level XID 来决定 tuple version 的最终命运？
6. [SubXID overflow 与可见性成本](06-subxid-overflow-visibility-cost.md)：当一个 backend 的子事务超过 snapshot 可内联记录的数量后，为什么可见性判断必须回查 `pg_subtrans`，以及这个退化如何影响长事务和高并发负载？

第 2 项 `Snapshot 语义、生命周期与可见性边界` 建议先拆成 6 个独立主题，后续生成课程时按“一个主问题一课”处理：

1. [SnapshotData 字段语义](07-snapshotdata-xmin-xmax-xip-subxip.md)：`xmin`、`xmax`、`xip`、`subxip`、`suboverflowed` 和 `takenDuringRecovery` 分别回答什么可见性问题，为什么它们足以描述一个 MVCC 读视图？
2. [READ COMMITTED 语句级 snapshot](08-read-committed-statement-snapshot.md)：为什么 READ COMMITTED 每条语句重新获取 snapshot，同一事务内相邻语句如何因此看到不同的已提交版本？
3. [REPEATABLE READ / SERIALIZABLE 事务级 snapshot](09-repeatable-read-transaction-snapshot.md)：为什么更高隔离级别要复用事务第一个 snapshot，后续语句如何在同一读视图下保持稳定结果？
4. [CommandId 与同事务内可见性](10-commandid-self-visibility.md)：同一个事务内部的 INSERT、UPDATE、DELETE 为什么还需要 `cmin`、`cmax`、combo CID 和 command counter 来判断当前命令能否看到自己刚写的 tuple？
5. [Active / registered snapshot 生命周期](11-active-registered-snapshot-lifetime.md)：executor、portal、cursor 和 function 调用为什么要 pin 住 snapshot，过早释放或长期持有 snapshot 分别会破坏什么可见性和回收边界？
6. [xmin horizon 与 cleanup snapshot 语义](12-snapshot-xmin-cleanup-horizon.md)：一个读 snapshot 的 `xmin` 如何限制 VACUUM 回收旧版本，这里关注 MVCC 语义上的“仍可能被看见”，而不是 `ProcArray` 本身的共享内存实现。

第 3 项 `Heap tuple visibility 与 tuple header 判定` 建议先拆成 7 个独立主题，后续生成课程时按“一个主问题一课”处理：

1. [tuple header 中的 MVCC 信息](13-heap-tuple-header-mvcc-fields.md)：`xmin`、`xmax`、`t_ctid`、infomask、hint bit 和 command id 如何共同描述一个 tuple version 的出生、死亡和版本链关系？
2. [HeapTupleSatisfiesMVCC 判定框架](14-heap-tuple-satisfies-mvcc.md)：一个普通 SELECT 如何按 xmin 可见、xmax 不可见或已删除等分支，最终决定 tuple version 是否属于当前 snapshot？
3. [hint bit 与可见性缓存](15-hint-bit-visibility-cache.md)：为什么 backend 可以把 CLOG 查询结果写回 tuple header，hint bit 如何减少重复事务状态查询，又为什么它不能改变事务语义？
4. [同事务写入、删除与版本替换](16-self-updated-deleted-visibility.md)：当前事务自己 INSERT、UPDATE、DELETE 的 tuple version 为什么不能只按 XID 判断，还必须结合 command id 和 `HeapTupleSatisfiesSelf` 等特殊规则？
5. [SnapshotAny / Dirty / Toast 等特殊可见性](17-special-snapshot-visibility-rules.md)：系统维护、索引检查、toast 访问和锁冲突探测为什么需要不同于普通 MVCC SELECT 的 visibility routine？
6. [UPDATE 版本链与 t_ctid 追踪](18-update-chain-tctid-following.md)：UPDATE 为什么生成新 tuple version 而不是原地覆盖，`t_ctid` 如何把旧版本、当前版本和并发更新冲突连接起来？
7. [HOT chain 可见性](19-hot-chain-visibility.md)：HOT update 如何把同页版本链藏在 heap page 内部，index scan 为什么需要沿 HOT chain 找到对当前 snapshot 可见的版本？

第 4 项 `Row lock / MultiXact / update conflict` 建议先拆成 6 个独立主题，后续生成课程时按“一个主问题一课”处理：

1. [xmax 的删除、更新与锁定多重含义](20-xmax-delete-update-lock-overload.md)：为什么 `xmax` 既可能表示删除者、更新者，也可能只是行锁持有者，可见性代码如何通过 infomask 区分这些语义？
2. [row lock 模式与 tuple lock 冲突矩阵](21-row-lock-modes-conflict-matrix.md)：`FOR KEY SHARE`、`FOR SHARE`、`FOR NO KEY UPDATE`、`FOR UPDATE` 分别保护什么不变量，哪些组合可以共存，哪些必须等待？
3. [MultiXact 成员与共享行锁](22-multixact-members-row-locks.md)：多个事务同时锁定同一行时，为什么单个 `xmax` 不够用，MultiXact 如何记录成员 XID、锁模式和最终冲突关系？
4. [UPDATE / DELETE 并发冲突判定](23-update-delete-concurrency-conflict.md)：两个事务同时 UPDATE 或 DELETE 同一行时，PostgreSQL 如何在等待、重查可见性、EvalPlanQual 和报错之间选择？
5. [foreign key 行锁与 key update 冲突](24-foreign-key-key-update-conflict.md)：外键检查为什么依赖 key-share 类行锁，主键或唯一键被 UPDATE 时如何避免引用完整性和 MVCC 可见性之间出现空洞？
6. [MultiXact wraparound 与冻结边界](25-multixact-wraparound-freeze-boundary.md)：MultiXact ID 也会回卷时，VACUUM 为什么必须冻结或清理旧 MultiXact 引用，哪些仍可能被并发事务解释的行锁信息不能提前丢弃？

第 5 项 `VACUUM / pruning / freeze / visibility map 与回收边界` 建议先拆成 7 个独立主题，后续生成课程时按“一个主问题一课”处理：

1. [dead tuple、recently dead 与可回收边界](26-dead-recently-dead-cleanup-boundary.md)：一个被 DELETE 或 UPDATE 淘汰的 tuple version 什么时候只是对当前 snapshot 不可见，什么时候才对所有可能读者都不可见并允许回收？
2. [heap page pruning 与 HOT chain 修剪](27-heap-page-pruning-hot-chain.md)：普通访问路径为什么也会触发 page pruning，PostgreSQL 如何在不破坏仍可见版本链的前提下裁剪 dead line pointer 和 HOT chain？
3. [VACUUM heap scan 与 index cleanup](28-vacuum-heap-index-cleanup.md)：VACUUM 如何从 heap tuple visibility 出发决定哪些 TID 需要从索引中删除，为什么 heap 回收和 index cleanup 必须围绕同一个 cleanup horizon 对齐？
4. [freeze 的语义与 anti-wraparound](29-freeze-anti-wraparound-semantics.md)：XID 会回卷时，为什么老 tuple 的 `xmin` / `xmax` 必须被改写或标记为 frozen，frozen tuple 在后续 visibility 判断中代表什么？
5. [visibility map 的 all-visible 位](30-visibility-map-all-visible.md)：一个 heap page 什么时候可以标记 all-visible，index-only scan 为什么可以相信 visibility map，又为什么任何可能改变可见性的写入都必须清除这个位？
6. [visibility map 的 all-frozen 位](31-visibility-map-all-frozen.md)：all-frozen 与 all-visible 的语义差别是什么，VACUUM 如何利用 all-frozen page 跳过重复 freeze 工作？
7. [长事务、复制槽与回收推迟](32-long-xact-replication-slot-cleanup-delay.md)：长期 snapshot、prepared transaction 和 replication slot 如何延后 dead tuple 回收，本课只讨论它们对 MVCC 回收边界的语义影响。

第 6 项 `Serializable / predicate lock 与可串行化冲突检测` 建议先拆成 6 个独立主题，后续生成课程时按“一个主问题一课”处理：

1. [SSI 与 snapshot isolation 异常](33-ssi-snapshot-isolation-anomalies.md)：为什么事务级 snapshot 仍可能产生 write skew，Serializable Snapshot Isolation 试图补上的到底是哪类可串行化语义缺口？
2. [rw-conflict 与 dangerous structure](34-rw-conflict-dangerous-structure.md)：SSI 为什么重点追踪读写依赖而不是简单加锁，`rw-conflict in/out` 和 dangerous structure 如何决定是否必须 abort 某个事务？
3. [predicate lock 的范围表达](35-predicate-lock-targets.md)：SELECT 没有读到的行也可能影响可串行化结果时，predicate lock 如何用 relation、page、tuple 和 index range 表达“我依赖这个谓词范围没有变化”？
4. [SIREAD lock 生命周期与内存压力](36-siread-lock-lifetime.md)：SIREAD lock 为什么不会阻塞写入，却必须在事务结束后继续保留一段时间，什么时候可以安全释放或合并这些 predicate lock？
5. [索引访问方法与 predicate lock 粒度](37-index-predicate-lock-granularity.md)：btree、heap scan 和缺少合适索引的查询为什么会产生不同粒度的 predicate lock，错误的粒度会如何影响误 abort 和可串行化正确性？
6. [只读事务、safe snapshot 与 deferrable](38-safe-snapshot-deferrable-readonly.md)：Serializable 只读事务为什么有时可以等待一个 safe snapshot 来避免后续冲突检测，`DEFERRABLE` 读事务在语义上换取了什么？
