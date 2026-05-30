# PostgreSQL DSA segment 增长、回收与 named DSM registry

## 课程定位

前置知识：已经理解 DSA area / `dsa_pointer` 的跨进程地址模型，也理解 DSA allocator 如何用 size class、span、superblock、pagemap 和 `FreePageManager` 管理小对象与大对象。

本节唯一主问题：

```text
DSA area 如何按需创建、pin、trim 和释放多个 DSM segment，
freed_segment_counter 如何防止 segment slot 重用造成旧映射误读，
GetNamedDSMSegment() / GetNamedDSA()
又如何把这种动态共享状态提升为可复用的命名基础设施？
```

核心矛盾：DSA 不能只管理第一个 DSM segment。共享 heap 需要按 workload 增长，空闲后尽量把整段 DSM 归还；但 segment index 可能复用，不同 backend 又可能还映射着旧 segment。同时，某些共享对象不是一次 parallel query 的临时 heap，而是扩展或子系统想“按名字懒创建、长期复用”的共享状态。PostgreSQL 需要在动态增长、回收安全、长期 pin 和命名发现之间取得平衡。

学完后应能判断：

```text
为什么 DSA 新 segment 创建后要 dsm_pin_segment()；
为什么 segment 0 不能像后续 segment 那样回收；
为什么 make_new_segment() 用几何增长而不是每次刚好分配；
为什么 freed_segment_counter 是跨 backend stale mapping 的防线；
为什么 dsa_trim() 只能释放 spare memory，不能释放仍有 live object 的 superblock；
为什么 named DSM registry 自己要用 pinned DSA + dshash；
为什么 GetNamedDSMSegment() / GetNamedDSA() 必须在 TopMemoryContext 下 attach 并 pin mapping；
为什么 named registry 适合长期共享对象，不适合替代普通事务内临时 DSM。
```

本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `2c4bd2bf5700`。

## 1. 本节在总主线中的位置

第 32 节讲了 DSA 的地址模型：

```text
dsa_handle:
  找到 area

dsa_pointer:
  segment index + offset

segment_maps[]:
  当前 backend 的本地映射表
```

第 33 节讲了 DSA 的 allocator 模型：

```text
small object:
  size class -> span -> superblock

large object:
  FreePageManager page run

free:
  pagemap -> span -> pool / FPM
```

本节回答最后两个边界问题：

```text
1. DSA area 增长和回收多个 DSM segment 时，如何保证其它 backend 不把旧映射当成新 segment？
2. 如果一个共享对象需要被很多 backend 按名字懒 attach，而不是通过某个已有 DSM/TOC 传 handle，系统如何发现它？
```

这就引出 `dsm_registry.c`：一个基于 pinned DSA + dshash 的命名共享状态注册表。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
DSA area 在空间不足时用 make_new_segment() 创建并 pin 新 DSM segment，
把 segment handle 写入 shared segment_handles[index]；
superblock 整体空闲后可能把非 0 segment unpin / detach 并清空 slot；
freed_segment_counter 让其它 backend 在解释新 dsa_pointer 前先清理旧 mapping；
named DSM registry 用一个长期 pinned DSA + dshash 保存 name -> DSM/DSA/DSHash handle。
```

本节 tension 是：

```text
动态共享状态要能增长、回收、长期复用
  vs
跨进程 mapping 可能滞后，segment slot 可能复用，命名对象必须只初始化一次
```

如果 DSA 只增长不回收：

```text
长生命周期 workload 会把空闲 DSM segment 永远留在系统里。
```

如果 DSA 回收 segment 但不通知其它 backend：

```text
backend A 仍映射着旧 segment slot 5；
backend B 释放 slot 5 后又创建新 segment 也放 slot 5；
A 收到一个指向新 slot 5 的 dsa_pointer，却可能用旧 mapping 解释。
```

如果命名共享对象没有 registry：

```text
每个扩展都要预先 request static shmem，或自己发明 name -> handle 表。
```

PostgreSQL 的选择是：

```text
DSA 内部用 freed_segment_counter 处理 slot reuse；
registry 层用 pinned DSA + dshash 处理 name -> handle 发现。
```

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 作用 |
| --- | --- | --- |
| 1 | `src/backend/utils/mmgr/dsa.c` | `make_new_segment()`、`destroy_superblock()`、`check_for_freed_segments()`、`dsa_trim()`、pin / detach / release。 |
| 2 | `src/include/utils/dsa.h` | `dsa_pin()`、`dsa_pin_mapping()`、`dsa_set_size_limit()`、`dsa_trim()`、`dsa_get_total_size()` API。 |
| 3 | `src/backend/storage/ipc/dsm_registry.c` | named DSM / DSA / dshash registry 实现。 |
| 4 | `src/include/storage/dsm_registry.h` | `GetNamedDSMSegment()`、`GetNamedDSA()`、`GetNamedDSHash()` 对外 API。 |
| 5 | `src/backend/lib/dshash.c` | registry 底层 hash table 如何用 DSA 存 entry。 |
| 6 | `src/backend/utils/activity/pgstat_shmem.c` | in-place DSA、size limit、manual release 的长期共享例子。 |

推荐阅读顺序：

```text
先读 make_new_segment()
  -> 再读 destroy_superblock()
  -> 再读 check_for_freed_segments()
  -> 再读 dsa_trim()
  -> 再读 dsm_registry.c 顶部注释和 init_dsm_registry()
  -> 最后跟 GetNamedDSMSegment() / GetNamedDSA() / GetNamedDSHash()
