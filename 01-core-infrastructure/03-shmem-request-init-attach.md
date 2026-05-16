# PostgreSQL Shmem request / init / attach 生命周期

## 课程定位

前置知识：已经理解 main shared memory segment 必须在启动期定容，也知道 `ShmemIndex` 如何把名字绑定到一块稳定的 shared memory allocation。

本节唯一主问题：

```text
postmaster、普通 fork backend 和 EXEC_BACKEND backend，如何在不同进程中建立同一组 shared state 指针？
```

核心矛盾：PostgreSQL 希望共享状态指针在 C 代码里像普通全局变量一样便宜、直接、少封装；但不同平台的进程创建模型并不相同。Unix fork 可以继承 postmaster 已经填好的全局指针，`EXEC_BACKEND` 则会重新 exec 一个新进程，不能依赖 postmaster 的 backend-local 内存和函数指针。

学完后应能判断：一个新 shared memory 子系统应该把工作放在 `request_fn`、`init_fn` 还是 `attach_fn`；为什么只在 postmaster 里能做一次性初始化；为什么 `EXEC_BACKEND` 会暴露“隐藏依赖 fork 继承”的 bug。

本课基于本地 `~/postgres` 源码，分支 `master`，提交 `8eba2edb801`。

## 1. 本节在总主线中的位置

上一节讲的是一个名字如何落到 `ShmemIndex` 中，并最终变成稳定地址。本节把时间线拉长：同一批 shared memory request 在 postmaster、普通 fork backend、`EXEC_BACKEND` backend 中分别经历什么阶段。

本节不再重点讨论 allocator 为什么不能 free，而是关注两个层次的状态如何重建：

```text
shared memory 中的真实对象
  vs
每个进程自己的 C 全局指针、HTAB handle、per-backend 辅助状态
```

下一节会进入扩展边界：`shared_preload_libraries`、legacy hook、late allocation 和什么时候该转向 DSM。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
request_fn 只声明“需要哪些 named shared areas”，
postmaster 在创建 segment 后执行 init_fn 做一次性 shared-state 初始化，
fork backend 继承 postmaster 的指针，
EXEC_BACKEND backend 重新跑 request_fn，再用 ShmemIndex 执行 attach_fn 重建本进程指针和本地状态。
```

这里的 tension 是：

```text
把 shared state 暴露成便宜的全局指针
  vs
