# PostgreSQL shm_toc 与 DSM 内对象发现

## 课程定位

前置知识：已经理解 DSM segment 的 create / attach / detach 生命周期，知道同一个 `dsm_handle` 可以被多个 backend attach，但 `dsm_segment_address()` 返回的是当前 backend 的本地虚拟地址。

本节唯一主问题：

```text
同一个 DSM 段在不同 backend 中可能映射到不同虚拟地址时，为什么不能直接共享普通指针，
shm_toc_create() / shm_toc_allocate() / shm_toc_insert() / shm_toc_lookup()
如何用 magic、key 和相对 offset 完成最小 bootstrap？
```

核心矛盾：DSM 只提供“一段跨进程共享字节”；上层代码需要在这段字节里找到 fixed state、message queue、serialized snapshot、DSA control object 等多个对象。但不同进程的 base address 可能不同，普通 C 指针不能跨进程传播；如果一开始就引入复杂 allocator 或 hash table，又会把 bootstrap 问题本身变成依赖更多共享对象的问题。

学完后应能判断：

```text
为什么 DSM 内对象目录存 offset 而不是 pointer；
为什么 shm_toc 是小规模 bootstrap 目录，不是通用共享 hash table；
为什么 allocate 从 segment 尾部向前长，TOC entry 从头部向后长；
为什么 insert 需要 write barrier，lookup 可以不加锁；
为什么 magic number 是跨模块边界检查，而不是安全认证；
什么时候该只把一个根对象放进 TOC，再在根对象下组织复杂结构。
```

本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `2c4bd2bf5700`。

## 1. 本节在总主线中的位置

前两节回答了两个 DSM 生命周期问题：

```text
第 27 节:
  DSM segment 如何创建、attach、detach，并用 refcount 避免早释放？

第 28 节:
  DSM mapping pin、segment pin 和 detach callback 如何处理长期共享边界？
```

现在进入 DSM 内部：

```text
拿到一段 DSM base address 后，多个 backend 如何对同一批对象达成共识？
```

典型场景是 parallel query：

```text
leader:
  创建 DSM
  -> 在 DSM 开头创建 shm_toc
  -> 分配 FixedParallelState、GUC、snapshot、error queue 等 chunk
  -> 用 well-known key 插入 TOC

worker:
  attach DSM
  -> 在 DSM base address 上 attach shm_toc
  -> 用同样 key lookup chunk
  -> 恢复自己的 backend-local 状态
```

下一节会讲 `shm_mq`，也就是 TOC 找到的对象之一如何承载消息流。本节只讲“如何发现 DSM 内对象”。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
shm_toc 把一段共享内存切成两部分：
TOC header 和 entry 从 segment 起点向后增长；
payload chunk 从 segment 终点向前分配；
entry 中保存 key -> offset，而不是 key -> pointer；
lookup 时用当前 backend 的 toc base address + offset 还原本地 pointer。
```

本节 tension 是：

```text
DSM 内对象必须能被多个进程重新发现
  vs
跨进程不能相信普通地址，bootstrap 目录也不能依赖复杂共享基础设施
```

PostgreSQL 的选择非常克制：

```text
shm_toc 只解决 bootstrap。
它不 free，不 resize，不做大量 key lookup 优化，不维护复杂 ownership。
```

如果你需要很多指针，正确模式不是塞很多 key：

```text
TOC 中只放一个 root object；
root object 内部自己维护数组、hash、DSA pointer 或其它结构。
```

这不是功能缺失，而是边界清晰。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 作用 |
| --- | --- | --- |
| 1 | `src/include/storage/shm_toc.h` | 对外接口、estimator API、注释明确说明 TOC 不适合大量 key。 |
| 2 | `src/backend/storage/ipc/shm_toc.c` | TOC layout、单向 allocation、key -> offset、barrier 和 lockless lookup 实现。 |
| 3 | `src/backend/access/transam/parallel.c` | parallel context 的主调用链：估算、创建、insert、worker attach / lookup。 |
| 4 | `src/backend/access/common/session.c` | per-session DSM 的小型 TOC：DSA 和 record typmod registry 两个 key。 |
| 5 | `src/backend/replication/logical/applyparallelworker.c` | logical parallel apply 的 TOC：shared header、message queue、error queue。 |
| 6 | `src/backend/executor/execParallel.c` | executor 如何把 plan、param、instrumentation、tuple queue、DSA 等对象加入 parallel DSM。 |
| 7 | `src/backend/commands/vacuumparallel.c` | parallel vacuum 如何把 shared state、index stats、buffer/WAL usage 放进 TOC。 |

推荐阅读顺序：

```text
先读 shm_toc.c 的 struct 和四个核心函数
  -> 再读 parallel.c 的 InitializeParallelDSM()
  -> 再读 ParallelWorkerMain() 如何 attach + lookup
  -> 最后横向看 session/apply/executor 的 key 使用习惯