```

本节要把两条线连起来：

```text
DSA segment lifecycle:
  dynamic storage mechanics

DSM registry:
  named discovery and long-lived ownership
```

## 4. 关键数据结构与状态

### 4.1 DSA segment table

`dsa_area_control` 中与 segment 增长 / 回收相关的字段：

| 字段 | 语义 |
| --- | --- |
| `segment_handles[]` | 每个 segment index 对应的 DSM handle。 |
| `segment_bins[]` | 按最大连续空闲 pages 分类的 segment 链表。 |
| `init_segment_size` | 初始 segment size。 |
| `max_segment_size` | 单个 segment 最大 size。 |
| `total_segment_size` | 当前 active backing segments 总大小。 |
| `max_total_segment_size` | area 可增长到的总上限。 |
| `high_segment_index` | area 历史上用过的最高 segment index。 |
| `freed_segment_counter` | segment 被释放的全局计数。 |
| `lock` | 保护 segment table、bins、FPM 等 area-wide 状态。 |

每个 segment 自己有 `dsa_segment_header`：

| 字段 | 语义 |
| --- | --- |
| `magic` | `DSA_SEGMENT_HEADER_MAGIC ^ area_handle ^ index`，校验 segment 属于哪个 area/index。 |
| `usable_pages` | metadata 之外可分配页数。 |
| `size` | segment 总字节数。 |
| `prev` / `next` | segment bin 链表。 |
| `bin` | 当前所在 bin。 |
| `freed` | 该 segment 已经被释放，其他 backend 应 detach 旧 mapping。 |

segment 0 特殊：

```text
它包含 dsa_area_control；
即使空闲，也不能像后续 segment 那样释放。
```

### 4.2 backend-local segment map

每个 backend 的 `dsa_area` 中有：

```text
segment_maps[DSA_MAX_SEGMENTS]
high_segment_index
freed_segment_counter
```

这三个字段表达当前 backend 的本地视图：

```text
segment_maps[index]:
  当前 backend 是否已经 attach 这个 segment，以及本地 base address 是什么。

area->high_segment_index:
  当前 backend 曾经映射过的最高 index。

area->freed_segment_counter:
  当前 backend 已经处理到哪个 freed_segment_counter。
```

共享 control 的 `freed_segment_counter` 和本地 area 的 `freed_segment_counter` 不相等时：

```text
当前 backend 可能还映射着已经释放的 old segment；
必须扫描 segment_maps[]，detach header->freed 的 mapping。
```

### 4.3 DSM registry entry

`dsm_registry.c` 中每个 entry 的 key 是 name，value 按类型分三种：

```text
DSM:
  dsm_handle handle
  size_t size

DSA:
  dsa_handle handle
  int tranche

DSHash:
  dsa_handle dsa_handle
  dshash_table_handle dsh_handle
  int tranche
```

entry 还保存：

```text
name[NAMEDATALEN]
type = DSM / DSA / DSH
```

registry 自己的根状态在 static shared memory 中只有两个 handle：

```text
DSMRegistryCtx->dsah
DSMRegistryCtx->dshh
```

实际 registry table 是：

```text
pinned DSA
  -> dshash table
       -> DSMRegistryEntry records
