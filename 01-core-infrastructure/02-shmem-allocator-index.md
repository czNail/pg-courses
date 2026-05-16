# PostgreSQL Shmem allocator 与 ShmemIndex

## 课程定位

前置知识：已经理解 main shared memory segment 为什么必须在启动期定容，知道 PostgreSQL 是 postmaster 创建共享内存、backend 通过 fork 或 attach 访问同一片 shared state。

本节唯一主问题：

```text
一个名字如何绑定到一块跨进程共享状态，为什么传统 main shmem 只能分配、不能释放？
```

核心矛盾：PostgreSQL 希望 shared state 像普通 C 指针一样被所有 backend 低成本访问，但它又需要一个跨进程一致的命名目录来避免重复分配、支持 attach 和诊断；一旦允许任意释放，悬空指针、并发使用者、碎片整理和 crash 后状态一致性都会进入 hot path。

学完后应能判断：什么时候应该用 named main shmem allocation，什么时候只是在 shared memory 区域里实现自己的对象池或 freelist，什么时候该转向 DSM / DSA。

本课基于本地 `~/postgres` 源码，分支 `master`，提交 `8eba2edb801`。

## 1. 本节在总主线中的位置

上一节回答了 main shared memory segment 为什么要启动期定容。本节进入 segment 内部：`InitShmemAllocator()` 如何建立一个只向前推进的 allocator，`ShmemIndex` 如何把字符串名字绑定到地址，`ShmemInitStruct()` / `ShmemRequestStruct()` 如何避免多个进程重复初始化同一块 shared state。

下一节会专门讲 request / init / attach 生命周期。本节只在 allocator 和 name binding 必要处提到这些阶段，不展开完整 backend attach 协议。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
main shmem segment 头部之后放一个 ShmemAllocatorData，
allocator 用 free_offset 单调切分空间，
每个命名 allocation 在 ShmemIndex 中记录 name -> location / size，
backend 后续通过稳定指针直接访问这块共享状态。
```

这里的系统 tension 是：

```text
跨进程稳定指针和极低访问成本
  vs
运行期动态创建、释放、整理和重用共享内存的灵活性
```

PostgreSQL 在传统 main shmem 上选择前者。它可以分配固定生命周期的大块结构，但不提供 `ShmemFree()`。如果某个子系统需要删除元素，通常在自己那块固定区域里维护 freelist、slot array、hash bucket 或 refcount，而不是把整块 main shmem 还给全局 allocator。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 作用 |
| --- | --- | --- |
| 1 | `src/include/storage/shmem.h` | 对子系统暴露 request、legacy init、hash 和低层 allocation 接口；说明 pointer、size、alignment 的语义。 |
| 2 | `src/backend/storage/ipc/shmem.c` | main shmem allocator、`ShmemIndex`、named allocation、late allocation 和 `pg_shmem_allocations` 的核心实现。 |
| 3 | `src/include/storage/shmem_internal.h` | postmaster / backend 启动路径调用的内部接口，如 `InitShmemAllocator()`、`ShmemInitRequested()`。 |
| 4 | `src/backend/storage/ipc/shmem_hash.c` | 在一整块 named shmem allocation 内部创建固定大小 shared hash table。 |
| 5 | `src/backend/storage/ipc/ipci.c` | segment 创建后调用 allocator 初始化和 requested allocations 初始化。 |
| 6 | `src/backend/storage/buffer/buf_init.c` | 典型大对象：buffer descriptor、buffer blocks、condition variable array 如何 request 后拿到稳定指针。 |
| 7 | `src/backend/catalog/system_views.sql` | `pg_shmem_allocations` 视图如何暴露 main shmem allocation 分布。 |

阅读时不要把 `ShmemAlloc()` 当成普通 `malloc()`。它的语义更接近“从启动期固定资源池中切出永久区域”。

## 4. 关键数据结构与状态

### `ShmemAllocatorData`

`ShmemAllocatorData` 是 main segment 内容区的第一个结构，位置由 `PGShmemHeader.content_offset` 指向。它不是 backend-local allocator handle，而是所有 backend 共享的 allocator 状态。

| 字段 | 语义 |
| --- | --- |
| `free_offset` | 从 `ShmemBase` 起算的第一个未分配偏移。传统 shmem allocator 的核心状态。 |
| `shmem_lock` | 保护 `free_offset`。分配时短暂持有 spinlock。 |
| `index` | `ShmemIndex` 的 shared hash header 地址。 |
| `index_size` | `ShmemIndex` 这块区域的大小。 |
| `index_lock` | 保护 `ShmemIndex` 的 LWLock。 |

这个结构解释了为什么传统 shmem 没有全局释放：allocator 只记住“下一个可分配位置”，不维护 per-allocation metadata、空闲区链表、coalescing 信息或对象所有者。

### `ShmemIndex`

`ShmemIndex` 是一个存放在 main shmem 里的 shared hash table。它把名字映射到 allocation 元数据：

| 字段 | 语义 |
| --- | --- |
| `key` | shared memory area 的字符串名字。 |
| `location` | 这块区域在 main segment 中的地址。传统 shmem 要求各进程映射到同一地址，因此普通指针可跨进程使用。 |
| `size` | 调用者请求的大小。 |
| `allocated_size` | 实际消耗的大小，包含对齐 padding。 |

raw field 不是语义。`location` 只有在 main segment 已 attach、`ShmemIndexLock` 保护下查到、`size` 校验通过、调用者完成 init 或 attach 回调之后，才变成“这个子系统可使用的共享状态指针”。

### `pending_shmem_requests`

`pending_shmem_requests` 是进程私有 list。request 阶段把“未来需要的 named area”记在这里；segment 创建后，`ShmemInitRequested()` 才把这些请求真正落到 `ShmemIndex` 和 allocator 上。

因此要分清三层状态：

```text
pending request：backend-local 计划
ShmemIndex entry：跨进程命名目录
location 指向的区域：真正的 shared state
```

## 5. 主流程源码 walkthrough

### 5.1 allocator bootstrap

主流程从上一节的 segment 创建之后开始：

```text
CreateSharedMemoryAndSemaphores()
  -> PGSharedMemoryCreate()
  -> InitShmemAllocator(seghdr)
     -> 把 ShmemAllocatorData 放在 content_offset
     -> 初始化 shmem_lock / free_offset / ShmemIndexLock
     -> ShmemAlloc(index_size)
     -> shmem_hash_create(..., "ShmemIndex", ...)
     -> 把 "ShmemIndex" 自己登记进 ShmemIndex
  -> ShmemInitRequested()
