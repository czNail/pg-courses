# PostgreSQL DSA allocator：size class、span 与 superblock

## 课程定位

前置知识：已经理解 DSA area 是建立在一个或多个 DSM segment 上的共享 heap，知道 `dsa_pointer` 是 segment index + offset，必须通过 `dsa_get_address()` 转成本 backend 的本地地址。

本节唯一主问题：

```text
dsa_allocate() / dsa_free()
如何把小对象、large allocation、FreePageManager、pagemap、span freelist
和 per-size-class LWLock 组合成一个共享 allocator，
哪些锁竞争和碎片化成本会随 workload 放大？
```

核心矛盾：DSA 要服务跨进程共享数据结构，它不能像 `palloc()` 那样只面对单 backend，也不能像 DSM 那样每个对象一个 segment。它必须在共享内存中支持大量小对象、偶发大对象、释放后复用、segment 增长和 segment 回收；同时 allocator 元数据本身也在共享内存里，需要并发保护。PostgreSQL 的选择是：小对象按 size class 放进 64KB superblock，由 span 管理 freelist；大对象直接按 page run 向 `FreePageManager` 申请；free 时用 pagemap 找回 span，再把对象或 page run 归还。

学完后应能判断：

```text
为什么小对象不直接向 FreePageManager 申请 4KB pages；
为什么 DSA 用 size class，而不是为每个请求保存精确大小；
为什么每个 superblock 需要一个 span 描述符；
为什么 pagemap 是 dsa_free() 找回对象归属的关键；
为什么 large allocation 仍然需要 span；
为什么已有 superblock 的小对象分配只拿 per-size-class lock；
为什么新 superblock 和 large allocation 会走 area lock；
为什么碎片、size class 热点和 segment slot 增长会随 workload 放大。
```

本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `2c4bd2bf5700`。

## 1. 本节在总主线中的位置

上一节回答的是 DSA 的地址模型：

```text
dsa_area_control:
  共享 root state

dsa_area:
  backend-local mapping cache

dsa_pointer:
  可跨进程传递的 pseudo pointer

dsa_get_address():
  当前 backend 内的地址解释
```

本节进入 allocator 内部：

```text
dsa_allocate(area, size)
  到底如何找到一块共享空间？

dsa_free(area, dp)
  又如何知道这块空间属于哪个 pool、哪个 superblock、该怎么归还？
```

这一课只讲 DSA allocator 的状态推进和成本边界，不展开上层数据结构如 `dshash`、parallel hash join 的业务协议。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
DSA 把每个 DSM segment 切成 4KB pages；
每个 segment 内用 FreePageManager 管理连续空闲 page run；
小对象按 size class 从 16-page / 64KB superblock 中切出；
每个 superblock 由 span 记录 size class、freelist 和 fullness class；
pagemap 把 page number 映射到 span pointer，让 dsa_free() 能找到归属；
大对象直接从 FreePageManager 申请连续 pages，并用特殊 span 描述。
```

本节 tension 是：

```text
小对象分配需要低锁、低碎片、可并发
  vs
共享 allocator 必须能从任意 dsa_pointer 反查归属并安全回收 page / segment
```

如果所有对象都直接走 page allocator：

```text
几十字节的 hash item 会消耗 4KB page；
空间浪费严重；
大量 page-level metadata 操作会集中在 area lock 上。
```

如果所有对象都用一个全局 freelist：

```text
不同大小对象混在一起；
释放时不知道该按哪个大小复用；
并发分配会集中争用一把锁。
```

DSA 的折中是：

```text
小对象:
  size class -> pool lock -> span -> superblock freelist

大对象:
  area lock -> segment bin -> FreePageManager page run

free:
  dsa_pointer -> segment_map -> pagemap -> span -> pool or FreePageManager
```

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 作用 |
| --- | --- | --- |
| 1 | `src/include/utils/dsa.h` | `dsa_allocate()` / `dsa_free()` API、allocation flags、`dsa_pointer`。 |
| 2 | `src/backend/utils/mmgr/dsa.c` | size class 表、span、pool、superblock、large allocation、segment growth / trim。 |
| 3 | `src/include/utils/freepage.h` | `FPM_PAGE_SIZE`、`FreePageManagerGet()` / `Put()` / `Dump()` 接口。 |
| 4 | `src/backend/utils/mmgr/freepage.c` | page run freelist、btree 合并相邻空闲范围、largest contiguous pages 维护。 |
| 5 | `src/backend/lib/dshash.c` | DSA 小对象和 bucket array 的典型用户。 |
| 6 | `src/backend/nodes/tidbitmap.c` | `DSA_ALLOC_HUGE` / `DSA_ALLOC_ZERO` 等 flags 的典型使用。 |

推荐阅读顺序：

```text
先读 dsa.c 顶部模块注释
  -> 读 dsa_size_classes[] 和 dsa_area_span
  -> 跟 dsa_allocate_extended() 的 large / small 分支
  -> 跟 alloc_object()
  -> 跟 ensure_active_superblock()
  -> 跟 dsa_free()
  -> 跟 destroy_superblock()
  -> 最后看 FreePageManager 的页面级职责
