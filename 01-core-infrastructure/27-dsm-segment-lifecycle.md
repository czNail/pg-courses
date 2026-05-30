# PostgreSQL DSM segment 创建、attach 与 refcount 生命周期

## 课程定位

前置知识：已经理解 main shared memory 的启动期定容、`ShmemIndex` 的命名绑定、`ResourceOwner` 的 ERROR-safe cleanup，以及 LWLock / spinlock / latch 的基本边界。

本节唯一主问题：

```text
为什么 PostgreSQL 需要运行期创建的共享内存段，dsm_create() / dsm_attach() / dsm_detach()
如何用 control segment、handle、refcount 和 ResourceOwner 在“跨进程可见”与
“ERROR-safe cleanup”之间取得平衡？
```

核心矛盾：并行执行、逻辑复制 parallel apply、DSA、扩展和临时协作状态都需要“现在才知道大小、用完就释放”的跨进程内存；但一块共享内存一旦把 handle 交给其它 backend，就必须防止早释放、悬空映射、ERROR 泄漏和 postmaster crash 后遗留对象。

学完后应能判断：

```text
什么时候 main shmem 不够，应该创建 DSM；
为什么 DSM handle 可以跨进程传递，但 mapped_address 不能；
为什么 refcnt = 2 表示只有一个普通 mapping；
为什么 refcnt = 1 是 moribund，不允许新 attach；
为什么 dsm_detach() 要先 unmap，再减少 refcount；
为什么 DSM mapping 默认挂 ResourceOwner，而 segment 生命周期还需要 control segment 兜底。
```

本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `2c4bd2bf5700`。

## 1. 本节在总主线中的位置

前面几节已经建立了三条基础线：

```text
main shmem:
  启动期定容，适合所有 backend 长期共享的固定结构。

MemoryContext:
  管 backend-local 内存生命周期。

ResourceOwner:
  管 buffer pin、lock、DSM mapping 这类“不是普通内存”的外部资源。
```

本节进入第四类共享状态：

```text
运行期才创建、多个 backend 临时共享、用完可以销毁的 shared memory segment。
```

这一类对象的典型使用现场是 parallel query：

```text
leader:
  dsm_create()
  -> 在 DSM 里放 shm_toc、shm_mq、序列化状态
  -> 把 dsm_handle 作为 bgw_main_arg 传给 worker

worker:
  dsm_attach(handle)
  -> 通过 shm_toc 找到共享状态
  -> 工作结束或进程退出时 dsm_detach()
```

下一节会专门讨论 pin、detach callback 和长期共享状态边界。本节只讲 DSM segment 最核心的创建、attach、detach、refcount 生命周期。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
postmaster 创建一个 DSM control segment；
每个普通 DSM segment 在 control segment 中占一个 slot；
slot 用 handle 标识对象，用 refcnt 记录 mapping/pin 生命周期；
每个 backend 用 backend-local dsm_segment 描述自己的 mapping；
ResourceOwner 负责 ERROR 时自动 dsm_detach()；
最后一个 mapping detach 后，refcnt 从 2 降到 1，触发 destroy，成功后 slot 变成 0 可复用。
```

本节 tension 是：

```text
运行期灵活创建和销毁共享内存
  vs
跨进程 handle 已经传播后，不能让任何 backend attach 到半死不活或已经重用的 segment
```

PostgreSQL 的答案不是把 DSM 当成“共享版 malloc”。它把问题拆成两层：

```text
control segment:
  记录全局可见的 segment handle、refcount、slot 状态。

backend-local dsm_segment:
  记录当前 backend 的 mapping 地址、大小、ResourceOwner、detach callback。
```

所以 DSM 的语义不是一个字段决定的：

```text
handle + control_slot + refcnt + mapped_address + ResourceOwner
  才共同表示“当前 backend 正在安全使用这个 DSM segment”。