```

也就是说，named registry 自己也是 DSA + dshash 的用户。

## 5. 主流程源码 walkthrough

### 5.1 `make_new_segment()`：按需增长 DSA area

当 `get_best_segment(area, npages)` 找不到足够连续 pages，DSA 会尝试：

```text
make_new_segment(area, requested_pages)
```

前提：

```text
caller 持有 DSA_AREA_LOCK(area)
```

第一步，找空 segment slot：

```text
for new_index = 1; new_index < DSA_MAX_SEGMENTS; ++new_index:
  if segment_handles[new_index] == DSM_HANDLE_INVALID:
    break
```

注意从 1 开始，segment 0 是 control segment。

第二步，检查总大小限制：

```text
if total_segment_size >= max_total_segment_size:
  return NULL
```

第三步，计算新 segment size：

```text
total_size =
  init_segment_size *
  (1 << (new_index / DSA_NUM_SEGMENTS_AT_EACH_SIZE))

total_size = min(total_size, max_segment_size)
total_size = min(total_size, max_total_segment_size - total_segment_size)
```

当前 `DSA_NUM_SEGMENTS_AT_EACH_SIZE = 2`，所以 segment size 大致每两个 slot 翻倍。

为什么不每次只创建刚好够用的 segment？

```text
DSM 系统不适合维护过多 segment；
几何增长减少 segment 数量和 segment table 压力；
但也会带来较大 segment 的尾部浪费风险。
```

第四步，计算 metadata：

```text
dsa_segment_header
FreePageManager
pagemap entries
padding 到 page boundary
usable_pages = (total_size - metadata_bytes) / 4KB
```

如果普通几何 size 不够满足 requested_pages，就创建 odd-sized segment，按 requested_pages 反推 metadata 和 total size。

第五步，真正创建并发布：

```text
segment = dsm_create(total_size, 0)
dsm_pin_segment(segment)
segment_handles[new_index] = dsm_segment_handle(segment)
high_segment_index = max(high_segment_index, new_index)
total_segment_size += total_size
```

第六步，初始化本 backend 的 local map 和 segment metadata：

```text
segment_map->mapped_address = dsm_segment_address(segment)
FreePageManagerInitialize(segment_map->fpm, base)
FreePageManagerPut(fpm, metadata_pages, usable_pages)
header->magic = DSA_SEGMENT_HEADER_MAGIC ^ area_handle ^ new_index
header->bin = contiguous_pages_to_segment_bin(usable_pages)
header->freed = false
插入 segment_bins[bin]
```

到这里，新 segment 已经可供 allocator 使用。

### 5.2 `get_best_segment()`：为什么 segment 要分 bin

DSA 不是线性扫描所有 segment 找空间。它维护 `segment_bins[]`：

```text
bin = contiguous_pages_to_segment_bin(npages)
```

`get_best_segment()` 从最低可能满足请求的 bin 开始：

```text
for bin in [needed_bin, DSA_NUM_SEGMENT_BINS):
  遍历该 bin 的 segment 链表
  contiguous_pages = fpm_largest(segment_map->fpm)
  如果 contiguous_pages >= npages:
    return segment_map
  如果 segment 实际已经不该在这个 bin:
    rebin_segment()
```

bin 不是精确大小表，而是启发式过滤：

```text
低于 needed_bin 的 segment 肯定不够；
高于或等于 needed_bin 的 segment 可能够，需要检查 fpm_largest。
```

这减少了 segment 数多时的搜索成本，也让释放后 `rebin_segment()` 能把 segment 放回合适位置。

### 5.3 `destroy_superblock()`：从空 superblock 到 segment 回收

第 33 节已经讲过：小对象 superblock 全空时，可能调用：

```text
destroy_superblock(area, span_pointer)
```

这一步先把 span 从 pool fullness 链表移除，然后拿 area lock，把 superblock pages 归还给该 segment 的 FPM：

```text
FreePageManagerPut(fpm,
                   offset(span->start) / FPM_PAGE_SIZE,
                   span->npages)
```

如果此时：

```text
fpm_largest(fpm) == segment_header->usable_pages
```

说明整个 segment 的 usable pages 都空了。若 `index != 0`，DSA 会把这个 segment 还给 DSM：

```text
unlink_segment(area, segment_map)
segment_map->header->freed = true
total_segment_size -= header->size
dsm_unpin_segment(dsm_segment_handle(segment))
dsm_detach(segment)
segment_handles[index] = DSM_HANDLE_INVALID
freed_segment_counter++
segment_map->segment = NULL
segment_map->header = NULL
segment_map->mapped_address = NULL
```

这里最重要的是顺序：

```text
先标记 header->freed = true；
再清 shared segment_handles[index]；
再增加 freed_segment_counter。
```

其它 backend 可能仍然映射着该 segment，但看到 counter 改变后会检查并 detach。

### 5.4 `freed_segment_counter`：防止 slot reuse 误读

危险场景：

```text
T1:
  backend A 映射 segment index 7，base = old segment