```

不要一开始就钻 `freepage.c` 的 btree。先记住它在 DSA 中的角色：

```text
给我 N 个连续 4KB pages；
我释放从 first_page 开始的 N 个 pages；
告诉我当前最大连续空闲 page run。
```

## 4. 关键数据结构与状态

### 4.1 size class 表

DSA 小对象走固定 size class：

```text
8, 16, 24, 32, ... 8192
```

`dsa_size_classes[]` 里前两个是特殊 class：

| class | 含义 |
| --- | --- |
| `DSA_SCLASS_BLOCK_OF_SPANS` | 用来分配 span descriptor 的特殊 pool。 |
| `DSA_SCLASS_SPAN_LARGE` | 用来管理 large allocation span 的特殊 pool。 |

普通小对象最大 size class 是 8192 bytes。超过最大 size class 的请求进入 large allocation 分支。

小于 1KB 的请求用 `dsa_size_class_map[]` 快速映射：

```text
把 size 向上按 8 bytes 对齐；
查表得到 size_class。
```

较大的小对象用 binary search 找到第一个能容纳 size 的 class。

这意味着：

```text
请求 33 bytes 可能落到 40-byte class；
请求 8200 bytes 不再是小对象，而是 large allocation。
```

内部碎片首先来自这个向上取整。

### 4.2 superblock

普通小对象不是逐个向 segment 申请 page，而是从 superblock 中切出：

```text
DSA_PAGES_PER_SUPERBLOCK = 16
FPM_PAGE_SIZE = 4096
DSA_SUPERBLOCK_SIZE = 64KB
```

一个普通 superblock 只服务一个 size class。它可以被切成：

```text
64KB / object_size
```

个对象。

为什么需要 superblock？

```text
小对象分配可以只在 pool lock 下从当前 active span 中取一个 slot；
不用每次都拿 area lock、找 segment、操作 FreePageManager。
```

代价是：

```text
一个 size class 至少按 superblock 粒度向 page allocator 要空间；
如果 workload 对很多 size class 只分配少量对象，会产生 superblock 级外部碎片。
```

### 4.3 span

`dsa_area_span` 是一个 superblock 或 large allocation 的描述符：

| 字段 | 语义 |
| --- | --- |
| `pool` | 所属 pool 的 `dsa_pointer`。 |
| `prevspan` / `nextspan` | fullness class 链表指针。 |
| `start` | superblock 或 large allocation 起点的 `dsa_pointer`。 |
| `npages` | 占用 page 数。 |
| `size_class` | 对象 size class；large allocation 使用特殊 class。 |
| `ninitialized` | 已经初始化过的对象数上界。 |
| `nallocatable` | 当前可分配对象数。 |
| `firstfree` | superblock 内第一个 free object index。 |
| `nmax` | 该 span 最多可分配对象数。 |
| `fclass` | 当前 fullness class。 |

span 的关键作用：

```text
alloc:
  从 span->firstfree 或 span->ninitialized 得到对象 index。

free:
  通过 pagemap 找到 span，再把对象 index 放回 span->firstfree。

reclaim:
  判断 span 是否整个 superblock 都空了，可以还给 FreePageManager。
```

### 4.4 span-of-spans

span 自己也是共享对象，也需要分配。普通 superblock 创建前需要先有 span descriptor；这会产生循环：

```text
要创建 superblock，需要 span；
要分配 span，又需要 superblock。
```

DSA 用特殊 size class `DSA_SCLASS_BLOCK_OF_SPANS` 解决这个 bootstrap：

```text
span-of-spans block 是一个 4KB page；
描述这个 block 的 span 放在该 page 开头 inline；
第一个对象不可用，因为它就是 span descriptor 本身。
```

这不是优雅小技巧，而是 allocator bootstrap 必需的自举边界。

### 4.5 pool 与 fullness class

每个 size class 对应一个 `dsa_area_pool`：

```text
LWLock lock
dsa_pointer spans[DSA_FULLNESS_CLASSES]
```

当前 `DSA_FULLNESS_CLASSES = 4`。这些链表大致表达 superblock 的使用程度：

```text
class 0:
  接近空

