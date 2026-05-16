# PostgreSQL allocator 类型、成本与内存诊断

## 课程定位

前置知识：已经理解 MemoryContext tree 表达 backend-local ownership；理解短生命周期 context 为什么主要靠 reset 批量回收；也理解事务、Portal 与 ERROR cleanup 如何决定一个 context 何时 reset 或 delete。

本节唯一主问题：

```text
AllocSet、Generation、Slab 等 allocator 如何匹配分配模式，如何用 pg_backend_memory_contexts、MemoryContextStats() 和 gdb 区分 leak、retention 与正常峰值？
```

核心矛盾：PostgreSQL 想把内存生命周期抽象成统一的 `MemoryContext`，让上层代码只关心“挂到哪个生命周期”；但不同 workload 的分配模式差异很大。小块任意 pfree、FIFO 批量释放、同尺寸对象池、只在 reset 时释放，这些模式如果都交给一个通用 allocator，会在 CPU、chunk header、fragmentation、free list 复用和诊断解释上付出额外成本。

学完后应能判断：一个新 context 应该用 `AllocSet`、`Generation`、`Slab` 还是当前源码里的 `Bump`；看到 `pg_backend_memory_contexts` 或 `MemoryContextStats()` 中某个 context 变大时，能先按 allocator 和生命周期解释它，而不是立刻判断为 leak；也能用 gdb 从 `MemoryContextData`、allocator 私有字段和调用栈确认内存增长来自哪里。

本课基于本地 `~/postgres` 源码，分支 `master`，提交 `8eba2edb8010`。

## 1. 本节在总主线中的位置

前几节回答的是：

```text
内存挂在哪里？
什么时候 reset？
什么时候 delete？
ERROR 后谁兜底？
```

本节进入另一个层次：

```text
同样挂在 MemoryContext tree 上，context 内部用什么 allocator 管理 block 和 chunk？
```

这不是实现细节 trivia。诊断 PostgreSQL 内存问题时，常见现场不是“某个 `palloc()` 明显忘了 `pfree()`”，而是：

```text
pg_backend_memory_contexts 里某个 context total_bytes 很高
RSS 没有回落
MemoryContextStats() 显示 free bytes 很多
某个 SQL 执行中 used_bytes 到达高峰，结束后消失
逻辑复制、HashAgg、tuplestore、tuplesort 的内存曲线不像普通 AllocSet
```

这些现象必须同时解释三层状态：

```text
生命周期层:
  context 何时 reset/delete

allocator 层:
  pfree 后内存是否回到 malloc
  是否保留 empty block 供复用
  是否有 chunk header 和 freelist
  是否允许逐个 pfree/realloc

观测层:
  total_bytes、free_bytes、used_bytes、nblocks 在当前 allocator 下分别意味着什么
```

本节不是重新讲 MemoryContext ownership，而是回答：

```text
为什么同样的 context 生命周期，在不同 allocator 下会表现出不同的内存曲线？
```

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
MemoryContext 提供统一生命周期入口；
AllocSet、Generation、Slab、Bump 用不同 block/chunk 策略匹配分配模式；
诊断时先看 context 生命周期和 allocator 类型，再解释 total/free/used 的变化。
```

核心 tension 是：

```text
上层代码希望统一 palloc/pfree/reset API
  vs
allocator 必须针对真实分配模式牺牲一部分通用性换取低成本和低碎片
```

如果只用一个 allocator，常见问题会变成：

```text
小对象任意释放:
  需要 freelist，否则 pfree 之后难以复用

FIFO 或同代释放:
  freelist 管理可能比按 block 回收更贵

大量同尺寸对象:
  power-of-two bucket 会浪费空间，block 内稀疏对象会造成碎片

只整体 reset 的对象集合:
  每个 chunk 都带 header、支持 pfree/realloc 是额外成本
```

所以当前源码把 allocator 做成 `MemoryContextMethods` 虚表：

```text
palloc()
  -> CurrentMemoryContext
     -> context->methods->alloc(context, size, flags)

pfree(ptr)
  -> 从 chunk 找 owning context
     -> context->methods->free_p(ptr)

MemoryContextReset(context)
  -> context->methods->reset(context)
```

这让上层保持统一抽象，同时让 context 内部选择更合适的 block/chunk 策略。

诊断时要记住一个反直觉点：

```text
MemoryContext 变大不等于 leak；
free_bytes 很大不一定是坏事；
used_bytes 很大也不一定是泄漏；
关键要看该 context 的生命周期是否应该结束，以及 allocator 是否正在故意保留内存供复用。
```

本节最后要沉淀的规律是：

```text
内存诊断不是看一个数值；
而是把 allocator 策略、生命周期边界和 workload 时间线对齐。
```

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 作用 |
| --- | --- | --- |
| 1 | `src/include/nodes/memnodes.h` | `MemoryContextData`、`MemoryContextMethods`、`MemoryContextCounters`；统一抽象和统计字段从这里开始。 |
| 2 | `src/include/utils/memutils.h` | allocator 创建 API：`AllocSetContextCreate`、`GenerationContextCreate`、`SlabContextCreate`、`BumpContextCreate`；默认 block size 参数和 `ALLOCSET_SEPARATE_THRESHOLD`。 |
| 3 | `src/backend/utils/mmgr/README` | MemoryContext 设计总览；解释 context 生命周期、current context、reset/delete callback 和全局知名 context。 |
| 4 | `src/backend/utils/mmgr/mcxt.c` | 类型无关入口：`MemoryContextReset()`、`MemoryContextMemAllocated()`、`MemoryContextMemConsumed()`、`MemoryContextStats()`、`ProcessLogMemoryContextInterrupt()`。 |
| 5 | `src/backend/utils/mmgr/aset.c` | `AllocSet` 通用 allocator；小块 freelist、大块 dedicated block、keeper block、context freelist。 |
| 6 | `src/backend/utils/mmgr/generation.c` | `Generation` allocator；按 block 追踪 allocated/free chunk，适合 FIFO 或同生命周期组。 |
| 7 | `src/backend/utils/mmgr/slab.c` | `Slab` allocator；固定 chunk size，大量同尺寸对象，按 block 空闲程度选择分配位置。 |
| 8 | `src/backend/utils/mmgr/bump.c` | `Bump` allocator；只追加分配，普通构建下 chunk 无 header，不支持 `pfree`/`realloc`/chunk context 查询，只能 reset/delete。 |
| 9 | `src/backend/utils/adt/mcxtfuncs.c` | `pg_get_backend_memory_contexts()`、`pg_log_backend_memory_contexts()`；SQL 和日志诊断入口。 |
| 10 | `src/backend/catalog/system_views.sql` / `doc/src/sgml/system-views.sgml` | `pg_backend_memory_contexts` 视图定义、列语义和权限。 |
| 11 | 典型调用点：`reorderbuffer.c`、`tuplestore.c`、`tuplesort.c`、`nodeAgg.c`、`execGrouping.c` | allocator 选择如何和真实 workload 模式绑定。 |

阅读顺序不要从某个 allocator 的 `Alloc()` 函数一头扎进去。先看 `MemoryContextMethods`：

```text
统一 API:
  alloc / free_p / realloc / reset / delete_context / stats

