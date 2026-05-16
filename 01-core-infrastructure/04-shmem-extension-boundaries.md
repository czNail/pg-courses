# PostgreSQL Shmem 扩展与边界

## 课程定位

前置知识：已经理解 main shared memory segment 为什么必须启动期定容，`ShmemIndex` 如何绑定名字与地址，以及 request / init / attach 如何在 postmaster、fork backend 和 `EXEC_BACKEND` backend 中重建同一组 shared state 指针。

本节唯一主问题：

```text
扩展想使用跨进程共享状态时，哪些工作必须在 shared_preload_libraries 启动期完成，哪些可以 late allocation，什么时候应该转向 DSM？
```

核心矛盾：PostgreSQL 希望扩展能按需加载、独立演化、少侵入内核；但传统 shmem 是 postmaster 启动期一次性定容、全生命周期不释放、通常依赖 fork 继承指针的固定资源。

学完后应能判断：一个扩展需要共享状态时，应该使用 `RegisterShmemCallbacks()`、legacy `shmem_request_hook` / `shmem_startup_hook`、late allocation，还是 `GetNamedDSMSegment()` / `GetNamedDSA()` / 普通 DSM。

本课基于本地 `~/postgres` 源码，分支 `master`，提交 `8eba2edb8010`。

## 1. 本节在总主线中的位置

前三节已经建立了传统 shmem 的核心模型：

```text
先声明总大小
  -> postmaster 创建固定 main segment
  -> ShmemIndex 绑定名字
  -> 各进程通过 fork 或 attach 获得本进程指针
```

本节把视角从内置子系统移到扩展。扩展的问题不只是“如何分配一块 shared memory”，而是：

```text
扩展什么时候被加载？
它是否能参与启动期 sizing？
它需要的共享对象是否固定、长期、不可释放？
其它 backend 如何找到这个对象？
对象失败、退出、重连、postmaster crash 后谁清理？
```

如果答案是“固定、很早、集群生命周期内都在”，传统 shmem 很合适。如果答案是“按需、可变、可能释放、由某个任务或 session 创建”，继续使用传统 shmem 往往是在跟系统边界对抗。

下一节进入 broader infrastructure 时，这条边界会再次出现：MemoryContext、ResourceOwner、DSM/DSA 都是在回答同一个问题：谁拥有资源，什么时候能安全释放。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
shared_preload_libraries 让扩展在 postmaster sizing 前进入生命周期；
RegisterShmemCallbacks 是新式 request/init/attach 接口；
shmem_request_hook + shmem_startup_hook 是 legacy 两段式接口；
late allocation 只是在启动后尝试使用少量传统 shmem 余量；
DSM/DSA 才是按需、可释放、动态容量共享状态的边界外方案。
```

这里有两组边界需要分开：

| 机制 | 适合什么 | 不适合什么 |
| --- | --- | --- |
| `shared_preload_libraries` + `RegisterShmemCallbacks()` | 扩展有固定 shared state，需要在 postmaster 启动期纳入 sizing，并兼容 `EXEC_BACKEND` attach | 用户 session 临时需要的一块可变共享内存 |
| `shmem_request_hook` + `shmem_startup_hook` | 老扩展兼容路径，配合 `RequestAddinShmemSpace()` 和 `ShmemInitStruct()` | 新代码的首选生命周期抽象 |
| `SHMEM_CALLBACKS_ALLOW_AFTER_STARTUP` | 非预加载扩展启动后小规模注册传统 shmem，对容量不足可接受 | 大对象、强可靠资源、需要释放或频繁创建的对象 |
| DSM / DSA / DSM registry | 按需创建、可 detach、可由 ResourceOwner 管理、容量或生命周期不适合 main segment 的对象 | 需要在 shared struct 中存放普通 C 指针并被所有进程直接复用的固定内核状态 |

最容易出错的地方是把“能在启动后分配一点传统 shmem”误读成“传统 shmem 支持一般意义上的动态内存”。源码注释说得很直接：允许启动后小分配主要是照顾 extension module，而且余量很小，没有可用内存保证。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 作用 |
| --- | --- | --- |
| 1 | `src/backend/postmaster/postmaster.c` | postmaster 启动期把内置 shmem callback、`shared_preload_libraries`、legacy request hook、新式 request callback 串成一条 sizing 时间线。 |
| 2 | `src/backend/utils/init/miscinit.c` | `process_shared_preload_libraries()` 加载扩展，`process_shmem_requests()` 调用 legacy `shmem_request_hook`。 |
| 3 | `src/backend/storage/ipc/ipci.c` | `RequestAddinShmemSpace()`、`CreateSharedMemoryAndSemaphores()`、`AttachSharedMemoryStructs()` 和 `shmem_startup_hook` 的位置。 |
| 4 | `src/include/storage/shmem.h` | `RegisterShmemCallbacks()`、`SHMEM_CALLBACKS_ALLOW_AFTER_STARTUP`、legacy `ShmemInitStruct()` / `ShmemInitHash()` 的公开边界。 |
| 5 | `src/backend/storage/ipc/shmem.c` | late allocation、`CallShmemCallbacksAfterStartup()`、legacy `ShmemInitStruct()`、`pg_get_shmem_allocations()` 的真实实现。 |
| 6 | `src/backend/storage/lmgr/lwlock.c` | `RequestNamedLWLockTranche()` 为什么只能在 legacy request hook 阶段调用。 |
| 7 | `src/backend/storage/ipc/dsm.c` | DSM 的创建、attach、ResourceOwner、pin 和 postmaster cleanup 模型。 |
| 8 | `src/backend/storage/ipc/dsm_registry.c` | `GetNamedDSMSegment()` / `GetNamedDSA()` / `GetNamedDSHash()` 如何给扩展提供按名字查找的动态共享对象。 |
| 9 | `src/test/modules/test_shmem/test_shmem.c` | after-startup traditional shmem 的最小测试扩展示例。 |
| 10 | `src/test/modules/test_dsm_registry/test_dsm_registry.c` | DSM registry 的最小测试扩展示例。 |

阅读时建议不要先背 hook 名字，而是先追问同一件事：

```text
这个接口有没有参与 startup sizing？
它创建的是 main segment 里的永久对象，还是 DSM 里的动态对象？
其它进程靠 fork 继承、ShmemIndex、DSM handle，还是 registry 名字找到它？
```

## 4. 关键数据结构与状态

### `shared_preload_libraries`

`shared_preload_libraries` 不是性能优化开关，而是生命周期边界。

扩展出现在这个 GUC 中时，postmaster 会在计算 shared memory 大小之前加载它。扩展的 `_PG_init()` 因此可以做三类事情：

```text
注册新式 ShmemCallbacks
安装 legacy shmem_request_hook / shmem_startup_hook
注册必须在 postmaster 启动期确定的其它全局资源
```

例如一些测试模块会检查：

```text
if (!process_shared_preload_libraries_in_progress)
    return 或报错