让同一套代码同时适配 fork inheritance 和 exec-after-spawn
```

如果只有 fork，一切似乎很简单：postmaster 把 `BufferDescriptors`、`ProcGlobal`、`MainLWLockArray` 等全局变量设好，子进程继承地址空间即可。但 PostgreSQL 还要支持 `EXEC_BACKEND` 构建，例如 Windows。exec 后，进程私有内存重新开始，预加载库也要重新加载，C 全局变量和回调函数地址不能假设从 postmaster 继承。

因此 PostgreSQL 把生命周期拆成三个语义不同的阶段：

| 阶段 | 语义 | 典型执行者 |
| --- | --- | --- |
| request | 声明需要哪些 named shmem area，以及要写回哪个本进程指针变量 | postmaster；`EXEC_BACKEND` backend |
| init | 第一次创建 shared memory object，并初始化其中的共享内容 | postmaster 或 standalone backend |
| attach | 已有 shared memory object，当前进程把自己的指针和本地辅助状态接上去 | `EXEC_BACKEND` backend；late allocation attach 路径 |

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 作用 |
| --- | --- | --- |
| 1 | `src/include/storage/shmem.h` | 定义 `ShmemCallbacks`、`ShmemStructOpts`、`request_fn` / `init_fn` / `attach_fn` 的公开语义。 |
| 2 | `src/backend/storage/ipc/shmem.c` | request 状态机、`pending_shmem_requests`、`ShmemCallRequestCallbacks()`、`ShmemInitRequested()`、`ShmemAttachRequested()` 的核心实现。 |
| 3 | `src/backend/storage/ipc/ipci.c` | postmaster 创建 shared memory 的主入口，以及 `EXEC_BACKEND` 子进程 attach 的统一入口。 |
| 4 | `src/include/storage/subsystemlist.h` | 内置子系统注册顺序；顺序本身是 correctness 边界的一部分。 |
| 5 | `src/backend/postmaster/postmaster.c` | postmaster 启动期：注册 callback、加载 `shared_preload_libraries`、request、计算 GUC、创建 shared memory。 |
| 6 | `src/backend/postmaster/launch_backend.c` | `EXEC_BACKEND` 子进程启动期：重新加载配置和预加载库，重新注册 request，恢复 allocator 指针。 |
| 7 | `src/backend/storage/lmgr/proc.c` | `EXEC_BACKEND` 下 attach 必须等到 `PGPROC` / LWLock access 可用后的例子。 |
| 8 | `src/backend/storage/buffer/buf_init.c` | 一个完整子系统例子：buffer manager 同时定义 request、init、attach。 |
| 9 | `src/backend/utils/init/miscinit.c` | `process_shared_preload_libraries()` 与 legacy `process_shmem_requests()` 的位置。 |

阅读这条链路时要特别注意：`request_fn` 不是 allocation，`attach_fn` 也不是 shared object 初始化。它们分别服务不同进程里的不同状态。

## 4. 关键数据结构与状态

### `ShmemCallbacks`

`ShmemCallbacks` 是一个子系统对 shared memory 生命周期的声明：

| 字段 | 语义 |
| --- | --- |
| `request_fn` | 在还没有创建或还没有重新 attach shared area 之前调用；它调用 `ShmemRequestStruct()` / `ShmemRequestHash()` 登记名字、大小和要设置的指针变量。 |
| `init_fn` | postmaster 创建 shared memory 后调用；用于初始化真正共享的内容。 |
| `attach_fn` | 当前进程已经找到 existing shared area 并写回指针后调用；用于初始化本进程私有状态。 |
| `opaque_arg` | shmem 框架不解释的 callback 参数。 |
| `flags` | 例如 `SHMEM_CALLBACKS_ALLOW_AFTER_STARTUP`，允许启动后注册并立即 init 或 attach。 |

一个典型例子是 `BufferManagerShmemCallbacks`：

```text
request_fn = BufferManagerShmemRequest
init_fn    = BufferManagerShmemInit
attach_fn  = BufferManagerShmemAttach
```

`BufferManagerShmemRequest()` 声明 `"Buffer Descriptors"`、`"Buffer Blocks"`、`"Buffer IO Condition Variables"`、`"Checkpoint BufferIds"`，并把 `.ptr` 指向 `BufferDescriptors`、`BufferBlocks` 等全局变量。`BufferManagerShmemInit()` 初始化 buffer descriptor 和 condition variable；`BufferManagerShmemAttach()` 只初始化 per-backend 的 writeback context。

这个例子正好说明边界：

```text
shared buffer descriptor 的内容初始化：init_fn
每个 backend 自己的 file flush context：init_fn 和 attach_fn 都要覆盖
```

### `pending_shmem_requests`

`pending_shmem_requests` 是进程私有 list，不在 shared memory 里。每次 `request_fn` 调用 `ShmemRequestStruct()`，都会把一个 `ShmemRequest` 追加进去。

它的生命周期很短：

```text
ShmemCallRequestCallbacks()
  -> 各子系统 request_fn 填充 pending_shmem_requests
  -> postmaster: ShmemInitRequested() 消费并释放 list
  -> EXEC_BACKEND backend: ShmemAttachRequested() 消费并释放 list
```

因此它不是“系统登记表”。真正跨进程持久存在的登记表仍然是上一节讲的 `ShmemIndex`。`pending_shmem_requests` 只是当前进程对“我需要哪些名字，以及我要把地址写到哪个本地变量”的临时计划。

### `shmem_request_state`

`shmem_request_state` 是进程私有状态机，用来约束调用顺序。源码注释给出了三条路径：

```text
Postmaster:
  INITIAL -> REQUESTING -> INITIALIZING -> DONE

Backends in EXEC_BACKEND mode:
  INITIAL -> REQUESTING -> ATTACHING -> DONE

Late request:
  DONE -> REQUESTING -> AFTER_STARTUP_ATTACH_OR_INIT -> DONE
