# PostgreSQL DSA area、dsa_pointer 与跨进程共享 heap

## 课程定位

前置知识：已经理解 DSM segment 的 create / attach / detach 生命周期，知道 `shm_toc` 用 offset 解决 DSM 内对象发现，也理解 `shm_mq` 如何在 DSM 中放置一个固定大小的共享对象。

本节唯一主问题：

```text
为什么 DSM 只适合整段共享而不适合小对象动态分配，
dsa_create() / dsa_attach() / dsa_get_address()
如何用 area handle、segment map 和 dsa_pointer
支持可在进程间传递但不能直接解引用的共享指针？
```

核心矛盾：DSM 给 PostgreSQL 提供的是“可被多个 backend attach 的共享内存段”，但很多并行执行和共享结构需要的是“运行期不断分配和释放的小对象”：hash item、radix tree node、listener array、bitmap page table、executor 参数等。普通 C 指针不能跨进程传播；一个 DSM segment 又无法优雅支持增长、回收和小对象碎片管理。DSA 在 DSM 之上建立共享 heap，但它必须同时尊重跨进程地址不稳定、segment 可增减、ERROR-safe cleanup 和共享对象 ownership。

学完后应能判断：

```text
为什么 dsa_pointer 是 pseudo pointer，而不是 void *；
为什么 dsa_pointer 可以跨进程保存和传递，但必须用 dsa_get_address() 转成本地地址；
为什么 dsa_area 在每个 backend 中都有一个 backend-local dsa_area 对象；
为什么 dsa_area_control 在共享内存中保存 segment_handles、refcnt、locks 和 allocator 状态；
为什么 dsa_create() 创建的是一个 area，而不是只创建一个 DSM segment；
为什么 dsa_attach() 只需要 area handle，就能按需 attach 后续 DSM segment；
为什么 dsa_detach()、dsa_release_in_place()、dsa_pin()、dsa_pin_mapping() 是不同生命周期动作。
```

本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `2c4bd2bf5700`。

## 1. 本节在总主线中的位置

第 27 到 31 节已经完成了 DSM 这一层的主要模型：

```text
DSM:
  动态创建 / attach / detach 一整段共享内存

shm_toc:
  在一段 DSM 内发现固定对象

shm_mq:
  在 DSM 内放置固定大小的双端消息队列
```

现在的问题变成：

```text
如果共享对象数量和大小在运行时才知道，怎么办？
```

典型例子：

```text
parallel hash join:
  需要共享 bucket、batch、tuple chunk、work queue。

dshash:
  需要共享 hash table control、bucket array 和 item 链表。

tidbitmap / tidstore:
  需要把 bitmap/page table 变成可被 worker 共享的对象。

LISTEN/NOTIFY channel table:
  需要在长期共享结构中保存动态 listener array。
```

这些场景都不适合只靠 `shm_toc_allocate()`：

```text
TOC 适合 bootstrap 固定对象；
DSA 适合在共享区域里继续动态分配对象。
```

本节先讲 DSA 的使用模型和 pointer ownership。下一节再深入 allocator 内部的 size class、span、superblock 和 FreePageManager。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
DSA area 是一个由一个或多个 DSM segment 支撑的共享 heap；
共享的 dsa_area_control 保存 area handle、segment handle 表、locks、refcount 和 allocator 元数据；
每个 backend attach 后建立自己的 backend-local dsa_area 和 segment_map；
dsa_allocate() 返回可跨进程传递的 dsa_pointer；
dsa_get_address() 用当前 backend 的 segment_map 把 dsa_pointer 转成本地 void *。
```

本节 tension 是：

```text
共享对象需要像 heap 一样动态分配和释放
  vs
