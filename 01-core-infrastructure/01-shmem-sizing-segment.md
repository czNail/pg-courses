# PostgreSQL Shmem 启动期 sizing 与一次性 segment 创建

## 课程定位

前置知识：熟悉 PostgreSQL postmaster / backend 多进程模型，知道 `shared_buffers`、`max_connections`、`shared_preload_libraries` 这类 GUC 会改变服务器启动形态。

本节唯一主问题：

```text
为什么 PostgreSQL 的 main shared memory 必须在启动期先汇总大小，再由 postmaster 一次性创建固定 segment？
```

核心矛盾：所有 backend 都要低成本访问同一批 shared state，但这些 shared state 的大小依赖配置、扩展和平台能力；如果允许运行期随意增长，指针稳定性、启动一致性、崩溃清理和跨平台 attach 都会变复杂。

学完后应能判断：一个状态该进入 main shared memory、DSM、backend-local memory，还是文件系统状态；也能解释为什么某些 GUC 或扩展必须在 server start 前确定。

本课基于本地 `~/postgres` 源码，分支 `master`，提交 `8eba2edb801`。

## 1. 本节在总主线中的位置

`Shmem 初始化与 shared state 边界` 可以继续拆成 allocator、request/init/attach、扩展边界等主题。本节只讲第一步：postmaster 如何在任何普通 backend 运行前，决定 main shared memory segment 到底有多大，并把它创建出来。

后续课程会继续追问：segment 创建后，一个名字如何绑定到一块 shared memory；每个 backend 如何拿到同一个状态指针；哪些场景必须转向 DSM。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
postmaster 先让核心子系统和 preload 扩展申报 shared memory 需求，
再把这些需求加总成 shared_memory_size，
最后调用 PGSharedMemoryCreate() 创建一个固定大小的 main segment。
```

这里的系统 tension 是：

```text
稳定共享地址和低运行期开销
  vs
配置、扩展、后台进程数量、平台 shared memory 能力带来的启动期不确定性
```

PostgreSQL 选择把不确定性集中到启动期处理。启动后，main shared memory 的总大小不再重新规划；运行期如果需要弹性共享内存，应该使用 DSM，而不是假装 main shmem 可以无限扩展。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 作用 |
| --- | --- | --- |
| 1 | `src/backend/postmaster/postmaster.c` | 正常 server 启动顺序：加载 preload、计算 `MaxBackends`、request shmem、计算 GUC、创建 segment。 |
| 2 | `src/backend/tcop/postgres.c` | single-user mode 的相似路径，用来对照“即使不是真的多进程，也要建立同一套 shared state”。 |
| 3 | `src/backend/storage/ipc/ipci.c` | `CalculateShmemSize()` 与 `CreateSharedMemoryAndSemaphores()` 的集中入口。 |
| 4 | `src/backend/storage/ipc/shmem.c` | request callback、pending requests、`ShmemGetRequestedSize()` 和 allocator 初始化前的状态机。 |
| 5 | `src/include/storage/subsystemlist.h` | 内置子系统 shared memory callback 的注册顺序。 |
| 6 | `src/include/storage/pg_shmem.h` | `PGShmemHeader` 描述 main segment 的稳定边界。 |
| 7 | `src/backend/port/sysv_shmem.c` / `src/backend/port/win32_shmem.c` | 平台相关 segment 创建、huge pages、旧 segment 清理和 attach 语义。 |

本节不要从 `ShmemAlloc()` 往下读。那是下一节 allocator 主题。本节只读到“总大小如何确定”和“segment 如何成为所有后续 shared state 的容器”。

## 4. 关键数据结构与状态

### `PGShmemHeader`

`PGShmemHeader` 是 segment 头部，不是普通业务状态。它定义边界：

| 字段 | 语义 |
| --- | --- |
| `magic` | 判断是不是 PostgreSQL shared memory segment。 |
| `creatorPID` | 创建者 PID。 |
| `totalsize` | main segment 总大小。后续 allocator 不能越过这个边界。 |
| `content_offset` | 头部后第一个可分配内容的偏移。 |
| `dsm_control` | DSM control segment 的句柄，用来把 main shmem 和 DSM 体系接起来。 |
| `device` / `inode` | 非 Windows 平台用 data directory 身份识别旧 segment 是否属于当前集群。 |

raw field 不是语义。`totalsize` 只有和 `ShmemSegHdr`、allocator 的 `free_offset`、`pg_shmem_allocations` 里的 unused 行一起看，才表示“main segment 还能容纳多少启动期或少量 late allocation”。

### request 阶段的 backend-local 状态

在 `shmem.c` 中，`registered_shmem_callbacks` 和 `pending_shmem_requests` 是进程私有内存里的 list。它们不是 shared state 本身，而是创建 shared state 之前的清单。

关键状态机：

```text
SRS_INITIAL
  -> SRS_REQUESTING
  -> SRS_INITIALIZING
  -> SRS_DONE
