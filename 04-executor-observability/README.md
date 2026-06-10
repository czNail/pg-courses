# Executor 与可观测性

本目录约 11 个执行器与观测主题组，目标是理解 Plan 如何变成运行中的 PlanState，tuple 如何在节点间流动，以及 EXPLAIN / pg_stat / wait event / hook / profiler 指标从哪里来。

课程安排：

1. Executor 生命周期：ExecutorStart / Run / End。
2. PlanState / TupleTableSlot / ExprContext。
3. Executor MemoryContext。
4. 常见执行节点源码 walkthrough。
5. EXPLAIN ANALYZE 与 Instrumentation。
6. pg_stat 统计体系。
7. Wait Event 与等待诊断。
8. auto_explain / executor hook。
9. 并行查询观测。
10. 慢 SQL 从 EXPLAIN 到源码。
11. 轻量 Executor profiler extension。

第 1 项 `Executor 生命周期：ExecutorStart / Run / End` 建议先拆成 5 个独立主题，后续生成课程时按“一个主问题一课”处理：

1. [ExecutorStart 如何把 Plan 变成 EState](01-executorstart-estate-bootstrap.md)：`ExecutorStart()` 为什么要先创建 `EState`、初始化 snapshot / range table / ResultRelInfo / trigger 状态，再递归构造 `PlanState` tree？
2. [ExecutorRun 的方向、计数与 Portal 边界](02-executorrun-direction-count-portal.md)：`ExecutorRun()` 如何处理 forward / backward scan、`count` 限制、`DestReceiver` 输出和 Portal 生命周期，为什么执行器不能只暴露一个“跑完整棵树”的接口？
3. [ExecutePlan 与 demand-driven tuple 拉取模型](03-executeplan-demand-driven-pull.md)：为什么大多数节点通过 `ExecProcNode()` 一次返回一个 tuple，`ExecutePlan()` 如何把调用栈、tuple 计数、command id 和 junk filter 组合成顶层执行循环？
4. [ExecutorFinish 与副作用收尾](04-executorfinish-side-effect-drain.md)：`ExecutorFinish()` 为什么要在 `ExecutorEnd()` 前单独存在，ModifyTable、AFTER trigger、foreign table、queued side effect 和 CTE cleanup 哪些必须先 drain？
5. [ExecutorEnd 与 ERROR-safe teardown](05-executorend-cleanup-ordering.md)：`ExecutorEnd()` 如何按 node shutdown、tuple table 清理、ResultRelInfo / trigger / partition 状态释放、MemoryContext 删除的顺序收尾，哪些资源依赖 ResourceOwner 兜底？

第 2 项 `PlanState / TupleTableSlot / ExprContext` 建议先拆成 6 个独立主题，后续生成课程时按“一个主问题一课”处理：

1. [Plan 到 PlanState 的状态化边界](06-planstate-runtime-boundary.md)：为什么 planner 输出的 `Plan` 不能直接执行，`ExecInitNode()` 如何为每类 plan node 构造带函数指针、表达式状态、子节点和运行时私有字段的 `PlanState`？
2. [ExecProcNode 函数指针与节点调度](07-execprocnode-dispatch.md)：`PlanState->ExecProcNode` 如何把统一的 tuple 拉取协议分派到 SeqScan、Join、Agg、Sort、Gather 等节点，为什么还需要 `ExecProcNodeFirst()` 做首调用初始化？
3. [TupleTableSlot 的虚拟 tuple 与物理 tuple 边界](08-tupletableslot-virtual-physical.md)：`TupleTableSlot` 为什么同时支持 virtual、heap、minimal、buffer-backed slot，slot ownership、pin、materialize 和 clear 规则如何影响节点间 tuple 传递？
4. [TupleDesc、投影与 junk attribute](09-slot-tupledesc-projection-junk.md)：执行器如何用 `TupleDesc`、`ProjectionInfo` 和 junk filter 把节点内部 tuple 形状转换成上层需要的输出形状，UPDATE / DELETE 的 row identity 为什么依赖 junk column？
5. [ExprContext 与 per-tuple 表达式状态](10-exprcontext-per-tuple-state.md)：`ExprContext` 如何承载 scan tuple、inner / outer tuple、params、aggregate context 和 per-tuple memory，为什么表达式求值必须有明确的 reset 边界？
6. [Param、SubPlan 与表达式执行状态](11-param-subplan-exprstate.md)：`PARAM_EXEC`、`ParamListInfo`、`ExprState` 和 `SubPlanState` 如何把外层变量、InitPlan 结果和相关子查询传递给表达式执行器？