```

`InitShmemAllocator()` 有一个自举点：`ShmemIndex` 本身也需要 shared memory，但创建 named allocation 又依赖 `ShmemIndex` 已经存在。因此源码直接用低层 `ShmemAlloc()` 给 `ShmemIndex` 切出一块区域，再调用 `shmem_hash_create()` 建立 shared hash table，最后手工插入名为 `"ShmemIndex"` 的 entry。

这解释了 `pg_shmem_allocations` 里为什么能看到 `ShmemIndex`：它不是普通子系统 request 出来的，但初始化完成后被补录到目录里。

### 5.2 named allocation 初始化

以 `BufferManagerShmemRequest()` 为例：

```text
BufferManagerShmemRequest()
  -> ShmemRequestStruct(.name = "Buffer Descriptors", .size = ..., .ptr = &BufferDescriptors)
  -> ShmemRequestStruct(.name = "Buffer Blocks", .size = ..., .ptr = &BufferBlocks)
```

这些调用不会立刻分配 shared memory，只是进入 `pending_shmem_requests`。真正分配发生在：

```text
ShmemInitRequested()
  -> foreach pending request
     -> InitShmemIndexEntry(request)
        -> hash_search(ShmemIndex, name, HASH_ENTER_NULL)
        -> ShmemAllocRaw(size, alignment, &allocated_size)
        -> index_entry->size = size
        -> index_entry->allocated_size = allocated_size
        -> index_entry->location = structPtr
        -> *ptr = location
  -> foreach registered callback
     -> init_fn()
```

这里有一个重要顺序：`*ptr` 在 subsystem 的 `init_fn` 之前已经设置好。因此 buffer manager 的 init 回调可以直接使用 `BufferDescriptors`、`BufferBlocks` 等全局指针来初始化每个 descriptor。

### 5.3 attach 或已有对象查找

legacy 接口 `ShmemInitStruct(name, size, &found)` 直接围绕 `ShmemIndex` 工作：

```text
ShmemInitStruct()
  -> LWLockAcquire(ShmemIndexLock, LW_EXCLUSIVE)
  -> 如果是普通 backend，先 AttachShmemIndexEntry(..., missing_ok = true)
  -> 如果没找到，InitShmemIndexEntry()
  -> LWLockRelease(ShmemIndexLock)
  -> 返回 location，并通过 foundPtr 告诉调用者是否需要初始化
