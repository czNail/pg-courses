# 内核基础设施

本目录约 8 个基础主题组，目标是建立 PostgreSQL 多进程共享状态、资源生命周期和同步模型。

课程安排：

1. Shmem 初始化与 shared state 边界。
2. MemoryContext 与 backend-local 内存生命周期。
3. ResourceOwner 与 ERROR-safe cleanup。
4. PGPROC / ProcArray 与 backend 状态。
5. SpinLock / LWLock / Latch / Condition Variable / Barrier。
6. DSM / shm_toc / shm_mq / DSA。
7. ParallelContext 与并行执行基础设施。
8. CatCache / RelCache / Shared Invalidation。

第 1 项 `Shmem 初始化与 shared state 边界` 建议先拆成 4 个独立主题，后续生成课程时按“一个主问题一课”处理：

1. [启动期 sizing 与一次性 segment 创建](01-shmem-sizing-segment.md)：为什么 shared state 必须先声明大小，再由 postmaster 创建固定 shared memory segment？
2. [Shmem allocator 与 `ShmemIndex`](02-shmem-allocator-index.md)：一个名字如何绑定到一块跨进程共享状态，为什么传统 shmem 只能分配、不能释放？
3. [request / init / attach 生命周期](03-shmem-request-init-attach.md)：postmaster、fork backend 和 `EXEC_BACKEND` 如何在不同进程中建立同一组 shared state 指针？
4. [扩展与边界](04-shmem-extension-boundaries.md)：`shared_preload_libraries`、`shmem_request_hook`、`shmem_startup_hook` 和 late allocation 的边界在哪里，什么时候该转向 DSM？

第 2 项 `MemoryContext 与 backend-local 内存生命周期` 建议先拆成 5 个独立主题，后续生成课程时按“一个主问题一课”处理：

1. [MemoryContext tree 与 backend-local ownership 边界](05-memorycontext-tree-ownership.md)：为什么 PostgreSQL 把 backend-local 内存挂到 context tree 上，而不是依赖调用者逐个 `pfree()`？
2. [短生命周期上下文与 reset 边界](06-memorycontext-short-lived-reset.md)：一条 SQL 执行过程中，per-query、per-tuple、per-expression 等短生命周期内存为什么主要靠 reset 批量回收，哪些指针不能跨过这个边界？
3. [事务、Portal 与长生命周期状态](07-memorycontext-transaction-portal.md)：为什么有些 backend-local 状态必须活过单条语句，但不能无限期挂在 `TopMemoryContext` 下？
4. [ERROR 路径与 MemoryContext cleanup](08-memorycontext-error-cleanup.md)：`ERROR` longjmp 后，哪些内存会被 context reset/delete 自动回收，哪些资源不能只靠 MemoryContext 兜底？
5. [allocator 类型、成本与内存诊断](09-memorycontext-allocator-diagnostics.md)：`AllocSet`、`Generation`、`Slab` 等 allocator 如何匹配分配模式，如何用 `pg_backend_memory_contexts`、`MemoryContextStats()` 和 gdb 区分 leak、retention 与正常峰值？

第 3 项 `ResourceOwner 与 ERROR-safe cleanup` 建议先拆成 5 个独立主题，后续生成课程时按“一个主问题一课”处理：