class 1:
  active allocation target

class 2:
  更满一些

class 3:
  完全满
```

源码特别说明：只有最高 class 严格表示“完全满”；其它 class 是启发式分组。active block 固定在 fullness class 1 的链表头，即使它实际已经比较满，也不会移动，直到完全满。

这样做的目的：

```text
优先从中等满的 block 分配；
尽量让接近空的 block 保持接近空，从而未来能整个释放。
```

### 4.6 pagemap

每个 DSA segment 都有 `pagemap`：

```text
page number -> span pointer
```

`dsa_free(area, dp)` 没有被传入 size，也没有被传入 span。它只能从 `dp` 反查：

```text
segment = DSA_EXTRACT_SEGMENT_NUMBER(dp)
offset = DSA_EXTRACT_OFFSET(dp)
pageno = offset / FPM_PAGE_SIZE
span_pointer = segment_map->pagemap[pageno]
```

没有 pagemap，free 时就不知道：

```text
这个对象属于哪个 size class？
该放回哪个 pool lock 保护的 freelist？
是不是 large allocation？
占了多少 pages？
```

pagemap 是 DSA free path 的导航表。

## 5. 主流程源码 walkthrough

### 5.1 `dsa_allocate_extended()` 入口分流

入口先校验 size：

```text
size > 0
如果没有 DSA_ALLOC_HUGE:
  用 AllocSizeIsValid(size)
如果有 DSA_ALLOC_HUGE:
  用 AllocHugeSizeIsValid(size)
```

然后分两路：

```text
if size > largest_size_class:
  large allocation
else:
  small object allocation
```

largest size class 当前是 8192 bytes。

allocation flags：

| flag | 行为 |
| --- | --- |
| `DSA_ALLOC_HUGE` | 允许 >= 1GB 的 huge request。 |
| `DSA_ALLOC_NO_OOM` | OOM 时返回 `InvalidDsaPointer`，不 `ERROR`。 |
| `DSA_ALLOC_ZERO` | 成功后把分配区域清零。 |

### 5.2 小对象：size -> size_class

小对象路径先映射 size class：

```text
if size < sizeof(dsa_size_class_map) * 8:
  mapidx = ceil(size / 8) - 1
  size_class = dsa_size_class_map[mapidx]
else:
  binary search dsa_size_classes[]
```

然后：

```text
result = alloc_object(area, size_class)
```

如果返回 `InvalidDsaPointer`：

```text
DSA_ALLOC_NO_OOM:
  return InvalidDsaPointer
否则:
  ERROR out of memory
```

如果 `DSA_ALLOC_ZERO`：

```text
memset(dsa_get_address(area, result), 0, size)
```

注意清零只清请求的 size，不是整个 size class slot。

### 5.3 `alloc_object()`：已有 superblock 上的 hot path

`alloc_object(area, size_class)` 先拿该 size class 的 pool lock：

```text
LWLockAcquire(DSA_SCLASS_LOCK(area, size_class), LW_EXCLUSIVE)
```

如果 `pool->spans[1]` 没有 active superblock：

```text
ensure_active_superblock(area, pool, size_class)
```

如果有 active span：

```text
span = dsa_get_address(area, pool->spans[1])
block = span->start
size = dsa_size_classes[size_class]

if span->firstfree != DSA_SPAN_NOTHING_FREE:
  result = block + firstfree * size
  object = dsa_get_address(area, result)
  span->firstfree = NextFreeObjectIndex(object)
else:
  result = block + ninitialized * size
  ninitialized++

nallocatable--
```

这就是小对象 hot path：

```text
一把 per-size-class LWLock；
一个 active span；
一个 freelist pop 或 bump ninitialized。
```

如果这个 span 被分配满：

```text
transfer_first_span(area, pool, 1, DSA_FULLNESS_CLASSES - 1)
```

也就是从 active class 移到 full class。

### 5.4 `ensure_active_superblock()`：选择或创建 active span

如果 class 1 没有 active span，DSA 先尝试复用已有 span。

第一步，扫描较高 fullness class：

```text
for fclass = 2 .. DSA_FULLNESS_CLASSES - 2:
  重新计算这个 span 理应属于哪个 fullness class
  如果利用率已经下降，就移动到更低 class