```

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 作用 |
| --- | --- | --- |
| 1 | `src/include/storage/dsm.h` | 对外暴露 `dsm_create()`、`dsm_attach()`、`dsm_detach()`、pin 和 detach callback 接口。 |
| 2 | `src/backend/storage/ipc/dsm.c` | DSM 高层生命周期：control segment、refcount、ResourceOwner、backend-local descriptor、cleanup。 |
| 3 | `src/include/storage/dsm_impl.h` | 定义低层 implementation API、`dsm_handle`、`DSM_OP_CREATE/ATTACH/DETACH/DESTROY`。 |
| 4 | `src/backend/storage/ipc/dsm_impl.c` | POSIX / SysV / Windows / mmap 等平台实现；高层 DSM 不直接依赖某一种 OS primitive。 |
| 5 | `src/include/storage/pg_shmem.h` | `PGShmemHeader.dsm_control` 保存 control segment handle，供重启清理和 EXEC_BACKEND attach 使用。 |
| 6 | `src/backend/port/sysv_shmem.c`、`src/backend/port/win32_shmem.c` | postmaster 如何把旧 control segment handle 用于 crash 后 cleanup，backend 如何恢复 control handle。 |
| 7 | `src/backend/access/transam/parallel.c` | 典型调用者：leader 创建 DSM，worker 用 handle attach，结束时 detach。 |
| 8 | `src/backend/access/common/session.c`、`src/backend/storage/ipc/dsm_registry.c`、`src/backend/utils/mmgr/dsa.c` | 其它调用者：per-session DSM、named DSM / DSA、DSA 多 segment 管理。 |

推荐阅读顺序：

```text
dsm.h 接口语义
  -> dsm.c 顶部结构体
  -> dsm_postmaster_startup()
  -> dsm_create()
  -> dsm_attach()
  -> dsm_detach()
  -> ResourceOwner callback
  -> parallel.c 中的实际调用链
```

不要从 `dsm_impl.c` 开始。OS primitive 很容易把注意力带到 `shm_open()`、`shmget()` 或 `mmap()` 细节上，但 DSM 的核心抽象在 `dsm.c`：谁能发现 segment、谁持有引用、谁能销毁。

## 4. 关键数据结构与状态

### 4.1 `dsm_control_header`：全局目录

`dsm_control_header` 存在于独立的 DSM control segment 中。它不是某个普通 backend 的私有状态，而是所有能使用 DSM 的进程共同访问的目录：

| 字段 | 语义 |
| --- | --- |
| `magic` | 判断 control segment 是否像当前版本期望的 DSM control segment。 |
| `nitems` | 已经使用过的 slot 上界；slot 可能因为 `refcnt = 0` 可复用。 |
| `maxitems` | control segment 能记录的最大 DSM segment 数。当前由固定 slot 加 `MaxBackends` 比例计算。 |
| `item[]` | 每个普通 DSM segment 的全局状态。 |

control segment 自己不走普通 `dsm_segment` descriptor，也不走 refcount。源码注释直接说明：control segment 持续整个 postmaster 生命周期。

### 4.2 `dsm_control_item`：segment 的 shared lifecycle state

每个 `dsm_control_item` 是一个普通 DSM segment 在全局目录里的状态：

| 字段 | 语义 |
| --- | --- |
| `handle` | 可以跨进程传递的 DSM ID。普通 external segment 使用偶数；preallocated main region pseudo-segment 使用奇数。 |
| `refcnt` | 生命周期核心状态：`2+` active，`1` moribund，`0` unused。 |
| `first_page` / `npages` | 如果 DSM 来自 `min_dynamic_shared_memory` 预留的 main shmem 区域，记录其页范围。 |
| `impl_private_pm_handle` | 某些平台 pin segment 到 postmaster shutdown 需要的 implementation-private 状态。 |
| `pinned` | segment 是否被额外 pin 住，避免没有普通 mapping 时立刻销毁。下一节展开。 |

最重要的是 `refcnt` 的三态：

```text
refcnt = 0:
  slot 未使用，可以复用。

refcnt = 1:
  segment 已经没有普通 mapping，但还没有完成 destroy；
  这是 moribund 状态，dsm_attach() 必须跳过。

refcnt >= 2:
  segment active；
  普通 mapping 数 = refcnt - 1 - pin引用数。