```

这不是形式主义。它是在防止扩展错过 sizing 窗口后，还假装自己可以安全拿到固定 shmem。

### `shmem_request_hook`

`shmem_request_hook` 是 legacy 扩展入口，类型定义在 `miscadmin.h`。它由扩展在 `_PG_init()` 中安装，postmaster 后续通过 `process_shmem_requests()` 调用。

它的典型职责是：

```text
RequestAddinShmemSpace(size);
RequestNamedLWLockTranche(name, nlocks);
调用前一个 hook，维持 hook chain
```

`RequestAddinShmemSpace()` 只累计一段额外空间到 `total_addin_request`，它不会创建 named allocation，也不会设置扩展自己的指针。真正创建通常发生在 `shmem_startup_hook` 里。

### `shmem_startup_hook`

`shmem_startup_hook` 定义在 `storage/ipc.h`，全局变量在 `ipci.c` 中。

它的典型职责是：

```text
ShmemInitStruct(name, size, &found);
如果 !found，初始化 shared struct；
如果 found，只把当前进程的本地指针接上已有对象。
```

postmaster 创建 main segment 并完成内置 shmem 初始化后，会调用一次 `shmem_startup_hook`。在 `EXEC_BACKEND` 子进程中，`AttachSharedMemoryStructs()` 完成 `ShmemAttachRequested()` 后，也会调用 `shmem_startup_hook`，让 legacy 扩展重新建立本进程状态。

这就是 legacy 接口的 awkwardness：它没有显式 request/init/attach 三段 callback，只能靠 `found` 结果和调用者自律区分第一次初始化与后续 attach。

### `RegisterShmemCallbacks()` 与 `SHMEM_CALLBACKS_ALLOW_AFTER_STARTUP`

新式接口把生命周期拆成：

```text
request_fn -> init_fn -> attach_fn
```

扩展在 `_PG_init()` 中调用 `RegisterShmemCallbacks()`。如果发生在 postmaster 启动期，它会被记到 `registered_shmem_callbacks`，随后由 `ShmemCallRequestCallbacks()`、`ShmemInitRequested()` 或 `ShmemAttachRequested()` 消费。

如果发生在 backend 启动后，默认会报错：

```text
cannot request shared memory at this time
```

只有设置 `SHMEM_CALLBACKS_ALLOW_AFTER_STARTUP` 时，`RegisterShmemCallbacks()` 才会进入 `CallShmemCallbacksAfterStartup()`，立即执行 request，然后在 `ShmemIndexLock` 保护下选择 init 或 attach。

### DSM registry

DSM registry 是扩展走出传统 shmem 边界后的常见落点。

它自身只在 main shmem 里保存一个很小的 context：

```text
DSMRegistryCtx:
  DSA handle
  dshash handle