```

如果这样找到了 class 1，直接返回。

第二步，如果仍然没有 class 1：

```text
从 class 2..非 full class 转一个 span 到 class 1；
最后才从 class 0 转一个几乎空的 span 到 class 1。
```

源码注释解释了偏好：

```text
通常从中等满的 block 分配更好；
尽量避免从快要完全空的 block 分配，因为它可能很快能整个释放。
```

第三步，如果没有任何现成 span 可用，创建新 superblock。

普通 size class 创建新 superblock 前，要先分配 span descriptor：

```text
span_pointer = alloc_object(area, DSA_SCLASS_BLOCK_OF_SPANS)
npages = DSA_PAGES_PER_SUPERBLOCK
```

然后拿 area lock：

```text
LWLockAcquire(DSA_AREA_LOCK(area), LW_EXCLUSIVE)
segment_map = get_best_segment(area, npages)
if segment_map == NULL:
  segment_map = make_new_segment(area, npages)
FreePageManagerGet(segment_map->fpm, npages, &first_page)
LWLockRelease(DSA_AREA_LOCK(area))
```

最后：

```text
start_pointer = DSA_MAKE_POINTER(segment_index, first_page * FPM_PAGE_SIZE)
init_span(area, span_pointer, pool, start_pointer, npages, size_class)
for each page in superblock:
  pagemap[first_page + i] = span_pointer
```

现在 `pool->spans[1]` 有 active span，`alloc_object()` 可以继续从中切对象。

### 5.5 large allocation：直接申请 page run

如果请求大于最大 size class：

```text
npages = fpm_size_to_pages(size)
```

large allocation 仍然需要一个 span descriptor：

```text
span_pointer = alloc_object(area, DSA_SCLASS_BLOCK_OF_SPANS)
```

然后拿 area lock，选择 segment：

```text
segment_map = get_best_segment(area, npages)
if NULL:
  segment_map = make_new_segment(area, npages)
FreePageManagerGet(segment_map->fpm, npages, &first_page)
```

构造起始 pointer：

```text
start_pointer = DSA_MAKE_POINTER(segment_index,
                                 first_page * FPM_PAGE_SIZE)
```

用 large pool 初始化特殊 span：

```text
LWLockAcquire(DSA_SCLASS_LOCK(area, DSA_SCLASS_SPAN_LARGE))
init_span(area, span_pointer, large_pool,
          start_pointer, npages, DSA_SCLASS_SPAN_LARGE)
pagemap[first_page] = span_pointer
LWLockRelease(...)
```

为什么 large allocation 仍需要 span？

```text
dsa_free(dp) 仍然只拿到 dp；
它必须通过 pagemap 找到 span；
span 告诉它这是 large allocation，以及 npages 是多少。
```

large allocation 不用普通 superblock freelist，但复用同一套“pagemap -> span -> free policy”模型。

### 5.6 `dsa_free()`：从 pagemap 找回归属

free 入口：

```text
dsa_free(area, dp)
```

先处理 stale segment：

```text
check_for_freed_segments(area)
```

然后从 dp 反查：

```text
segment_map = get_segment_by_index(area, segment_index(dp))
pageno = offset(dp) / FPM_PAGE_SIZE
span_pointer = segment_map->pagemap[pageno]
span = dsa_get_address(area, span_pointer)
superblock = dsa_get_address(area, span->start)
object = dsa_get_address(area, dp)
size_class = span->size_class
```

这一步解释了 pagemap 的必要性：free 不需要调用者传 size，也不需要上层记 pool。

### 5.7 free small object：归还到 span freelist

如果不是 large span，free 进入 size class pool：

```text
LWLockAcquire(DSA_SCLASS_LOCK(area, size_class), LW_EXCLUSIVE)
```

先把对象放回 span freelist：

```text
NextFreeObjectIndex(object) = span->firstfree
span->firstfree = (object - superblock) / size
span->nallocatable++
```

这里的 free list 指针直接写在被释放对象的开头。这也是很多 allocator 的常见做法：释放对象后，它的 payload 已经不再属于调用者，可以用作 allocator metadata。

然后判断 fullness class：

```text
如果这个 span 刚从 full 变成非 full:
  从 full class 移到较低 class

如果整个 span 都空了，并且不是唯一 active block:
  destroy_superblock()