```

为什么创建时 `refcnt = 2`，而不是 1？

```text
因为 1 被保留为“最后一个 mapping 已经离开，destroy 正在发生或将要发生”。
如果只有一个 backend attach，slot 必须仍然是 active，所以用 2 表示一个普通 mapping。
```

### 4.3 `dsm_segment`：backend-local mapping descriptor

`dsm_segment` 不是共享结构。每个 backend attach 同一个 handle 时，都会有自己的 `dsm_segment`：

| 字段 | 语义 |
| --- | --- |
| `resowner` | 当前 mapping 是否挂在 `CurrentResourceOwner` 下；为 `NULL` 表示 session lifespan。 |
| `handle` | 对应 control slot 的全局 handle。 |
| `control_slot` | 当前 mapping 已经在 control segment 中占到的 slot；`INVALID_CONTROL_SLOT` 表示不需要降 refcount。 |
| `impl_private` | OS implementation 的 per-backend 私有状态。 |
| `mapped_address` / `mapped_size` | 当前 backend 的虚拟地址和映射长度；不能跨进程传递。 |
| `on_detach` | detach 前要执行的 callback 链。 |

这里的边界非常关键：

```text
dsm_handle:
  可以跨进程传递，用于找到同一个 DSM segment。

mapped_address:
  只对当前 backend 有意义，不能写进 DSM 后让别的 backend 解引用。
```

这也是下一节 `shm_toc` 存 offset、不存裸指针的前提。

## 5. 主流程源码 walkthrough

### 5.1 postmaster 创建 control segment

DSM 系统启动发生在 postmaster 生命周期早期：

```text
dsm_postmaster_startup(PGShmemHeader *shim)
  -> 如果 dynamic_shared_memory_type = mmap，先清理 pg_dynshmem 旧文件
  -> maxitems = PG_DYNSHMEM_FIXED_SLOTS
               + PG_DYNSHMEM_SLOTS_PER_BACKEND * MaxBackends
  -> 随机生成 dsm_control_handle，避开 DSM_HANDLE_INVALID
  -> dsm_impl_op(DSM_OP_CREATE, dsm_control_handle, segsize, ...)
  -> shim->dsm_control = dsm_control_handle
  -> 初始化 magic / nitems / maxitems
  -> 注册 dsm_postmaster_shutdown()
```

control segment handle 被写入 `PGShmemHeader.dsm_control`，有两个作用：

```text
EXEC_BACKEND:
  子进程重新 attach main shmem 后，可以通过 header 找回 control segment handle。

crash cleanup:
  新 postmaster 可以尝试 attach 旧 control segment，遍历旧 item[]，销毁遗留 DSM。
```

### 5.2 leader 创建普通 DSM segment

典型调用者在 `parallel.c`：

```text
InitializeParallelDSM(pcxt)
  -> segsize = shm_toc_estimate(&pcxt->estimator)
  -> pcxt->seg = dsm_create(segsize, DSM_CREATE_NULL_IF_MAXSEGMENTS)
  -> pcxt->toc = shm_toc_create(..., dsm_segment_address(pcxt->seg), ...)
```

`dsm_create()` 的核心路径：

```text
dsm_create(size, flags)
  -> 如果本 backend 尚未初始化 DSM，dsm_backend_startup()
  -> dsm_create_descriptor()
     -> 在 TopMemoryContext 分配 backend-local dsm_segment
     -> 加入 dsm_segment_list
     -> 如果 CurrentResourceOwner 非 NULL，ResourceOwnerRememberDSM()

  -> 优先尝试从 min_dynamic_shared_memory 预留区域切一块
     -> 成功则使用 main region pseudo-handle

  -> 否则循环生成偶数 handle
     -> dsm_impl_op(DSM_OP_CREATE, handle, size, ...)

  -> 持有 DynamicSharedMemoryControlLock
     -> 找 refcnt = 0 的旧 slot，或者使用新 slot
     -> 写 handle / pinned = false / impl_private_pm_handle = NULL
     -> refcnt = 2
     -> seg->control_slot = slot
```

这里有两个 state transition：

```text
backend-local:
  没有 descriptor
    -> dsm_segment_list 中有一个 descriptor
    -> descriptor 挂到 ResourceOwner

shared control:
  slot refcnt = 0
    -> slot handle = 新 handle
    -> slot refcnt = 2
```

为什么 `dsm_create_descriptor()` 先于 OS segment 和 control slot？

```text
因为 create/attach 过程中可能 ERROR。
descriptor 一旦挂进 ResourceOwner，后续失败路径就有机会通过 dsm_detach()
清理已经成功获得的部分资源。
```

### 5.3 把 handle 交给其它 backend

DSM 不知道你怎么传 handle。它只提供：

```text
dsm_segment_handle(seg) -> dsm_handle
```

在 parallel worker 场景中，leader 把 handle 放进 `BackgroundWorker` 参数：

```text
worker.bgw_main_arg = UInt32GetDatum(dsm_segment_handle(pcxt->seg));
```

这个设计刻意只传 `uint32` handle，不传地址。worker 的进程地址空间可能不同，尤其是 `EXEC_BACKEND` 平台；就算 fork 模式继承了某些 mapping，也不能把“恰好同地址”当成 DSM 抽象的一部分。

### 5.4 worker attach 已有 DSM

worker 启动后：

```text
ParallelWorkerMain(main_arg)
  -> seg = dsm_attach(DatumGetUInt32(main_arg))
  -> toc = shm_toc_attach(PARALLEL_MAGIC, dsm_segment_address(seg))