T2:
  backend B 释放 index 7 的 segment，segment_handles[7] = invalid，
  freed_segment_counter++

T3:
  backend C 创建新 segment，复用 index 7，
  生成一个新的 dsa_pointer(segment=7, offset=X)

T4:
  backend A 收到这个新的 dsa_pointer
```

如果 A 不先清理旧 mapping：

```text
dsa_get_address(area, dp)
  -> segment_maps[7].mapped_address 仍然非 NULL
  -> 直接 old_base + offset
  -> 把新 pointer 解释到旧 segment
```

`check_for_freed_segments()` 正是为这个场景服务：

```text
pg_read_barrier()
freed = area->control->freed_segment_counter
if area->freed_segment_counter != freed:
  LWLockAcquire(DSA_AREA_LOCK)
  check_for_freed_segments_locked()
  LWLockRelease(...)
```

注释里的 ordering 关键点是：

```text
释放 segment 时在 LWLock 下递增 freed_segment_counter；
复用 slot 创建新 segment 也在 LWLock 下发生；
调用者把 dsa_pointer 传给本 backend 必须通过某种同步手段；
读侧用 read barrier 确保先看见 pointer，再看见 counter 状态。
```

`check_for_freed_segments_locked()` 扫描当前 backend 已映射的 segment：

```text
if segment_maps[i].header != NULL &&
   segment_maps[i].header->freed:
  dsm_detach(segment_maps[i].segment)
  segment_maps[i] = NULL mapping
```

然后更新本地：

```text
area->freed_segment_counter = control->freed_segment_counter
```

这样 `dsa_get_address()` 再遇到 index 7 时，会发现本地 mapping 为空，重新按 `segment_handles[7]` attach 新 segment。

### 5.5 `dsa_trim()`：主动释放 spare memory

普通 `dsa_free()` 会在条件满足时释放 empty superblock，但 DSA 还提供：

```text
dsa_trim(area)
```

它更积极地释放 spare memory：

```text
for size_class = DSA_NUM_SIZE_CLASSES - 1 down to 0:
  跳过 DSA_SCLASS_SPAN_LARGE
  只扫描 fullness class 1
  如果 span->nallocatable == span->nmax:
    destroy_superblock(area, span_pointer)
```

为什么 reverse order？

```text
先处理普通 pools；
最后处理 span-of-spans；
因为释放普通 superblock 可能释放 span descriptor，
让 span-of-spans pool 也变得可回收。
```

`dsa_trim()` 不能释放仍有 live object 的 superblock，也不能释放 segment 0。它只是把“已经空了但为避免 hysteresis 被保留”的 spare memory 尽量归还。

### 5.6 registry 初始化：用 DSA + dshash 建一张命名表

`dsm_registry.c` 的 static shared memory 只保存：

```text
DSMRegistryCtx->dsah
DSMRegistryCtx->dshh
```

真正 registry table 在第一次使用时懒初始化：

```text
init_dsm_registry()
```

流程：

```text
if dsm_registry_table 已经存在:
  return

LWLockAcquire(DSMRegistryLock, LW_EXCLUSIVE)

if DSMRegistryCtx->dshh == DSHASH_HANDLE_INVALID:
  dsm_registry_dsa = dsa_create(LWTRANCHE_DSM_REGISTRY_DSA)
  dsm_registry_table = dshash_create(dsm_registry_dsa, ...)
  dsa_pin(dsm_registry_dsa)
  dsa_pin_mapping(dsm_registry_dsa)
  DSMRegistryCtx->dsah = dsa_get_handle(dsm_registry_dsa)
  DSMRegistryCtx->dshh = dshash_get_hash_table_handle(dsm_registry_table)
else:
  dsm_registry_dsa = dsa_attach(DSMRegistryCtx->dsah)
  dsa_pin_mapping(dsm_registry_dsa)
  dsm_registry_table = dshash_attach(...)

LWLockRelease(DSMRegistryLock)
```

这里有两层 pin：

```text
dsa_pin:
  registry DSA 即使没有 backend attach，也继续存在。

dsa_pin_mapping:
  当前 backend 的 mapping 不随短事务 ResourceOwner 释放。