```

它解决的不是并发同步问题，而是 API misuse 问题：`ShmemRequestStruct()` 只能在 request callback 中调用，不能散落在任意初始化代码里。否则普通 fork 模式可能“看起来能跑”，但 `EXEC_BACKEND` 会因为无法重建相同 request 列表而失败。

### `ShmemStructOpts.ptr`

`ShmemStructOpts.ptr` 是本节最容易被低估的字段。它不是指向 shared memory 的指针，而是“指向一个本进程 C 指针变量的指针”。

```text
.ptr = (void **) &BufferDescriptors
```

postmaster init 时，框架会把 `BufferDescriptors` 设为 shared memory 中的 `"Buffer Descriptors"` 地址。普通 fork backend 继承这个变量值。`EXEC_BACKEND` backend 则重新执行 request，`ShmemAttachRequested()` 查 `ShmemIndex` 后再把自己的 `BufferDescriptors` 设为同一个 shared memory 地址。

这就是本节的核心抽象：

```text
名字和 size 是跨进程协议；
ptr 是每个进程本地的接线点。
```

## 5. 主流程源码 walkthrough

### 5.1 postmaster：从 callback 注册到 shared object 初始化

postmaster 启动期的主链路在 `postmaster.c` 和 `ipci.c` 中：

```text
PostmasterMain()
  -> RegisterBuiltinShmemCallbacks()
     -> include storage/subsystemlist.h
     -> RegisterShmemCallbacks(&EachSubsystemCallbacks)
  -> process_shared_preload_libraries()
     -> 扩展 _PG_init() 可注册 ShmemCallbacks
  -> InitializeMaxBackends()
  -> InitializeFastPathLocks()
  -> process_shmem_requests()
     -> legacy shmem_request_hook
  -> ShmemCallRequestCallbacks()
     -> shmem_request_state = SRS_REQUESTING
     -> 调用所有 request_fn
     -> pending_shmem_requests 收集 name / size / ptr
  -> InitializeShmemGUCs()
  -> CreateSharedMemoryAndSemaphores()
```

进入 `CreateSharedMemoryAndSemaphores()` 后：

```text
CreateSharedMemoryAndSemaphores()
  -> CalculateShmemSize()
     -> ShmemGetRequestedSize()
     -> total_addin_request
  -> PGSharedMemoryCreate(size, &shim)
  -> InitShmemAllocator(seghdr)
  -> ShmemInitRequested()
     -> 对每个 pending request 执行 InitShmemIndexEntry()
     -> 设置 options->ptr 指向 shared memory location
     -> 调用所有 init_fn
     -> shmem_request_state = SRS_DONE
  -> dsm_postmaster_startup(shim)
  -> shmem_startup_hook()
```

这条路径里有两个“必须早于”的顺序：

| 顺序 | 原因 |
| --- | --- |
| `process_shared_preload_libraries()` 早于 `ShmemCallRequestCallbacks()` | 预加载扩展需要先注册 callbacks，才有机会参与统一 request。 |
| `ShmemCallRequestCallbacks()` 早于 `CalculateShmemSize()` / `PGSharedMemoryCreate()` | 必须先知道所有 named areas 的 size，才能创建一次性 main segment。 |

`ShmemInitRequested()` 有一个重要特征：它先把所有 requested area 的基础指针都建立好，再调用各子系统 `init_fn`。这意味着某个 `init_fn` 可以引用其它已经 request 过、已经写回 `.ptr` 的 shared areas。但这也让 `subsystemlist.h` 的顺序成为隐含契约：例如 LWLock callbacks 排在前面，因为其它 init 代码可能用 LWLocks。

### 5.2 普通 fork backend：为什么不需要 attach 主流程

在非 `EXEC_BACKEND` 的 Unix-like 环境中，backend 是从 postmaster fork 出来的。fork 复制父进程的虚拟地址空间，因此：

```text
postmaster 中的 BufferDescriptors = 0x...
fork backend 后的 BufferDescriptors = 同一个数值
main shmem segment 也映射在同一地址
```

所以普通 fork backend 不需要再次调用 `ShmemAttachRequested()` 来给每个全局变量接线。它继承了 postmaster 已经填好的 backend-local 指针变量。

这里容易产生一个误解：不是 shared memory 自动知道 `BufferDescriptors` 这个 C 变量，而是 fork 复制了 postmaster 的 C 变量值。shared memory 中保存的是 named allocation 和真正对象；`BufferDescriptors` 这样的全局变量仍然是进程私有内存。

因此普通 fork 模式虽然省掉了 attach 阶段，但它不是另一套语义。它只是用 OS fork inheritance 作为 attach 的快捷路径。

### 5.3 `EXEC_BACKEND` backend：重新 request，再 attach

`EXEC_BACKEND` 路径出现在 `launch_backend.c`。新进程不是简单继承 postmaster 的 backend-local 内存，而是重新启动可执行文件并读入必要参数。

核心链路是：

```text
SubPostmasterMain()
  -> PGSharedMemoryReAttach()
  -> read_nondefault_variables()
  -> LocalProcessControlFile(false)
  -> RegisterBuiltinShmemCallbacks()
  -> process_shared_preload_libraries()
  -> InitShmemAllocator(UsedShmemSegAddr)
  -> ShmemCallRequestCallbacks()
  -> child_process_kinds[child_type].main_fn(...)