```

`dsm_attach()` 的核心路径：

```text
dsm_attach(handle)
  -> 确认不是 postmaster / standalone backend 乱用
  -> 如果本 backend 尚未初始化 DSM，dsm_backend_startup()
  -> 扫 dsm_segment_list，禁止同一 backend 重复 attach 同一个 handle
  -> dsm_create_descriptor()
  -> seg->handle = handle
  -> 持有 DynamicSharedMemoryControlLock
     -> 扫 control item[]
     -> 跳过 refcnt <= 1 的 slot
     -> 找 handle 匹配的 active slot
     -> refcnt++
     -> seg->control_slot = i
     -> 如果是 main region pseudo-segment，直接计算 mapped_address
  -> 释放锁
  -> 如果不是 main region，dsm_impl_op(DSM_OP_ATTACH, handle, ...)
```

注意 attach 先增加 `refcnt`，再做 OS-level attach。这个顺序不是随意的：

```text
如果先做 OS attach，再加 refcount：
  原创建者可能同时 detach，看到自己是最后一个 mapping，于是 destroy segment。

先加 refcount：
  control segment 先承认“有人正在尝试 attach”；
  即便后面的 OS attach ERROR，ResourceOwner 仍能在 abort cleanup 中调用 dsm_detach()
  把 refcount 降回来。
```

### 5.5 detach 和最后引用销毁

`dsm_detach()` 是本节最重要的 cleanup 路径：

```text
dsm_detach(seg)
  -> HOLD_INTERRUPTS()
  -> 逐个执行 on_detach callback
  -> RESUME_INTERRUPTS()

  -> 如果 mapped_address != NULL
     -> 对 external segment 调 dsm_impl_op(DSM_OP_DETACH)
     -> 清空 mapped_address / mapped_size / impl_private

  -> 如果 control_slot 有效
     -> 持有 DynamicSharedMemoryControlLock
     -> --refcnt
     -> seg->control_slot = INVALID_CONTROL_SLOT
     -> 释放锁

  -> 如果 refcnt 降到 1
     -> destroy segment
     -> 成功后重新持锁，把 slot refcnt 置 0

  -> 如果 seg->resowner != NULL，ResourceOwnerForgetDSM()
  -> 从 dsm_segment_list 删除 descriptor
  -> pfree(seg)
```

这里的关键顺序是：

```text
先 unmap 当前 backend 的 mapping；
再降低 shared refcount；
最后一个 detach 者负责 destroy；
destroy 成功后 slot 才能变成 refcnt = 0。
```

源码注释明确说，先 remove mapping 再降 refcount，是为了让“看到 refcount 变为最后引用”的进程可以相信没有剩余 mapping。

## 6. 生命周期 / ownership / cleanup

### 谁创建

postmaster 创建 DSM control segment。普通 backend 或 worker leader 通过 `dsm_create()` 创建普通 DSM segment。

典型创建者：

```text
parallel query leader:
  InitializeParallelDSM()

logical apply launcher:
  pa_allocate_worker()

DSA:
  dsa_create_ext()

DSM registry:
  GetNamedDSMSegment()
```

### 谁持有

DSM 有两种 ownership，不能混在一起：

| 层次 | 持有者 | 状态位置 | 作用 |
| --- | --- | --- | --- |
| segment 生命周期 | control segment 中的 `refcnt` / `pinned` | shared | 决定 segment 是否还能 attach、是否该 destroy。 |
| 当前 backend mapping | `dsm_segment.resowner` / `dsm_segment_list` | backend-local | 决定本 backend ERROR / exit 时是否自动 detach。 |

一个 backend attach 到 DSM 后，必须有一个 `dsm_segment` descriptor。这个 descriptor 通常挂到当前 `ResourceOwner` 下：

```text
dsm_create_descriptor()
  -> ResourceOwnerEnlarge(CurrentResourceOwner)
  -> ResourceOwnerRememberDSM(CurrentResourceOwner, seg)