```

这个状态机的语义是：只有在 request 阶段，子系统才允许声明“我需要多大的一块 shared memory”。过了这个阶段再声明，就不再是正常启动路径。

### `MaxBackends` 与规模变量

`InitializeMaxBackends()` 在 shared memory size 计算前运行：

```text
MaxBackends =
  MaxConnections
  + autovacuum_worker_slots
  + max_worker_processes
  + max_wal_senders
  + NUM_SPECIAL_WORKER_PROCS
```

这会影响 `PGPROC`、`ProcArray`、lock table、backend status 等 shared state 的大小。也就是说，main shmem 的大小不是只由 `shared_buffers` 决定，很多“进程槽位上限”也会进入启动期 sizing。

## 5. 主流程源码 walkthrough

主流程以 postmaster 正常启动为轴：

```text
PostmasterMain()
  -> RegisterBuiltinShmemCallbacks()
  -> process_shared_preload_libraries()
  -> InitializeMaxBackends()
  -> InitializeFastPathLocks()
  -> process_shmem_requests()
  -> ShmemCallRequestCallbacks()
  -> InitializeShmemGUCs()
  -> CreateSharedMemoryAndSemaphores()
     -> CalculateShmemSize()
     -> PGSharedMemoryCreate()
     -> InitShmemAllocator()
     -> ShmemInitRequested()
     -> dsm_postmaster_startup()
     -> shmem_startup_hook()