不同实现:
  AllocSet
  Generation
  Slab
  Bump

诊断入口:
  stats() 填 MemoryContextCounters
  SQL view 和 log 把 counters 展示出来
```

然后再读每个 allocator 的开头注释。PostgreSQL 这里的注释非常关键，因为它直接告诉你该 allocator 是为哪种分配模式设计的。

## 4. 关键数据结构与状态

### `MemoryContextData`

`src/include/nodes/memnodes.h` 中的 `MemoryContextData` 是所有 context 类型的公共头：

```text
type:
  具体 context 类型，例如 AllocSetContext、GenerationContext、SlabContext、BumpContext

isReset:
  创建或上次 reset 后是否没有分配过新空间

mem_allocated:
  该 context 当前向 malloc 层持有的内存量，类型无关快速统计会看它

methods:
  allocator 虚表

parent / firstchild / prevchild / nextchild:
  context tree 链接

name / ident:
  诊断展示名和可变标识

reset_cbs:
  reset/delete 前调用的 callback 链
```

这里要区分两类统计：

```text
context->mem_allocated:
  allocator 维护的“当前从 malloc 持有多少内存”

MemoryContextCounters:
  stats() 计算出来的 nblocks / freechunks / totalspace / freespace
```

这两个入口服务不同场景：

```text
MemoryContextMemAllocated(context, recurse):
  快速汇总 mem_allocated

MemoryContextMemConsumed(context, &counters):
  调用每个 context 的 stats()，得到更接近 MemoryContextStats 输出的统计

MemoryContextStats(context):
  打印 context tree 和 totals，调试用
```

### `MemoryContextMethods`

`MemoryContextMethods` 是本节最重要的抽象点：

```text
alloc:
  分配 chunk

free_p:
  释放 chunk，或在某些 allocator 中拒绝释放

realloc:
  改变已有 chunk 大小

reset:
  让 context 回到可复用或空状态

delete_context:
  释放 context 自身占用的所有内存

get_chunk_context:
  从 pointer 找 owning context

get_chunk_space:
  估算 pointer 在 context 中消耗的空间

is_empty:
  判断 context 是否空

stats:
  填写 MemoryContextCounters 并可打印人类可读字符串
```

这解释了两个现象：

```text
1. 上层代码大多数时候只看到 palloc/pfree/reset。
2. 不同 allocator 可以让同一个 API 有不同成本和限制。
```

例如 `Bump` 在普通构建下没有 chunk header，所以 `pfree()`、`realloc()`、`GetMemoryChunkContext()`、`GetMemoryChunkSpace()` 都不是正常可用能力。这个限制不是 bug，而是它换取更紧凑布局的条件。

### `MemoryContextCounters`

`MemoryContextCounters` 有四个字段：

```text
nblocks:
  allocator 持有的 malloc block 数

freechunks:
  allocator 认为空闲 chunk 的数量

totalspace:
  从 malloc 层持有或可统计的总空间

freespace:
  allocator 统计出来的空闲空间
```

`pg_backend_memory_contexts` 最终展示：

```text
total_bytes  = totalspace
total_nblocks = nblocks
free_bytes   = freespace
free_chunks  = freechunks
used_bytes   = totalspace - freespace
```

注意 `memnodes.h` 注释提醒：这组 counters 历史上偏向 `AllocSet` 的展示格式。如果 allocator 策略不同，字段语义会有细微差异。

最重要的例子是 `GenerationStats()`：

```text
freespace 只统计 block 尾部尚未使用的空间；
已经 pfree 的 chunk 空间并不全部计入 freespace，因为那部分大小不可直接得知。
```

所以不能把所有 allocator 下的 `used_bytes` 都解释成“仍被用户对象持有的精确字节数”。它是诊断入口，不是精确 heap profiler。

### `AllocSetContext`

`AllocSet` 是通用 allocator，也是最常见的 MemoryContext 实现。

核心状态：

```text
blocks:
  从 malloc 取得的 block 链表

freelist[ALLOCSET_NUM_FREELISTS]:
  小块按 power-of-two size class 复用

initBlockSize / maxBlockSize / nextBlockSize:
  block 增长参数

allocChunkLimit:
  小块走 freelist 的上限

freeListIndex:
  context 自身是否可进入全局 context freelist
```

关键策略：

```text
小块:
  按 size class 放进 freelist，pfree 后通常不还给 malloc

大块:
  dedicated block，pfree 时可直接还给 malloc

新增 block:
  通常按 initBlockSize 开始，逐步翻倍到 maxBlockSize

reset:
  释放除 keeper block 外的 block，并清空 freelist
```

诊断含义：

```text
AllocSet 的 free_bytes 高:
  可能表示 context 内有很多可复用空间

AllocSet 的 total_bytes 高但生命周期未结束:
  可能是正常峰值后的 retention

AllocSet 长期挂在 TopMemoryContext/CacheMemoryContext 下并持续增长:
  需要继续判断是缓存增长、工作集增长还是 leak
```

### `GenerationContext`

`Generation` 适合“分配和释放大致同代”的对象。源码注释直接说明它假设 chunk 大致按 FIFO 或相近生命周期分组释放。

核心状态：

```text
blocks:
  GenerationBlock 双向链表

block:
  当前分配使用的 block

freeblock:
  一个已空 block，可优先复用

每个 GenerationBlock:
  nchunks
  nfree
  freeptr
  endptr
```

关键策略：

```text
pfree:
  更新 block 的 nfree

当 block 中所有 chunk 都释放:
  这个 block 变成可复用 block 或还给 malloc

freeblock:
  context 最多重点保留一个空 block 供复用