```

源码特别避免对 active block 过度释放：

```text
如果唯一 active block 反复只分配/释放一个对象，
每次都 destroy/recreate 会产生 hysteresis。
```

所以 DSA 会保留 active block，除非满足释放条件。

### 5.8 free large object：直接归还 pages

如果 `span->size_class == DSA_SCLASS_SPAN_LARGE`：

```text
LWLockAcquire(DSA_AREA_LOCK(area), LW_EXCLUSIVE)
FreePageManagerPut(segment_map->fpm,
                   offset(span->start) / FPM_PAGE_SIZE,
                   span->npages)
rebin_segment(area, segment_map)
LWLockRelease(DSA_AREA_LOCK(area))
```

然后从 large span 链表中 unlink：

```text
LWLockAcquire(DSA_SCLASS_LOCK(area, DSA_SCLASS_SPAN_LARGE))
unlink_span(area, span)
LWLockRelease(...)
```

最后释放 span descriptor：

```text
dsa_free(area, span_pointer)
```

所以 large allocation free 会触发两层回收：

```text
用户大对象 pages -> FreePageManager
large span descriptor -> span-of-spans pool
```

### 5.9 `destroy_superblock()`：superblock -> FreePageManager -> maybe segment

当某个普通 superblock 整体空了，`destroy_superblock()` 被调用。前提是调用者已经持有对应 pool lock。

先从 fullness class 链表中移除 span：

```text
unlink_span(area, span)
```

然后拿 area lock：

```text
check_for_freed_segments_locked(area)
segment_map = get_segment_by_index(area, segment_index(span->start))
FreePageManagerPut(segment_map->fpm,
                   offset(span->start) / FPM_PAGE_SIZE,
                   span->npages)
```

如果整个 segment 都空了：

```text
if fpm_largest(segment_map->fpm) == segment_map->header->usable_pages:
  if index != 0:
    unlink_segment(area, segment_map)
    header->freed = true
    total_segment_size -= header->size
    dsm_unpin_segment(handle)
    dsm_detach(segment)
    segment_handles[index] = DSM_HANDLE_INVALID
    freed_segment_counter++
```

segment 0 不释放，因为它包含 area control object。

最后，如果这个 span 不是 span-of-spans inline descriptor：

```text
dsa_free(area, span_pointer)
```

这会递归释放 span descriptor 本身。锁顺序上源码明确约束：

```text
持某个普通 pool lock
  -> 可拿 area lock
  -> 可递归释放 span pool 对象

避免反向拿锁，防止死锁。
```

## 6. 生命周期 / ownership / cleanup

### 6.1 对象生命周期

一个小对象的生命周期：

```text
dsa_allocate(size)
  -> size class
  -> pool lock
  -> active span
  -> 返回 dsa_pointer

上层把 dsa_pointer 存入共享结构

dsa_free(dp)
  -> pagemap 找 span
  -> pool lock
  -> object index 放回 freelist
  -> span 可能移动 fullness class
  -> superblock 可能归还给 FPM
```

DSA 不维护 per-object refcount。上层必须保证对象没人再访问时才调用 `dsa_free()`。

### 6.2 superblock 生命周期

superblock 生命周期：

```text
ensure_active_superblock()
  -> 从 FPM 申请 16 pages
  -> init_span()
  -> pagemap[page] = span_pointer
  -> pool->spans[1] 持有 span

alloc/free 反复改变 nallocatable / firstfree / fullness class

destroy_superblock()
  -> 从 pool 链表移除
  -> pages 归还 FPM
  -> span descriptor 释放
```

superblock 是小对象 allocator 和 page allocator 之间的批量单位。

### 6.3 segment 生命周期

segment 生命周期：

```text
make_new_segment()
  -> dsm_create()
  -> dsm_pin_segment()
  -> control->segment_handles[index] = handle
  -> 初始化 FPM / pagemap / segment header
  -> 放入 segment_bins

destroy_superblock()
  -> 如果 segment 全空且不是 segment 0:
       dsm_unpin_segment()
       dsm_detach()
       segment_handles[index] = invalid
       freed_segment_counter++