跨进程不能共享普通地址，segment 还可能按需增长和回收
```

如果直接在 DSM 里保存 `void *`：

```text
leader 映射 DSM 在 0x7f...A
worker 映射同一 DSM 在 0x7f...B
leader 写入的 pointer 在 worker 中没有意义
```

如果只保存 offset：

```text
单个 DSM segment 内可以工作；
但 DSA 可能有多个 segment，还会创建新 segment。
```

因此 DSA 的 pointer 必须表达：

```text
segment index
offset within segment
```

这就是 `dsa_pointer` 的本质。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 作用 |
| --- | --- | --- |
| 1 | `src/include/utils/dsa.h` | 对外 API、`dsa_pointer` 类型、area handle、allocation flags、指针宽度限制。 |
| 2 | `src/backend/utils/mmgr/dsa.c` | DSA control block、backend-local area、segment map、create / attach / get_address / detach 主实现。 |
| 3 | `src/backend/executor/execParallel.c` | parallel query 如何在 TOC 中创建 in-place DSA，并让 worker attach。 |
| 4 | `src/backend/lib/dshash.c` | 典型上层：control object、bucket array 和 item 链表都放在 DSA 中，用 `dsa_pointer` 连接。 |
| 5 | `src/backend/storage/ipc/dsm_registry.c` | named DSM registry 如何用 pinned DSA + dshash 形成长期共享 registry。 |
| 6 | `src/backend/commands/async.c` | LISTEN/NOTIFY channel table 如何保存 DSA handle 和 dynamic listener array。 |
| 7 | `src/backend/nodes/tidbitmap.c` / `src/backend/access/common/tidstore.c` | 位图和 tidstore 如何把本地结构转成共享 DSA 结构。 |

推荐阅读顺序：

```text
先读 dsa.h 中 dsa_pointer 和 dsa_handle 注释
  -> 读 dsa.c 顶部模块注释
  -> 读 dsa_area_control / dsa_segment_map / struct dsa_area
  -> 跟 dsa_create_ext() -> create_internal()
  -> 跟 dsa_attach() -> attach_internal()
  -> 跟 dsa_get_address()
  -> 最后看 execParallel.c 和 dshash.c 的真实使用方式
```

本节不要急着读完整 allocator。先把这三个问题看清楚：

```text
area 的共享根状态在哪里？
dsa_pointer 为什么能跨进程传递？
本 backend 如何把 dsa_pointer 变成本地地址？
```

## 4. 关键数据结构与状态

### 4.1 `dsa_pointer`

`dsa_pointer` 是 DSA 的 pseudo pointer。它可以放进共享内存、atomic 字段、hash item、TOC entry 指向的对象中，也可以跨进程传递。

在 `dsa.c` 中，pointer 编码由两个宏表达：

```text
DSA_MAKE_POINTER(segment_number, offset)
DSA_EXTRACT_SEGMENT_NUMBER(dp)
DSA_EXTRACT_OFFSET(dp)
```

也就是说，`dsa_pointer` 的语义是：

```text
高位:
  segment index

低位:
  offset inside that segment
```

`dsa.h` 根据平台选择 32 位或 64 位 `dsa_pointer`：

```text
32-bit:
  DSA_OFFSET_WIDTH = 27
  最多 32 个 segment，每个 segment 最大约 128MB

64-bit:
  DSA_OFFSET_WIDTH = 40
  最多 1024 个 segment，每个 segment 最大约 1TB
```

`InvalidDsaPointer` 是 0。任何真实分配返回的 `dsa_pointer` 都必须用：

```text
DsaPointerIsValid(dp)
```

检查后再使用。

最重要的边界：

```text
dsa_pointer 可以跨进程传递；
void * 不可以跨进程传递。
```

`dsa_pointer` 也不能脱离所属 `dsa_area` 使用，因为 segment index 只在该 area 的 segment handle 表中有意义。

### 4.2 `dsa_handle`

`dsa_handle` 是进程间 attach 一个 area 的 handle：

```text
typedef dsm_handle dsa_handle;
```

源码注释提醒：当前实现中它是第一个 DSM segment 的 `dsm_handle`，但调用者不应该依赖这个实现细节。

调用关系是：

```text
creator:
  area = dsa_create(...)
  handle = dsa_get_handle(area)
  把 handle 存到共享状态或传给 worker

attacher:
  area = dsa_attach(handle)