```

不要把 `shm_toc` 读成一套通用 allocator。它的核心是“跨进程对象发现”，不是“动态内存管理”。

## 4. 关键数据结构与状态

### 4.1 `shm_toc`

`shm_toc` 位于 DSM segment 的起始地址。它是 opaque type，对外只通过 API 操作：

| 字段 | 语义 |
| --- | --- |
| `toc_magic` | 调用者定义的 magic，用于确认这段 DSM 内容属于预期协议。 |
| `toc_mutex` | spinlock，保护 allocate / insert 修改 TOC 状态。 |
| `toc_total_bytes` | TOC 管理的总字节数，向下按 `BUFFERALIGN` 对齐。 |
| `toc_allocated_bytes` | 已从 segment 尾部向前分配的 payload bytes。 |
| `toc_nentry` | 已发布的 entry 数量。 |
| `toc_entry[]` | 变长数组，存放 key -> offset。 |

这里要注意：`toc_allocated_bytes` 只记录从尾部向前切出的 payload，总 TOC entry 空间则从头部按 `toc_nentry` 推进。两者相向增长，直到空间不足。

### 4.2 `shm_toc_entry`

TOC entry 很小：

| 字段 | 语义 |
| --- | --- |
| `key` | 调用者约定的 64-bit 标识。通常是模块内的 `#define` 常量。 |
| `offset` | 对象地址相对 TOC 起点的字节偏移。 |

为什么 offset 足够？

```text
leader:
  address = local_base + offset
  insert key -> offset

worker:
  local_base 可能不同
  lookup key -> offset
  address = worker_local_base + offset
```

这就是 DSM 内“相对寻址”的第一层。后面 DSA 的 `dsa_pointer` 会把这个思想扩展到多 segment heap。

### 4.3 `shm_toc_estimator`

estimator 是 backend-local 估算工具：

| 字段 | 语义 |
| --- | --- |
| `space_for_chunks` | 所有 payload chunk 的对齐后大小之和。 |
| `number_of_keys` | 预计需要的 TOC entry 数。 |

它不是运行期分配器状态，而是帮助调用者在 `dsm_create()` 前算出 segment 大小：

```text
shm_toc_initialize_estimator()
shm_toc_estimate_chunk()
shm_toc_estimate_keys()
shm_toc_estimate()
```

这符合 DSM 的约束：segment 创建时需要知道大小；TOC 只在这段固定空间里切分，不会自动扩容。

## 5. 主流程源码 walkthrough

### 5.1 leader 估算 DSM 大小

`parallel.c` 的 `InitializeParallelDSM()` 先估算所有需要放进 DSM 的对象：

```text
shm_toc_estimate_chunk(&pcxt->estimator, sizeof(FixedParallelState))
shm_toc_estimate_keys(&pcxt->estimator, 1)

如果有 workers:
  estimate library / GUC / combo CID / snapshot / transaction state / ...
  shm_toc_estimate_keys(&pcxt->estimator, 12)
  estimate error queue space
  estimate entrypoint info
```

源码注释里有一句很重要：

```text
If you add more chunks here, you probably need to add keys.
```

这说明 TOC 的 sizing 是调用者协议的一部分。你不能随便 `allocate()` 后忘了给对应 key，也不能新增 key 后忘记估算 entry 空间。

### 5.2 创建 DSM 并在起点建立 TOC

估算完成后：

```text
segsize = shm_toc_estimate(&pcxt->estimator)
pcxt->seg = dsm_create(segsize, DSM_CREATE_NULL_IF_MAXSEGMENTS)
pcxt->toc = shm_toc_create(PARALLEL_MAGIC,
                           dsm_segment_address(pcxt->seg),
                           segsize)
```

`shm_toc_create()` 初始化：

```text
toc = address
toc->toc_magic = magic
SpinLockInit(&toc->toc_mutex)
toc->toc_total_bytes = BUFFERALIGN_DOWN(nbytes)
toc->toc_allocated_bytes = 0
toc->toc_nentry = 0
```