```

真正的动态目录在 DSM/DSA 中。`GetNamedDSMSegment()`、`GetNamedDSA()`、`GetNamedDSHash()` 负责：

```text
按名字查找 registry entry
不存在则创建并初始化
已存在则 attach 到当前 backend
保证只有一个 backend 做初始化
把 handle 存在 registry 中供其它 backend 复用
```

这跟 `ShmemIndex` 很像，但语义不同：`ShmemIndex` 管 main segment 中不可释放的传统 shmem allocation；DSM registry 管动态 shared memory 对象的名字到 handle 映射。

## 5. 主流程源码 walkthrough

### 5.1 预加载扩展的新式 shmem callback 路径

postmaster 启动期的关键顺序在 `PostmasterMain()` 附近：

```text
RegisterBuiltinShmemCallbacks()
  -> process_shared_preload_libraries()
  -> InitializeMaxBackends()
  -> InitPostmasterChildSlots()
  -> InitializeFastPathLocks()
  -> process_shmem_requests()
  -> ShmemCallRequestCallbacks()
  -> InitializeShmemGUCs()
  -> CreateSharedMemoryAndSemaphores()
```

扩展如果在 `_PG_init()` 里调用：

```text
RegisterShmemCallbacks(&MyCallbacks);
```

它会被追加到 `registered_shmem_callbacks`。此时还没有创建 shared memory segment，只是在 postmaster 进程私有内存里登记“将来要调用哪些 callback”。

随后 `ShmemCallRequestCallbacks()` 把状态切到 `SRS_REQUESTING`，依次调用每个 `request_fn`。扩展的 `request_fn` 调用：

```text
ShmemRequestStruct(.name = "...",
                   .size = ...,
                   .ptr = (void **) &MySharedPtr);
```

这一步仍然不是 allocation。它只是在 `pending_shmem_requests` 中记录：

```text
名字
大小
alignment
当前进程里要写回的指针变量地址
```

`CreateSharedMemoryAndSemaphores()` 之后才进入真正创建：

```text
CalculateShmemSize()
  -> ShmemGetRequestedSize()
  -> total_addin_request
PGSharedMemoryCreate()
InitShmemAllocator()
ShmemInitRequested()
  -> InitShmemIndexEntry()
  -> callbacks->init_fn()
dsm_postmaster_startup()
shmem_startup_hook()
```

这里要注意一个顺序细节：新式 `RegisterShmemCallbacks()` 的 `init_fn` 在 `ShmemInitRequested()` 中执行；legacy `shmem_startup_hook` 则在这之后执行。

因此新扩展优先使用 `RegisterShmemCallbacks()` 的好处是：它自然纳入 request/init/attach 状态机，也自然兼容 `EXEC_BACKEND`。

### 5.2 legacy hook 路径：request 只预留，startup 才绑定名字

老扩展通常在 `_PG_init()` 里做 hook chaining：

```text
prev_shmem_request_hook = shmem_request_hook;
shmem_request_hook = my_request_hook;

prev_shmem_startup_hook = shmem_startup_hook;
shmem_startup_hook = my_startup_hook;
```

request 阶段：

```text
process_shmem_requests()
  -> shmem_request_hook()
     -> RequestAddinShmemSpace(size)
     -> RequestNamedLWLockTranche(name, nlocks)
```

`RequestAddinShmemSpace()` 只允许在 `process_shmem_requests_in_progress` 为 true 时调用。如果扩展在普通 backend 中调用它，会直接 FATAL：

```text
cannot request additional shared memory outside shmem_request_hook
```

`RequestNamedLWLockTranche()` 有类似限制，因为 named LWLocks 需要进入 main LWLock array，并影响 LWLock sizing。

startup 阶段：

```text
CreateSharedMemoryAndSemaphores()
  -> ShmemInitRequested()
  -> dsm_postmaster_startup()
  -> shmem_startup_hook()
     -> ShmemInitStruct(name, size, &found)
```

`ShmemInitStruct()` 会在 `ShmemIndexLock` 下查找或创建 named entry。调用者必须检查 `found`：

```text
found == false:
  这是第一次创建，必须初始化 shared struct 内容

found == true:
  对象已经存在，本进程只 attach 到它