```

第一段，`RegisterBuiltinShmemCallbacks()` 读取 `subsystemlist.h`，把核心子系统的 `ShmemCallbacks` 注册起来。这个列表里有 LWLock、XLOG、CLOG、buffer manager、ProcArray、background worker、stats、AIO 等。顺序有语义，比如 LWLocks 先注册，因为其他初始化过程可能要使用 LWLock。

第二段，`process_shared_preload_libraries()` 让 preload 扩展有机会改变 GUC、注册新 callbacks 或 legacy hooks。随后 `InitializeMaxBackends()` 和 `InitializeFastPathLocks()` 才能根据最终配置算出进程槽位和 fast-path lock 数组大小。

第三段，`process_shmem_requests()` 调用 legacy `shmem_request_hook`。这里必须早于 `ShmemCallRequestCallbacks()`，因为扩展可能通过 `RequestNamedLWLockTranche()` 影响 LWLock array 的大小。

第四段，`ShmemCallRequestCallbacks()` 进入 `SRS_REQUESTING`，逐个调用子系统的 `request_fn`。例如 buffer manager 会请求：

```text
Buffer Descriptors
Buffer Blocks
Buffer IO Condition Variables
Checkpoint BufferIds
```

这些 request 还没有分配内存，只是记录名字、大小、对齐要求和要回填的指针地址。

第五段，`InitializeShmemGUCs()` 再次调用 `CalculateShmemSize()`，把计算结果写入 internal GUC：

```text
shared_memory_size
shared_memory_size_in_huge_pages
```

这解释了为什么 `postgres -C shared_memory_size` 必须走完 preload 和 request 阶段，才能给出有意义的值。

第六段，`CreateSharedMemoryAndSemaphores()` 真正创建 segment。它再次计算 size，调用平台层 `PGSharedMemoryCreate(size, &shim)`，得到 `PGShmemHeader`，然后才进入 allocator 和各子系统 init。

时间线上最重要的不变量：

```text
所有 main shmem 大对象的大小，必须在 PGSharedMemoryCreate() 之前已知。
PGSharedMemoryCreate() 之后，totalsize 成为硬边界。
```

## 6. 生命周期 / ownership / cleanup

### 谁创建

正常 server 模式下，postmaster 创建 main shared memory segment。single-user mode 也会调用 `CreateSharedMemoryAndSemaphores()`，因为即使没有多个 OS 进程，很多内核代码仍假设这些 shared state 存在。

### 谁持有

segment 的系统资源由 postmaster 启动路径建立，backend 通过 fork 继承映射；在 `EXEC_BACKEND` 平台，child process 需要重新 attach。`PGShmemHeader` 和平台层的 `UsedShmemSegID` / `UsedShmemSegAddr` 共同描述“哪个 segment 是当前集群的 main shmem”。

### 谁释放

平台层 `PGSharedMemoryCreate()` 会注册 exit callback。postmaster 退出或崩溃重启时，平台层负责 detach / remove 相关资源。SysV 路径还会用 data directory 的 `device` / `inode` 识别旧 segment：如果旧 segment 属于当前数据目录且已经无人 attach，可以清理后重建；如果仍在使用，会 FATAL，避免两个 postmaster 同时操作同一个集群状态。

### ERROR / FATAL 时怎么办

本节这个阶段大多发生在普通 backend 启动前，因此失败通常是启动失败，不是事务级 ERROR cleanup。典型结果是：

| 失败点 | 结果 |
| --- | --- |
| request 阶段大小溢出 | `add_size()` / `mul_size()` 路径报错，阻止错误 size 进入创建阶段。 |
| `RequestAddinShmemSpace()` 不在 `shmem_request_hook` 内调用 | FATAL。 |
| `ShmemRequestStruct()` 不在 request callback 内调用 | ERROR。 |
| OS 无法创建 segment 或 huge pages 不满足要求 | ERROR / FATAL，server 不进入可服务状态。 |
| 发现仍在使用的旧 SysV segment | FATAL，提示终止旧 server processes。 |

## 7. 正确性机制层次

本节的正确性不是靠 WAL 或 MVCC，而是靠启动顺序、地址稳定性和资源边界。

| 层次 | 机制 | 作用 |
| --- | --- | --- |
| 启动顺序 | preload -> GUC -> request -> calculate -> create | 保证所有配置和扩展需求先进入 size 计算。 |
| 状态机 | `SRS_INITIAL` / `SRS_REQUESTING` / `SRS_INITIALIZING` / `SRS_DONE` | 防止子系统在错误阶段声明 shared memory。 |
| 溢出检查 | `add_size()` / `mul_size()` | 防止 size_t wraparound 让 segment 被低估。 |
| 平台 segment 身份 | SysV key、data directory `device` / `inode`、`PGShmemMagic` | 区分当前集群旧 segment、外部 segment 和仍在使用的 segment。 |
| 地址稳定性 | fork 继承或 `EXEC_BACKEND` reattach | 让 backend 能访问同一个 main shared memory。 |
| 大小硬边界 | `PGShmemHeader.totalsize` | allocator 和观测入口都以它作为上限。 |

## 8. 错误路径 / 异常路径 / fallback

### preload 扩展过晚请求 shared memory

legacy 扩展如果要 main shmem，必须在 `shared_preload_libraries` 中加载，并在 `shmem_request_hook` 中调用 `RequestAddinShmemSpace()`。源码用 `process_shmem_requests_in_progress` 强制这个边界。

这是一个设计信号：main shmem 是启动期协议，不是普通 backend 想申请就申请的堆。

### huge pages fallback

Linux mmap 路径可以尝试 huge pages。`InitializeShmemGUCs()` 会计算 `shared_memory_size_in_huge_pages`，平台层创建时再决定 `huge_pages_status`。如果 `huge_pages=on` 但平台或 `shared_memory_type` 不支持，启动会失败；如果是 try 类语义，可能 fallback 到普通页。

### stale SysV segment

SysV 路径不是简单“key 冲突就失败”。它会检查旧 segment 是否属于当前 data directory、是否仍被进程 attach。无人使用的旧 segment 可以清理，仍在使用的旧 segment 必须阻止启动。

### single-user mode

single-user mode 走类似初始化路径。这说明 main shmem 不只是多进程通信工具，也是一批内核模块共享假设的初始化基座。

## 9. 成本、资源与跨模块传播

main shmem size 主要随这些变量扩张：

| 变量 | 影响方向 |
| --- | --- |
| `shared_buffers` | 直接放大 `Buffer Blocks`、`Buffer Descriptors`、buffer IO condition variables、checkpoint sort array。 |
| `max_connections` / worker / wal sender 上限 | 放大 `MaxBackends`，进而影响 `PGPROC`、`ProcArray`、backend status、lock 相关数组。 |
| `max_locks_per_transaction` | 影响 lock table、fast-path lock grouping 和 LWLock 相关需求。 |
| preload 扩展 | 通过 callbacks、`RequestAddinShmemSpace()`、named LWLock tranches 增加启动期需求。 |
| `wal_level` / hot standby 相关配置 | 会影响某些 recovery / ProcArray 相关结构是否请求。 |
| AIO、stats、replication slots 等功能配置 | 影响各自 subsystem request 的大小。 |

跨模块传播的关键点在 `subsystemlist.h`：很多模块并不知道最终 segment 多大，只声明自己的局部需求；`ipci.c` 把这些局部需求折叠成一个全局资源请求。

这也是本节可迁移的模型：

```text
局部模块声明需求，全局启动协议统一定容；
运行期 hot path 只消费稳定地址，不重新谈判容量。
```

## 10. 观测与诊断入口

能直接观测：

```sql
SHOW shared_memory_size;
SHOW shared_memory_size_in_huge_pages;
SHOW huge_pages_status;
```

能看 main shmem 内部 allocation 分布：

```sql
SELECT name, size, allocated_size
FROM pg_shmem_allocations
ORDER BY allocated_size DESC
LIMIT 20;
```

看 unused 和 anonymous：

```sql
SELECT name, off, size, allocated_size
FROM pg_shmem_allocations
WHERE name IS NULL OR name = '<anonymous>';
```

能在未启动 server 前观察 runtime-computed GUC：

```bash
postgres -D "$PGDATA" -C shared_memory_size
postgres -D "$PGDATA" -C shared_memory_size -c shared_buffers=512MB
postgres -D "$PGDATA" -C shared_memory_size -c max_connections=300
```

能从日志看到：

```text
DEBUG3: invoking IpcMemoryCreate(size=...)
```

但要注意边界：`pg_shmem_allocations` 不包含 DSM；它只展示 main shared memory segment 的分配。`shared_memory_size` 是向上取整到 MB 的值，不是每个子结构 size 的逐项精确加总。

## 11. 常见误区

误区一：`shared_memory_size` 约等于 `shared_buffers`。

实际不是。`shared_buffers` 往往是大头，但 ProcArray、PGPROC、locks、WAL、stats、replication、AIO、扩展 request 都会进入 main shmem。

误区二：启动后还可以把 main shmem 当 malloc 用。

传统 main shmem 可以有少量 late allocation 余量，但设计上不应该依赖它。稳定、大块、生命周期和 backend 集合有关的共享状态，应在启动期声明；弹性和短生命周期共享状态应考虑 DSM / DSA。

误区三：`ShmemRequestStruct()` 已经分配了内存。

request 阶段只是登记需求。真正的 segment 创建发生在 `PGSharedMemoryCreate()`，具体区域分配发生在 `ShmemInitRequested()` 之后的 allocator 路径。

误区四：single-user mode 不需要 shared memory。

源码仍会创建和初始化这些结构，因为大量内核路径依赖同一套 shared state 抽象。

## 12. 课堂实验

### 实验 1：观察配置如何改变启动期 size

在同一个数据目录上比较：

```bash
postgres -D "$PGDATA" -C shared_memory_size -c shared_buffers=128MB
postgres -D "$PGDATA" -C shared_memory_size -c shared_buffers=512MB
postgres -D "$PGDATA" -C shared_memory_size -c max_connections=300
```

预期：`shared_buffers` 改变会显著影响结果；`max_connections` 也会影响结果，但增长来自 backend 槽位、ProcArray、lock 等结构，而不是 buffer blocks。

### 实验 2：把观测值拆回 allocation 分布

启动 server 后执行：

```sql
SHOW shared_memory_size;