```

注意这时只是重新建立了 allocator / `ShmemIndex` 的基本访问能力，并重新收集了 `pending_shmem_requests`。真正对所有 named areas 执行 attach，不在 `launch_backend.c` 里立即完成。

普通 backend 和 auxiliary process 都在获得 `PGPROC` 并初始化 LWLock access 之后调用：

```text
InitProcess()
  -> 分配当前进程的 PGPROC
  -> InitLWLockAccess()
  -> InitDeadLockChecking()
  -> AttachSharedMemoryStructs()

AttachSharedMemoryStructs()
  -> InitializeFastPathLocks()
  -> ShmemAttachRequested()
     -> shmem_request_state = SRS_ATTACHING
     -> LWLockAcquire(ShmemIndexLock, LW_SHARED)
     -> 对每个 pending request 执行 AttachShmemIndexEntry()
     -> 设置 options->ptr 指向 existing location
     -> 调用所有 attach_fn
     -> shmem_request_state = SRS_DONE
  -> shmem_startup_hook()
```

为什么 attach 要等到 `InitProcess()` 之后？因为 `ShmemAttachRequested()` 要拿 `ShmemIndexLock`，而 backend 要等到有了 `PGPROC`、初始化 LWLock access 后，才能安全参与 LWLock 等待和错误处理。`proc.c` 里的注释直接点出这个约束：`EXEC_BACKEND` 下不能在 `ShmemAttachRequested()` 之前使用依赖完整 shared pointers 的 `ProcArrayAdd()`。

这条路径说明：`EXEC_BACKEND` backend 启动早期可能已经“物理 attach 到 shared memory segment”，但多数本地全局指针还没接好。能否碰 shared memory，不只看 mapping 是否存在，还要看本进程是否已经完成 pointer wiring 和所需的同步基础设施。

### 5.4 同一个 BufferManager 例子贯穿三种进程

以 buffer manager 为例：

```text
postmaster request:
  BufferManagerShmemRequest()
    -> request "Buffer Descriptors"
    -> request "Buffer Blocks"
    -> request "Buffer IO Condition Variables"
    -> request "Checkpoint BufferIds"

postmaster init:
  ShmemInitRequested()
    -> 设置 BufferDescriptors / BufferBlocks / BufferIOCVArray / CkptBufferIds
    -> BufferManagerShmemInit()
       -> 初始化每个 BufferDesc
       -> 初始化每个 ConditionVariable
       -> 初始化 BackendWritebackContext

fork backend:
  继承 BufferDescriptors / BufferBlocks 等变量值
  不调用 BufferManagerShmemAttach()

EXEC_BACKEND backend:
  重新执行 BufferManagerShmemRequest()
  ShmemAttachRequested()
    -> 设置本进程的 BufferDescriptors / BufferBlocks 等变量值
    -> BufferManagerShmemAttach()
       -> 初始化 BackendWritebackContext
```

这里的重点不是 buffer manager 本身，而是职责分层：

| 动作 | 放在哪里 |
| --- | --- |
| 声明需要多少共享内存 | `request_fn` |
| 初始化所有 backend 共享的 buffer descriptor 状态 | `init_fn` |
| 初始化当前 backend 私有的 writeback context | `attach_fn`，如果 postmaster/standalone 也需要则 `init_fn` 也做 |

## 6. 生命周期 / ownership / cleanup

### 谁创建

postmaster 拥有传统 main shmem 的创建时机：

```text
Register callbacks
  -> collect requests
  -> calculate size
  -> create segment
  -> init requested areas