```

`dsa_handle` 只解决“如何找到 area 的控制 segment”。真正对象地址仍然要靠 `dsa_pointer`。

### 4.3 共享的 `dsa_area_control`

`dsa_area_control` 位于 area 的第一个 segment 开头。它是所有 backend 共享的控制块：

| 字段 | 语义 |
| --- | --- |
| `segment_header` | 第一个 segment 的 header，也包含 magic / size / bin / freed 状态。 |
| `handle` | area handle。 |
| `segment_handles[]` | area 拥有的 DSM segment handle 表。 |
| `segment_bins[]` | 按最大连续空闲页分类的 segment 链表。 |
| `pools[]` | 各 size class 的 object pool。 |
| `init_segment_size` / `max_segment_size` | 新 segment 增长策略边界。 |
| `total_segment_size` / `max_total_segment_size` | 当前 backing storage 总大小和上限。 |
| `high_segment_index` | area 历史上用到过的最高 segment index。 |
| `refcnt` | attach / release 生命周期计数。 |
| `pinned` | area 是否即使没有 backend attach 也保持存在。 |
| `freed_segment_counter` | segment 被释放次数，用于发现本地 stale mapping。 |
| `lwlock_tranche_id` / `lock` | DSA locks 所属 tranche 和全局 area lock。 |

这里要强调：

```text
dsa_area_control 是共享状态；
struct dsa_area 是 backend-local 状态。
```

不要把两者混成一个对象。

### 4.4 backend-local `dsa_area`

每个 backend attach 一个 area 后，都会有自己的 `struct dsa_area`：

| 字段 | 语义 |
| --- | --- |
| `control` | 指向共享 `dsa_area_control`，当前 backend 的本地地址。 |
| `resowner` | 当前 backend 中 DSM mappings 的 ResourceOwner；`dsa_pin_mapping()` 后为 NULL。 |
| `segment_maps[]` | 当前 backend 对各 segment 的本地映射表。 |
| `high_segment_index` | 当前 backend 曾经映射过的最高 segment index。 |
| `freed_segment_counter` | 当前 backend 上次观察到的 freed counter。 |

`segment_maps[]` 是理解 DSA 的关键。每个 slot 保存：

| 字段 | 语义 |
| --- | --- |
| `segment` | 本 backend attach 的 `dsm_segment *`。 |
| `mapped_address` | 该 segment 在本 backend 的虚拟地址。 |
| `header` | 该 segment 的 header 地址。 |
| `fpm` | 该 segment 内的 FreePageManager 地址。 |
| `pagemap` | 该 segment 内 page -> span 的映射表地址。 |

这说明：

```text
dsa_pointer -> segment index + offset
segment_maps[index].mapped_address -> 本 backend 的 base address
最终地址 = mapped_address + offset
```

同一个 `dsa_pointer` 在不同 backend 中会转成不同的 `void *`，但指向同一个共享对象。

## 5. 主流程源码 walkthrough

### 5.1 `dsa_create()`：创建一个 area，而不是一个普通 segment

宏入口：

```text
dsa_create(tranche_id)
  -> dsa_create_ext(tranche_id,
                    DSA_DEFAULT_INIT_SEGMENT_SIZE,
                    DSA_MAX_SEGMENT_SIZE)
```

`dsa_create_ext()` 先创建第一个 DSM segment：

```text
segment = dsm_create(init_segment_size, 0)
dsm_pin_segment(segment)
```

为什么要 pin segment？

```text
DSA area 会显式管理它拥有的 segment 生命周期；
如果某个 segment 因为当前唯一映射 backend 退出而被 DSM 自动销毁，
area 的 segment_handles[] 中就会留下指向不存在 segment 的状态。
```

随后调用：

```text
create_internal(dsm_segment_address(segment),
                init_segment_size,
                tranche_id,
                dsm_segment_handle(segment),
                segment,
                init_segment_size,
                max_segment_size)
```

这一步在第一个 segment 起点初始化 `dsa_area_control`，并创建当前 backend 的 `dsa_area`。

最后注册 DSM detach callback：

```text
on_dsm_detach(segment,
              dsa_on_dsm_detach_release_in_place,
              dsm_segment_address(segment))
```

这让控制 segment detach 时自动 release area refcount。

### 5.2 `create_internal()`：建立共享 control 和本地 area

`create_internal()` 做两类初始化。

共享 control：

```text
control = place
control->segment_header.magic = DSA_SEGMENT_HEADER_MAGIC ^ handle ^ 0
control->handle = control_handle
control->segment_handles[0] = control_handle
control->refcnt = 1
control->total_segment_size = size
control->max_total_segment_size = (size_t) -1
control->lwlock_tranche_id = tranche_id
LWLockInitialize(&control->lock, tranche_id)
初始化各 size class pool lock
```

backend-local area：

```text
area = palloc_object(dsa_area)
area->control = control
area->resowner = CurrentResourceOwner
area->segment_maps[0].segment = control_segment
area->segment_maps[0].mapped_address = place
area->segment_maps[0].fpm = place + metadata offset
area->segment_maps[0].pagemap = place + metadata offset
```

所以创建者并不是“拿到共享 control 指针”就结束了。它还需要自己的 local `dsa_area`，因为本 backend 需要知道各 segment 映射到哪里。

### 5.3 `dsa_get_handle()`：把 area 暴露给其它 backend

创建者调用：

```text
handle = dsa_get_handle(area)
```

这个 handle 可以放到：

```text
static shared memory
DSM 中的 TOC object
另一个共享结构的 control block
```

例子一：`async.c` 中 LISTEN/NOTIFY 的 global channel table：

```text
globalChannelDSA = dsa_create(...)
dsa_pin(globalChannelDSA)
dsa_pin_mapping(globalChannelDSA)
asyncQueueControl->globalChannelTableDSA =
  dsa_get_handle(globalChannelDSA)