1. [ResourceOwner tree 与外部资源 ownership 边界](10-resourceowner-tree-ownership.md)：为什么内存可以交给 `MemoryContext` 批量 reset，而 buffer pin、lock、snapshot、文件句柄、cache ref 这类资源必须挂到独立的 `ResourceOwner` tree 上？
2. [Remember / Forget hot path 与 acquire-before-ERROR 安全](11-resourceowner-remember-forget-hotpath.md)：为什么 `ResourceOwnerEnlarge()` 必须在真正获取资源前调用，`ResourceOwnerRemember()` / `ResourceOwnerForget()` 如何在 buffer pin、tupledesc refcount、临时文件等路径上把“获取成功但随后 ERROR”的窗口收住？
3. [事务、子事务与 Portal owner 传播](12-resourceowner-xact-portal-propagation.md)：`CurrentResourceOwner` 如何在 top transaction、subtransaction 和 Portal 执行之间切换，为什么子事务提交时锁要转移给父 owner，而 abort 时必须释放？
4. [三阶段 release 与锁释放顺序](13-resourceowner-release-ordering.md)：为什么 `ResourceOwnerRelease()` 要分成 before-locks、locks、after-locks 三段，buffer pin、relcache ref、DSM、JIT、catcache ref、snapshot、文件等资源的释放顺序如何服务并发可见性和 backend-local cleanup？
5. [ERROR-safe cleanup、`PG_TRY` 与诊断边界](14-resourceowner-error-safe-cleanup.md)：`PG_TRY` / `PG_CATCH` 负责恢复 `CurrentResourceOwner` 等全局执行状态，事务 abort 和 `ResourceOwnerRelease()` 负责兜底释放资源；commit warning、`ResourceOwnerDesc` callback 和 gdb / 日志能看到什么、看不到什么？

第 4 项 `PGPROC / ProcArray 与 backend 状态` 建议先拆成 5 个独立主题，后续生成课程时按“一个主问题一课”处理：

1. [PGPROC slot 与 backend identity 生命周期](15-pgproc-backend-identity.md)：为什么每个 backend 必须先获得一个 `PGPROC` 槽位，才能参与 shared memory 中的锁、等待、事务状态发布和退出清理？
2. [ProcArray membership 与事务状态发布](16-procarray-membership-xact-state.md)：一个 backend 什么时候进入 `ProcArray`，如何把 XID、SubXID、xmin、vacuum flags 等事务状态发布给其它 backend，为什么这些字段不能只保存在 backend-local 状态里？
3. [Snapshot 获取与 ProcArray scan 扩展性](17-procarray-snapshot-scalability.md)：为什么一个普通 snapshot 需要扫描全局 backend 状态，`GetSnapshotData()` 如何在 visibility correctness 与 `MaxBackends` 扩展性之间折中？
4. [xmin horizon 与 cleanup 边界](18-procarray-xmin-horizon-cleanup.md)：为什么一个 backend 的 active snapshot、replication / recovery 相关状态会阻止 VACUUM 移除旧 tuple version，ProcArray 如何把局部 xmin 汇总成全局 cleanup horizon？
5. [事务结束、backend 退出与 stale state 清理](19-pgproc-procarray-exit-cleanup.md)：commit、abort、FATAL exit 和 postmaster cleanup 如何把 `PGPROC` / `ProcArray` 状态恢复到可复用边界，哪些状态必须先对其它 backend 不再可见，哪些资源仍要交给后续 cleanup 阶段？

第 5 项 `SpinLock / LWLock / Latch / Condition Variable / Barrier` 建议先拆成 7 个独立主题，后续生成课程时按“一个主问题一课”处理：

1. [SpinLock 与极短 shared state 临界区](20-spinlock-short-critical-section.md)：为什么 PostgreSQL 只允许 spinlock 保护几条指令级的共享字段更新，`s_lock.h` / `spin.h` / `s_lock.c` 如何在 CPU 原子指令、memory barrier、退避和 stuck spinlock PANIC 之间取舍？
2. [LWLock state、tranche 与共享读写互斥](21-lwlock-state-tranche-rw.md)：为什么很多 shared memory 结构不能只靠 spinlock，而需要支持 shared / exclusive 模式、wait event 命名和 cache-line padding 的 `LWLock`，`LWLockAttemptLock()` 如何用一个 atomic state 表达读者数、独占持有者和等待标志？
3. [LWLock 等待队列、唤醒协议与 ERROR-safe release](22-lwlock-waitqueue-wakeup-cleanup.md)：`LWLockAcquire()` 为什么必须“先尝试、入队、再尝试、再睡眠”，`PGPROC.lwWaiting`、process semaphore、`LW_FLAG_WAKE_IN_PROGRESS` 和 `held_lwlocks` 如何共同避免 missed wakeup、重复唤醒和 ERROR 后遗留锁？
4. [Latch 与进程级异步唤醒](23-latch-process-wakeup.md)：为什么后台进程不能靠周期性 `pg_usleep()` 轮询信号和共享标志，`SetLatch()` / `ResetLatch()` / `WaitLatch()` 以及 `PGPROC.procLatch` 如何把 signal、postmaster death、timeout 和 socket readiness 统一成可靠可观测的等待点？
5. [ConditionVariable 与谓词等待循环](24-condition-variable-predicate-wait.md)：为什么等待“某个条件变真”不能只暴露一个 latch，`ConditionVariableSleep()` / `Signal()` / `Broadcast()` 如何用 `PGPROC.cvWaitLink`、spinlock 保护的 wait list 和 spurious wakeup 规则，让 buffer I/O、checkpoint、replication slot 等路径等待共享状态变化？
6. [Barrier 与多进程阶段推进](25-barrier-phase-synchronization.md)：并行 hash join 这类多阶段算法为什么需要所有参与者在阶段边界对齐，`BarrierArriveAndWait()`、`BarrierAttach()`、`BarrierDetach()` 如何用 `phase`、`participants`、`arrived` 和 condition variable 支持 static party 与 dynamic party？
7. [同步原语选择与组合边界](26-sync-primitive-composition.md)：面对一个新的共享状态等待点，如何判断应该使用 spinlock、LWLock、latch、condition variable 还是 barrier，PostgreSQL 为什么经常把它们分层组合成“短临界区修改状态、长等待交给 latch/CV、阶段推进交给 barrier”的模式？