```

如果 `CurrentResourceOwner == NULL`，则 mapping 是 session / process lifespan，需要显式 detach 或等 backend exit。

### 谁释放

普通路径：

```text
调用者显式 dsm_detach(seg)
  -> 当前 backend unmap
  -> shared refcnt--
  -> 最后一个普通 mapping 触发 destroy
```

ERROR / abort 路径：

```text
ResourceOwnerRelease(... RESOURCE_RELEASE_BEFORE_LOCKS ...)
  -> ResOwnerReleaseDSM()
  -> dsm_detach(seg)
```

进程退出路径：

```text
dsm_backend_shutdown()
  -> while dsm_segment_list not empty
     -> dsm_detach(head)
```

postmaster shutdown 或 crash 后重启：

```text
dsm_postmaster_shutdown()
  -> 遍历当前 control segment，destroy 剩余 external DSM

dsm_cleanup_using_control_segment(old_control_handle)
  -> attach 旧 control segment
  -> sanity check
  -> 遍历旧 item[]
  -> destroy 旧 ordinary DSM
  -> destroy 旧 control segment
```

### ERROR / abort 时谁兜底

DSM mapping 的兜底是 `ResourceOwner`，不是 `MemoryContext`。

原因很简单：

```text
dsm_segment descriptor 本身在 TopMemoryContext 里；
真正重要的资源是 OS mapping 和 control segment refcount；
只 reset MemoryContext 不会自动 munmap，也不会 refcnt--。
```

因此 DSM 注册了 `ResourceOwnerDesc`：

```text
name = "dynamic shared memory segment"
release_phase = RESOURCE_RELEASE_BEFORE_LOCKS
release_priority = RELEASE_PRIO_DSMS
ReleaseResource = ResOwnerReleaseDSM
```

DSM 放在 before-locks 阶段，是因为 DSM 可能承载并行执行、sharedfileset、message queue 或执行期共享状态。事务释放锁前，应先清掉这些跨 backend 可见的资源占用。

## 7. 正确性机制层次

| 机制 | 保证 | 不保证 |
| --- | --- | --- |
| `DynamicSharedMemoryControlLock` | control segment 中 `item[]`、`refcnt`、`nitems` 的并发修改一致。 | DSM 内部用户数据的一致性。 |
| `refcnt` 三态 | segment active / moribund / unused 的生命周期边界。 | 对象内容语义有效，也不表示某个 higher-level worker 还活着。 |
| `dsm_handle` | 跨进程发现同一 segment。 | 跨进程地址相同；也不保证 attach 一定成功。 |
| `ResourceOwner` | ERROR / abort 时本 backend mapping 自动 detach。 | 长期 pinned segment 的业务失效；callback 内部逻辑正确。 |
| `on_dsm_detach` | DSM detach 前让上层对象释放依赖 DSM 的资源。 | callback 不应依赖再次访问已经销毁的上层状态。 |
| postmaster cleanup | shutdown / hard kill 后尽量清理遗留 OS DSM 对象。 | control segment 损坏时完整恢复所有遗留对象。 |
| `dsm_impl_op` | 平台相关 create / attach / detach / destroy。 | 高层生命周期协议；它不知道 ResourceOwner 和 refcount 语义。 |

本节不涉及 WAL 或 MVCC。DSM segment 是共享内存资源，不是持久化数据结构；crash 后的目标是删除遗留 OS 资源，而不是 redo 到某个一致业务状态。

## 8. 错误路径 / 异常路径 / fallback

### 8.1 control slot 用尽

control segment 大小在 postmaster startup 时确定：

```text
maxitems = PG_DYNSHMEM_FIXED_SLOTS
           + PG_DYNSHMEM_SLOTS_PER_BACKEND * MaxBackends
```

`dsm_create()` 如果发现 `nitems >= maxitems` 且没有 `refcnt = 0` 的可复用 slot，会清理已经创建的 OS segment 和 backend descriptor，然后：

```text
如果 flags 包含 DSM_CREATE_NULL_IF_MAXSEGMENTS:
  return NULL

否则:
  ERROR "too many dynamic shared memory segments"