```

适用场景：

```text
tuplestore tuple memory:
  tuple 按近似 FIFO 模式进入和离开内存

logical decoding variable-length tuple data:
  变量长度数据以相近生命周期分组释放
```

诊断含义：

```text
Generation 的 used_bytes 可能高估仍活跃对象，因为 freed chunks 不一定计入 freespace。
判断它是不是问题，要看 block 是否能在工作流推进后变空并复用或释放。
```

### `SlabContext`

`Slab` 适合大量同尺寸对象。

核心状态：

```text
chunkSize:
  调用者请求的固定对象大小

fullChunkSize:
  加上 chunk header 和对齐后的内部 chunk 大小

blockSize:
  每个 block 大小

chunksPerBlock:
  每个 block 能容纳多少 chunk

emptyblocks:
  空 block 缓存

blocklist[]:
  按 free chunk 数量范围分组的 block 链表
```

关键策略：

```text
所有 chunk 等大:
  减少 power-of-two bucket 浪费

优先从“最满的可用 block”分配:
  避免很多 block 都稀疏地留着少量活对象

block 变空:
  进入 emptyblocks，超过上限再释放
```

适用场景：

```text
logical decoding ReorderBufferChange:
  固定长度 change 结构

logical decoding ReorderBufferTXN:
  固定长度 transaction 结构

radixtree 节点 slab:
  同类节点对象池
```

诊断含义：

```text
Slab 的 free_chunks 很多:
  说明固定大小对象池里有空闲槽

Slab 的 total_bytes 高:
  可能是对象池为峰值保留 block，也可能是长期活对象分散导致 block 不能完全释放
```

### `BumpContext`

虽然 README 主题里只点名 `AllocSet`、`Generation`、`Slab`，但当前源码已经有 `Bump`，诊断时经常会在 `pg_backend_memory_contexts.type = 'Bump'` 中看到它。

`Bump` 的设计约束很强：

```text
只追加分配
不需要逐个 pfree
不需要 realloc
不需要从 pointer 查 owning context
只通过 reset/delete 整体释放
```

普通构建下，`Bump` chunk 没有 `MemoryChunk` header，因此每个对象少了 header 成本，也能更紧凑地塞进 block。

适用场景：

```text
HashAgg hashed tuples:
  hash table entries 不逐个释放，整个 hash table reset 时一起丢弃

SubPlan hashed tuples:
  hash 表条目不需要单独 pfree

RecursiveUnion hashed tuples:
  rescan 时整体丢弃

tuplesort caller tuples:
  非 bounded sort 可以用 bump 减少浪费和 CPU 成本
```

诊断含义：

```text
Bump used_bytes 高:
  通常表示该批对象仍在当前批次生命周期内

Bump free_bytes 高:
  多数是当前 block 尾部尚未使用空间

不能尝试用 pfree 单个 Bump chunk 来“修 leak”:
  正确边界是 reset/delete owning context
```

## 5. 主流程源码 walkthrough

本节用三条主流程串起来：分配路径、统计路径、真实模块选择 allocator 的路径。

### 5.1 `palloc()` 如何落到具体 allocator

类型无关的主链路是：

```text
palloc(size)
  -> context = CurrentMemoryContext
  -> context->methods->alloc(context, size, flags)
  -> 返回 allocator-specific chunk pointer
```

`palloc()` 本身不关心这是 `AllocSet`、`Generation`、`Slab` 还是 `Bump`。它只依赖 `CurrentMemoryContext` 指向一个合法 `MemoryContext`。

所以诊断某次分配来自哪里，第一步不是看 `palloc()`，而是看：

```text
当前调用链是否切换了 CurrentMemoryContext
目标 context 的 name/type 是什么
该 context 的 parent 生命周期是什么
```

例如执行器表达式求值中，很多临时分配发生在 per-tuple context；HashAgg hash tuple 则可能落在 `HashAgg hashed tuples` 的 `BumpContext`。

### 5.2 `pfree()` 如何回到 owning context

通用 allocator 下，`pfree(ptr)` 不依赖当前 context：

```text
pfree(ptr)
  -> 从 chunk header 找 owning context
  -> context->methods->free_p(ptr)
```

这对 `AllocSet`、`Generation`、`Slab` 很重要，因为它们都能从 chunk 找回 context。

但 `Bump` 是刻意的例外：

```text
普通构建:
  Bump chunk 没有 header
  不能普通 pfree/realloc

MEMORY_CONTEXT_CHECKING 构建:
  增加 header 用于发现误用
```

这提醒我们：

```text
allocator 选择会改变该 context 允许的 API 语义。
```

如果一个数据结构未来需要逐个删除条目，就不能简单为了省 header 成本改成 `BumpContext`。

### 5.3 `MemoryContextStats()` 如何生成输出

源码路径：

```text
MemoryContextStats(context)
  -> MemoryContextStatsDetail(context, 100, 100, true)
     -> MemoryContextStatsInternal(...)
        -> context->methods->stats(context, MemoryContextStatsPrint, ...)
        -> 递归或汇总 children
     -> 打印 Grand total
```

`MemoryContextStats()` 默认打到 stderr；`pg_log_backend_memory_contexts(pid)` 通过 procsignal 让目标 backend 在安全点执行：

```text
HandleLogMemoryContextInterrupt()
  -> 设置 LogMemoryContextPending

ProcessLogMemoryContextInterrupt()
  -> 防递归
  -> LOG_SERVER_ONLY "logging memory contexts of PID ..."
  -> MemoryContextStatsDetail(TopMemoryContext, 100, 100, false)
```

这里有几个诊断细节：

```text
1. 日志每个 context 一条 LOG_SERVER_ONLY，不发给客户端。
2. 深度和每个 parent 的 child 数默认限制为 100。
3. 如果 child 很多，会输出 summary，而不是所有子 context。
4. 打印本身可能很长，所以频繁调用会有明显日志和 I/O 成本。
```

### 5.4 `pg_backend_memory_contexts` 如何生成行

SQL view 定义很薄：

```sql
CREATE VIEW pg_backend_memory_contexts AS
    SELECT * FROM pg_get_backend_memory_contexts();