```

这就是 legacy hook 的主矛盾：request hook 只预留空间，startup hook 才做命名 allocation；框架不能替扩展区分 init 与 attach，也不能替扩展自动处理 `EXEC_BACKEND` 的 per-process attach callback。

### 5.3 late allocation：启动后立即 init 或 attach

`RegisterShmemCallbacks()` 有一条特殊分支：

```text
if (shmem_request_state == SRS_DONE && IsUnderPostmaster)
```

这表示当前已经是 postmaster child，也就是服务器启动后某个 backend 正在加载扩展。默认情况下，注册 shared memory callback 会报错。只有扩展显式设置：

```text
SHMEM_CALLBACKS_ALLOW_AFTER_STARTUP
```

才会进入 `CallShmemCallbacksAfterStartup()`。

这个函数的时间线是：

```text
SRS_DONE
  -> SRS_REQUESTING
     -> callbacks->request_fn()
  -> SRS_AFTER_STARTUP_ATTACH_OR_INIT
     -> LWLockAcquire(ShmemIndexLock, LW_EXCLUSIVE)
     -> 检查 requested areas 是全部存在，还是全部不存在
     -> 全部存在：AttachShmemIndexEntry()
     -> 全部不存在：InitShmemIndexEntry()
     -> callbacks->attach_fn() 或 callbacks->init_fn()
     -> LWLockRelease(ShmemIndexLock)
  -> SRS_DONE
```

它有一个重要不变量：

```text
同一个 callback 请求的一组 areas 必须全部已经存在，或者全部不存在。
```

如果一部分存在、一部分不存在，源码会报：

```text
found some but not all
```

原因是框架无法判断应该调用 `init_fn` 还是 `attach_fn`。对于扩展来说，这意味着 after-startup callback 应该把一组 shared areas 设计成同一个原子单元，不要把不相关对象塞进同一个 late allocation request。

`src/test/modules/test_shmem/test_shmem.c` 正好是这个路径的最小例子。它设置 `SHMEM_CALLBACKS_ALLOW_AFTER_STARTUP`，在 `_PG_init()` 中注册 callbacks。测试里不把它放进 `shared_preload_libraries`，而是在运行期 `CREATE EXTENSION test_shmem;`，再用新连接观察 attach callback 计数增加。

### 5.4 DSM registry：启动后按名字创建动态对象

如果扩展不想依赖 postmaster 启动期，或者对象生命周期不适合 main segment，可以走 DSM registry。

典型调用是：

```text
ptr = GetNamedDSMSegment("my extension state",
                         sizeof(MyState),
                         my_init_callback,
                         &found,
                         arg);
```

源码主链路在 `dsm_registry.c`：

```text
GetNamedDSMSegment()
  -> init_dsm_registry()
     -> 如果 registry dshash 尚未创建，创建 DSA + dshash，并 pin
     -> 否则 attach existing DSA + dshash
  -> dshash_find_or_insert(name, &found)
  -> 如果 entry 不存在：
       dsm_create(size, 0)
       init_callback(dsm_segment_address(seg), arg)
       dsm_pin_segment(seg)
       dsm_pin_mapping(seg)
       entry->handle = dsm_segment_handle(seg)
  -> 如果 entry 已存在：
       dsm_find_mapping(handle)
       不在当前进程则 dsm_attach(handle)
       dsm_pin_mapping(seg)
  -> 返回当前进程中的 mapping address
```

这个路径与 traditional shmem 的根本区别是：

```text
traditional shmem:
  name -> ShmemIndex -> main segment address
  lifetime: postmaster main segment lifetime
  pointer: main segment 在所有进程同地址映射，shared struct 里可用普通指针

DSM registry:
  name -> registry dshash -> DSM handle -> 当前进程 mapping
  lifetime: DSM refcount / pin / postmaster cleanup
  pointer: 不保证各进程同地址映射，shared struct 内需要 offset、handle 或 DSA pointer
```

`src/test/modules/test_dsm_registry/test_dsm_registry.c` 演示了按需创建 DSM、DSA、dshash。SQL 测试先查询 `pg_dsm_registry_allocations` 为空，调用扩展函数后再查询，能看到 `segment`、`area`、`hash` 三类 registry entry。

## 6. 生命周期 / ownership / cleanup

### traditional shmem 生命周期

传统 shmem 的 owner 是 postmaster 创建的 main shared memory segment：

```text
创建：postmaster 启动期 PGSharedMemoryCreate()
命名：ShmemIndex entry
访问：所有 backend 通过 fork 继承或 attach 重建本地指针
释放：没有单对象 free；postmaster 生命周期结束时整个 segment 消失
```

extension late allocation 虽然发生在 backend 中，但对象仍然属于 main segment。它不是 backend-private，也不会在事务 abort、session exit 或 extension drop 时释放。

因此 traditional shmem 适合：

```text
集群生命周期级别的固定控制块
固定大小或可保守上限的 hash table
所有 backend 都可能访问的轻量全局状态
需要普通 C 指针在 shared struct 内可用的对象
```

不适合：

```text
每次任务创建一块共享状态
大小与 workload 强相关且启动期无法估计
需要释放、缩容、重建的对象
只被少数 backend 临时共享的数据通道
```

### legacy hook 生命周期

legacy 扩展需要自己维护两段 hook：

```text
request hook:
  预留 size 和 named LWLock tranche