第 3 项 `Executor MemoryContext` 建议先拆成 4 个独立主题，后续生成课程时按“一个主问题一课”处理：

1. [QueryDesc、EState 与 executor context 生命周期](12-executor-memory-lifecycle.md)：一次查询执行中哪些内存挂在 `QueryDesc` / `EState` / executor context 下，为什么它们应该随 `ExecutorEnd()` 整体释放？
2. [Per-tuple context reset 与表达式泄漏边界](13-per-tuple-context-reset.md)：`ResetExprContext()` 和 per-tuple context 如何让表达式求值、函数调用、投影和过滤器产生的短生命周期内存批量回收，哪些值必须 materialize 才能跨节点保存？
3. [节点私有 context 与可重扫状态](14-node-private-context-rescan.md)：Sort、Hash、Agg、WindowAgg、Material 等节点为什么需要自己的内存上下文或 tuplestore，`ExecReScan()` 时哪些状态应该重置、保留或重建？
4. [Executor 内存诊断与峰值解释](15-executor-memory-diagnostics.md)：如何结合 `EXPLAIN (ANALYZE, MEMORY)`、`pg_backend_memory_contexts`、gdb 和节点源码区分正常峰值、跨 tuple retention、Hash / Sort spill 前的膨胀和真正 leak？

第 4 项 `常见执行节点源码 walkthrough` 建议先拆成 9 个独立主题，后续生成课程时按“一个主问题一课”处理：

1. [SeqScan / IndexScan 的访问方法边界](16-scan-node-access-method-boundary.md)：扫描节点如何把 MVCC 可见性、table AM / index AM、qual 过滤和 slot 输出连在一起，为什么执行器不直接理解 heap page 细节？
2. [BitmapHeapScan 的两阶段访问模型](17-bitmapheapscan-two-phase-access.md)：为什么 BitmapIndexScan 先构造 TID bitmap，再由 BitmapHeapScan 回表取 tuple，lossy page、recheck qual 和 prefetch 如何影响执行路径？
3. [Nested Loop 的参数化内侧扫描](18-nestloop-parameterized-inner.md)：Nested Loop 如何用 outer tuple 驱动 inner rescan，join qual、other qual、`PARAM_EXEC` 和 index parameterization 如何形成常见的点查计划？
4. [Hash Join 的 build / probe 与 batch spill](19-hashjoin-build-probe-spill.md)：Hash Join 如何选择 build side、构造 hash table、处理 skew / batch / spill，为什么内存限制会把一个 join 变成多批磁盘路径？
5. [Merge Join 的有序流推进](20-mergejoin-mark-restore.md)：Merge Join 为什么需要排序输入、mark / restore 和 join key 比较状态，outer join 语义如何影响匹配组推进？
6. [Agg 的 group、hash 与 transition state](21-agg-transition-state.md)：Agg 节点如何在 sorted aggregation、hash aggregation 和 mixed strategy 间组织 transition value、aggregate context、group boundary 和 spill？
7. [Sort / Incremental Sort / Material 的阻塞节点特征](22-blocking-nodes-sort-material.md)：为什么 Sort 和 Material 会打断 tuple-by-tuple 流水线，tuplesort、tuplestore、work_mem 和 rescan 需求如何决定它们的运行时行为？
8. [Limit / Result / ProjectSet 的轻节点路径](23-lightweight-nodes-limit-result-projectset.md)：这些看似简单的节点如何处理 tuple 截断、one-time qual、targetlist 投影、set-returning function 和执行器状态切换？
9. [ModifyTable 与 DML 执行边界](24-modifytable-dml-boundary.md)：INSERT / UPDATE / DELETE / MERGE 如何在 `ModifyTable` 中串起 junk attr、partition routing、constraint、trigger、RETURNING 和 FDW / table AM 回调？

第 5 项 `EXPLAIN ANALYZE 与 Instrumentation` 建议先拆成 5 个独立主题，后续生成课程时按“一个主问题一课”处理：