```

其它 backend 之后用：

```text
globalChannelDSA =
  dsa_attach(asyncQueueControl->globalChannelTableDSA)
```

例子二：`dsm_registry.c` 中 named DSM registry：

```text
dsm_registry_dsa = dsa_create(...)
dsa_pin(dsm_registry_dsa)
dsa_pin_mapping(dsm_registry_dsa)
DSMRegistryCtx->dsah = dsa_get_handle(dsm_registry_dsa)
```

这里 DSA handle 是长期共享基础设施的入口。

### 5.4 `dsa_attach()`：从 handle attach 控制 segment

`dsa_attach(handle)` 的第一步是：

```text
segment = dsm_attach(handle)
```

因为当前实现中 area handle 指向第一个 DSM segment。attach 成功后：

```text
area = attach_internal(dsm_segment_address(segment), segment, handle)
```

`attach_internal()` 校验：

```text
control->handle == handle
control->segment_handles[0] == handle
magic == DSA_SEGMENT_HEADER_MAGIC ^ handle ^ 0
```

然后创建本 backend 的 `dsa_area`，并设置 segment 0 的本地 map：

```text
area->control = control
area->resowner = CurrentResourceOwner
segment_maps[0].segment = segment
segment_maps[0].mapped_address = place
segment_maps[0].fpm = ...
segment_maps[0].pagemap = ...
```

随后在 area lock 下：

```text
如果 refcnt == 0:
  ERROR "could not attach to dynamic shared area"
否则:
  refcnt++
  area->freed_segment_counter = control->freed_segment_counter
```

这里的 refcount 保护的是 area 生命周期，不是单个 allocation 的引用计数。

### 5.5 `dsa_create_in_place()`：在已有共享空间中放置 control

parallel query 常用 in-place DSA。`execParallel.c` 中 leader 已经有一个 parallel DSM，于是：

```text
area_space = shm_toc_allocate(pcxt->toc, dsa_minsize)
shm_toc_insert(pcxt->toc, PARALLEL_KEY_DSA, area_space)
pei->area = dsa_create_in_place(area_space,
                                dsa_minsize,
                                LWTRANCHE_PARALLEL_QUERY_DSA,
                                pcxt->seg)
```

worker attach 时：

```text
area_space = shm_toc_lookup(toc, PARALLEL_KEY_DSA, false)
area = dsa_attach_in_place(area_space, seg)
```

这种模式下，DSA control object 放在 parallel DSM 的一个 chunk 中：

```text
TOC 负责找到 DSA control 的位置；
DSA 负责在这个 control 之下动态分配更多共享对象；
必要时 DSA 仍可以继续创建额外 DSM segment。
```

`dsa_create_in_place()` 不能用 `dsa_get_handle()` 暴露 area，因为它没有独立控制 segment handle。调用者必须通过其它协议知道 `place`，比如 parallel query 用 TOC key。

### 5.6 `dsa_allocate()`：返回的是 `dsa_pointer`

上层调用：

```text
dp = dsa_allocate(area, size)
```

返回值不是 `void *`，而是 `dsa_pointer`。

小对象和大对象的 allocator 细节留到下一节。本节只需要理解这个边界：

```text
dsa_allocate() 选定一个 segment 和 offset；
把它编码成 dsa_pointer；
调用者可以把 dsa_pointer 存入共享结构。
```

比如 `dshash_create()`：

```text
control = dsa_allocate(area, sizeof(dshash_table_control))
hash_table->control = dsa_get_address(area, control)
hash_table->control->handle = control
hash_table->control->buckets =
  dsa_allocate_extended(area,
                        sizeof(dsa_pointer) * DSHASH_NUM_PARTITIONS,
                        DSA_ALLOC_NO_OOM | DSA_ALLOC_ZERO)