startup hook:
  ShmemInitStruct()/ShmemInitHash()
  found=false 时初始化
  found=true 时 attach
```

ERROR 语义也比较粗糙：request 阶段错过窗口会 FATAL；startup 阶段 allocation 失败通常是 shared memory 不足；扩展自身初始化失败会中断 postmaster 启动或当前 backend attach。

### late allocation 生命周期

late allocation 的 owner 仍然是 main segment，但触发者是某个已启动 backend。

它的 cleanup 边界是：

```text
pending_shmem_requests:
  request 后在当前进程私有内存中短暂存在，init/attach 后释放

ShmemIndex entry + allocation:
  一旦成功进入 main segment，就没有单对象释放

当前 backend 的 pointer / HTAB handle:
  由 init_fn 或 attach_fn 建立，进程退出后消失
```

这就解释了为什么 late allocation 不能作为通用动态资源管理器：它没有 per-object reclaim，也不能把失败后的部分初始化随意留给其它 backend 猜。

### DSM 生命周期

DSM 的 owner 模型更细：

```text
mapping:
  当前 backend 的 dsm_segment descriptor，通常受 ResourceOwner 管理

segment:
  DSM control segment 中的 handle、refcnt、pinned 标志

cleanup:
  ResourceOwner cleanup detach mapping
  refcnt 降到销毁状态时清理 segment
  postmaster shutdown 或 crash restart 清理遗留 DSM
```

`dsm_pin_mapping()` 会把 mapping 从当前 ResourceOwner 中移出，让它保持到 session 生命周期；`dsm_pin_segment()` 则让 segment 即使没有普通 backend attach，也能保留到 postmaster shutdown 或显式 unpin。

DSM registry 使用了 pin，因为它要提供“按名字找到长期动态对象”的语义。

## 7. 正确性机制层次

### 启动期 ordering

`shared_preload_libraries` 必须早于 sizing：

```text
process_shared_preload_libraries()
  before
process_shmem_requests()
  before
ShmemCallRequestCallbacks()
  before
CalculateShmemSize()
```

这个顺序保证扩展的固定需求能被纳入 `shared_memory_size`、huge pages、OS shared memory allocation 和 main segment layout。

### hook chaining

legacy hook 是单个全局函数指针，不是 list。扩展必须保存 previous hook 并调用它：

```text
prev = shmem_request_hook;
shmem_request_hook = my_hook;

my_hook():
  if (prev)
      prev();
  my_request();