```

`pg_get_backend_memory_contexts()` 做了几件事：

```text
从 TopMemoryContext 开始广度优先遍历
为每个 context 分配 transient context_id
用 ancestor context_id 构造 path
调用 context->methods->stats() 填 MemoryContextCounters
把 type 映射成 AllocSet / Generation / Slab / Bump
输出 total_bytes / total_nblocks / free_bytes / free_chunks / used_bytes
```

`path` 是诊断 parent-child 关系的关键字段，但它不是稳定 ID。文档明确提醒：context 会在查询运行中创建和销毁，同一条 SQL 多次求值时 path 可能不一致。因此聚合某个子树时，推荐先用 CTE 固定一次视图结果。

示例：

```sql
WITH memory_contexts AS (
  SELECT * FROM pg_backend_memory_contexts
)
SELECT
  c2.name AS root,
  sum(c1.total_bytes) AS subtree_total_bytes,
  sum(c1.used_bytes) AS subtree_used_bytes
FROM memory_contexts c1
JOIN memory_contexts c2
  ON c1.path[c2.level] = c2.path[c2.level]
WHERE c2.name = 'CacheMemoryContext'
GROUP BY c2.name;
```

注意这个 SQL 观察的是当前 session 的 backend-local contexts，不是全实例所有 backend 的内存。

### 5.5 `AllocSet` 的分配和释放时间线

典型 `AllocSet` 时间线：

```text
Create:
  AllocSetContextCreate(...)
  分配 context header 和 keeper block

Small alloc:
  从 freelist 找合适 size class
  或从当前 block 切一块
  或分配新 block

pfree small chunk:
  chunk 放回对应 freelist
  block 通常不还给 malloc

Large alloc:
  dedicated block

pfree large chunk:
  dedicated block 可直接释放

Reset:
  释放非 keeper block
  清 freelist
  保留 context 自身和 keeper block 供复用

Delete:
  释放整个 context
```

这解释了为什么：

```text
一个长期 AllocSet context 经历峰值后，total_bytes 可能不马上回落；
free_bytes 增大可能只是说明 freelist 中有可复用 chunk。
```

如果该 context 会在每条消息、每条 query 或事务结束 reset，这通常不是 leak。它是 allocator 为减少 malloc/free 成本做的 retention。

### 5.6 `Generation` 的同代释放时间线

`Generation` 时间线：

```text
Create:
  GenerationContextCreate(...)
  创建 keeper block

Alloc:
  在当前 block 尾部追加 chunk
  大对象可能使用 dedicated block

pfree:
  标记该 chunk 所在 block 的 nfree 增加

Block all free:
  如果 freeblock 为空，保留为 freeblock
  否则释放该空 block

Reset:
  释放除 keeper block 外的 block
```

这比 `AllocSet` 更适合 FIFO/同代释放，因为一个 block 内对象如果成组失效，就能整体回收或复用，避免很多小 freelist 操作。

典型源码现场：

```text
tuplestore_begin_common()
  -> state->context = GenerationContextCreate(..., "tuplestore tuples", ...)

ReorderBufferAllocate()
  -> buffer->tup_context = GenerationContextCreate(..., "Tuples", ...)
```

对于逻辑解码，注释还特别提到：如果 evict oldest changes，更容易让 generational allocator 真正释放内存，因为旧 change 的生命周期更接近同代。

### 5.7 `Slab` 的固定尺寸对象池时间线

`Slab` 时间线：

```text
Create:
  SlabContextCreate(parent, name, blockSize, chunkSize)
  计算 fullChunkSize 和 chunksPerBlock

Alloc:
  选择当前“最满但仍有空位”的 block
  优先复用已经用过后释放的 chunk
  不够时分配新 block

pfree:
  chunk 放到 block-level freelist
  更新 block 的 nfree
  必要时移动到新的 blocklist bucket

Block empty:
  放入 emptyblocks
  emptyblocks 超过上限后释放多余 block
```

典型源码现场：

```text
ReorderBufferAllocate()
  -> change_context = SlabContextCreate(..., sizeof(ReorderBufferChange))
  -> txn_context    = SlabContextCreate(..., sizeof(ReorderBufferTXN))
```

这里对象大小固定，`Slab` 可以避免 `AllocSet` power-of-two size class 带来的内部浪费，也能减少长期对象散落在多个 block 上导致的碎片。

### 5.8 `Bump` 的只追加分配时间线

`Bump` 时间线：

```text
Create:
  BumpContextCreate(...)
  创建 keeper block

Alloc:
  从当前 block 尾部推进 freeptr
  普通构建下不写 chunk header

pfree / realloc / get_chunk_context:
  不作为正常能力

Reset:
  一次释放或重置整个对象集合
```

典型源码现场：

```text
nodeAgg.c:
  HashAgg hashed tuples 用 BumpContext

execGrouping.c:
  TupleHashTable 的 tuplescxt 通常希望是 BumpContext

nodeSubplan.c:
  SubPlan hashed tuples 用 BumpContext

nodeRecursiveunion.c:
  RecursiveUnion hashed tuples 用 BumpContext
```

`Bump` 非常适合 hash table entries 这类对象：

```text
创建很多
查找很多
不单独删除
rescan/end 时整体 reset/delete
```

## 6. 生命周期 / ownership / cleanup

allocator 只决定 context 内部如何管理 block 和 chunk；它不决定 context 应该活多久。

这句话是诊断中最容易漏掉的边界：

```text
Context lifetime is owned by parent tree and module protocol.
Allocator policy is owned by context type.
```

### 创建者负责匹配生命周期和分配模式

创建 context 时，模块同时做两件事：

```text
选择 parent:
  决定生命周期边界

选择 allocator:
  决定 context 内部成本模型
```

例如：

```text
ExecutorState:
  parent 通常在 query 生命周期内
  常用 AllocSet，因为 executor state 中有多种大小和释放模式

per-tuple ExprContext:
  parent 在 query 内
  常用 AllocSet，小而频繁 reset

tuplestore tuples:
  parent 是调用者 context
  allocator 用 Generation，因为 tuple 内存模式接近 FIFO

HashAgg hashed tuples:
  parent 在 executor query context 下
  allocator 用 Bump，因为条目整体释放

ReorderBuffer fixed structs:
  parent 是 ReorderBuffer context
  allocator 用 Slab，因为对象大小固定且数量多
```

### reset 和 delete 的语义仍然由 MemoryContext 统一

无论 allocator 类型如何，`MemoryContextReset()` 都是：

```text
调用 reset callbacks
删除 children
调用 context->methods->reset(context)
```

`MemoryContextDelete()` 则删除整个 subtree，再调用 allocator 的 delete 方法。

这意味着：

```text
AllocSet / Generation / Slab / Bump 都可以被 reset/delete 批量回收。
区别只在于 reset 前的 pfree/realloc 支持、内部复用策略和统计解释。
```

### pfree 能力不是所有 context 都等价

这是 allocator 选择最硬的边界：

```text
如果对象需要单独 pfree:
  不要把它放进 BumpContext

