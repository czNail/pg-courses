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