TOC header 放在 segment 起点。payload 还没分配。

### 5.3 分配 chunk，并发布 key -> offset

以 `FixedParallelState` 为例：

```text
fps = shm_toc_allocate(pcxt->toc, sizeof(FixedParallelState))
初始化 fps 字段
shm_toc_insert(pcxt->toc, PARALLEL_KEY_FIXED, fps)
```

`shm_toc_allocate()` 的状态变化：

```text
nbytes = BUFFERALIGN(nbytes)
SpinLockAcquire(toc_mutex)
  total_bytes = toc_total_bytes
  allocated_bytes = toc_allocated_bytes
  nentry = toc_nentry
  toc_bytes = header + nentry * entry_size + allocated_bytes
  如果 toc_bytes + nbytes > total_bytes:
     ERROR "out of shared memory"
  toc_allocated_bytes += nbytes
SpinLockRelease(toc_mutex)
return toc + (total_bytes - old_allocated_bytes - nbytes)
```

也就是说第一次 allocation 通常出现在 segment 末尾附近，后续 allocation 继续向前。

`shm_toc_insert()` 的状态变化：

```text
offset = address - toc
SpinLockAcquire(toc_mutex)
  检查 entry 空间
  toc_entry[nentry].key = key
  toc_entry[nentry].offset = offset
  pg_write_barrier()
  toc_nentry++
SpinLockRelease(toc_mutex)
```

write barrier 的语义是：

```text
先让其它 CPU 看到 entry 内容已经写好；
再发布 toc_nentry 增加。
```

这样 lookup 可以先读 `toc_nentry`，再无锁扫描 entry，而不会看到“entry 数增加了但 entry 内容还没写完”的状态。

### 5.4 worker attach TOC 并 lookup

parallel worker 收到 DSM handle 后：

```text
seg = dsm_attach(main_arg)
toc = shm_toc_attach(PARALLEL_MAGIC, dsm_segment_address(seg))
fps = shm_toc_lookup(toc, PARALLEL_KEY_FIXED, false)
error_queue_space = shm_toc_lookup(toc, PARALLEL_KEY_ERROR_QUEUE, false)
```

`shm_toc_attach()` 只做最小校验：

```text
toc = address
if toc->toc_magic != magic:
  return NULL
assert total >= allocated
return toc
```

这里的 magic 用于防止“拿错 DSM 或协议不匹配”：

```text
parallel worker 期待 PARALLEL_MAGIC；
logical apply worker 期待 PG_LOGICAL_APPLY_SHM_MAGIC；
session DSM 期待 SESSION_MAGIC。
```

`shm_toc_lookup()`：

```text
nentry = toc->toc_nentry
pg_read_barrier()
for i in [0, nentry):
  if toc_entry[i].key == key:
     return toc + toc_entry[i].offset
if noError:
  return NULL
else:
  elog(ERROR)
```

lookup 不加锁，是因为：

```text
entry 发布顺序由 insert 的 write barrier 保证；
lookup 只读已经发布的 entry；
TOC 预期 key 数很小，线性扫描足够。
```

### 5.5 optional key 与 `noError`

parallel worker 读取 transaction snapshot 时：

```text
tsnapspace = shm_toc_lookup(toc, PARALLEL_KEY_TRANSACTION_SNAPSHOT, true)
```

`noError = true` 表示这个 key 可能不存在。例如事务隔离级别不需要 transaction snapshot 时，leader 不会插入这个 key。

这也是 TOC 的一个重要协议点：

```text
key 是否必需，不由 TOC 知道；
调用者必须用 noError 表达自己的协议。
```

## 6. 生命周期 / ownership / cleanup

### 谁创建

TOC 通常由 DSM 创建者在 segment 起点创建：

```text
dsm_create()
-> shm_toc_create(magic, dsm_segment_address(seg), segsize)
```

也可以创建在 backend-private memory 中。parallel context 在没有 worker 或 DSM 创建失败 fallback 时就是这样：

```text
pcxt->private_memory = MemoryContextAlloc(TopMemoryContext, segsize)
pcxt->toc = shm_toc_create(PARALLEL_MAGIC, pcxt->private_memory, segsize)
```

这说明 `shm_toc` 本质上是“管理一段内存的布局”，不要求底层一定是 DSM；只是它主要服务 DSM。