```

新接口把 request / init / attach 拆得更清楚；legacy 接口仍然广泛影响扩展生态。无论接口新旧，核心语义都是同一个：

```text
name 负责去重和重新发现；
location 负责跨进程访问；
found / init_fn 负责避免重复初始化。
```

## 6. 生命周期 / ownership / cleanup

### 谁创建

postmaster 或 standalone backend 在 `InitShmemAllocator()` 中创建 allocator 和 `ShmemIndex`。随后 `ShmemInitRequested()` 依次创建所有 request 出来的 named areas。

### 谁持有

传统 main shmem allocation 没有“调用者可释放的 owner”。所有 backend 都可能通过全局指针、`ShmemIndex` 或 subsystem 私有结构访问它。ownership 更接近：

```text
segment 归 postmaster 生命周期所有；
named area 归子系统协议所有；
area 内部对象归子系统自己的并发与回收机制所有。
```

### 谁释放

单个 named area 不释放。main segment 在 postmaster 退出、崩溃重启或平台层清理时整体释放。某个 shared hash table 删除元素时，释放的是 hash table 内部 entry，回到该 hash table 自己的 freelist，不会回到全局 shmem allocator。

### ERROR / abort 时怎么办

传统 main shmem allocation 不跟事务生命周期绑定，也不由 `ResourceOwner` 自动回滚。启动期失败通常导致 server 启动失败；late allocation 失败会在当前 backend 报错，但已经成功创建并初始化的 shared state 不会像事务内存一样自动撤销。

这也是为什么 init 回调要谨慎：一旦把名字插入 `ShmemIndex` 并分配出区域，后续错误路径不能假设有一个全局 `free` 可以把状态完整倒回去。

## 7. 正确性机制层次

| 层次 | 机制 | 保证 | 不保证 |
| --- | --- | --- | --- |
| segment 边界 | `PGShmemHeader.totalsize`、`ShmemBase`、`ShmemEnd` | allocator 不越界；指针可判断是否在 main shmem 内 | 对象语义有效 |
| 空间切分 | `free_offset` + `shmem_lock` | 并发分配不会分到同一段空间 | 可释放、可整理、可缩容 |
| 命名目录 | `ShmemIndex` + `ShmemIndexLock` | name 到 location 的一致映射；防止重复创建 | area 内部并发安全 |
| 大小校验 | `AttachShmemIndexEntry()` | attach 时发现同名不同 size | 调用者结构版本完全兼容 |
| 初始化顺序 | request -> allocate -> set pointer -> init callback | init 回调看到稳定指针 | init 回调内部逻辑正确 |
| 子系统内部协议 | LWLock、spinlock、atomic、freelist、slot state | area 内元素并发访问和复用 | 全局 allocator 回收 |

本节不依赖 WAL 或 MVCC。它处理的是进程间内存地址、启动顺序和并发分配目录，不处理 tuple visibility 或 crash redo。

## 8. 错误路径 / 异常路径 / fallback

### out of shared memory

`ShmemAllocRaw()` 在持有 `shmem_lock` 时检查 `newFree <= ShmemSegHdr->totalsize`。如果空间不足，返回 `NULL`；`ShmemAlloc()` 报 `out of shared memory`，`ShmemAllocNoError()` 则把失败交给调用者。

`InitShmemIndexEntry()` 有一个细节：它先向 `ShmemIndex` 插入 entry，再分配实际区域。如果实际分配失败，会把刚插入的 entry 从 `ShmemIndex` 移除，然后报错。这只处理“目录插入成功但空间分配失败”的局部一致性，不等于支持一般性 free。

### duplicate name

request 阶段会拒绝同一进程内重复注册同名 request；初始化阶段 `InitShmemIndexEntry()` 如果发现 `ShmemIndex` 已有同名 entry，也会报错。名字是全局目录键，不是注释或显示标签。

### same name, different size

attach 时如果 `ShmemIndex` 中已有 entry，但请求 size 不一致，`AttachShmemIndexEntry()` 会报错。`SHMEM_ATTACH_UNKNOWN_SIZE` 是特殊 attach 逃生口，主要服务某些 `EXEC_BACKEND` 或动态 attach 场景，不应把它理解成“size 不重要”。

### late allocation

`RegisterShmemCallbacks()` 在 postmaster 启动后通常拒绝 request。只有带 `SHMEM_CALLBACKS_ALLOW_AFTER_STARTUP` 的 callback 才能进入 `CallShmemCallbacksAfterStartup()`，并且源码注释明确说不保证还有足够 shared memory。

这不是鼓励运行期扩容，而是为少量 extension 兼容性留下的余量。

## 9. 成本、资源与跨模块传播

### allocator 成本

`ShmemAllocRaw()` 是非常短的 critical section：

```text
SpinLockAcquire(shmem_lock)
  -> 读 free_offset
  -> 对齐
  -> 检查 totalsize
  -> 写回 free_offset