1. [Instrumentation 对象如何挂到 PlanState](25-instrumentation-planstate-attachment.md)：`EXPLAIN ANALYZE` 开启后，`Instrumentation` 如何按计划节点分配、挂载和启动，为什么普通执行不能无条件承担这些计时成本？
2. [Actual rows、loops 与 timing 的采样边界](26-explain-actual-rows-loops-timing.md)：节点的 actual rows、loops、startup time、total time 分别在 `InstrStartNode()` / `InstrStopNode()` / `InstrEndLoop()` 哪些边界上累计，如何解读 loops 放大的时间？
3. [Buffers、WAL、JIT 与 memory 指标来源](27-explain-buffers-wal-jit-memory.md)：`EXPLAIN (ANALYZE, BUFFERS, WAL, JIT, MEMORY)` 中的指标分别来自哪些计数器和上下文，为什么它们有些是节点级、有些是查询级？
4. [ExplainNode 如何把执行状态格式化输出](28-explainnode-output-format.md)：`ExplainNode()` 如何遍历 `PlanState` tree，把 plan-time 信息、runtime instrumentation、worker 信息和 text / json / yaml / xml 格式拼成最终结果？
5. [EXPLAIN ANALYZE 的观测扰动](29-explain-analyze-observer-effect.md)：计时、buffer 统计、JIT 统计、并行 worker 汇总和输出格式化会给执行增加哪些额外成本，什么时候应关闭 timing 或换用 pg_stat / auto_explain？

第 6 项 `pg_stat 统计体系` 建议先拆成 4 个独立主题，后续生成课程时按“一个主问题一课”处理：

1. [pg_stat 架构与 backend 本地计数](30-pgstat-local-counter-flush.md)：执行器路径上的 tuple、block、function、transaction 等计数如何先累计在 backend-local 状态，再按事务边界或 flush 时机进入共享统计体系？
2. [pg_stat_statements 的 query id 与执行指标](31-pgstat-statements-queryid-metrics.md)：`pg_stat_statements` 如何用 query id 聚合执行次数、时间、行数、block I/O、WAL 和 JIT 指标，为什么它适合发现热点但不能替代单次执行剖析？
3. [表、索引、函数统计与执行节点对应关系](32-pgstat-relation-index-function.md)：`pg_stat_all_tables`、`pg_stat_all_indexes`、`pg_stat_user_functions` 等视图里的计数如何对应 scan、DML、index fetch 和函数调用路径？
4. [统计刷新延迟与诊断误区](33-pgstat-lag-reset-scope.md)：为什么 pg_stat 视图可能存在刷新延迟、事务可见性和 reset scope 差异，排查执行器问题时如何避免把累计指标误读成单条 SQL 事实？

第 7 项 `Wait Event 与等待诊断` 建议先拆成 4 个独立主题，后续生成课程时按“一个主问题一课”处理：

1. [wait_event_info 的设置与清理边界](34-wait-event-set-clear-boundary.md)：`pgstat_report_wait_start()` / `pgstat_report_wait_end()` 如何把 backend 当前等待点暴露给 `pg_stat_activity`，为什么每条等待路径都必须保证清理？
2. [Executor 常见等待类型定位](35-executor-wait-event-taxonomy.md)：执行 SQL 时常见的 LWLock、Lock、BufferPin、IO、Client、IPC 等 wait event 分别对应哪些执行器、存储和通信路径？
3. [等待、CPU 与阻塞链的区分](36-wait-cpu-blocking-chain.md)：如何把 `pg_stat_activity`、`pg_locks`、wait event、`EXPLAIN ANALYZE` 和系统 profiler 结合起来，判断慢 SQL 是锁等待、I/O、CPU 还是上游客户端背压？
4. [自定义等待点与扩展可观测性](37-custom-wait-event-extension.md)：扩展或新执行路径何时应该注册 wait event，如何命名、设置和清理，避免让用户只能看到模糊的 active 状态？

第 8 项 `auto_explain / executor hook` 建议先拆成 4 个独立主题，后续生成课程时按“一个主问题一课”处理：

1. [Executor hook 链与扩展插桩边界](38-executor-hook-chain.md)：`ExecutorStart_hook`、`ExecutorRun_hook`、`ExecutorFinish_hook`、`ExecutorEnd_hook` 如何形成扩展插桩入口，hook 链为什么必须保存并调用 previous hook？
2. [auto_explain 如何捕获慢查询计划](39-auto-explain-capture-plan.md)：`auto_explain` 如何在 executor hook 中测量执行、判断阈值、调用 explain 输出计划，并处理 nested statement、log level、buffers、timing 等选项？
3. [Hook 中的异常安全与开销控制](40-hook-error-safety-overhead.md)：扩展 hook 如何避免在 ERROR 路径破坏 executor 状态，采样率、阈值、GUC、内存上下文和递归调用保护如何控制线上开销？
4. [Executor hook 与 planner / ProcessUtility hook 的边界](41-hook-boundaries-planner-utility.md)：什么问题应该放在 executor hook 观测，什么问题必须在 planner hook、ProcessUtility hook 或 event trigger 处理，如何避免在错误层次修补行为？