### 谁持有

TOC 本身没有独立 owner：

```text
如果底层是 DSM:
  生命周期由 DSM mapping / segment 控制。

如果底层是 backend-private memory:
  生命周期由 MemoryContext 控制。
```

TOC 内分配出来的 chunk 也没有独立 owner。它们跟整段内存同生共死。

### 谁释放

`shm_toc` 没有 `shm_toc_free()`。原因与它的设计目标一致：

```text
它是 bootstrap-time layout 工具；
只负责把一段固定大小 memory 切成若干对象；
对象释放由整个 DSM detach/destroy 或 MemoryContext reset 一次性完成。
```

如果你需要对象级释放，应该在 TOC 中放一个 DSA area、shared hash table root 或自定义 allocator 的根对象。

### ERROR / abort 时怎么办

`shm_toc_allocate()` 和 `shm_toc_insert()` 可能 ERROR：

```text
out of shared memory
missing required key
wrong magic number
```

它们不自己回滚已分配 chunk。调用者依赖外层 DSM / ResourceOwner cleanup：

```text
parallel leader 初始化失败:
  pcxt->seg 的 ResourceOwner cleanup 或 DestroyParallelContext 清理 DSM。

worker attach 后 lookup 失败:
  worker ERROR，进程退出路径 detach DSM。
```

这也是为什么 TOC 通常在 DSM 初始化阶段使用，而不是在复杂并发业务路径中反复增删对象。

## 7. 正确性机制层次

| 机制 | 保证 | 不保证 |
| --- | --- | --- |
| magic number | attach 方确认这段内存大概率符合预期协议。 | 安全认证；版本完整兼容；对象内容有效。 |
| key | 调用者约定的对象标识。 | 自动避免跨模块冲突；key 的业务含义。 |
| offset | 跨进程地址重定位。 | 指向对象仍然语义有效；对象内部指针可跨进程。 |
| spinlock | `allocate()` / `insert()` 对 TOC 状态的短临界区互斥。 | payload 内容并发安全。 |
| write/read barrier | 无锁 lookup 不读到半发布 entry。 | 对 payload 初始化顺序的全部语义；复杂对象内部同步。 |
| estimator | 创建 segment 前估算空间。 | 运行期自动扩容；估算错误自动修复。 |
| DSM lifecycle | TOC 所在内存存在且当前 backend 已映射。 | TOC 内每个对象有独立释放。 |

特别要注意 payload 初始化：

```text
通常调用者先 shm_toc_allocate()
  -> 初始化 payload
  -> shm_toc_insert() 发布 key
```

TOC 的 barrier 只保护 entry 发布顺序。复杂 payload 内部如果需要并发读写，还必须使用自己的锁、atomic、barrier 或状态机。

## 8. 错误路径 / 异常路径 / fallback

### 8.1 magic mismatch

`shm_toc_attach(magic, address)` 如果发现 `toc_magic` 不匹配，返回 `NULL`。调用者通常 ERROR：

```text
invalid magic number in dynamic shared memory segment
```

这类错误通常说明：

```text
传错 dsm_handle；
worker 和 leader 对 DSM 协议不一致；
attach 到了已经不是预期用途的 memory。
```

### 8.2 估算不足

如果 `shm_toc_estimate_chunk()` 或 `shm_toc_estimate_keys()` 少算了，后续 `shm_toc_allocate()` 或 `shm_toc_insert()` 会报：

```text
out of shared memory
```

这种错误不是全局 shared memory 用尽，而是当前 TOC 管理的这段 DSM layout 不够大。

### 8.3 duplicate key

`shm_toc_insert()` 只在 assert build 中检查重复 key：

```text
#ifdef USE_ASSERT_CHECKING
  Assert(vtoc->toc_entry[i].key != key);
#endif
```

release build 中重复 key 不会被强制阻止，lookup 会返回扫描到的第一个匹配项。课程中要把 key 当成调用者协议：不要依赖 TOC 替你维护 key namespace。

### 8.4 lookup missing key

`shm_toc_lookup(key, noError)`：

```text
noError = false:
  缺失时报 ERROR。

noError = true:
  缺失返回 NULL。
```

调用者必须明确这个 key 是必需还是 optional。TOC 不知道业务语义。

### 8.5 并发 insert 与 lookup