```

共享 hash table control 中保存的 handle 仍然是 `dsa_pointer`。其它 backend attach hash table 时，再用 `dsa_get_address()` 得到本地地址。

### 5.7 `dsa_get_address()`：把 pseudo pointer 转成本地地址

`dsa_get_address(area, dp)` 是本节最核心函数：

```text
if dp == InvalidDsaPointer:
  return NULL

check_for_freed_segments(area)

index = DSA_EXTRACT_SEGMENT_NUMBER(dp)
offset = DSA_EXTRACT_OFFSET(dp)

if segment_maps[index].mapped_address == NULL:
  get_segment_by_index(area, index)

return segment_maps[index].mapped_address + offset
```

它做了三件事：

```text
1. 把 InvalidDsaPointer 转成 NULL。
2. 检查是否有其它 backend 释放过 segment，需要先清掉本地 stale mapping。
3. 如果目标 segment 当前 backend 尚未 attach，就按 segment index 找 handle 并 attach。
```

`get_segment_by_index()` 从共享 control 中取：

```text
handle = area->control->segment_handles[index]
segment = dsm_attach(handle)
segment_maps[index].mapped_address = dsm_segment_address(segment)
```

所以 DSA segment mapping 是 lazy 的：

```text
attach area 时只一定映射 control segment；
访问某个 dsa_pointer 时，才可能 attach 该 pointer 所在的后续 segment。
```

### 5.8 普通 pointer 只适合短期本地缓存

很多上层结构会同时保存：

```text
dsa_pointer shared_handle;
void *local_pointer;
```

例如 `dshash_table`：

```text
control handle:
  dsa_pointer，可传给其它 backend。

hash_table->control:
  本 backend local pointer，方便当前 backend 访问。
```

这是一种常见模式：

```text
共享结构中保存 dsa_pointer；
backend-local wrapper 中缓存 void *。
```

但必须记住：

```text
local pointer 不能写进共享内存；
local pointer 不能在另一个 backend 中使用；
如果对象被 dsa_free()，local pointer 也变成悬空引用。
```

DSA 不会为每个 `dsa_get_address()` 返回的地址提供自动 pin。对象是否仍然 live，是上层锁、引用计数或协议保证的。

## 6. 生命周期 / ownership / cleanup

DSA 生命周期要拆成四层看。

### 6.1 area backing segments

`dsa_create()` 创建第一个 DSM segment，并 `dsm_pin_segment()`。后续 `make_new_segment()` 也会：

```text
segment = dsm_create(...)
dsm_pin_segment(segment)
control->segment_handles[index] = dsm_segment_handle(segment)
```

这些 segment 是 area 的 backing storage。它们的销毁由 DSA 控制，而不是由某个 backend 的普通 DSM mapping 自然结束控制。

### 6.2 area refcount

`dsa_area_control->refcnt` 表示 area attach / release 生命周期：

```text
create_internal():
  refcnt = 1

attach_internal():
  refcnt++

dsa_release_in_place():
  refcnt--
  如果 refcnt == 0:
    unpin area 拥有的所有 segment
```

`dsa_release_in_place()` 会作为 DSM detach callback 被调用：

```text
dsa_on_dsm_detach_release_in_place()
  -> dsa_release_in_place(place)
```

对于 in-place area，如果没有提供 containing DSM segment，就需要调用者显式 release，否则可能永久泄漏 segment。

### 6.3 backend-local mapping

`dsa_detach(area)` 做的是当前 backend 的 mapping cleanup：

```text
for each mapped segment:
  dsm_detach(segment)
pfree(area)
```

源码注释强调：

```text
detaching DSM segments
  不等于
adjusting area refcount
```

refcount release 是通过控制 segment 的 detach callback 或显式 `dsa_release_in_place()` 完成的。这个拆分有点绕，但它避免在错误路径里依赖调用者一定会走到显式 `dsa_detach()`。

### 6.4 mapping pin 和 area pin

`dsa_pin_mapping(area)`：

```text
把本 backend 的 area mapping 从当前 ResourceOwner 生命周期提升到 session / explicit detach 生命周期；
对已经映射的 DSM segment 调用 dsm_pin_mapping()；
area->resowner = NULL。
```

`dsa_pin(area)`：

```text
在共享 control 中设置 pinned = true；
refcnt++；
即使所有 backend detach，area 仍可通过 handle 重新 attach。
```

两者名字很像，但含义不同：

```text
dsa_pin_mapping:
  本 backend 的 mapping 生命周期。