```

parallel query 使用了这个 fallback：

```text
pcxt->seg = dsm_create(segsize, DSM_CREATE_NULL_IF_MAXSEGMENTS);
if (pcxt->seg == NULL)
{
    pcxt->nworkers = 0;
    pcxt->private_memory = MemoryContextAlloc(TopMemoryContext, segsize);
}
```

这解释了一个可见现象：DSM segment slot 压力过大时，某些并行路径可能退化为不启动 worker，而不是直接把用户查询打失败。

### 8.2 attach 时 segment 已经消失

`dsm_attach(handle)` 可能返回 `NULL`。源码注释说明：调用者拿到 handle 后，原持有者可能已经全部 detach 并 destroy 了 segment。

attach 只扫描 `refcnt > 1` 的 slot：

```text
refcnt = 0:
  slot unused，不能 attach。

refcnt = 1:
  segment 正在消亡，不能 attach。

refcnt > 1:
  active，可以尝试 attach。
```

如果找不到匹配 slot，`dsm_attach()` 会 detach 掉刚创建的 backend-local descriptor，并返回 `NULL`。调用者决定是 ERROR、fallback，还是认为对端已经结束。

### 8.3 attach 过程中 ERROR

`dsm_attach()` 在调用 `dsm_impl_op(DSM_OP_ATTACH)` 前已经把 control `refcnt++`。如果 OS attach 报 ERROR，当前 backend 已经有：

```text
seg->control_slot = i
seg->resowner = CurrentResourceOwner
mapped_address = NULL
```

ERROR cleanup 时 `ResourceOwnerRelease()` 会调用 `dsm_detach()`，它发现没有 mapping 可 unmap，但 `control_slot` 有效，于是仍然会把 refcount 降回来。

这就是 “acquire-before-ERROR safety” 在 DSM 上的具体形态。

### 8.4 destroy 失败或进程被 kill

当最后一个 mapping detach，`refcnt` 降到 1 后，当前进程尝试 destroy segment。源码承认 destroy 可能失败，或者进程可能在 destroy 前被 kill：

```text
refcnt 会停留在 1；
新 attach 会跳过这个 slot；
postmaster shutdown 或下次 postmaster startup 会再次尝试 cleanup。
```

所以 `refcnt = 1` 不是泄漏的同义词，而是“不能再被业务 attach 的待销毁状态”。它的存在是为了避免正在销毁或销毁失败的 segment 被新 backend 错误接入。

### 8.5 main region pseudo-segment

如果配置了 `min_dynamic_shared_memory`，DSM 可以优先从 main shared memory 预留区域中切分空间。此时：

```text
handle 是奇数；
不会调用 dsm_impl_op() 创建 OS segment；
destroy 时把页范围还给 FreePageManager；
attach 时根据 first_page / npages 直接计算 mapped_address。
```

这条路径仍然使用 control segment 和 refcount，只是底层存储不再是单独 OS DSM 对象。

## 9. 成本、资源与跨模块传播

### 创建 / attach 成本

`dsm_create()` 和 `dsm_attach()` 都要短暂持有 `DynamicSharedMemoryControlLock`，并线性扫描 `item[]`：

```text
dsm_create():
  找 refcnt = 0 的 slot，或者追加 nitems。

dsm_attach():
  找 handle 匹配且 refcnt > 1 的 slot。
```

因此 DSM 不是设计给每个 tuple、每个 buffer 或每个小对象动态创建的。它适合粗粒度生命周期：

```text
一次 parallel operation；
一个 session DSM；
一个 DSA area 的 backing segment；
一个 named DSM / registry object。
```

小对象动态分配应交给 DSA 或 DSM 内部自己的 allocator，而不是频繁创建 DSM segment。

### 访问成本

一旦 attach 成功，访问 DSM 内容就是普通内存读写：

```text
void *base = dsm_segment_address(seg);
```

但是 DSM 不提供内容同步。segment 内部对象需要自己使用 spinlock、LWLock、atomic、latch、barrier 或更高层协议。

这就是为什么 parallel query 在 DSM 里先放 `shm_toc`，再放 `shm_mq`、fixed state、serialized snapshot 等结构：

```text
DSM:
  提供共享 address space。

shm_toc:
  提供 segment 内对象发现。

shm_mq / locks / atomics:
  提供对象自己的并发协议。