第 6 项 `DSM / shm_toc / shm_mq / DSA` 建议先拆成 8 个独立主题，后续生成课程时按“一个主问题一课”处理：

1. [DSM segment 创建、attach 与 refcount 生命周期](27-dsm-segment-lifecycle.md)：为什么 PostgreSQL 需要运行期创建的共享内存段，`dsm_create()` / `dsm_attach()` / `dsm_detach()` 如何用 control segment、handle、refcount 和 `ResourceOwner` 在“跨进程可见”与“ERROR-safe cleanup”之间取得平衡？
2. [DSM pin、detach callback 与长期共享状态边界](28-dsm-pin-callback-registry-boundary.md)：为什么 DSM 要区分 mapping pin 和 segment pin，`dsm_pin_mapping()`、`dsm_pin_segment()`、`on_dsm_detach()` 以及 postmaster cleanup 如何避免短事务资源释放、后台进程退出和长期共享对象互相踩边界？
3. [shm_toc 与 DSM 内对象发现](29-shm-toc-object-discovery.md)：同一个 DSM 段在不同 backend 中可能映射到不同虚拟地址时，为什么不能直接共享普通指针，`shm_toc_create()` / `allocate()` / `insert()` / `lookup()` 如何用 magic、key 和相对 offset 完成最小 bootstrap？
4. [shm_mq 单生产者单消费者 ring buffer](30-shm-mq-ring-buffer-flow-control.md)：为什么共享消息队列限定为 single-reader / single-writer，`mq_bytes_read`、`mq_bytes_written`、ring wrap、memory barrier 和 latch 唤醒如何在无锁数据搬运、背压和消息边界之间折中？
5. [shm_mq attach、detach 与 worker 失败传播](31-shm-mq-attach-detach-failure.md)：发送方或接收方可能尚未启动、提前退出或 ERROR 时，`shm_mq_wait_for_attach()`、`BackgroundWorkerHandle`、`SHM_MQ_WOULD_BLOCK` / `SHM_MQ_DETACHED` 和 on-detach callback 如何把“等不到对端”变成可诊断的状态？
6. [DSA area、dsa_pointer 与跨进程共享 heap](32-dsa-area-pointer-ownership.md)：为什么 DSM 只适合整段共享而不适合小对象动态分配，`dsa_create()` / `dsa_attach()` / `dsa_get_address()` 如何用 area handle、segment map 和 `dsa_pointer` 支持可在进程间传递但不能直接解引用的共享指针？
7. [DSA allocator：size class、span 与 superblock](33-dsa-allocator-spans-superblocks.md)：`dsa_allocate()` / `dsa_free()` 如何把小对象、large allocation、FreePageManager、pagemap、span freelist 和 per-size-class LWLock 组合成一个共享 allocator，哪些锁竞争和碎片化成本会随 workload 放大？
8. [DSA segment 增长、回收与 named DSM registry](34-dsa-segment-reclaim-registry.md)：DSA area 如何按需创建、pin、trim 和释放多个 DSM segment，`freed_segment_counter` 如何防止 segment slot 重用造成旧映射误读，`GetNamedDSMSegment()` / `GetNamedDSA()` 又如何把这种动态共享状态提升为可复用的命名基础设施？