dsa_pin:
  共享 area 本身的生命周期。
```

`async.c` 和 `dsm_registry.c` 都会同时使用：

```text
dsa_pin(area)
dsa_pin_mapping(area)
```

因为它们需要长期存在的共享 registry，并且当前 backend 也要长期持有 mapping。

## 7. 正确性机制层次

| 层次 | 机制 | 保护的问题 |
| --- | --- | --- |
| 跨进程指针 | `dsa_pointer = segment index + offset` | 避免把本地 `void *` 传播给其它 backend。 |
| area 发现 | `dsa_handle` / TOC in-place address | 让其它 backend 找到 DSA control object。 |
| 本地地址转换 | backend-local `segment_maps[]` | 同一 pointer 在不同 backend 转成本地虚拟地址。 |
| 懒 attach | `get_segment_by_index()` | 只在访问某 segment 时 attach 对应 DSM。 |
| 生命周期 | `refcnt` / `dsa_release_in_place()` | area 不再被引用时 unpin backing segments。 |
| mapping 生命周期 | `ResourceOwner` / `dsa_pin_mapping()` | ERROR 或事务边界时清理本 backend mappings。 |
| 长期共享 | `dsa_pin()` | 没有 backend attach 时 area 仍可保留。 |
| stale mapping 防护 | `freed_segment_counter` / `check_for_freed_segments()` | segment slot 复用前先让 backend detach 旧 mapping。 |
| 并发 allocator | area LWLock / pool LWLock | 保护 segment table、free page manager、size class pool。 |

本节最重要的 correctness boundary：

```text
dsa_get_address() 只负责地址转换；
不负责证明对象还活着。
```

对象 live-ness 必须由上层协议保证。例如：

```text
dshash 持有 partition lock 时遍历 item；
parallel hash join 用 barrier 和 batch 状态管理共享对象；
tidbitmap shared iterator 有自己的迭代协议。
```

## 8. 错误路径 / 异常路径 / fallback

### 8.1 `dsa_attach()` 找不到 area

`dsa_attach(handle)` 内部调用：

```text
dsm_attach(handle)
```

如果失败：

```text
ERROR "could not attach to dynamic shared area"
```

常见原因是 handle 指向的控制 segment 已经不存在，或者 area 已经被释放。

### 8.2 attach 时 refcount 已经为 0

`attach_internal()` 在 area lock 下检查：

```text
if control->refcnt == 0:
  ERROR
```

这表示 area 已经进入 destroyed 边界。即使还能看到某段内存，也不能把它当作可 attach area。

### 8.3 OOM 与 allocation flags

`dsa_allocate_extended()` 支持：

```text
DSA_ALLOC_HUGE
DSA_ALLOC_NO_OOM
DSA_ALLOC_ZERO
```

其中：

```text
DSA_ALLOC_NO_OOM:
  失败返回 InvalidDsaPointer，不 ERROR。

默认:
  OOM 时 ereport(ERROR)。
```

上层必须检查 `DsaPointerIsValid()`。`dshash_create()` 就会在 bucket array 分配失败时释放 control object 并报 OOM。

### 8.4 `dsa_get_address()` 访问已释放对象

源码注释很明确：

```text
如果调用者传入的 dsa_pointer 对象已经不 live，
这是调用者 bug，行为未定义。
```

DSA 能防一类问题：

```text
segment slot 被释放后又被新 segment 复用；
当前 backend 仍映射旧 segment。
```

为此 `dsa_get_address()` 先调用 `check_for_freed_segments()`。但它不能防：

```text
同一 segment 内对象已经 dsa_free；
上层还拿着旧 dsa_pointer 访问。
```

### 8.5 in-place area 没有自动 release

`dsa_create_in_place()` 的注释提醒：

```text
如果 place 在 DSM 中，可以传入 segment 注册 on_dsm_detach callback；
否则调用者必须用 dsa_release_in_place 或 on_shmem_exit callback 管理 release。
```

否则 area 创建的额外 DSM segment 可能永久泄漏。

## 9. 成本、资源与跨模块传播

DSA 的成本来自三个层次：

| 成本 | 场景 | 影响 |
| --- | --- | --- |
| pointer 转换 | 每次 `dsa_get_address()` | 需要拆 segment index / offset，可能检查 freed counter。 |
| lazy segment attach | 首次访问某 segment | 可能调用 `dsm_attach()`，进入 DSM mapping 和 ResourceOwner 路径。 |
| allocator locks | `dsa_allocate()` / `dsa_free()` | area lock 或 size class pool lock，具体竞争留到第 33 节。 |
| segment 增长 | 空间不足时 `make_new_segment()` | 创建并 pin 新 DSM segment，更新 shared segment handle table。 |
| local pointer cache | 上层缓存 `void *` | 减少重复 `dsa_get_address()`，但需要上层保证对象 live。 |

跨模块传播例子：

```text
execParallel.c:
  TOC 中只放 DSA control address；
  executor 参数和 worker 共享对象放进 DSA。