```

### 跨模块传播

DSM 是很多后续机制的地基：

| 模块 | DSM 的角色 |
| --- | --- |
| `parallel.c` | leader 和 worker 共享 parallel context 状态。 |
| `shm_toc.c` | 在 DSM 内建立轻量目录。 |
| `shm_mq.c` / `pqmq.c` | 在 DSM 内传递 tuple、错误消息和协议消息。 |
| `dsa.c` | 用一个或多个 DSM segment 组成可增长共享 heap。 |
| `dsm_registry.c` | 把 named DSM / DSA / dshash 做成 extension 可复用基础设施。 |
| `sharedfileset.c` | DSM detach 时清理和共享文件集合关联的临时文件。 |

本节只讲第一层：segment 生命周期。后面几节会逐层解释“拿到一块 DSM 后，如何在里面组织对象和通信”。

## 10. 观测与诊断入口

### 日志

把日志级别调到足够低时，DSM startup / cleanup 有 DEBUG2 日志：

```text
dynamic shared memory system will support N segments
created dynamic shared memory control segment H (S bytes)
cleaning up orphaned dynamic shared memory with ID H
```

这些日志来自 `dsm_postmaster_startup()`、`dsm_postmaster_shutdown()` 和 `dsm_cleanup_using_control_segment()`。

### SQL 层间接观察

普通 DSM segment 没有一个全局 `pg_stat_dsm_segments` 视图。可观察入口通常是间接的：

```text
EXPLAIN 中是否启动 parallel workers；
pg_stat_activity 的 parallel query / wait event；
日志中 worker 是否启动失败；
DSM registry 相关对象可通过 pg_get_dsm_registry_allocations() 观察；
main shmem 预留的 "Preallocated DSM" 可在 pg_shmem_allocations 里看到。
```

如果一个 parallel query 由于 DSM slot 用尽而 fallback，用户通常看到的是 worker 数减少，而不是直接看到 DSM control slot 状态。

### gdb 断点

调试生命周期时，最有用的断点不是 `shm_open()`，而是：

```gdb
break dsm_create
break dsm_attach
break dsm_detach
break ResOwnerReleaseDSM
break dsm_postmaster_shutdown
```

建议观察：

```gdb
p *seg
p dsm_control->item[seg->control_slot]
p dsm_control->nitems
p dsm_control->maxitems
```

重点看状态变化：

```text
dsm_create 后:
  refcnt = 2

worker dsm_attach 后:
  refcnt = 3, 4, ...

一个 backend dsm_detach 后:
  refcnt 逐步下降

最后一个 detach 后:
  refcnt = 1 -> destroy -> refcnt = 0
```

### 文件系统 / OS 侧观察

不同 `dynamic_shared_memory_type` 使用不同 OS primitive。`mmap` 实现使用 `pg_dynshmem` 下的文件；POSIX / SysV / Windows 路径则依赖相应平台对象。

不要把 OS 侧对象名当成 PostgreSQL 高层生命周期语义。真正决定 segment 是否 active 的是 control segment 中的 slot 状态。

## 11. 常见误区

### 误区 1：DSM 是 shared memory 版 malloc

不是。DSM 是 segment 级 abstraction。频繁申请小对象应该用 DSA 或 segment 内 allocator。

### 误区 2：`refcnt = 1` 表示还有一个 backend attach

不是。普通 mapping 的 active 状态从 `refcnt = 2` 开始。`refcnt = 1` 是 moribund，表示没有普通 mapping，segment 正在等待 destroy 或 cleanup。

### 误区 3：拿到 `mapped_address` 后可以把它写给其它进程

不能。跨进程只能传 `dsm_handle`，segment 内对象也应使用 offset、key、`dsa_pointer` 或其它相对引用协议。

### 误区 4：ResourceOwner 管的是 segment 本身

更准确地说，ResourceOwner 管当前 backend 的 mapping descriptor。segment 是否被销毁由 control segment 的 refcount / pin 决定。

### 误区 5：`dsm_detach()` 只是本 backend `munmap`

不止。它还会执行 detach callback、更新 shared refcount、在最后引用时 destroy segment，并从 ResourceOwner 和 `dsm_segment_list` 中移除 descriptor。

### 误区 6：postmaster cleanup 能修复所有泄漏

不能。它能根据 control segment 尽力删除遗留 OS DSM 对象；如果 control segment 损坏或平台对象已经不一致，只能记录日志并尽量避免更糟。

## 12. 课堂实验

### 实验 1：跟踪 parallel query 的 DSM 生命周期

目标：看到一次 parallel query 如何创建 DSM、worker 如何 attach、结束时如何 detach。

步骤：

```sql
SET max_parallel_workers_per_gather = 2;
SET min_parallel_table_scan_size = 0;
SET parallel_setup_cost = 0;
SET parallel_tuple_cost = 0;