`shm_toc_lookup()` 可以无锁读，但这并不意味着可以把 TOC 当成高并发动态目录。源码注释说得很清楚：TOC 不打算支持大量 key，lookup 只是为了让一组 worker 在启动附近读取一些 entry 时少拿锁。

如果需要动态、高并发、可删除的共享目录，应使用 dshash / DSA / 自定义 shared structure。

## 9. 成本、资源与跨模块传播

### 空间成本

TOC 总空间：

```text
offsetof(shm_toc, toc_entry)
+ number_of_keys * sizeof(shm_toc_entry)
+ sum(BUFFERALIGN(each chunk))
```

`shm_toc_estimate()` 只做这个加法并 `BUFFERALIGN`。它不预留额外增长空间，除非调用者自己估算。

### 时间成本

`allocate()` 和 `insert()` 持有 spinlock，临界区很短。`lookup()` 是线性扫描：

```text
O(number_of_keys)
```

这就是为什么注释反复说 “not intended to scale to a large number of keys”。

### 跨模块传播

TOC 是很多模块的共同 bootstrap 层：

| 模块 | TOC 中放什么 |
| --- | --- |
| `parallel.c` | fixed parallel state、error queue、GUC、snapshot、transaction state、entrypoint 等。 |
| `execParallel.c` | query text、planned statement、param list、tuple queues、instrumentation、executor DSA。 |
| `session.c` | per-session DSA、record typmod registry。 |
| `applyparallelworker.c` | parallel apply shared header、message queue、error queue。 |
| `vacuumparallel.c` | shared vacuum state、index stats、buffer usage、WAL usage。 |

这些模块共同遵守一个模式：

```text
TOC 只放少数 well-known roots；
root object 内部再承载模块自己的复杂状态。
```

## 10. 观测与诊断入口

### gdb

推荐断点：

```gdb
break shm_toc_create
break shm_toc_allocate
break shm_toc_insert
break shm_toc_attach
break shm_toc_lookup
```

观察 parallel query 时：

```gdb
break InitializeParallelDSM
break ParallelWorkerMain
```

在 `shm_toc_insert()` 中看：

```gdb
p key
p offset
p *toc
```

在 worker `shm_toc_lookup()` 中看：

```gdb
p key
p toc
p toc->toc_entry[0]
p/x toc->toc_magic
```

你会看到同一个 key 对应的 offset 相同，但 leader 和 worker 的 `toc` 本地地址可能不同。

### SQL

TOC 本身没有 SQL 视图。可以通过触发 parallel query 间接观察：

```sql
SET max_parallel_workers_per_gather = 2;
SET min_parallel_table_scan_size = 0;
SET parallel_setup_cost = 0;
SET parallel_tuple_cost = 0;

CREATE TABLE shm_toc_demo AS
SELECT g AS id, md5(g::text) AS payload
FROM generate_series(1, 1000000) g;
ANALYZE shm_toc_demo;

EXPLAIN (ANALYZE, VERBOSE)
SELECT count(*) FROM shm_toc_demo;
```

如果 worker 成功启动并执行，说明：

```text
leader 创建 DSM + TOC；
worker attach DSM；
worker 用 TOC 找到 fixed state、error queue、entrypoint 等对象。
```

### 日志与错误

常见错误线索：

```text
invalid magic number in dynamic shared memory segment
could not find key ... in shm TOC
out of shared memory
```

诊断时先问：

```text
是不是传错 DSM handle？
magic 是否和创建者一致？
新增 chunk 时是否同时新增 estimate_chunk 和 estimate_keys？
lookup 使用 noError 的语义是否正确？
```

## 11. 常见误区

### 误区 1：DSM 内可以直接共享普通指针

不能。普通指针只在当前进程地址空间有效。TOC 存 offset，就是为了让每个 backend 用自己的 base address 重建本地 pointer。

### 误区 2：shm_toc 是通用 shared memory allocator

不是。它没有 free，没有 coalescing，没有扩容，也不适合大量 key。它是 bootstrap layout 工具。

### 误区 3：magic number 可以保证对象内容正确

不能。magic 只能挡住明显拿错协议的 attach。对象内部版本、大小、初始化完成状态仍由调用者协议保证。

### 误区 4：lookup 无锁说明 TOC 可以高并发更新

不对。lookup 无锁依赖 insert 的发布顺序和 TOC 小规模使用假设。高并发动态目录应该用其它结构。

### 误区 5：`shm_toc_allocate()` 返回的地址能单独释放