如果对象大小不固定:
  Slab 不合适

如果释放顺序随机且对象大小混杂:
  Generation 可能不如 AllocSet

如果对象同尺寸、大量、反复分配释放:
  Slab 往往比 AllocSet 更可解释
```

### cleanup 时不要把 allocator retention 误认为没有 cleanup

典型误判：

```text
看到 total_bytes 没回到 0
  -> 以为 context 没 cleanup
```

但很多 context reset 后仍会保留 context header 和 keeper block。长期 context 为了复用，也可能保留一定 block。

更合理的判断顺序：

```text
1. 这个 context 是否应该被 delete？
2. 如果不 delete，它是否应该 reset？
3. reset 后 total_bytes 是否降到该 allocator 的合理基线？
4. 如果长期不 reset，它的 parent 是否就是长期 context？
5. 增长是否随 workload 无界增加？
```

## 7. 正确性机制层次

本节主题主要是 backend-local memory，不涉及 shared memory lock、WAL-before-data 或 visibility horizon。但它仍然有几个正确性边界。

### `MemoryContextMethods` 是 allocator 契约

每个 allocator 必须符合公共契约：

```text
alloc:
  处理 MCXT_ALLOC_HUGE 和 MCXT_ALLOC_NO_OOM

realloc:
  处理 size change 语义，除非 allocator 明确不支持

reset/delete:
  释放该 context 管理的内存

stats:
  正确累加 MemoryContextCounters

is_empty:
  告诉上层 context 是否为空
```

这保证上层 MemoryContext tree 可以统一遍历、reset、delete、stats。

### chunk ownership 依赖 header 或 allocator 特殊约束

`pfree()`、`repalloc()`、`GetMemoryChunkContext()` 依赖从 pointer 找回 owning context。多数 allocator 通过 chunk header 支持这个能力。

`Bump` 为了省掉 header，牺牲了这些能力。因此它只能放在调用范围受控、不会把 pointer 暴露给大量通用代码的场景。

这是一条工程判断规则：

```text
越通用的 pointer，越应该来自支持普通 pfree/realloc/diagnostic 的 allocator。
越局部、越批量释放的 pointer，越可以考虑 Bump。
```

### `MaxAllocSize` 和 huge allocation

`memutils.h` 中有两个上限：

```text
MaxAllocSize:
  普通 palloc 上限，约 1GB - 1

MaxAllocHugeSize:
  更高上限，供 MemoryContextAllocHuge() 一类小心使用的调用者
```

这不是 allocator 任意决定的。很多 varlena 类型、长度字段和计算逻辑假设普通可分配大小不会超过 `MaxAllocSize`。

因此内存诊断中看到 OOM 或大对象分配失败时，不要只看系统可用内存，还要看调用路径是否触碰 PostgreSQL 自己的分配大小约束。

### stats 是观测接口，不是精确对象图

`MemoryContextStats()` 和 `pg_backend_memory_contexts` 的输出来自 allocator 的 `stats()`。它们能告诉你：

```text
context tree 形状
allocator 类型
block 数
总持有空间
allocator 认为的空闲空间
估算 used 空间
```

它们不能直接告诉你：

```text
哪个 SQL 源码行分配了某个 chunk
哪个对象仍然持有 pointer
free_bytes 是否一定能还给 OS
used_bytes 是否等于用户级活对象大小
```

要回答这些问题，需要结合 gdb、断点、调用栈、workload 时间线和 context 生命周期边界。

## 8. 错误路径 / 异常路径 / fallback

### OOM 时 allocator 会走 ERROR

`palloc()` 默认不会返回 NULL。OOM 时一般会：

```text
MemoryContextStats(TopMemoryContext)
ereport(ERROR, "out of memory")
```

具体 allocator 的 create 或 alloc 路径里也会在 malloc 失败时打印 memory context stats，再抛 ERROR。这是为什么 OOM 日志常常能看到一串 MemoryContextStats 输出。

例外是：

```text
palloc_extended(..., MCXT_ALLOC_NO_OOM)
```

调用者明确允许返回 NULL，用来实现可降级路径。

诊断 OOM 时，第一步通常不是直接看 core，而是先找日志中的：

```text
TopMemoryContext
Grand total
最大 total/used 的 context
这些 context 的 name/type/level
```

### `MemoryContextStats()` 本身要控制输出规模

`MemoryContextStatsDetail()` 有 `max_level` 和 `max_children` 参数。默认 `MemoryContextStats()` 使用硬编码的 100/100。

原因很现实：

```text
内存已经很大时，完整打印 context tree 可能非常长；
如果某个 parent 下有大量 sibling，逐个打印诊断价值下降，日志成本上升。
```

所以看到：

```text
N more child contexts containing ...
```

不要以为 stats 丢失了总量。它只是省略了部分 child 的逐项输出，但会把 summary 加进 totals。

### `pg_log_backend_memory_contexts()` 是异步信号触发

`pg_log_backend_memory_contexts(pid)` 不直接进入目标 backend 的内存结构。它：

```text
查找 PID 是否是 backend 或 auxiliary process
发送 PROCSIG_LOG_MEMORY_CONTEXT
目标进程在 CHECK_FOR_INTERRUPTS 或进程特定 interrupt handler 中处理
```

所以它有几个 fallback 语义：

```text
目标进程可能已经退出:
  返回 false 并 WARNING

目标进程长时间不处理 interrupt:
  日志不会立刻出现

频繁调用:
  可能制造大量日志和 I/O 开销

递归调用风险:
  ProcessLogMemoryContextInterrupt() 用 LogMemoryContextInProgress 防重入
```

### `Bump` 误用会在 debug build 中更早暴露

普通构建下 `Bump` 没有 chunk header，很多 chunk-level 操作不可行。`MEMORY_CONTEXT_CHECKING` 构建会给 Bump chunk 增加 header，用来发现误用。

这带来一个诊断差异：

```text
debug build 中 Bump 的空间成本和生产构建不同；
但 debug build 能更早发现 pfree/realloc Bump chunk 这类错误。
```

不要拿 `MEMORY_CONTEXT_CHECKING` 下的精确 byte 数去推导生产环境的 planner 或 memory accounting 行为。源码里 `execGrouping.c` 对 Bump 空间估算也特别说明：debug build 会有额外 chunk header，但不希望 debug build 改变生产规划选择。

## 9. 成本、资源与跨模块传播

### allocator 成本维度

选择 allocator 时，至少看这些成本：

```text
chunk header 成本:
  每个对象是否需要 header