```

忘记 hook chain 会让其它扩展的 request 消失。这类 bug 很难从 SQL 层直接看出，通常表现为启动后某个扩展缺少 shared memory 或 LWLock tranche。

### ShmemIndexLock

启动期 `ShmemInitRequested()` 不需要锁，因为还没有并发 backend。

late allocation 和 legacy `ShmemInitStruct()` 则必须使用 `ShmemIndexLock`，因为多个 backend 可能同时尝试 attach 或第一次创建同名对象。

`CallShmemCallbacksAfterStartup()` 使用 exclusive lock，是因为它可能创建新的 `ShmemIndex` entry 和更新 allocator free offset。

### all-or-none late allocation

after-startup callbacks 的 all-or-none 检查是为了保护 callback 语义：

```text
全部不存在 -> init_fn
全部存在   -> attach_fn
部分存在   -> ERROR
```

这个机制不保证扩展内部状态一定正确，只保证 shmem 框架不会在“半初始化单元”上猜测生命周期。

### DSM control lock 与 refcount

DSM 使用 `DynamicSharedMemoryControlLock` 保护 control segment。`dsm_create()`、`dsm_attach()`、`dsm_detach()` 都围绕 control slot、refcnt 和 pinned 标志维护生命周期。

与 traditional shmem 相比，DSM 多了 cleanup 能力，也多了运行期锁、handle、mapping、refcount 的成本。

## 8. 错误路径 / 异常路径 / fallback

### 错过 request 窗口

如果扩展在普通 backend 中调用：

```text
RequestAddinShmemSpace()
RequestNamedLWLockTranche()
```

会报错，因为 `process_shmem_requests_in_progress` 不为 true。它们只能在由 `shared_preload_libraries` 加载的扩展的 `shmem_request_hook` 中调用。

这类资源影响 startup sizing，错过窗口后不能补救。

### 启动后注册传统 shmem callback

不带 `SHMEM_CALLBACKS_ALLOW_AFTER_STARTUP` 的 `RegisterShmemCallbacks()` 在启动后会报：

```text
cannot request shared memory at this time
```

带了 flag 也不代表一定成功。失败可能来自：

```text
ShmemIndex 没有可用 entry 空间
main segment 余量不足
同一组 request 中部分存在、部分不存在
size/name/alignment 不匹配
```

### `EXEC_BACKEND` 暴露隐藏依赖

普通 fork backend 继承 postmaster 中已经设置好的扩展指针。`EXEC_BACKEND` backend 需要重新加载 `shared_preload_libraries`，重新注册 callbacks，然后通过 `ShmemAttachRequested()` 或 `shmem_startup_hook` 重建本进程指针。

如果扩展把初始化逻辑写成“只在 postmaster `_PG_init()` 中设置一个 backend-local 静态变量”，在 fork 构建中可能看起来正常，在 `EXEC_BACKEND` 构建中就会暴露。

### DSM attach 失败与 max segments

DSM 不是无限资源。`dsm_create()` 需要 control slot，也可能受 `dynamic_shared_memory_type`、OS 资源、`min_dynamic_shared_memory` 预分配区、segment 数量限制影响。

如果达到 control segment 能支持的最大 segment 数，普通 `dsm_create()` 会报：

```text
too many dynamic shared memory segments
```

调用者使用 `DSM_CREATE_NULL_IF_MAXSEGMENTS` 时，可以选择返回 NULL 作为 fallback 信号。

### 指针语义 fallback

traditional shmem 中 main segment 必须在各进程映射到相同地址，所以 shared struct 中可以放普通指针。

DSM 不保证这一点。跨进程 DSM 数据结构如果需要引用内部对象，应使用：

```text
offset
handle
DSA pointer
shm_toc key
dshash key
```

不能把当前进程的裸指针写进 DSM 后期待其它 backend 可用。

## 9. 成本、资源与跨模块传播

### fixed shmem 成本

传统 shmem 的成本在启动期支付：

```text
shared_memory_size 增大
huge page 需求变化
OS shared memory allocation 变大
postmaster startup 更容易因为资源不足失败
所有 backend 都映射同一个 main segment
```

好处是运行期访问便宜：通常就是全局指针加普通 load/store，再由扩展自己选择 LWLock、spinlock、atomic 或 condition variable 做并发控制。

### legacy addin request 成本

`RequestAddinShmemSpace()` 把 size 加进 `total_addin_request`。它不会出现在 `ShmemIndex` 中，也不会告诉你这段预留最终被哪个扩展消耗。

因此从 observability 看，legacy 预留和匿名 allocation 更难诊断。`pg_shmem_allocations` 中会有 `<anonymous>` 统计未通过 named ShmemIndex 归属的空间。

### named LWLock tranche 成本

`RequestNamedLWLockTranche()` 会影响 main LWLock array。它必须在 `ShmemCallRequestCallbacks()` 前完成，因为 LWLock 子系统自己的 sizing 需要知道这些 extension tranche。

这个顺序解释了 postmaster 中为什么先调用 legacy `process_shmem_requests()`，再调用新式 `ShmemCallRequestCallbacks()`。

### DSM 成本

DSM 的成本在运行期支付：

```text
control segment slot
DynamicSharedMemoryControlLock
OS mapping / unmapping
ResourceOwner tracking
refcount / pin 管理
跨进程 handle 传递或 registry lookup
```

好处是：

```text
不要求 postmaster 启动期知道所有大小
可以按任务、session、扩展函数调用动态创建
可以 detach 或随 ResourceOwner cleanup
可以用 DSA 承载动态增长的数据结构
```

### 选择规则

一个实用判断：

| 问题 | 更可能选择 |
| --- | --- |
| 每个集群固定一个小控制块，所有 backend 长期访问 | `shared_preload_libraries` + `RegisterShmemCallbacks()` |
| 需要兼容老扩展，已有 `ShmemInitStruct()` 代码 | `shmem_request_hook` + `shmem_startup_hook` |
| 非预加载扩展只需很小、可失败的传统 shmem 状态 | `SHMEM_CALLBACKS_ALLOW_AFTER_STARTUP` |
| 运行期按名字创建共享对象，多个 backend 复用 | DSM registry |
| 对象大小会增长、需要 allocator、需要释放或长期动态结构 | DSA / dshash |
| query 或 background worker 之间传递一组临时状态 | DSM + `shm_toc` / `shm_mq` |

## 10. 观测与诊断入口

### 传统 shmem allocation

查看 main shmem 中的 named allocation：

```sql
SELECT name, size, allocated_size
FROM pg_shmem_allocations
ORDER BY allocated_size DESC
LIMIT 20;
```

观察某个扩展是否进入 traditional shmem，可以查它的 named area：

```sql
SELECT *
FROM pg_shmem_allocations
WHERE name ILIKE '%test_shmem%';
```

注意：legacy `RequestAddinShmemSpace()` 只是预留总空间，不一定有清晰的 named entry。

### DSM registry allocation

查看 registry 中的动态对象：

```sql
SELECT name, type, size
FROM pg_dsm_registry_allocations
ORDER BY name;
```

`test_dsm_registry` 的回归测试展示了一个可观测现象：

```text
CREATE EXTENSION 前：
  test_dsm_registry% entries 为 0