SpinLockRelease(shmem_lock)
```

因为大多数分配发生在启动期，运行期 hot path 不应该频繁打到这个 spinlock。PostgreSQL 用“启动期集中分配”换来了之后每次访问 shared state 时不需要查 allocator、不需要 pin allocation、不需要处理移动或释放。

### ShmemIndex 成本

`ShmemIndex` 查找受 `ShmemIndexLock` 保护。启动期没有并发 backend，主要是初始化顺序问题；late allocation 和诊断视图会在运行期读取它。`pg_shmem_allocations` 会以 shared mode 持有 `ShmemIndexLock` 遍历目录，所以它是诊断入口，不应该被当作业务 hot path。

### 为什么不能释放

传统 main shmem 不能释放不是因为“写一个 free list 很难”，而是因为释放会要求回答这些问题：

```text
还有哪些 backend 持有这个指针？
是否有 backend 把指针保存在全局变量、缓存 entry、hash value 或等待队列里？
释放后同一地址能否被复用？旧指针如何检测 generation 变化？
如果需要搬移或 compact，跨进程普通 C 指针如何更新？
ERROR / FATAL / postmaster crash restart 中，半释放状态如何恢复？
```

一旦这些问题进入通用 allocator，就会把 refcount、generation、hazard pointer、epoch 或 handle indirection 的成本传播给所有 shared state。PostgreSQL 选择把这些复杂性下放到真正需要动态对象生命周期的子系统内部，或者使用 DSM / DSA 这类更适合动态共享内存的机制。

## 10. 观测与诊断入口

查看 named allocation：

```sql
SELECT name, off, pg_size_pretty(size) AS size,
       pg_size_pretty(allocated_size) AS allocated