```

每个 shared memory area 的“创建者”不是具体子系统直接调用 `malloc()`，而是 shmem 框架在 `ShmemInitRequested()` 中统一创建并登记。子系统只通过 request callback 声明需求。

### 谁持有

shared memory object 的实际生命周期跟 main shmem segment 绑定。进程本地的全局指针只是 handle，不拥有对象。

```text
shared object owner: postmaster 创建的 main shmem segment 生命周期
process-local handle owner: 当前进程自己的全局变量 / HTAB handle / attach_fn 本地状态
```

因此普通 backend 退出时不会释放 `"Buffer Descriptors"`；它最多释放或清理自己的 process-local 状态。真正的 shared memory segment 在 postmaster 生命周期结束或异常清理路径中整体释放。

### 谁释放

request 阶段产生的 `pending_shmem_requests` 会在 init 或 attach 后释放：

```text
ShmemInitRequested()
  -> pfree(request->options)
  -> list_free_deep(pending_shmem_requests)

ShmemAttachRequested()
  -> pfree(request->options)
  -> list_free_deep(pending_shmem_requests)
```

shared memory allocation 本身没有 per-area release。课程第 2 节已经解释过：传统 main shmem allocator 是单调分配，退出时整体回收。

### ERROR / abort 时怎么办

postmaster 启动期 request/init 失败通常是 `FATAL` 或启动失败，不会产生“继续运行但少一块 shared state”的状态。`InitShmemIndexEntry()` 若创建了 `ShmemIndex` entry 后发现 raw allocation 失败，会移除刚插入的 entry 再报错，避免目录留下半初始化名字。

`EXEC_BACKEND` backend attach 失败则是当前子进程启动失败；已存在的 shared memory object 不由该 backend 清理。因为 attach 只是在当前进程重建本地指针，不拥有全局对象。

## 7. 正确性机制层次

### 状态机约束调用位置

`ShmemRequestStruct()` 会检查当前进程的 `shmem_request_state`。如果不在 `SRS_REQUESTING`，会报错：

```text
ShmemRequestStruct can only be called from a shmem_request callback
```

这个限制把“声明需求”集中到 request callback，使 postmaster sizing 和 `EXEC_BACKEND` 重新 attach 使用同一套注册逻辑。它防止代码只在某个局部启动路径里偷偷 request，导致不同进程看到不同 request set。

### `ShmemIndex` 是跨进程 attach 协议

postmaster init 时，`InitShmemIndexEntry()` 把 name、size、location 写入 `ShmemIndex`。`EXEC_BACKEND` attach 时，`AttachShmemIndexEntry()` 用同一个 name 查 entry，并校验 size。

这形成了一个简单但强的协议：

```text
name 必须一致
size 必须一致，除非显式使用 SHMEM_ATTACH_UNKNOWN_SIZE
kind 必须一致，否则 ptr/HTAB/SLRU attach 语义不匹配
```

size check 对 `EXEC_BACKEND` 很重要。它能发现“postmaster 和 backend 重新计算出的 request 不一致”，例如某个 request size 依赖了没有正确传递到子进程的配置或本地变量。

### callback 顺序是初始化依赖

内置子系统通过 `src/include/storage/subsystemlist.h` 注册。文件注释明确说某些顺序很重要。例如 LWLocks 排在最前面，因为其它 shmem init 代码可能会使用 LWLocks。

这不是漂亮的抽象层次，而是真实内核代码常见的启动期依赖图：在并发进程尚未启动前，很多锁不用于互斥，却仍然需要先完成数据结构初始化。

### LWLock 保护 attach 查找

`ShmemAttachRequested()` 会持有 `ShmemIndexLock` 的 shared mode 来遍历并 attach 所有 pending requests。late allocation 路径会使用 exclusive mode，因为可能插入新 entry。

postmaster `ShmemInitRequested()` 不需要并发锁，因为此时还没有其它 backend 并发访问这些新对象。这也是启动期代码常见的 correctness 边界：

```text
single-process bootstrap 可以省掉运行期互斥；
一旦进入 backend attach / late allocation，就必须回到正常锁协议。
```

## 8. 错误路径 / 异常路径 / fallback

### request 出现在错误阶段

如果代码在 request callback 之外调用 `ShmemRequestStruct()`，会触发状态机错误。这类错误在非 `EXEC_BACKEND` fork 模式下有时会被误判为“只是初始化顺序问题”，但真正的问题是：这段代码无法保证在 exec 后的 backend 中用同一套逻辑重建 request。

### `EXEC_BACKEND` 找不到 `ShmemIndex` entry

如果 backend 重新 request 了一个 postmaster 初始化时不存在的名字，`AttachShmemIndexEntry()` 会报错：

```text
could not find ShmemIndex entry for data structure "..."
```

这通常说明 postmaster 与 child 的 request set 不一致。常见原因包括：扩展没有通过 `shared_preload_libraries` 在 postmaster 阶段加载；request size 或 request name 依赖进程本地状态；只在 backend-local 初始化路径注册 callback。

### 同名但 size 不一致

如果 `ShmemIndex` 里已有 entry，但 backend request 的 size 不同，`AttachShmemIndexEntry()` 会报错：

```text
shared memory struct "..." was created with different size
```

`ProcGlobalShmemRequest()` 中 `"Fast-Path Lock Array"` 是一个特殊例子。postmaster 可以计算真实 size；`EXEC_BACKEND` backend 不继承 postmaster 那个计算结果，所以 attach 时使用 `SHMEM_ATTACH_UNKNOWN_SIZE`，避免把“当前进程无法重新计算”误判成错误。

这不是通用逃生门。`SHMEM_ATTACH_UNKNOWN_SIZE` 只适合 attach 阶段确实无法知道 size、但名字和对象结构由其它机制保证一致的场景。

### attach 太早

在 `EXEC_BACKEND` 下，子进程可能已经重新映射了 shared memory，但还不能访问所有 shared structures。`AttachSharedMemoryStructs()` 要等到 `InitProcess()` 分配 `PGPROC` 并初始化 LWLock access 后调用。太早 attach 会撞上 LWLock / process identity 尚未建立的问题。

### extension legacy hook 的边界

老接口仍然存在：

```text
shmem_request_hook -> RequestAddinShmemSpace()
shmem_startup_hook -> ShmemInitStruct()
```

这条路径没有 typed `init_fn` / `attach_fn`，调用者需要自己检查 `found` 并决定是否初始化。它还能工作，但更容易把“全局共享对象初始化”和“本进程 attach 后的本地状态初始化”混在一起。新代码应优先使用 `RegisterShmemCallbacks()`。

## 9. 成本、资源与跨模块传播

### 启动期成本

request/init/attach 主要不是 SQL hot path 成本，而是启动期和 backend fork/exec 成本：

| 路径 | 成本形态 |
| --- | --- |
| postmaster request | 遍历所有 callbacks，构造 `pending_shmem_requests`。 |
| postmaster init | 为每个 request 插入 `ShmemIndex`、切分内存、调用 init callback。 |
| fork backend | 几乎没有 shmem pointer wiring 成本，依赖 OS fork 继承。 |
| `EXEC_BACKEND` backend | 每个 backend 重新注册 callbacks、重新 request、遍历 pending list、查 `ShmemIndex`、调用 attach callback。 |

这就是为什么 `attach_fn` 应该轻量，且只做本进程必要工作。把昂贵的全局初始化放进 attach，会在 `EXEC_BACKEND` 下被每个 backend 重复执行。

### 资源传播

request 阶段会影响 shared memory 总量，进而影响：

```text
shared_memory_size
shared_memory_size_in_huge_pages
huge_pages_status
OS shared memory / mmap / semaphore 资源
```

`InitializeShmemGUCs()` 必须在 request 完成后执行，因为 runtime-computed GUC 依赖最终 size。legacy `process_shmem_requests()` 也必须早于 `ShmemCallRequestCallbacks()`，因为扩展可能 request named LWLock tranche，影响 LWLock array sizing。

### 跨模块依赖

本节主链路连接了多个基础设施：

| 模块 | 依赖关系 |
| --- | --- |
| postmaster | 控制 request/init 的全局顺序。 |
| preload library loader | 决定扩展是否有机会参与 postmaster sizing。 |
| LWLock | attach 和 late allocation 需要锁保护 `ShmemIndex`。 |
| PGPROC / ProcArray | `EXEC_BACKEND` attach 必须与 backend process identity 初始化排序。 |
| DSM | main shmem 先初始化，再启动 dynamic shared memory facilities。 |

## 10. 观测与诊断入口

### SQL 观察 allocation 是否存在

上一节用过的 `pg_shmem_allocations` 仍然是第一入口：

```sql
SELECT name, size, allocated_size
FROM pg_shmem_allocations
WHERE name IN (
  'Buffer Descriptors',
  'Buffer Blocks',
  'Proc Header',
  'Fast-Path Lock Array'
)
ORDER BY name;
```

能看到这些名字，说明 postmaster 已经在 init 阶段把 named allocation 放进了 `ShmemIndex`。但 SQL 看不到某个 backend 的 `BufferDescriptors` 全局变量是否已经接好；那是进程私有状态，只能通过 gdb、日志或断点推断。

### 查看是否为 `EXEC_BACKEND` 构建

当前源码有 runtime-computed GUC：

```sql
SHOW exec_backend;
```

如果返回 `on`，每个 backend 都会走重新 request / attach 的路径；如果返回 `off`，普通 backend 主要依赖 fork inheritance。

### 建议断点

调试 postmaster 初始化：

```text
break ShmemCallRequestCallbacks
break ShmemInitRequested
break InitShmemIndexEntry
break BufferManagerShmemInit
```

调试 `EXEC_BACKEND` attach：

```text
break SubPostmasterMain
break ShmemCallRequestCallbacks
break AttachSharedMemoryStructs
break ShmemAttachRequested
break AttachShmemIndexEntry
break BufferManagerShmemAttach
```

观察重点：

```text
pending_shmem_requests 的长度
request->options->name
request->options->ptr 指向哪个全局变量
*(request->options->ptr) 在 attach 前后是否从 NULL 变成 shared memory 地址
shmem_request_state 的变化
```

### 日志和错误信息

这类问题常见的可见错误不是“segmentation fault”，而是启动或 backend 创建失败：

```text
cannot request shared memory at this time
ShmemRequestStruct can only be called from a shmem_request callback
could not find ShmemIndex entry for data structure "..."
shared memory struct "..." was created with different size
```

如果错误只在 Windows 或 `--enable-exec-backend` 构建出现，优先怀疑代码依赖了 fork inheritance，而不是正确实现了 request/init/attach 三段式生命周期。

## 11. 常见误区

### 误区一：`request_fn` 负责初始化 shared memory

`request_fn` 只声明需求。此时 main shared memory segment 可能还没有创建。真正的 shared object 初始化必须放到 `init_fn` 或 legacy `shmem_startup_hook` 中。

### 误区二：普通 fork backend 不 attach，所以 attach 逻辑不重要

普通 fork backend 不调用 `ShmemAttachRequested()` 是优化路径，不是语义豁免。代码仍然必须能在 `EXEC_BACKEND` 下重新建立同一组指针。否则这段代码就是平台相关的。

### 误区三：`ptr` 是 shared memory 里的字段

`ptr` 指向的是本进程的 C 指针变量。框架通过它把 named allocation 的地址写回每个进程自己的全局变量。

### 误区四：`init_fn` 和 `attach_fn` 可以做同样的事

`init_fn` 是一次性全局初始化；`attach_fn` 是每个进程的本地接线和本地状态初始化。把共享对象清零放进 `attach_fn`，在 `EXEC_BACKEND` 下会破坏已有共享状态。

### 误区五：`shared_preload_libraries` 只是性能优化

对需要传统 main shmem 的扩展来说，`shared_preload_libraries` 往往是 correctness 条件。只有 postmaster sizing 前加载，扩展才有机会可靠 request 足够 shared memory，并让所有 backend 按同一套 callback 注册。

## 12. 课堂实验

### 实验一：从 SQL 反推 request/init 结果

1. 启动一个本地 PostgreSQL 实例。
2. 执行：

```sql
SELECT name, size, allocated_size
FROM pg_shmem_allocations
WHERE name LIKE 'Buffer%'
   OR name IN ('Proc Header', 'Fast-Path Lock Array')