调用 set_val_in_shmem()/set_val_in_hash() 后：
  registry 中出现 segment、area、hash
```

这正是 DSM registry 与 startup shmem 的区别：对象可以在扩展函数第一次使用时出现。

### 日志与断点

诊断启动期扩展 shmem 时，可以在这些位置打断点：

```text
process_shared_preload_libraries()
process_shmem_requests()
ShmemCallRequestCallbacks()
RequestAddinShmemSpace()
RequestNamedLWLockTranche()
CreateSharedMemoryAndSemaphores()
ShmemInitRequested()
```

诊断 late allocation：

```text
RegisterShmemCallbacks()
CallShmemCallbacksAfterStartup()
InitShmemIndexEntry()
AttachShmemIndexEntry()
```

诊断 DSM registry：

```text
GetNamedDSMSegment()
GetNamedDSA()
GetNamedDSHash()
dsm_create()
dsm_attach()
dsm_pin_segment()
```

如果怀疑 `EXEC_BACKEND` 相关问题，可以在测试环境查看：

```sql
SHOW debug_exec_backend;
```

不同构建和配置下，预加载扩展的 attach callback 是否在每个 backend 调用会不同。`src/test/modules/test_shmem/t/001_late_shmem_alloc.pl` 专门覆盖了这个差异。

## 11. 常见误区

误区一：`shared_preload_libraries` 只是让扩展更早加载。

更准确地说，它让扩展进入 postmaster sizing 生命周期。没有这个时机，很多固定 shared memory 和 named LWLock tranche 根本不能正确声明。

误区二：`RequestAddinShmemSpace()` 分配了 shared memory。

它只增加总大小估算。真正 named allocation 通常发生在 `shmem_startup_hook` 里的 `ShmemInitStruct()`。

误区三：`shmem_startup_hook` 只在 postmaster 调一次。

在 `EXEC_BACKEND` 子进程中，`AttachSharedMemoryStructs()` 完成新式 attach 后也会调用它，让 legacy 扩展重建本进程状态。

误区四：late allocation 意味着 traditional shmem 可以动态管理。

late allocation 只是在 main segment 中尝试拿一块启动期留下的余量。成功后仍然不可释放，没有容量保证，也不适合频繁创建对象。

误区五：DSM 比 traditional shmem 更高级，所以都该用 DSM。

DSM 是更动态，不是更便宜。固定、长期、所有 backend 高频访问的内核控制块，traditional shmem 的全局指针模型仍然更直接。

误区六：DSM 中可以随便存普通指针。

DSM mapping 地址不保证各进程一致。跨进程引用必须用 offset、DSA pointer、handle 或 TOC/key 语义。

误区七：只要扩展函数能 `CREATE EXTENSION` 成功，共享状态就一定在所有 backend 正确 attach。

不一定。fork、`EXEC_BACKEND`、预加载、late allocation、DSM registry 的 attach 路径不同。要用新连接、重连、并发调用和 `pg_shmem_allocations` / `pg_dsm_registry_allocations` 验证。

## 12. 课堂实验

### 实验一：追踪预加载扩展的启动期窗口

阅读 `postmaster.c` 中 `process_shared_preload_libraries()` 到 `CreateSharedMemoryAndSemaphores()` 的顺序，回答：

```text
为什么 process_shmem_requests() 必须早于 ShmemCallRequestCallbacks()？
如果 RequestNamedLWLockTranche() 晚于 LWLock sizing，会破坏什么？
```

建议断点：

```text
process_shared_preload_libraries
process_shmem_requests
RequestNamedLWLockTranche
ShmemCallRequestCallbacks
LWLockShmemRequest
```

### 实验二：观察 late traditional shmem

阅读：

```text
src/test/modules/test_shmem/test_shmem.c
src/test/modules/test_shmem/t/001_late_shmem_alloc.pl
```

重点观察：

```text
SHMEM_CALLBACKS_ALLOW_AFTER_STARTUP
get_test_shmem_attach_count()
新连接中 attach_count 是否增加
debug_exec_backend 对预加载路径的影响
```

思考：为什么非 `EXEC_BACKEND` 且预加载时 attach callback 可能不在每个 backend 中调用？

### 实验三：把 legacy hook 翻译成新式 callback

阅读一个使用 `shmem_request_hook` 的例子：

```text
src/test/modules/test_lwlock_tranches/test_lwlock_tranches.c
```

尝试拆分：

```text
哪些逻辑是 sizing request？
哪些逻辑是 shared object init？
哪些逻辑需要每个 backend attach？
哪些逻辑因为涉及 named LWLock tranche 仍必须留在 legacy request hook？
```

这个实验的目的不是机械迁移，而是识别 legacy API 中混在一起的生命周期。

### 实验四：观察 DSM registry 的按需创建

阅读：

```text
src/test/modules/test_dsm_registry/test_dsm_registry.c
src/test/modules/test_dsm_registry/sql/test_dsm_registry.sql
```

执行思路：

```sql
SELECT name, type, size > 0 AS size_ok
FROM pg_dsm_registry_allocations
WHERE name LIKE 'test_dsm_registry%'
ORDER BY name;