```

segment slot 可复用，因此 `freed_segment_counter` 是跨 backend stale mapping 防线。`dsa_get_address()` 和 `dsa_free()` 都会先调用 `check_for_freed_segments()`。

## 7. 正确性机制层次

| 层次 | 机制 | 保护的问题 |
| --- | --- | --- |
| size class | `dsa_size_classes[]` / lookup table | free 时不需要保存精确请求大小。 |
| 小对象并发 | per-size-class `LWLock` | 同一 size class 的 span freelist 串行化，不同 size class 可并发。 |
| page run 并发 | area `LWLock` | segment bins、FPM、segment handle table 串行化。 |
| 对象归属 | segment `pagemap` | 从任意 `dsa_pointer` 找到对应 span。 |
| superblock 元数据 | `dsa_area_span` | 记录 start、npages、size_class、freelist、fullness。 |
| large allocation | special span class | 大对象也能用统一 free path 找到 npages。 |
| segment 回收 | `freed_segment_counter` | slot 复用前让其它 backend detach stale mapping。 |
| lock ordering | pool lock -> area lock；避免 area lock -> pool lock | 防止 allocator 内部死锁。 |
| OOM 语义 | `DSA_ALLOC_NO_OOM` / default ERROR | 允许上层选择返回 invalid pointer 或直接 ERROR。 |

需要特别记住：

```text
dsa_free() 的正确性不是来自调用者告诉 allocator size；
而是来自 pagemap + span。
```

这也是 DSA 能提供简单 API 的代价：每个 segment 需要维护 page-level pagemap。

## 8. 错误路径 / 异常路径 / fallback

### 8.1 size 非法

`dsa_allocate_extended()` 先检查 size：

```text
没有 DSA_ALLOC_HUGE:
  AllocSizeIsValid(size)

有 DSA_ALLOC_HUGE:
  AllocHugeSizeIsValid(size)
```

非法 size 直接 `elog(ERROR)`。

### 8.2 无法分配 span descriptor

large allocation 和新普通 superblock 都需要 span descriptor。如果：

```text
alloc_object(area, DSA_SCLASS_BLOCK_OF_SPANS)
```

失败，普通 allocation 或 large allocation 都无法继续。

这说明 span pool 是 allocator 的基础设施；它本身也可能成为 OOM 边界。

### 8.3 找不到足够 page run

新 superblock 或 large allocation 需要连续 pages：

```text
get_best_segment(area, npages)
make_new_segment(area, npages)
```

如果两个都失败：

```text
DSA_ALLOC_NO_OOM:
  return InvalidDsaPointer
否则:
  ERROR out of memory
```

失败原因可能是：

```text
达到 max_total_segment_size；
达到 DSA_MAX_SEGMENTS；
单 segment 最大大小限制；
没有足够连续空闲 pages 且不能创建新 segment。
```

### 8.4 FreePageManagerGet 理论上不应失败

在已经找到合适 segment 后：

```text
FreePageManagerGet(segment_map->fpm, npages, &first_page)
```

如果失败，源码使用 `elog(FATAL)`。因为 `get_best_segment()` / `make_new_segment()` 已经承诺 segment 有足够连续空间。失败意味着 backend-private state 或 allocator 元数据不一致。

### 8.5 double free / use after free

DSA 不尝试可靠检测对象级 double free 或 use-after-free。`dsa_get_address()` 注释已经说明：

```text
调用者传入不 live 的 dsa_pointer 是调用者 bug，行为未定义。
```

`check_for_freed_segments()` 只解决 segment slot stale mapping，不解决同一 segment 内对象已释放后再次访问。

## 9. 成本、资源与跨模块传播

### 9.1 hot path 成本

已有 active superblock 的小对象 allocation：

```text
per-size-class LWLock
dsa_get_address(span)
freelist pop 或 bump ninitialized
可能移动到 full class
```

这是 DSA 小对象最便宜的路径。

free 小对象：

```text
check_for_freed_segments()
pagemap lookup
per-size-class LWLock
freelist push
可能移动 fullness class
可能 destroy superblock
```

free 比 alloc 多一个 pagemap 反查和可能的回收逻辑。

### 9.2 慢路径成本

慢路径包括：

```text
没有 active superblock:
  扫描 fullness classes
  可能分配 span descriptor
  area lock 下找 segment / 创建 segment
  FPM 申请 pages
  初始化 pagemap

large allocation:
  分配 span descriptor
  area lock 下找连续 pages
  large pool lock 初始化 span

destroy superblock:
  pool lock + area lock
  FPM 归还 pages
  segment 可能 unpin / detach
  freed_segment_counter++