ORDER BY name;
```

3. 回到源码，找到这些名字分别在哪个 `request_fn` 中出现。
4. 判断每个名字的 `.ptr` 写回了哪个全局变量。

目标：把 SQL 看到的 named allocation、`ShmemRequestStruct()` 调用、进程内全局指针三者连起来。

### 实验二：用断点看 postmaster 初始化顺序

在 debug build 上设置断点：

```text
ShmemCallRequestCallbacks
ShmemInitRequested
BufferManagerShmemRequest
BufferManagerShmemInit
```

观察：

```text
BufferManagerShmemRequest() 执行时 BufferDescriptors 是否已经可用？
ShmemInitRequested() 中 InitShmemIndexEntry() 何时设置 BufferDescriptors？
BufferManagerShmemInit() 进入时 BufferDescriptors 是否已经指向 shared memory？
```

预期结论：

```text
request_fn 只登记；
InitShmemIndexEntry() 写回 ptr；
init_fn 看到的是已经完成基础接线的 shared pointer。
```

### 实验三：构造一个错误阶段 request

在一个实验分支中，把某个 `ShmemRequestStruct()` 临时挪到非 request callback 的初始化函数里，然后启动服务器。

观察错误：

```text
ShmemRequestStruct can only be called from a shmem_request callback
```

回到源码解释：

```text
shmem_request_state != SRS_REQUESTING
```

讨论：为什么这个限制不只是 API 洁癖，而是为了保证 postmaster sizing 和 `EXEC_BACKEND` attach 使用同一套 request 协议？

### 实验四：阅读 `ProcGlobalShmemRequest()` 的特殊 size 处理

阅读 `src/backend/storage/lmgr/proc.c` 中 `"Fast-Path Lock Array"` 的 request：

```text
postmaster: size = CalculateFastPathLockShmemSize()
EXEC_BACKEND backend: size = SHMEM_ATTACH_UNKNOWN_SIZE
```

回答：

```text
为什么 backend 不能简单复用 postmaster 计算出的 FastPathLockArrayShmemSize？
为什么 "Proc Header" 注释说 ProcGlobal 在 EXEC_BACKEND 下需要特殊传播？
```

这个实验帮助理解：request/init/attach 生命周期不是孤立机制，它与 backend 启动早期能访问哪些 shared pointers 紧密耦合。

## 13. 讨论题

1. 如果没有 `attach_fn`，`EXEC_BACKEND` backend 如何初始化类似 `BackendWritebackContext` 这样的 per-backend 状态？把它放到 `init_fn` 会有什么问题？
2. 为什么 `request_fn` 里传的是 `.ptr = &GlobalPointer`，而不是直接返回一个地址？
3. `ShmemIndex` 已经有 name -> location，为什么还需要每个 backend 重新跑 `request_fn`？
4. 一个扩展只在 session preload 时加载，并希望使用传统 main shmem。它可能遇到哪些 sizing、late allocation 和 attach 问题？
5. `subsystemlist.h` 中 callback 顺序为什么属于 correctness 设计，而不只是代码组织？

## 14. 本节小结

本节要带走的模型：

```text
request 是跨进程一致的声明协议；
init 是 postmaster 对 shared object 的一次性初始化；
attach 是当前进程把本地指针和本地状态接到已有 shared object 上。
```

普通 fork backend 之所以看不到显式 attach，是因为 fork 继承了 postmaster 的进程私有指针值；`EXEC_BACKEND` 没有这个条件，所以必须重新注册 callback、重新收集 request，再通过 `ShmemIndex` 找回同一组 named areas。

可迁移的系统规律是：

```text
当系统把共享对象包装成普通全局指针时，
必须另有一套生命周期协议保证每个进程的本地 handle 都能被一致重建；
否则代码只是偶然依赖某一种进程创建模型。
```

下一节会继续沿着这条边界往外看：扩展如何接入这套生命周期，`shared_preload_libraries` 为什么重要，late allocation 的余量在哪里，以及什么时候传统 main shmem 已经不是合适工具。