第 9 项 `并行查询观测` 建议先拆成 5 个独立主题，后续生成课程时按“一个主问题一课”处理：

1. [并行 PlanState 与 worker instrumentation 布局](42-parallel-planstate-worker-instrumentation.md)：`ExecInitParallelPlan()` 如何为并行查询准备计划、参数和 instrumentation 共享布局，哪些内容属于执行器观测边界而不是 DSM / worker bootstrap 基础设施？
2. [Gather / GatherMerge 的 tuple routing 与 leader participation](43-gather-gathermerge-observability.md)：`Gather` / `GatherMerge` 如何从 worker tuple queue 和 leader 本地执行路径收 tuple，`parallel_leader_participation` 如何影响 rows、loops 和时间解读？
3. [并行节点的 per-worker EXPLAIN 输出](44-parallel-explain-worker-output.md)：EXPLAIN 如何展示每个 worker 的 actual rows、loops、sort / hash / buffer / timing 信息，为什么 worker 间倾斜比总耗时更能解释并行效率？
4. [Parallel Hash / Parallel Append 的共享执行状态](45-parallel-node-shared-runtime-state.md)：并行感知节点如何在执行器层共享进度、批次、分配和完成状态，观测时如何区分算法共享状态与基础设施共享内存机制？
5. [并行查询 cleanup 与指标汇总顺序](46-parallel-cleanup-instrumentation-merge.md)：`ExecParallelFinish()` / `ExecParallelCleanup()` 为什么要先处理 tuple queue 和 worker finish，再汇总 instrumentation、buffer、WAL、JIT 和内存指标？

第 10 项 `慢 SQL 从 EXPLAIN 到源码` 建议先拆成 5 个独立主题，后续生成课程时按“一个主问题一课”处理：

1. [从计划节点名定位 Exec 函数](47-explain-node-to-source-entry.md)：如何把 EXPLAIN 中的 `Seq Scan`、`Hash Join`、`Aggregate`、`Gather` 等节点名映射到 `ExecInit*`、`Exec*`、`ExecEnd*` 源码入口？
2. [从 actual / estimate 偏差定位问题层次](48-estimate-actual-gap-triage.md)：rows 估计错误、loops 异常、过滤率异常和 join order 异常分别更可能属于统计信息、优化器选择、执行器行为还是数据分布问题？
3. [从 Buffers / I/O timing 定位存储访问路径](49-buffers-io-source-tracing.md)：如何沿着 scan node、buffer access、read timing 和 heap / index fetch 计数判断慢 SQL 是缓存未命中、随机 I/O、回表过多还是可见性检查成本？
4. [从等待事件定位并发瓶颈源码](50-wait-event-source-tracing.md)：当慢 SQL 卡在 lock、buffer pin、LWLock、IPC 或 client wait 时，如何从 wait event 名称反查源码等待点和相关共享状态？
5. [从 profile 栈回到执行器热点](51-profiler-stack-to-executor-hotspot.md)：如何用 perf、gprof、eBPF 或 extension profiler 的栈样本，把 CPU 热点映射回表达式求值、tuple deform、hash compare、sort compare、JIT 或节点调度？

第 11 项 `轻量 Executor profiler extension` 建议先拆成 5 个独立主题，后续生成课程时按“一个主问题一课”处理：

1. [Profiler extension 的最小 hook 设计](52-profiler-extension-hook-design.md)：如何用 executor hook 搭建一个只观测不改语义的轻量 profiler，记录 query id、计划节点、开始结束时间和错误路径？
2. [节点级计时与嵌套调用栈](53-profiler-node-timing-stack.md)：如何在 `ExecProcNode` 边界或 wrapper 中记录节点级耗时、调用次数和父子关系，避免把子节点时间重复解释成独占时间？
3. [共享内存、环形缓冲与采样输出](54-profiler-shmem-ring-buffer.md)：profiler 何时需要共享内存 ring buffer，如何控制事件大小、丢弃策略、锁竞争和 backend 退出后的数据可读性？
4. [GUC、采样率与线上保护](55-profiler-guc-sampling-guardrails.md)：如何设计 enable 开关、采样率、最小时长、用户 / database 过滤和最大记录数，让 profiler 能在线上临时开启而不制造新的瓶颈？
5. [Profiler 结果视图与源码闭环](56-profiler-result-view-source-loop.md)：如何把 profiler 采样结果暴露成 SRF / view，并把节点 id、plan id、source symbol 和 EXPLAIN 输出关联起来，形成从慢 SQL 到源码热点的闭环？