```

registry 自身必须长期存在，否则 name -> handle 表会随某个 backend 退出消失。

### 5.7 `GetNamedDSMSegment()`：name -> pinned DSM segment

`GetNamedDSMSegment(name, size, init_callback, found, arg)` 流程：

```text
切到 TopMemoryContext
init_dsm_registry()
entry = dshash_find_or_insert(dsm_registry_table, name, found)
```

如果 entry 不存在：

```text
entry->type = DSM
state->handle = invalid
state->size = size
```

如果 entry 已存在，要校验：

```text
type 必须是 DSM
size 必须一致
```

如果 `state->handle == DSM_HANDLE_INVALID`，说明还没真正创建：

```text
seg = dsm_create(size, 0)
if init_callback:
  init_callback(dsm_segment_address(seg), arg)
dsm_pin_segment(seg)
dsm_pin_mapping(seg)
state->handle = dsm_segment_handle(seg)
*found = false
```

否则 attach existing：

```text
seg = dsm_find_mapping(handle)
if seg == NULL:
  seg = dsm_attach(handle)
  dsm_pin_mapping(seg)
*found = true
```

最后释放 dshash entry lock，恢复 memory context，返回当前 backend 的 segment address。

这个 API 保证：

```text
同名 segment 只初始化一次；
其它 backend 只 attach；
mapping 是 backend-lifetime 级别，而不是事务结束就消失。
```

### 5.8 `GetNamedDSA()`：name -> pinned DSA area

`GetNamedDSA(name, found)` 类似，但 value 是 DSA handle 和 LWLock tranche：

```text
entry = dshash_find_or_insert(...)
```

新 entry：

```text
entry->type = DSA
state->handle = invalid
state->tranche = -1
```

如果 tranche 未创建：

```text
state->tranche = LWLockNewTrancheId(name)
*found = false
```

如果 handle invalid：

```text
ret = dsa_create(state->tranche)
dsa_pin(ret)
dsa_pin_mapping(ret)
state->handle = dsa_get_handle(ret)
*found = false
```

如果当前 backend 已经 attach：

```text
ERROR "requested DSA already attached to current process"
```

否则：

```text
ret = dsa_attach(state->handle)
dsa_pin_mapping(ret)
*found = true
```

这里的“每 backend 最多调用一次”很重要。因为 `dsa_pin_mapping()` 把 mapping 提升为长期生命周期，重复 attach 同一个 named DSA 会制造重复本地对象和混乱 cleanup 边界。

### 5.9 `GetNamedDSHash()`：name -> DSA + dshash

`GetNamedDSHash()` 是 registry 的第三层便利 API：

```text
name -> {dsa_handle, dshash_table_handle, tranche}
```

第一次：

```text
dsa = dsa_create(tranche)
dshash_create(dsa, params_copy, NULL)
dsa_pin(dsa)
dsa_pin_mapping(dsa)
保存 dsa handle 和 dshash handle
```

后续：

```text
dsa = dsa_attach(dsa_handle)
dsa_pin_mapping(dsa)
ret = dshash_attach(dsa, params, dsh_handle, NULL)
```

这说明 registry 不只是存 DSM segment，也把“共享 heap + 共享 hash table root object”封装成按名字获取的基础设施。

## 6. 生命周期 / ownership / cleanup

### 6.1 DSA segment 增长生命周期

```text
需要更多 pages
  -> area lock
  -> 找 empty segment slot
  -> dsm_create(total_size)
  -> dsm_pin_segment(segment)
  -> segment_handles[index] = handle
  -> 初始化 FPM / pagemap / header
  -> 放入 segment bin
```

owner 是 DSA area，不是创建该 segment 的普通 backend 调用栈。

### 6.2 DSA segment 回收生命周期

```text
superblock 全空
  -> FreePageManagerPut()
  -> 如果 entire segment free 且 index != 0
       unlink segment bin
       header->freed = true
       total_segment_size -= size
       dsm_unpin_segment()
       dsm_detach()
       segment_handles[index] = invalid
       freed_segment_counter++
```

这不是普通 `dsa_free()` 立刻释放整个 area。它只是在 allocator 发现某个 backing segment 完全空闲时，把这个 DSM segment 还给系统。

### 6.3 Named registry 生命周期

registry 的生命周期：

```text
postmaster static shared memory:
  DSMRegistryCtx holds dsah/dshh

first backend using registry:
  create DSA + dshash
  pin area and mapping
  store handles in DSMRegistryCtx

later backend:
  attach DSA + dshash
  pin mapping
  use name -> entry table
```

named DSM / DSA / DSHash entries 通常是长期对象：

```text
DSM:
  dsm_pin_segment + dsm_pin_mapping

