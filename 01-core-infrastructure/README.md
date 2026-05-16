# 内核基础设施

本目录约 8 节课，目标是建立 PostgreSQL 多进程共享状态、资源生命周期和同步模型。

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