alignment 浪费:
  请求大小到 MAXALIGN 或 power-of-two 的差距

freelist 成本:
  pfree 后是否维护 size class 或 block-level freelist

malloc/free 次数:
  是否能批量复用 block

fragmentation:
  长寿对象是否把大 block 卡住

reset/delete 成本:
  是否能整体快速释放

诊断可见性:
  stats 字段是否容易解释，chunk 是否能反查 context
```

### `AllocSet` 的成本画像

优点：

```text
通用
支持 pfree/realloc
小块复用成熟
大块 dedicated block 可及时释放
大多数 mixed-size workload 足够好
```

代价：

```text
每个 chunk 有 header
小块按 power-of-two size class 可能有内部浪费
随机释放后的 freelist retention 让 total_bytes 不一定马上下降
长期 context 中可能形成可复用但看起来偏大的空间
```

适合：

```text
对象大小混杂
需要逐个 pfree 或 repalloc
模块没有非常明确的释放模式
普通 executor/planner/cache 子结构
```

### `Generation` 的成本画像

优点：

```text
适合 FIFO 或同生命周期组
block 级回收比复杂 freelist 更简单
对 queue-like tuple 数据更容易减少碎片
```

代价：

```text
释放顺序随机时，block 不容易全空
freespace 统计不包含所有已 free chunk 空间，used_bytes 解释更粗
不适合大量长期随机保留对象
```

适合：

```text
tuplestore
logical decoding 中变量长度 tuple 数据
构造、处理、丢弃顺序相近的数据流
```

### `Slab` 的成本画像

优点：

```text
固定大小对象空间浪费低
block 内 free list 简单
优先填充最满 block，减少稀疏 block
free_chunks 对对象池状态很直观
```

代价：

```text
只能服务固定 chunk size
对象大小变化时需要多个 slab 或退回 AllocSet
emptyblocks 可能保留少量空 block 供复用
```

适合：

```text
大量同尺寸 struct
对象频繁分配释放
希望避免 AllocSet power-of-two 浪费
```

### `Bump` 的成本画像

优点：

```text
普通构建下无 per-chunk header
追加分配路径简单
空间更紧凑，cache line 更友好
整体 reset/delete 成本模型清晰
```

代价：

```text
不能逐个 pfree
不能普通 realloc
不能从 pointer 反查 context
pointer 暴露范围必须非常受控
debug build 空间成本不同
```

适合：

```text
hash table entries
批量创建、批量丢弃的数据
不会单独释放的短生命周期集合
```

### 跨模块传播：allocator 选择会影响 memory accounting

`nodeAgg.c` 中有一个很好的例子：

```text
HashAgg meta context:
  AllocSet
  因为 bucket array 可能扩容，旧 array 会被 pfree

HashAgg hashed tuples:
  Bump
  因为 hash entries 不逐个释放

Transition values:
  AllocSet
  因为大小和释放模式不同，还要考虑 chunk header 和 power-of-two allocation
```

这说明同一个 executor node 内部也可能同时使用多个 allocator。不要以为“HashAgg 内存大”就对应一个 context 或一种 allocator。

更好的诊断句式是：

```text
HashAgg 的哪类内存大？
  bucket/meta?
  hashed tuple?
  transition value?
  per-tuple temporary?

它们分别在哪个 context？
它们分别是什么 allocator？
它们各自什么时候 reset/delete？
```

## 10. 观测与诊断入口

### SQL: 当前 session 的 context tree

基础查看：

```sql
SELECT
  name,
  ident,
  type,
  level,
  total_bytes,
  free_bytes,
  used_bytes
FROM pg_backend_memory_contexts
ORDER BY total_bytes DESC
LIMIT 30;
```

看 allocator 分布：

```sql
SELECT
  type,
  count(*) AS contexts,
  pg_size_pretty(sum(total_bytes)) AS total,
  pg_size_pretty(sum(used_bytes)) AS used,
  pg_size_pretty(sum(free_bytes)) AS free
FROM pg_backend_memory_contexts
GROUP BY type
ORDER BY sum(total_bytes) DESC;
```

看某个子树，先用 CTE 固定 `path`：

```sql
WITH m AS (
  SELECT * FROM pg_backend_memory_contexts
),
root AS (
  SELECT * FROM m WHERE name = 'CacheMemoryContext'
)
SELECT
  pg_size_pretty(sum(m.total_bytes)) AS subtree_total,
  pg_size_pretty(sum(m.used_bytes)) AS subtree_used,
  pg_size_pretty(sum(m.free_bytes)) AS subtree_free
FROM m, root
WHERE m.path[root.level] = root.path[root.level];
```

看同名 context：

```sql
SELECT
  name,
  type,
  count(*) AS n,
  pg_size_pretty(sum(total_bytes)) AS total,
  pg_size_pretty(sum(used_bytes)) AS used
FROM pg_backend_memory_contexts
GROUP BY name, type
ORDER BY sum(total_bytes) DESC
LIMIT 30;
```

### SQL: 让目标 backend 打日志

当前 session：

```sql
SELECT pg_log_backend_memory_contexts(pg_backend_pid());
```

其他 backend：

```sql
SELECT pg_log_backend_memory_contexts(pid)
FROM pg_stat_activity
WHERE pid <> pg_backend_pid()
  AND state = 'active';
```

注意：

```text
默认需要 superuser，或具有相应授权。
输出在 server log 中，不在当前 SQL result 中。
目标进程要到 CHECK_FOR_INTERRUPTS 附近才会处理。
```

### gdb: 从 context 看公共字段

附加到 backend 后：

```gdb
(gdb) p TopMemoryContext
(gdb) p *TopMemoryContext
(gdb) p TopMemoryContext->firstchild
(gdb) p TopMemoryContext->mem_allocated
(gdb) p TopMemoryContext->name
```

如果知道某个 context 指针：

```gdb
(gdb) p ((MemoryContext) ctx)->type
(gdb) p ((MemoryContext) ctx)->name
(gdb) p ((MemoryContext) ctx)->ident
(gdb) p ((MemoryContext) ctx)->parent
(gdb) p ((MemoryContext) ctx)->mem_allocated
```

想判断 allocator 类型，可以先看 `type`，再 cast 到具体结构：

```gdb
(gdb) p ((AllocSetContext *) ctx)->blocks
(gdb) p ((AllocSetContext *) ctx)->allocChunkLimit