CREATE TABLE dsm_demo AS
SELECT g AS id, md5(g::text) AS payload
FROM generate_series(1, 1000000) g;

ANALYZE dsm_demo;

EXPLAIN (ANALYZE, VERBOSE)
SELECT count(*) FROM dsm_demo WHERE id > 0;
```

源码观察点：

```text
leader:
  InitializeParallelDSM()
  -> dsm_create()
  -> dsm_segment_handle()

worker:
  ParallelWorkerMain()
  -> dsm_attach(main_arg)
  -> shm_toc_attach()

cleanup:
  DestroyParallelContext()
  -> dsm_detach(pcxt->seg)
```

gdb 练习：

```gdb
break dsm_create
commands
  silent
  bt 5
  continue
end

break dsm_attach
break dsm_detach
```

思考：leader 创建后 `refcnt` 为什么是 2？第一个 worker attach 后为什么变 3？

### 实验 2：观察 `DSM_CREATE_NULL_IF_MAXSEGMENTS` fallback

目标：理解 DSM slot 用尽时，parallel context 为什么可以退回 backend-private memory。

阅读源码：

```text
src/backend/access/transam/parallel.c
  InitializeParallelDSM()
    -> dsm_create(segsize, DSM_CREATE_NULL_IF_MAXSEGMENTS)
    -> pcxt->seg == NULL 时 nworkers = 0
```

讨论：

```text
为什么这里选择“少用 parallelism”，而不是直接 ERROR？
如果调用者是 DSA 创建 backing segment，还能不能这样 fallback？
```

### 实验 3：单步 detach 的最后引用路径

目标：确认 `refcnt = 1` 只是中间销毁状态。

gdb 观察点：

```gdb
break dsm_detach
```

在断点处观察：

```gdb
p seg->handle
p seg->control_slot
p dsm_control->item[seg->control_slot].refcnt
next
```

当最后一个 mapping detach：

```text
--refcnt 得到 1
-> dsm_impl_op(DSM_OP_DESTROY, ...)
-> 成功后 refcnt = 0
```

如果你只在 `dsm_impl_op(DSM_OP_DESTROY)` 前后看 OS 对象，容易错过 control slot 的 moribund 语义。

## 13. 讨论题

1. 为什么 `dsm_attach()` 要先在 control segment 中 `refcnt++`，再调用 OS-level attach？如果反过来会出现什么 race？
2. 为什么 `dsm_detach()` 要先 unmap 当前 backend 的 mapping，再把 control refcount 降下来？
3. `refcnt = 1` 为什么不能允许新 backend attach？如果允许，会和 handle 重用、destroy 失败产生什么问题？
4. 为什么 DSM mapping 默认挂 `CurrentResourceOwner`，而不是只依赖 backend exit 时的 `dsm_backend_shutdown()`？
5. 为什么 DSM 不提供“segment 内 malloc/free”？如果提供，会和 control segment refcount 解决的问题有什么不同？
6. parallel query 可以在 DSM 创建失败时 fallback 到 backend-private memory；哪些 DSM 使用场景不能 fallback？

## 14. 本节小结

DSM 解决的是 main shmem 不适合解决的问题：

```text
运行期创建；
跨进程临时共享；
用完可销毁；
大小不必在 postmaster startup 前完全知道。
```

但它没有把共享内存变成“随便传指针、随便释放”的普通内存。PostgreSQL 用两个层次收住风险：

```text
control segment:
  handle -> slot -> refcnt / pinned，保证 segment 生命周期跨进程一致。

backend-local descriptor + ResourceOwner:
  mapped_address / impl_private / on_detach / resowner，保证当前 backend 在 ERROR 和退出时清理 mapping。
```

本节最可迁移的规律是：

```text
跨进程资源的生命周期不能只靠本地指针管理。
必须同时有全局可见的引用状态、进程本地的 ownership 记录、
以及失败路径中可重复执行且尽量不失败的 cleanup。
```

下一节在这个基础上继续：为什么“segment 还存在”和“当前 backend 还映射着它”不是一回事，DSM pin、detach callback 和 named DSM 如何把短期 DSM 扩展为长期共享基础设施。