dshash.c:
  hash table control、bucket array、item 链表都用 dsa_pointer 连接。

async.c / dsm_registry.c:
  pinned DSA + dshash 构成长生命周期共享 registry。

tidbitmap.c:
  shared bitmap iterator state 和 page/chunk arrays 放入 DSA。
```

这说明 DSA 是很多高层共享数据结构的“内存底座”，但不是并发语义本身。并发读写仍由上层 hash lock、barrier、LWLock 或业务协议负责。

## 10. 观测与诊断入口

### 10.1 gdb 断点

适合断点：

```text
break dsa_create_ext
break dsa_create_in_place_ext
break dsa_attach
break attach_internal
break dsa_allocate_extended
break dsa_get_address
break get_segment_by_index
break make_new_segment
break dsa_release_in_place
break dsa_detach
```

关键字段：

```text
p area->control->handle
p area->control->refcnt
p area->control->pinned
p area->control->total_segment_size
p area->control->high_segment_index
p area->control->segment_handles[0]
p area->high_segment_index
p area->segment_maps[index].mapped_address
p area->freed_segment_counter
p area->control->freed_segment_counter
```

观察 `dsa_pointer`：

```text
p/x dp
p DSA_EXTRACT_SEGMENT_NUMBER(dp)
p DSA_EXTRACT_OFFSET(dp)
```

如果宏在 gdb 中不方便展开，可以手工按 `DSA_OFFSET_WIDTH` 拆位。

### 10.2 判断是不是 stale pointer

当怀疑 DSA pointer 出错时，按这个顺序看：

```text
1. dp 是否为 InvalidDsaPointer。
2. dp 的 segment index 是否小于 DSA_MAX_SEGMENTS。
3. area->control->segment_handles[index] 是否有效。
4. 当前 backend 的 segment_maps[index] 是否已经 attach。
5. dsa_get_address() 前后 freed_segment_counter 是否变化。
6. 上层是否在对象 dsa_free 后还保存 dp。
```

第 6 点最难，因为 DSA 本身不保存 per-object refcount。

### 10.3 从上层现象回到 DSA

常见 runtime 现象：

```text
parallel query OOM:
  可能是 DSA area 增长失败或 size limit。

shared hash 表异常:
  先看 dshash lock / bucket，再看 bucket pointer 是否是有效 dsa_pointer。

worker attach 后访问共享参数失败:
  看 PARALLEL_KEY_DSA 是否 lookup 正确，fpes->param_exec 是否有效。

长期 registry 内存持续增长:
  看 pinned DSA 是否有对象未释放，而不是期待事务结束自动清理。
```

## 11. 常见误区

误区一：`dsa_pointer` 是可以直接解引用的指针。

不是。它只是 segment index + offset 的编码。必须通过 `dsa_get_address(area, dp)` 转成本 backend 的地址。

误区二：`dsa_get_address()` 返回的 `void *` 可以存进共享内存。

不可以。返回值只在当前 backend 当前 mapping 下有意义。

误区三：`dsa_handle` 和 `dsa_pointer` 是同一种东西。

不是。`dsa_handle` 用来 attach 整个 area；`dsa_pointer` 用来定位 area 内的 allocation。

误区四：attach area 会 attach 所有 segment。

不是。通常先 attach control segment。其它 segment 在 `dsa_get_address()` 或 allocator 查找时按需 attach。

误区五：`dsa_detach()` 一定等价于销毁 area。

不是。它释放当前 backend 的 mappings 和本地 area object。area 是否销毁取决于 shared refcount、pin 状态和 segment unpin。

误区六：DSA 能自动判断对象是否仍然 live。

不能。DSA 能避免一部分 segment slot stale mapping 问题，但对象级 live-ness 由上层同步协议保证。

误区七：DSA 替代了 `shm_toc`。

不完全。`shm_toc` 常用来 bootstrap DSA control 或 root object；DSA 用来在 root object 下动态分配更多对象。

## 12. 课堂实验

### 实验一：观察 parallel query 的 in-place DSA

在 `execParallel.c` 中设置断点：

```text
break dsa_create_in_place_ext
break dsa_attach_in_place
break dsa_get_address
```

运行一个 parallel query，观察：

```text
leader:
  area_space 来自 shm_toc_allocate
  PARALLEL_KEY_DSA 插入 TOC