FROM pg_shmem_allocations
ORDER BY off NULLS LAST
LIMIT 30;
```

观察 `ShmemIndex` 自举结果：

```sql
SELECT *
FROM pg_shmem_allocations
WHERE name = 'ShmemIndex';
```

观察对齐 padding：

```sql
SELECT name, size, allocated_size, allocated_size - size AS padding
FROM pg_shmem_allocations
WHERE name IS NOT NULL
ORDER BY padding DESC
LIMIT 20;
```

观察 anonymous 和 unused：

```sql
SELECT name, off, size, allocated_size
FROM pg_shmem_allocations
WHERE name IS NULL OR name = '<anonymous>';
```

解释边界：

| 行 | 含义 |
| --- | --- |
| 普通 `name` | `ShmemIndex` 中登记的 named allocation。 |
| `<anonymous>` | allocator 已经消耗，但没有通过 `ShmemIndex` 计入 named allocation 的空间。 |
| `name IS NULL` | `free_offset` 之后尚未使用的 main shmem 空间。 |

调试断点建议：

```text
InitShmemAllocator
ShmemAllocRaw
InitShmemIndexEntry
AttachShmemIndexEntry
ShmemInitStruct
pg_get_shmem_allocations
```

断在 `InitShmemIndexEntry()` 时重点看 `name`、`request->options->size`、`index_entry->location` 和 `ShmemAllocator->free_offset` 的变化。

## 11. 常见误区

误区一：`ShmemIndex` 是普通 backend-local hash table。

实际它本身也在 main shmem 中，是所有 backend 共享的命名目录。backend-local 的只是 `HTAB *ShmemIndex` 这个控制指针，以及某些 attach 时重建的 backend-private handle。

误区二：`ShmemAlloc()` 和 `malloc()` 类似，只是分配在共享内存里。

实际 `ShmemAlloc()` 没有对应 `ShmemFree()`，也没有事务 cleanup。它更像从固定 segment 里永久切出一段基础设施空间。

误区三：不能全局 free，所以 shared hash table 里的 entry 也不能删除。

两层要分开。全局 shmem allocator 不回收整块 named area；但某个 named area 内部可以实现自己的 freelist。`shmem_hash.c` 明确说明每个 hash table 有自己的 free list，hash entry 删除后可在该 table 内重用。

误区四：`pg_shmem_allocations` 能看到所有共享内存。

它只展示传统 main shmem allocator 和 `ShmemIndex` 视角，不展示 DSM registry 之外的全部动态共享内存生命周期，也不展示每个 named area 内部 slot 是否空闲。

误区五：unused 很多就一定是浪费。

unused 包含启动期估算余量、对齐和小量 late allocation 空间。是否浪费要结合 GUC、preload 扩展、平台页大小和 workload 判断。

## 12. 课堂实验

### 实验 1：从 SQL 还原 name -> address 目录

启动 server 后执行：

```sql
SELECT name, off, size, allocated_size
FROM pg_shmem_allocations
WHERE name IN ('ShmemIndex', 'Buffer Descriptors', 'Buffer Blocks', 'Proc Header')
ORDER BY off;
```

预期：这些名字都有稳定 offset。`off` 是相对 segment 头部的偏移，适合跨进程比较；真实 `location` 是进程地址空间中的指针，不直接在 SQL 里展示。

### 实验 2：观察 padding

执行：

```sql
SELECT name, size, allocated_size, allocated_size - size AS padding
FROM pg_shmem_allocations
WHERE name IS NOT NULL
ORDER BY padding DESC
LIMIT 10;
```

把结果和源码中的 alignment 联系起来。比如 buffer blocks 会请求 IO page alignment；普通 allocation 至少按 cache line 对齐。

### 实验 3：源码断点看 `free_offset` 单调增长

在调试构建中：

```text
b InitShmemAllocator
b InitShmemIndexEntry
b ShmemAllocRaw
```

观察每次 `ShmemAllocRaw()` 前后的 `ShmemAllocator->free_offset`。你会看到它只向前推进，不会因为某个子系统 init 完成而回退。

### 实验 4：对照全局 allocator 与 hash table 内部复用

阅读：

```text
src/backend/storage/ipc/shmem_hash.c
src/backend/utils/hash/dynahash.c
```

确认 shared hash table 是先拿到一整块 fixed-size region，再在内部管理 hash entries。删除一个 hash entry 不会改变 `ShmemAllocator->free_offset`，只影响该 hash table 自己的内部空闲结构。

## 13. 讨论题

1. 如果给 main shmem 增加 `ShmemFree(name)`，最先需要补哪些元数据和并发协议？
2. 为什么 `ShmemIndex` 记录的是普通指针，而 DSM / DSA 通常需要 handle、offset 或特殊 pointer？
3. `ShmemIndexLock` 保护了哪些状态？为什么它不能替代某个子系统内部的 LWLock？
4. 一个扩展需要维护可增删的共享对象集合，应该申请一整块 main shmem 后内部自管，还是直接用 DSM / DSA？判断依据是什么？
5. `ShmemInitStruct()` 的 `foundPtr` 为什么重要？如果每个 backend 都重新初始化同一块 shared state，会出现什么问题？

## 14. 本节小结

本节主链路是：

```text
PGSharedMemoryCreate()
  -> InitShmemAllocator()
  -> 创建 ShmemIndex
  -> ShmemInitRequested()
  -> InitShmemIndexEntry()
  -> ShmemAllocRaw()
  -> name 绑定到 location
```

核心状态边界是 `ShmemAllocatorData.free_offset` 和 `ShmemIndex`。前者定义 main shmem 的单调分配位置，后者定义跨进程可重新发现的 name -> location 目录。

传统 main shmem 不能释放的根本原因，是 PostgreSQL 把“普通 C 指针跨进程稳定可用”放在第一优先级。全局 allocator 不承担对象生命周期回收；需要动态生命周期的子系统必须在自己的 fixed region 内部实现复用，或改用 DSM / DSA。

可迁移规律：

```text
当系统选择用稳定裸指针换取共享状态访问效率时，
全局 allocator 往往只能提供粗粒度、长生命周期分配；
细粒度释放必须通过局部对象池、句柄层或另一套动态内存机制完成。
```

下一节继续沿着这条线追问：postmaster、fork backend 和 `EXEC_BACKEND` 如何在不同进程中建立同一组 shared state 指针。
