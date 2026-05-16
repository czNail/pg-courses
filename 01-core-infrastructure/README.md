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