worker:
  shm_toc_lookup(PARALLEL_KEY_DSA)
  dsa_attach_in_place(area_space, seg)
```

目标：理解 TOC 和 DSA 的分工。

### 实验二：拆一个 `dsa_pointer`

在 `dsa_allocate_extended()` 返回处记录：

```text
dp
segment index
offset
```

然后在另一个 backend 的 `dsa_get_address()` 中观察：

```text
同一个 dp
不同 mapped_address
不同 void *
```

目标：亲眼看到 pseudo pointer 跨进程稳定，本地 pointer 不稳定。

### 实验三：观察 lazy attach 后续 segment

构造足够大的 DSA allocation，让 area 创建新 segment。断点：

```text
break make_new_segment
break get_segment_by_index
```

观察：

```text
control->segment_handles[new_index] 被写入
另一个 backend 首次访问该 segment 的 dp 时才 dsm_attach(handle)
segment_maps[index].mapped_address 从 NULL 变为本地地址
```

目标：理解 segment map 是 backend-local cache。

### 实验四：比较 `dsa_pin()` 和 `dsa_pin_mapping()`

参考 `async.c` 或 `dsm_registry.c`：

```text
dsa_pin(area)
dsa_pin_mapping(area)
```

分别观察：

```text
dsa_pin:
  control->pinned = true
  control->refcnt++

dsa_pin_mapping:
  area->resowner = NULL
  已映射 segment 调用 dsm_pin_mapping()
```

目标：区分共享 area 生命周期和当前 backend mapping 生命周期。

### 实验五：找一个上层 root object

读 `dshash_create()`：

```text
control = dsa_allocate(...)
hash_table->control = dsa_get_address(area, control)
hash_table->control->handle = control
bucket array = dsa_allocate_extended(...)
```

画出：

```text
backend-local dshash_table
  -> local pointer to shared control

shared dshash_table_control
  -> dsa_pointer bucket array
  -> dsa_pointer item chain
```

目标：理解“共享结构保存 dsa_pointer，本地 wrapper 缓存 void *”这一常见模式。

## 13. 讨论题

1. 为什么 `dsa_pointer` 必须包含 segment index，而不能只保存 offset？
2. 为什么 `dsa_pointer` 不能脱离所属 `dsa_area` 解释？
3. 如果把 `dsa_get_address()` 返回的 `void *` 放进共享 hash item，会在哪些构建或运行环境下出错？
4. `dsa_handle` 当前实现是第一个 DSM segment handle，为什么 API 注释仍提醒调用者不要依赖这个事实？
5. 为什么 `dsa_pin()` 和 `dsa_pin_mapping()` 必须拆成两个动作？
6. DSA 为什么不为每个 allocation 提供自动 refcount？这样做会给 allocator hot path 带来什么成本？
7. `shm_toc`、`dsa_area`、`dshash` 三者分别解决哪一层问题？

## 14. 本节小结

DSA 在 DSM 之上补上的不是“更大的 DSM”，而是一套跨进程共享 heap 模型：

```text
dsa_area_control:
  共享的 area 根状态和 segment handle 表。

backend-local dsa_area:
  当前 backend 的 segment mapping cache 和 ResourceOwner 边界。

dsa_handle:
  让其它 backend 找到 area。

dsa_pointer:
  让共享对象互相引用，却不暴露本地虚拟地址。

dsa_get_address():
  在当前 backend 中把 pseudo pointer 转成本地 pointer。
```

本节最重要的判断：

```text
dsa_pointer 是可共享的地址描述；
void * 是当前 backend 的临时解释结果。
```

可迁移规律：

```text
只要共享内存可能映射到不同虚拟地址，跨进程数据结构就不能保存普通指针；
它必须保存某种稳定的相对地址或句柄，
并在每个进程本地通过映射表重新解释。
```

下一节会继续向下看 DSA allocator：`dsa_allocate()` / `dsa_free()` 如何用 size class、span、superblock、FreePageManager 和 per-pool LWLock 在共享 heap 中管理小对象、大对象、碎片和锁竞争。