(gdb) p ((GenerationContext *) ctx)->freeblock
(gdb) p ((GenerationContext *) ctx)->block

(gdb) p ((SlabContext *) ctx)->chunkSize
(gdb) p ((SlabContext *) ctx)->chunksPerBlock

(gdb) p ((BumpContext *) ctx)->blocks
```

这些结构定义在 `.c` 文件里，不在公开 header 中。gdb 能否直接识别类型取决于是否带 debug symbols，以及你是否在对应编译单元类型信息可见的构建中调试。

### gdb: 在分配入口打断点

常用断点：

```gdb
(gdb) b MemoryContextAlloc
(gdb) b MemoryContextAllocExtended
(gdb) b AllocSetAlloc
(gdb) b GenerationAlloc
(gdb) b SlabAlloc
(gdb) b BumpAlloc
(gdb) b MemoryContextReset
(gdb) b MemoryContextDelete
(gdb) b MemoryContextStats
```

如果断点太热，可以加条件：

```gdb
(gdb) b MemoryContextAlloc if strcmp(context->name, "HashAgg hashed tuples") == 0
```

或者在 allocator-specific 函数里按 size 过滤：

```gdb
(gdb) b AllocSetAlloc if size > 8192
```

诊断流程：

```text
1. 先用 pg_backend_memory_contexts 找 context name/type。
2. gdb 对该 allocator 的 Alloc/Reset/Delete 打断点。
3. 运行 workload。
4. 看 backtrace 确认增长来自哪个模块路径。
5. 看 reset/delete 是否按预期发生。
```

### 从日志读 `MemoryContextStats()`

典型输出形态：

```text
level: 2; MessageContext: 16384 total in 2 blocks; 5152 free (0 chunks); 11232 used
Grand total: 1651920 bytes in 201 blocks; 622360 free (88 chunks); 1029560 used
```

读法：

```text
level:
  context tree 深度

name:
  context name，dynahash 会优先显示 ident

total:
  allocator 统计的 totalspace

blocks:
  malloc block 数

free:
  allocator 统计的 freespace

chunks:
  free chunk 数

used:
  total - free
```

不同 allocator 可打印额外信息：

```text
Generation:
  可能包含 chunks 总数

Slab:
  包含 empty blocks

Bump:
  没有 free chunks 数，因为它只统计尾部 free space
```

## 11. 常见误区

### 误区一：`total_bytes` 增长就是 leak

不一定。

更准确的判断：

```text
如果 context 生命周期尚未结束:
  total_bytes 增长可能是正常工作集或峰值

如果 context reset 后仍保留 keeper block:
  小的 total_bytes 基线正常

如果 AllocSet free_bytes 很高:
  可能是 freelist retention

如果长期 context 随请求数无界增长:
  才更接近 leak 或缓存无界增长
```

### 误区二：`free_bytes` 高说明内存已经还给 OS

不是。

`free_bytes` 是 allocator 内部认为可复用的空间，不等于 OS 已经回收的 RSS。尤其是：

```text
AllocSet small chunk pfree:
  通常进入 freelist，不还给 malloc

Slab emptyblocks:
  可能保留少量空 block

Generation freeblock:
  可能保留一个空 block 供复用
```

### 误区三：所有 allocator 的 `used_bytes` 都能横向比较

不要这么做。

`used_bytes = totalspace - freespace`，而 `freespace` 由 allocator-specific `stats()` 决定。

例如：

```text
AllocSet:
  freelist chunk 能计入 free space

Generation:
  freespace 不完整反映 freed chunks

Bump:
  free 主要是 block 尾部未使用空间
```

所以 `used_bytes` 更适合在同一 context、同一 allocator、同一 workload 时间线上做比较。

### 误区四：为了省内存把 context 都换成 Slab 或 Bump

allocator 是模式匹配，不是越特殊越好。

```text
Slab:
  只适合同尺寸对象

Bump:
  不支持普通 pfree/realloc

Generation:
  随机长寿对象会降低效果

AllocSet:
  通用性强，很多场景的正确默认选择
```

如果模块 API 会把 pointer 交给通用代码，`Bump` 的限制尤其危险。

### 误区五：`pg_backend_memory_contexts` 能直接看所有 backend

它只显示当前 session 所在 server process 的 memory contexts。

要看其他 backend，只能：

```text
用 pg_log_backend_memory_contexts(pid) 请求目标进程打日志
或 attach gdb 到目标 backend
或结合 OS/RSS/perf 等外部工具
```

### 误区六：长期 context 下的增长一定是 bug

长期 context 包括：

```text
TopMemoryContext
CacheMemoryContext
各种 backend lifetime cache context
```

它们增长可能是：

```text
合法 cache warmup
连接生命周期内的工作集增长
extension/backend-local cache
真正 leak
```

判断关键是：

```text
是否有失效机制？
是否随 schema/workload 达到平台？
是否每次重复同样操作都会线性增长？
是否 ERROR/transaction end/message end 后仍不回落到合理平台？
```

## 12. 课堂实验

### 实验一：观察当前 backend 的 allocator 类型分布

目标：确认你的 PostgreSQL 版本中有哪些 MemoryContext allocator 类型。

步骤：

```sql
SELECT
  type,
  count(*) AS n,
  pg_size_pretty(sum(total_bytes)) AS total,
  pg_size_pretty(sum(used_bytes)) AS used
FROM pg_backend_memory_contexts
GROUP BY type
ORDER BY sum(total_bytes) DESC;
```

观察点：

```text
AllocSet 是否占大多数？
是否能看到 Bump、Generation 或 Slab？
不同阶段执行查询前后，类型分布是否变化？
```

源码回扣：

```text
src/backend/utils/adt/mcxtfuncs.c
  type switch 映射 T_AllocSetContext / T_GenerationContext / T_SlabContext / T_BumpContext