第 7 项 `ParallelContext 与并行执行基础设施` 建议先拆成 8 个独立主题，后续生成课程时按“一个主问题一课”处理：

1. [Parallel mode 与 ParallelContext ownership 边界](35-parallel-context-lifecycle.md)：为什么并行操作必须包在 parallel mode 中，`CreateParallelContext()`、`pcxt_list`、subtransaction id、`AtEOSubXact_Parallel()` / `AtEOXact_Parallel()` 和 `DestroyParallelContext()` 如何把 worker、DSM、error queue 等资源绑定到可清理的事务边界？
2. [InitializeParallelDSM 与 leader 状态序列化](36-parallel-dsm-state-serialization.md)：并行 worker 不能直接继承 leader 的 backend-local 指针时，`InitializeParallelDSM()` 如何用 `shm_toc_estimator`、`FixedParallelState`、snapshot、GUC、combo CID、transaction state、session DSM handle 和 entrypoint key 构造 worker 可恢复的最小共享启动包？
3. [parallel worker launch、attach barrier 与少 worker fallback](37-parallel-worker-launch-attach-fallback.md)：`LaunchParallelWorkers()` 如何用 dynamic background worker、`BackgroundWorkerHandle`、lock group leader 和 error queue 启动并行 worker，`WaitForParallelWorkersToAttach()` 又如何把注册失败、启动失败和实际 worker 数不足变成 executor 可以继续或报错的状态？
4. [ParallelWorkerMain 状态恢复与 worker backend bootstrap](38-parallel-worker-bootstrap-state-restore.md)：`ParallelWorkerMain()` 为什么要先 attach DSM / TOC 和 error queue，再恢复 database connection、role/security context、transaction snapshot、GUC、session DSM、serializable xact 与 lock group membership，最后才调用 `ParallelWorkerEntryPoint`？
5. [parallel message、ERROR 传播与 worker finish 协议](39-parallel-message-error-cleanup.md)：worker 中的 `ereport()`、NOTICE、NOTIFY、progress 和 terminate 消息如何经 `pq_redirect_to_shm_mq()`、`HandleParallelMessageInterrupt()`、`ProcessParallelMessages()` 与 `WaitForParallelWorkersToFinish()` 回到 leader，并在 `DestroyParallelContext()` 中完成中断、等待和队列清理？
6. [executor parallel DSM：plan、params、DSA 与 node shared state](40-exec-parallel-dsm-plan-param.md)：`ExecInitParallelPlan()` 为什么要序列化 `PlannedStmt`、`ParamListInfo`、`PARAM_EXEC`、query text、instrumentation 和 DSA area，`ExecParallelEstimate()` / `ExecParallelInitializeDSM()` 又如何让并行感知 plan node 在同一个 DSM 中建立自己的共享状态？
7. [Gather / GatherMerge tuple routing 与 leader participation](41-gather-tuple-routing-leader-participation.md)：`ExecGather()` / `ExecGatherMerge()` 为什么延迟到第一次执行才启动 worker，`ExecParallelSetupTupleQueues()`、`ExecParallelCreateReaders()`、`TupleQueueReader` 和 `parallel_leader_participation` 如何在 worker tuple stream、leader 本地执行、背压和有序 merge 之间取舍？
8. [parallel rescan、instrumentation 汇总与 cleanup ordering](42-parallel-reinitialize-instrumentation-cleanup.md)：`ExecParallelReinitialize()` 如何支持重新发起一批 worker，`ExecParallelFinish()` / `ExecParallelCleanup()` 为什么要先 detach tuple queue、再等待 worker、再汇总 buffer / WAL / JIT / instrumentation，最后释放 DSA 和 `ParallelContext`？