CREATE EXTENSION test_dsm_registry;
SELECT set_val_in_shmem(1236);
SELECT set_val_in_hash('test', '1414');

\c

SELECT get_val_in_shmem();
SELECT get_val_in_hash('test');
```

解释为什么重连后还能读到值：registry 里保存的是 DSM/DSA/dshash handle，当前 backend 会重新 attach。

### 实验五：设计题

给一个扩展需求：

```text
需要记录最近 10 万个跨 backend 事件；
事件大小不固定；
希望 CREATE EXTENSION 后第一次使用时才分配；
希望重启后清空；
希望多个 backend 并发写入。
```

判断 traditional shmem、late allocation、DSM registry、DSA/dshash 分别承担什么角色。说明为什么不应该把所有事件都塞进 startup fixed shmem。

## 13. 讨论题

1. 为什么 PostgreSQL 仍然保留 `shmem_request_hook` / `shmem_startup_hook`，而不是要求所有扩展立刻迁移到 `RegisterShmemCallbacks()`？
2. late allocation 已经能在启动后创建 traditional shmem，为什么源码仍然建议需要 shared memory 的扩展加入 `shared_preload_libraries`？
3. DSM registry 和 `ShmemIndex` 都是“名字 -> 共享对象”的目录，它们的生命周期和指针语义有什么根本不同？
4. 一个扩展如果需要等待事件名字可见，什么时候用 `RequestNamedLWLockTranche()`，什么时候用 `LWLockNewTrancheId()`？
5. 如果一个 shared object 必须被所有 backend 高频访问，但大小依赖 GUC，为什么把 GUC 设为 `PGC_POSTMASTER` 往往比 DSM 更合理？
6. 在 `EXEC_BACKEND` 下，为什么“函数指针、全局变量、动态库加载地址”不能被当作 postmaster 到 backend 的隐式共享状态？

## 14. 本节小结

本节的可迁移模型是：

```text
传统 shmem 是 startup-sized, cluster-lifetime, same-address, no-free 的固定共享状态；
扩展只有进入 shared_preload_libraries，才能可靠参与这个 sizing 生命周期；
legacy hook 把 request 和 startup init 分散在两个全局 hook 中，新式 ShmemCallbacks 把 request/init/attach 显式化；
late allocation 是小余量兼容机制，不是通用动态内存管理；
DSM/DSA/DSM registry 是按需、可 attach、可清理或可增长共享对象的边界外工具。
```

遇到扩展共享状态设计时，先问四个问题：

```text
对象是不是集群生命周期固定存在？
大小能否在 postmaster 启动期可靠估算？
是否需要释放、重建或动态增长？
其它 backend 如何重新找到并 attach？
```

如果前两个答案是“是”，后两个答案是“否 / 通过 fork 或 ShmemIndex 即可”，traditional shmem 很合适。只要答案开始变成“运行期才知道、按需创建、需要释放、需要动态结构”，就应该转向 DSM/DSA，而不是继续把传统 shmem 当成 malloc。