DSA:
  dsa_pin + dsa_pin_mapping

DSHash:
  pinned DSA holds hash table storage
```

registry 没有提供“按名字删除” API。这是一个有意的生命周期边界：它服务长期共享对象，不服务短生命周期临时对象。

## 7. 正确性机制层次

| 层次 | 机制 | 保护的问题 |
| --- | --- | --- |
| segment 增长 | area lock + `segment_handles[]` | 新 segment slot 发布与 allocator 状态一致。 |
| segment 选择 | `segment_bins[]` + `fpm_largest()` | 避免全表扫描，按 contiguous pages 找候选 segment。 |
| segment 回收 | `header->freed` + invalid handle + unpin/detach | 空 segment 归还 DSM，并标记旧 mapping 不再可用。 |
| stale mapping 防护 | `freed_segment_counter` + read barrier | 防止旧 mapping 解释复用 slot 的新 pointer。 |
|长期 area | `dsa_pin()` | 没有 backend attach 时 area 仍可重新 attach。 |
| 当前 backend mapping | `dsa_pin_mapping()` | named object mapping 不被事务 ResourceOwner 释放。 |
| registry 初始化 | `DSMRegistryLock` | 保证 registry DSA/dshash 只创建一次。 |
| entry 并发 | `dshash_find_or_insert()` entry lock | 保证同名 object 只初始化一次。 |
| 类型边界 | `DSMREntryType` | 同名不能既是 DSM 又是 DSA / DSHash。 |

最容易漏掉的边界：

```text
freed_segment_counter 不证明某个 dsa_pointer 指向的 object 仍然 live；
它只防止 segment slot reuse 时用旧 mapping 解释新 segment pointer。
```

对象级 live-ness 仍然由上层锁、引用计数或协议保证。

## 8. 错误路径 / 异常路径 / fallback

### 8.1 DSA 无法增长

`make_new_segment()` 可能返回 NULL：

```text
没有空 segment slot；
total_segment_size 已达到 max_total_segment_size；
requested_pages 超过 DSA_MAX_SEGMENT_SIZE；
新 segment 会超过总大小限制；
dsm_create() 失败。
```

上层 allocation path 根据 `DSA_ALLOC_NO_OOM` 决定：

```text
返回 InvalidDsaPointer
或 ERROR out of memory
```

### 8.2 `dsm_attach()` 后续 segment 失败

`get_segment_by_index()` 通过 `segment_handles[index]` attach 后续 segment：

```text
segment = dsm_attach(handle)
if segment == NULL:
  ERROR "dsa_area could not attach to segment"
```

如果 handle 已经 invalid：

```text
ERROR "dsa_area could not attach to a segment that has been freed"
```

这通常说明调用者使用了不再 live 的 `dsa_pointer`，或同步协议有问题。

### 8.3 Named registry name/type/size 错误

`GetNamedDSMSegment()` 会检查：

```text
name 不能为空
name 不能太长
size 不能为 0
同名 entry 类型必须是 DSM
已有 DSM size 必须一致
```

`GetNamedDSA()` / `GetNamedDSHash()` 也检查：

```text
name 不能为空
name 不能太长
同名 entry 类型必须匹配
当前 backend 不能重复 attach 同一个 named DSA/DSHash
```

这些错误不是 allocator 错误，而是 registry 命名协议错误。

### 8.4 init callback 失败

`GetNamedDSMSegment()` 第一次创建 DSM 后，会在持有 registry entry lock 期间调用：

```text
init_callback(dsm_segment_address(seg), arg)
```

如果 callback ERROR，会跳出初始化。调用者设计 init callback 时要注意：

```text
只做必要初始化；
不要在里面依赖同一个 registry entry 的递归获取；
ERROR 后要依赖 ResourceOwner / DSM cleanup 兜底。
```

### 8.5 registry allocation 观测

`pg_get_dsm_registry_allocations()` 会遍历 registry dshash：

```text
DSM:
  返回 size

DSA:
  dsa_get_total_size_from_handle(handle)

DSHash:
  dsa_get_total_size_from_handle(dsa_handle)