```

### 实验二：观察 MessageContext reset

目标：区分“峰值”和“跨消息 retention”。

步骤：

```sql
SELECT name, type, total_bytes, free_bytes, used_bytes
FROM pg_backend_memory_contexts
WHERE name = 'MessageContext';
```

然后执行一条较长 SQL 或带较大常量的语句，再重新观察。

预期：

```text
MessageContext 在 PostgresMain 主循环顶部 reset。
同一条消息处理中可能变大；
下一条消息开始后应回到较小基线。
```

源码回扣：

```text
src/backend/tcop/postgres.c
  main loop 顶部 MemoryContextReset(MessageContext)
```

### 实验三：观察排序或聚合中的 BumpContext

目标：看到 `Bump` 在 executor 内部的真实使用。

可以尝试构造排序或 hash aggregate workload：

```sql
CREATE TEMP TABLE t AS
SELECT g, md5(g::text) AS v
FROM generate_series(1, 500000) g;

SELECT g % 10000 AS k, count(*), min(v), max(v)
FROM t
GROUP BY 1;
```

在另一个 session 中找到该 backend PID 后：

```sql
SELECT pg_log_backend_memory_contexts(<pid>);
```

或在同一 session 的合适时机查询：

```sql
SELECT name, type, total_bytes, used_bytes
FROM pg_backend_memory_contexts
WHERE type IN ('Bump', 'Generation', 'Slab')
ORDER BY total_bytes DESC;
```

观察点：

```text
是否看到 HashAgg hashed tuples？
type 是否是 Bump？
query 结束后 context 是否消失或 reset？
```

源码回扣：

```text
src/backend/executor/nodeAgg.c
  hash_tuplescxt = BumpContextCreate(...)
```

### 实验四：观察 `pg_log_backend_memory_contexts()` 的日志输出

目标：理解 SQL view 和日志输出的差异。

步骤：

```sql
SELECT pg_log_backend_memory_contexts(pg_backend_pid());
```

到 server log 中查：

```text
logging memory contexts of PID ...
level: ...
Grand total: ...
```

观察点：

```text
日志是否每个 context 一条？
是否出现 Grand total？
是否有 "N more child contexts containing ..."？
```

源码回扣：

```text
src/backend/utils/mmgr/mcxt.c
  ProcessLogMemoryContextInterrupt()
  MemoryContextStatsDetail(TopMemoryContext, 100, 100, false)
```

### 实验五：用 gdb 判断增长来自哪条路径

目标：把 SQL 观测回到源码调用栈。

步骤：

1. 找目标 backend PID：

```sql
SELECT pg_backend_pid();
```

2. gdb attach：

```bash
gdb -p <pid>
```

3. 打断点：

```gdb
(gdb) b BumpAlloc
(gdb) b GenerationAlloc
(gdb) b SlabAlloc
(gdb) b AllocSetAlloc
```

4. 运行 workload 后看 backtrace：

```gdb
(gdb) bt
```

观察点：

```text
分配来自 executor、logical decoding、tuplestore 还是 cache？
CurrentMemoryContext 指向哪里？
context name/type 是否和 pg_backend_memory_contexts 对得上？
```

### 实验六：设计一个 allocator 选择练习

给出四种对象集合，让学员选择 allocator：

```text
A. 大量固定大小 struct，频繁创建释放，生命周期混杂。
B. variable-length tuple data，按队列顺序进入和离开。
C. hash table entries，构建后只查找，rescan 时整体丢弃。
D. plan/cache 数据，大小混杂，可能单独 pfree 或 repalloc。
```

参考答案：

```text
A -> Slab
B -> Generation
C -> Bump
D -> AllocSet
```

讨论关键不是记答案，而是说明：

```text
是否固定大小
是否需要 pfree/realloc
释放顺序是否同代
生命周期边界是否整体 reset/delete
pointer 是否暴露给通用代码
```

## 13. 讨论题

1. 为什么 `AllocSet` 的 `free_bytes` 增大不一定说明内存已经能还给 OS？

2. 如果一个 context 的 `total_bytes` 很大、`free_bytes` 也很大，你如何判断它是正常 retention 还是问题？

3. `Generation` 为什么适合 FIFO 或同生命周期组？如果释放顺序高度随机，会出现什么诊断现象？

4. `Slab` 为什么要优先从“最满”的 block 分配，而不是从最空的 block 分配？

5. 为什么 `Bump` 不能作为通用省内存替代品？它牺牲了哪些 API 能力？

6. `pg_backend_memory_contexts` 的 `path` 为什么要用 CTE 固定一次结果后再做子树聚合？

7. 看到 HashAgg 内存高时，为什么不能只说“HashAgg leak”？你会把它拆成哪些 context 和 allocator 来看？

8. OOM 日志里 `MemoryContextStats()` 的 Grand total 很高，但 OS RSS 更高或更低，可能有哪些原因？

9. 如果你要给一个 extension 设计 backend-local cache，什么情况下应该用 `CacheMemoryContext` 子 context，什么情况下应该挂在事务或 Portal 生命周期下？

10. 你如何用 gdb 证明某个 context 没有在预期边界 reset？

## 14. 本节小结

本节从一个诊断问题出发：

```text
同样看到 MemoryContext 变大，如何判断它是 leak、retention 还是正常峰值？
```

关键答案不是某个单独指标，而是三步：

```text
先看生命周期:
  这个 context 应该什么时候 reset/delete？

再看 allocator:
  AllocSet / Generation / Slab / Bump 分别怎样处理 block、chunk、pfree 和 stats？

最后看时间线:
  total/free/used 是否随 workload 达到平台，是否跨过应有 cleanup 边界继续增长？
```

四类 allocator 的核心判断：

```text
AllocSet:
  通用、mixed-size、支持 pfree/realloc，小块 freelist 会产生 retention。

Generation:
  FIFO 或同代释放，block 级回收更合适，但 used/free 统计要按 allocator 语义解释。

Slab:
  大量同尺寸对象，减少碎片和内部浪费，free_chunks 对对象池状态有诊断价值。

Bump:
  只追加、整体 reset/delete，省 header 和 CPU，但不支持普通 chunk-level 操作。
```

本节可迁移的系统规律：

```text
统一生命周期抽象不等于统一分配策略；
诊断内存问题时，数值只有放回 allocator 策略和 cleanup 边界中才有语义。
```

下一组主题进入 `ResourceOwner 与 ERROR-safe cleanup`。MemoryContext 能解释 backend-local palloc 内存的 ownership 和 allocator 成本，但 lock、buffer pin、snapshot、临时文件、DSM handle 等资源还需要另一套 ownership 机制，这正是 ResourceOwner 要解决的问题。