```

这些路径更容易在高并发或碎片化 workload 下显形。

### 9.3 锁竞争如何放大

锁竞争常见来源：

| 竞争点 | workload 特征 | 影响 |
| --- | --- | --- |
| 同一 size class pool lock | 大量同大小 hash item / tree node 并发分配 | 所有 backend 串行进入该 pool。 |
| span-of-spans pool lock | 频繁创建 / 销毁 superblock 或 large allocation | allocator 元数据分配变热点。 |
| area lock | 新 superblock、大对象、segment growth、superblock destroy | 不同 size class 也会在这里汇合。 |
| FPM metadata | 大量 large allocation 或 superblock churn | page run 查找、合并、rebin 成本上升。 |

源码注释提到当前是：

```text
one pool, and therefore one lock, per size class
```

per-core pools 这类更复杂设计仍是未来研究方向。这说明社区接受了当前复杂度与竞争成本之间的折中。

### 9.4 碎片化如何放大

碎片有几类：

| 类型 | 来源 |
| --- | --- |
| size class 内部碎片 | 请求 size 向上取整到 class size。 |
| superblock 外部碎片 | 某个 size class 只用掉少量对象，也会占一个 64KB superblock。 |
| page run 碎片 | FreePageManager 中连续空闲 pages 被打散，大对象需要 contiguous pages。 |
| segment 尾部浪费 | segment 按几何增长或为大对象创建 odd-sized segment。 |
| active block hysteresis | 为避免频繁 destroy/recreate，active empty block 可能被保留。 |

这些都不是 bug，而是 allocator trade-off。

## 10. 观测与诊断入口

### 10.1 gdb 断点

适合断点：

```text
break dsa_allocate_extended
break alloc_object
break ensure_active_superblock
break dsa_free
break destroy_superblock
break get_best_segment
break make_new_segment
break FreePageManagerGet
break FreePageManagerPut
break dsa_trim
break dsa_dump
```

小对象观察字段：

```text
p size
p size_class
p area->control->pools[size_class].spans[1]
p span->nallocatable
p span->ninitialized
p span->firstfree
p span->fclass
```

segment / page 观察字段：

```text
p area->control->total_segment_size
p area->control->max_total_segment_size
p area->control->high_segment_index
p segment_map->header->usable_pages
p fpm_largest(segment_map->fpm)
p area->control->freed_segment_counter
```

### 10.2 `dsa_dump()`

`dsa_dump(area)` 会打印：

```text
area handle
max_total_segment_size
total_segment_size
refcnt
pinned
segment bins
每个非空 pool 的 fullness class
span descriptor pointer
superblock pointer
pages
objects free / max
```

它不是一致快照，源码注释说明它会逐个拿锁再释放。但对理解 allocator 当前形状很有用。

### 10.3 从现象回到源码

如果看到 DSA OOM：

```text
看请求 size 是否进入 large path；
看是否设置 DSA_ALLOC_NO_OOM；
看 total_segment_size 是否接近 max_total_segment_size；
看是否达到 DSA_MAX_SEGMENTS；
看 fpm_largest 是否不足以满足 contiguous pages。
```

如果看到 DSA 锁等待：

```text
同一 size class 大量分配:
  pool lock 可能热点。

大对象或频繁创建 superblock:
  area lock / FPM 可能热点。