SELECT name, pg_size_pretty(size) AS size, pg_size_pretty(allocated_size) AS allocated
FROM pg_shmem_allocations
ORDER BY allocated_size DESC
LIMIT 15;
```

把最大的几项和 `shared_buffers`、`max_connections` 联系起来。重点不是背每一项，而是看到“配置变量 -> 子系统 request -> main segment allocation”的链路。

### 实验 3：确认 DSM 不在这个视图里

执行：

```sql
SELECT COUNT(*) FROM pg_shmem_allocations;
SELECT * FROM pg_dsm_registry_allocations;
```

不同 workload 下结果可能不同。目标是确认两个观测入口的边界：main shmem allocation 和 DSM registry allocation 不是同一个资源池。

### 实验 4：源码断点

在调试构建中给这些函数设断点：

```text
ShmemCallRequestCallbacks
CalculateShmemSize
PGSharedMemoryCreate
InitShmemAllocator
ShmemInitRequested
```

观察 `pending_shmem_requests` 何时变长，`CalculateShmemSize()` 何时读取它，`PGShmemHeader.totalsize` 何时成为硬边界。

## 13. 讨论题

1. 为什么 `InitializeMaxBackends()` 必须在 shared memory size 计算前，而不是在 backend 启动时按需扩容？
2. 如果允许 `ProcArray` 运行期增长，会影响哪些正确性和性能路径？
3. 为什么 `shared_preload_libraries` 里的扩展可以影响 main shmem，而普通 session preload 不应该影响？
4. `pg_shmem_allocations` 里 unused memory 很多，是否一定说明配置浪费？还需要看哪些 workload 和扩展因素？
5. 一个扩展需要跨 backend 共享一个大 hash table，应优先使用 main shmem 还是 DSM？判断依据是什么？

## 14. 本节小结

本节主链路是：

```text
注册 shared memory callbacks
  -> preload 扩展介入
  -> 计算 MaxBackends / lock 规模
  -> request 所有 main shmem 需求
  -> CalculateShmemSize()
  -> PGSharedMemoryCreate()
```

核心状态边界是 `PGShmemHeader.totalsize`。它把启动期所有局部 request 压缩成一个固定资源池，后续 allocator、`ShmemIndex`、各 subsystem init 都不能越过这个边界。

本节不变量：main shared memory 的大对象必须启动期定容；启动后普通 backend 消费稳定地址，而不是重新协商共享状态大小。

能观测的是 `shared_memory_size`、`shared_memory_size_in_huge_pages`、`huge_pages_status` 和 `pg_shmem_allocations`。不能从这些入口直接看到 DSM，也不能仅凭 `shared_memory_size` 判断哪个子系统浪费，需要结合 allocation 分布、GUC 和 workload。

可迁移规律：

```text
当多个进程需要长期、低成本、指针稳定地共享状态时，
系统通常会把容量协商前移到启动期；
运行期弹性则交给另一套带句柄、offset 或 registry 的机制。
```

这些判断依赖版本、平台、huge pages 配置、preload 扩展和 workload。不要把当前调用链误认为永恒接口；要抓住更稳定的边界：启动期定容、postmaster 创建、backend attach、main shmem 与 DSM 分工。