不能。chunk 生命周期绑定整段 DSM / private memory。需要对象级释放时，TOC 里放 DSA 或自定义 allocator。

### 误区 6：key 全局唯一

key 只在某个 TOC 协议内有意义。`PARALLEL_KEY_FIXED`、`SESSION_KEY_DSA` 可以值相同，只要它们属于不同 magic / 不同 DSM 协议即可。

## 12. 课堂实验

### 实验 1：跟踪 parallel TOC 创建与 lookup

目标：看到 leader insert 和 worker lookup 的 key / offset 对应关系。

SQL：

```sql
SET max_parallel_workers_per_gather = 2;
SET min_parallel_table_scan_size = 0;
SET parallel_setup_cost = 0;
SET parallel_tuple_cost = 0;

CREATE TABLE shm_toc_demo AS
SELECT g AS id, md5(g::text) AS payload
FROM generate_series(1, 1000000) g;
ANALYZE shm_toc_demo;

EXPLAIN (ANALYZE, VERBOSE)
SELECT count(*) FROM shm_toc_demo;
```

gdb：

```gdb
break shm_toc_insert
commands
  silent
  printf "insert key=%llu toc=%p address=%p\n", key, toc, address
  continue
end

break shm_toc_lookup
commands
  silent
  printf "lookup key=%llu toc=%p\n", key, toc
  continue
end
```

观察点：

```text
leader insert 的 key 和 worker lookup 的 key 对得上；
toc base address 可能不同；
entry 中保存 offset，因此 lookup 能返回 worker 本地地址。
```

### 实验 2：估算缺失的思维练习

目标：理解 `estimate_chunk` 和 `estimate_keys` 是调用者协议。

阅读 `parallel.c`：

```text
如果新增一个要传给 worker 的 chunk:
  需要新增 shm_toc_estimate_chunk()
  需要新增 shm_toc_estimate_keys()
  需要 shm_toc_allocate()
  需要 shm_toc_insert()
  worker 侧需要 shm_toc_lookup()
```

讨论：

```text
少 estimate_chunk 会在哪里失败？
少 estimate_keys 会在哪里失败？
insert 了 key 但 worker 没 lookup，会有什么影响？
worker lookup 必需 key 但 leader 没 insert，会看到什么错误？
```

### 实验 3：optional key

目标：理解 `noError = true` 的业务语义。

阅读：

```text
parallel.c:
  PARALLEL_KEY_TRANSACTION_SNAPSHOT
```

观察：

```text
leader 只在 IsolationUsesXactSnapshot() 时插入 transaction snapshot；
worker 用 shm_toc_lookup(..., true) 查找；
返回 NULL 是合法状态，不是 TOC 错误。
```

讨论：如果这个 lookup 改成 `false`，哪些 isolation level 或执行场景会出错？

## 13. 讨论题

1. 为什么 `shm_toc_insert()` 存的是 offset，而不是 `void *address`？
2. 为什么 payload 从 segment 尾部向前分配，而 TOC entry 从头部向后增长？
3. `shm_toc_lookup()` 为什么可以不拿 spinlock？它依赖 `shm_toc_insert()` 哪个 ordering？
4. 为什么 duplicate key 只在 assert build 中检查？这对调用者协议意味着什么？
5. 如果一个模块想在 DSM 内维护几千个对象指针，为什么不应该给每个对象一个 TOC key？
6. magic number、key、offset 分别解决什么问题？它们各自不能解决什么问题？

## 14. 本节小结

`shm_toc` 解决的是 DSM 使用中的第一个 bootstrap 问题：

```text
多个 backend attach 同一段 DSM 后，
如何用稳定协议找到里面的少数根对象？
```

它的答案很小：

```text
magic:
  确认 DSM 内容协议。

key:
  标识调用者约定的对象。

offset:
  把跨进程不稳定的 pointer 变成相对地址。

单向 allocator:
  在固定大小 segment 中切出少量 chunk。

barrier + lockless lookup:
  让启动附近的一组 worker 低成本发现已发布 entry。
```

最可迁移的系统规律：

```text
当一段共享内存可能映射到不同本地地址时，
共享结构里不能存裸指针；
先用一个极小的相对寻址目录找到 root object，
再把复杂状态交给 root object 自己的同步和内存管理协议。
```

下一节进入 `shm_mq`：TOC 找到 message queue 后，single-reader / single-writer 的 ring buffer 如何表达消息边界、背压、等待和对端退出。