```

如果看到内存没有快速归还：

```text
active block 可能被保留；
部分使用的 superblock 不能归还；
segment 只有全空且非 0 才能 unpin/detach；
dsa_trim() 只积极释放 spare memory，不会释放仍有 live object 的 superblock。
```

## 11. 常见误区

误区一：DSA 小对象每次都向 DSM 申请内存。

不是。小对象通常从已有 superblock 中切 slot，只在没有合适 superblock 时才走 FPM / segment growth。

误区二：`dsa_free()` 需要调用者传入 size。

不需要。它用 `pagemap` 找 span，再从 span 得到 size class 和对象大小。

误区三：large allocation 完全绕过 span。

不是。large allocation 绕过普通 superblock freelist，但仍用特殊 span 记录 `start` 和 `npages`。

误区四：per-size-class lock 意味着所有 DSA allocation 都能并行。

不完全。不同 size class 的已有 superblock allocation 可以并行；新 superblock、大对象、segment 回收仍会争用 area lock。

误区五：superblock 空了就一定马上释放。

不一定。active block 可能被保留以避免 hysteresis。`dsa_trim()` 可以更积极地释放 spare memory。

误区六：FreePageManager 解决所有碎片问题。

它会合并相邻 free page ranges，帮助大对象找到连续 pages；但无法消除 size class 内部碎片或部分使用 superblock 的外部碎片。

误区七：`dsa_get_address()` 能检查对象是否还没被 free。

不能。它能处理 freed segment slot 的 stale mapping 问题，不能证明单个 object live。

## 12. 课堂实验

### 实验一：观察小对象分配 hot path

在 `dsa_allocate_extended()` 和 `alloc_object()` 下断点，分配多个同大小对象。

观察：

```text
size -> size_class
pool->spans[1]
span->ninitialized 增长
span->nallocatable 下降
span 满后移动到 full class
```

目标：理解已有 active superblock 上的 allocation 几乎只动一个 pool。

### 实验二：触发新 superblock

持续分配某个 size class，直到 active span 用尽。断点：

```text
break ensure_active_superblock
break FreePageManagerGet
```

观察：

```text
先尝试从其它 fullness class 找 span；
找不到后分配 span descriptor；
从 FPM 申请 16 pages；
pagemap 连续 pages 都写成同一个 span_pointer。
```

目标：理解 superblock 是小对象和 page allocator 之间的批量桥梁。

### 实验三：释放对象并观察 freelist

在 `dsa_free()` 下断点，释放一个小对象：

```text
pageno = offset / FPM_PAGE_SIZE
span_pointer = pagemap[pageno]
NextFreeObjectIndex(object) = span->firstfree
span->firstfree = object_index
span->nallocatable++
```

目标：理解 free path 为什么不需要 size 参数。

### 实验四：触发 large allocation

分配大于 8192 bytes 的对象。断点：

```text
break dsa_allocate_extended
break FreePageManagerGet
break init_span
```

观察：

```text
npages = fpm_size_to_pages(size)
span size_class = DSA_SCLASS_SPAN_LARGE
pagemap[first_page] = span_pointer
```

再释放它，观察：

```text
FreePageManagerPut(... span->npages)
unlink large span
dsa_free(span_pointer)
```

目标：理解 large allocation 和小对象共享 span/pagemap 模型。

### 实验五：观察 segment 回收

构造能让某个非 0 segment 全空的 workload，或在 `destroy_superblock()` 中观察条件：

```text
fpm_largest(segment_map->fpm) == usable_pages
index != 0
```

观察：

```text
dsm_unpin_segment()
dsm_detach()
segment_handles[index] = DSM_HANDLE_INVALID
freed_segment_counter++
```

目标：理解 segment 回收为什么必须通知其它 backend 清理 stale mapping。

### 实验六：使用 `dsa_dump()`

在 gdb 中对某个 area 调用：

```text
call dsa_dump(area)
```

观察输出中的：

```text
segment bins
pool for size class N
fullness class
objects free / max
```

目标：把 allocator 内部状态从字段表变成可读的诊断快照。

## 13. 讨论题

1. 为什么 DSA 对小对象使用 64KB superblock，而不是每次按 4KB page 分配？
2. 为什么 active span 放在 fullness class 1，而不是 class 0？
3. 如果某 workload 大量分配 40-byte 对象，会争用哪些锁？
4. large allocation 为什么还要分配 span descriptor？
5. `pagemap` 的空间成本和 free path 简化之间是什么 trade-off？
6. 为什么 `destroy_superblock()` 可以归还整个 segment，但 segment 0 不能释放？
7. DSA 为什么不保证 `dsa_free()` 能检测 double free？
8. 如果要降低同一 size class 的锁竞争，可以引入什么设计？它会带来哪些碎片或复杂性？

## 14. 本节小结

DSA allocator 的核心不是“共享版 malloc”，而是一组分层折中：

```text
小对象:
  size class -> pool lock -> span -> superblock slot

大对象:
  area lock -> segment bin -> FreePageManager page run -> large span

free:
  dsa_pointer -> pagemap -> span -> pool freelist 或 FPM

回收:
  empty superblock -> FPM
  empty nonzero segment -> dsm_unpin_segment / dsm_detach
```

它把高频小对象操作限制在 per-size-class lock 上，把低频但更重的 page run、segment growth、segment reclaim 放到 area lock 和 FreePageManager 上。

可迁移规律：

```text
共享 allocator 的难点不是“找到一块内存”；
而是让分配、释放、归属反查、并发锁、碎片控制和跨进程 segment 生命周期
在同一套元数据中闭合。
```

下一节会继续看 DSA 的 segment 增长、回收与 named DSM registry：当一个 DSA area 不只是 parallel query 的临时 heap，而成为可复用的长期共享基础设施时，segment pin、trim、freed counter 和 registry handle 如何构成长生命周期边界。