```

如果 entry 尚未完成初始化，则 size 返回 NULL。它是诊断入口，不是 cleanup API。

## 9. 成本、资源与跨模块传播

### 9.1 DSA segment 成本

增长成本：

```text
area lock
dsm_create()
dsm_pin_segment()
初始化 FPM / pagemap
更新 segment_handles / bins
```

回收成本：

```text
pool lock + area lock
FreePageManagerPut()
dsm_unpin_segment()
dsm_detach()
freed_segment_counter++
其它 backend 下次 dsa_get_address/dsa_free 时清理 stale mapping
```

频繁增长/回收会放大：

```text
DSM segment 创建销毁成本；
area lock 竞争；
pagemap metadata 成本；
freed_segment_counter 检查和 stale mapping cleanup。
```

### 9.2 registry 成本

named registry 成本：

```text
首次使用:
  初始化 registry DSA + dshash。

每次按 name 获取:
  dshash lookup / insert；
  可能 attach DSM/DSA；
  pin mapping；
  TopMemoryContext 中分配本地 wrapper。
```

registry 适合：

```text
扩展或子系统懒创建长期共享对象；
不希望在 postmaster startup 提前 request 固定 shmem；
需要 name -> handle 的跨 backend 发现。
```

registry 不适合：

```text
每个查询创建大量短生命周期对象；
需要按事务自动释放的临时 DSM；
需要可删除、可回收 name entry 的缓存。
```

## 10. 观测与诊断入口

### 10.1 gdb 断点

DSA segment 生命周期：

```text
break make_new_segment
break destroy_superblock
break check_for_freed_segments
break check_for_freed_segments_locked
break dsa_trim
break dsa_get_address
```

关键字段：

```text
p area->control->total_segment_size
p area->control->max_total_segment_size
p area->control->high_segment_index
p area->control->segment_handles[index]
p area->control->freed_segment_counter
p area->freed_segment_counter
p area->segment_maps[index].mapped_address
p area->segment_maps[index].header->freed
```

registry：

```text
break init_dsm_registry
break GetNamedDSMSegment
break GetNamedDSA
break GetNamedDSHash
break pg_get_dsm_registry_allocations
```

### 10.2 SQL 观测

可以用：

```sql
SELECT * FROM pg_get_dsm_registry_allocations();
```

观察 registry 中的 named DSM / DSA / DSHash 及其大小。

这能回答：

```text
某个 named DSA 是否已经初始化？
registry 中长期对象的 DSA backing storage 约多大？
同名对象类型是否符合预期？
```

但它不能告诉你：

```text
DSA 内部每个 size class 的碎片；
某个 object 是否 leaked；
哪个 backend 持有对象引用。
```

这些需要 `dsa_dump()`、gdb 或上层模块诊断。

### 10.3 从现象回到源码

现象一：DSA 总大小持续增长。

排查：

```text
看 workload 是否持续持有 live object；
看 dsa_trim() 是否可释放 spare memory；
看是否有 pinned long-lived DSA；
看 segment 0 和部分使用 superblock 是否无法回收。
```

现象二：访问 DSA pointer 报 freed segment。

排查：

```text
调用者是否使用了已释放对象的 dsa_pointer；
跨 backend 传 pointer 是否有锁/atomic/IPC 同步；
freed_segment_counter 是否被处理；
segment_handles[index] 是否已 invalid。
```

现象三：named DSA already attached。

排查：

```text
同一 backend 是否重复调用 GetNamedDSA(name)；
是否应该缓存第一次返回的 dsa_area *；
是否误把 named DSA 当作短生命周期可重复 attach 对象。
```

## 11. 常见误区

误区一：DSA segment 一创建就由创建 backend 拥有。

不是。segment 被 `dsm_pin_segment()` 后属于 DSA area 生命周期，由 DSA 显式 unpin / detach。

误区二：segment handle slot 清空就足以防 stale mapping。

不够。其它 backend 本地 `segment_maps[index]` 可能仍有旧 base address；需要 `freed_segment_counter` 和 `header->freed` 让它们清理。

误区三：`freed_segment_counter` 能防 use-after-free object。

不能。它防的是 segment slot reuse 误读。对象级 use-after-free 仍是上层 bug。

误区四：`dsa_trim()` 能把 DSA 缩到最小。

不能。它只释放已经完全空闲的 spare superblock / segment；live object、部分使用 superblock 和 segment 0 都不能释放。

误区五：named registry 可以替代所有 DSM/DSA 传递方式。

不应该。parallel query 这类短生命周期对象更适合通过已有 DSM/TOC 传递。registry 适合长期、按名字发现的共享对象。

误区六：`GetNamedDSA()` 可以在同一 backend 中重复调用拿多个 handle。

源码会报错。调用者应缓存第一次返回的 `dsa_area *`。

误区七：registry entry 可以自动删除。

当前 API 不提供删除。命名对象按长期共享状态设计。

## 12. 课堂实验

### 实验一：观察 DSA segment 增长

在 DSA workload 中触发大量 allocation，断点：

```text
break make_new_segment
```

观察：

```text
new_index
requested_pages
total_size
metadata_bytes
usable_pages
segment_handles[new_index]
total_segment_size
segment header magic
segment bin
```

目标：理解几何增长、metadata 和 segment publish。

### 实验二：观察 segment 回收

断点：

```text
break destroy_superblock
```

构造让非 0 segment 全空的场景，观察：

```text
fpm_largest(fpm) == usable_pages
index != 0
header->freed = true
segment_handles[index] = invalid
freed_segment_counter++
```

目标：理解 segment 回收不是 free object，而是 backing storage 归还。

### 实验三：模拟 stale mapping 清理

在两个 backend 中 attach 同一个 DSA。让 backend B 释放某 segment，再让 backend A 调用：

```text
dsa_get_address(area, dp)
```

断点：

```text
break check_for_freed_segments
break check_for_freed_segments_locked
```

观察 A 的：

```text
area->freed_segment_counter
control->freed_segment_counter
segment_maps[index].header->freed
segment_maps[index].mapped_address
```

目标：理解本地 stale mapping 如何被发现和清理。

### 实验四：观察 registry 首次初始化

断点：

```text
break init_dsm_registry
```

调用任意 registry API，观察：

```text
DSMRegistryCtx->dsah
DSMRegistryCtx->dshh
dsa_create()
dshash_create()
dsa_pin()
dsa_pin_mapping()
```

目标：理解 registry 自己也是一个 pinned DSA + dshash。

### 实验五：观察 GetNamedDSA

断点：

```text
break GetNamedDSA
```

第一次调用：

```text
state->tranche = LWLockNewTrancheId(name)
dsa_create()
dsa_pin()
dsa_pin_mapping()
state->handle = dsa_get_handle()
found = false
```

第二个 backend 调用：

```text
dsa_attach(state->handle)
dsa_pin_mapping()
found = true
```

同一 backend 重复调用，观察报错路径。

### 实验六：查看 registry allocations

执行：

```sql
SELECT * FROM pg_get_dsm_registry_allocations();
```

结合 gdb 断点：

```text
break pg_get_dsm_registry_allocations
```

观察 DSM / DSA / DSHash entry 的 size 如何计算。

## 13. 讨论题

1. 为什么 DSA 后续 segment 必须 pin，而不能依赖当前 backend mapping 的 refcount？
2. 如果没有 `freed_segment_counter`，segment slot reuse 会出现哪种错误解释？
3. 为什么 `check_for_freed_segments()` 需要 read barrier？
4. 为什么 segment 0 不能被 `destroy_superblock()` 释放？
5. `dsa_trim()` 为什么按 size class 逆序处理？
6. named registry 为什么用 dshash，而不是固定大小的 shmem hash table？
7. `GetNamedDSMSegment()` 为什么要校验同名 segment 的 size？
8. named DSA 为什么不允许同一 backend 重复 attach？
9. registry 适合扩展长期共享状态，但为什么不适合 per-query 临时状态？

## 14. 本节小结

DSA 的 segment lifecycle 可以压缩成三条规则：

```text
增长:
  area lock 下创建并 pin 新 DSM segment，
  发布到 segment_handles[index]，
  放入 segment bin。

回收:
  empty superblock 归还 FPM；
  empty nonzero segment unpin/detach；
  清空 segment handle；
  递增 freed_segment_counter。

防误读:
  每个 backend 在解释 dsa_pointer 前检查 freed_segment_counter，
  先 detach header->freed 的旧 mapping，
  再按当前 handle attach 新 segment。
```

named DSM registry 则把这种动态共享状态提升到系统基础设施层：

```text
static shmem 中保存 registry root handles；
pinned DSA + dshash 保存 name -> handle；
GetNamedDSMSegment / GetNamedDSA / GetNamedDSHash
保证同名对象只初始化一次，之后 backend 只 attach。
```

可迁移规律：

```text
动态共享内存一旦支持“增长”和“命名复用”，就必须同时解决两个问题：
地址槽位复用时的旧视图清理，以及长期对象的唯一初始化与发现。
前者是 allocator 内部一致性，后者是系统级 naming protocol。
```

到这里，第 6 组 DSM / shm_toc / shm_mq / DSA 的主线闭合：从动态 segment 生命周期，到 DSM 内对象发现，到消息通道，再到共享 heap 和命名共享状态。
