# PostgreSQL IndexAM build 与 insert 路径

## 课程定位

前面几节已经讲过 B-tree page、search、insert、unique check、split、dedup 和 bottom-up deletion。
本节把视角从 nbtree 内部拉回到通用 IndexAM 边界。
前置知识：
- 已理解 heap tuple、HOT chain、heap TID 和索引 entry 的关系。
- 已理解 B-tree leaf page、high key、rightlink、page split 和 WAL-before-data。
- 已理解 MVCC snapshot、`SnapshotDirty`、事务等待和 speculative insertion 的基本语义。
- 已理解 relcache、`pg_class`、`pg_index` 至少是 catalog state，而不是普通内存字段。
本节唯一主问题：
PostgreSQL 如何让一个统一的 IndexAM 抽象同时承接 `CREATE INDEX` 的批量构建和 DML 的逐条索引维护，而不把 MVCC、HOT、唯一性、WAL、资源生命周期全部泄漏给上层调用者？
本节围绕的核心矛盾：
上层希望 index AM 是一个通用接口。
executor 只想给出 heap tuple 的索引键和 TID。
DDL 只想创建一个新 index relation 并调用 AM 把它填满。
但真实实现里，批量 build 和逐条 insert 的约束完全不同。
build path 追求吞吐、顺序写、排序加载、并行 heap scan 和尽量少的 buffer churn。
insert path 追求低延迟、页级并发、事务等待、唯一性原子检查和每条 tuple 的约束反馈。
同一个 `IndexInfo` 和同一个 `IndexAmRoutine` 必须覆盖这两类入口。
PostgreSQL 的折中是：
core 只定义调用合同、catalog 状态、snapshot 阶段和资源边界。
AM 自己实现物理结构、bulk load、retail insert、WAL record 和局部并发。
heap/table AM 负责告诉 build path 哪些 heap tuple 应该被索引，以及 HOT root TID 应该是什么。
executor 负责在 DML path 中枚举所有 ready index、计算 index datums、传递 unique check 模式和 update hint。
读完本节，你应该能独立判断：
- `ambuild` 和 `aminsert` 为什么是两个不同回调，而不是一个循环调用另一个。
- `index_build()` 为什么调用 `indexRelation->rd_indam->ambuild()`，而不是对 heap scan 结果逐条调用 `index_insert()`。
- `btbuild()` 为什么先 spool/sort，再用 bulk smgr 顺序加载 B-tree page。
- `ExecInsertIndexTuples()` 为什么在 heap tuple 已经插入之后才维护 index。
- `IndexInfo` 里哪些字段是 catalog 语义，哪些字段是 executor/build 的临时状态。
- `ii_ReadyForInserts`、`indisready`、`indisvalid` 的边界为什么不能混为一谈。
- 普通 build、concurrent build、validate pass 分别用什么 snapshot 语义。
- B-tree retail insert 如何把 `UNIQUE_CHECK_*` 模式变成等待、重查或返回待 recheck。
- 为什么 unique build 要把 dead tuples 放进第二个 spool。
- 为什么 concurrent build 的 validate 阶段还要调用 `index_insert()`。
- 哪些资源由 MemoryContext 清理，哪些由 relation close、tuplesort end、parallel context、bulk smgr finish 或 ERROR cleanup 收尾。
- `pg_stat_progress_create_index`、wait event、`pg_stat_wal`、`EXPLAIN` 和 gdb 各自能看到什么，看不到什么。

源码基线：本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`。行号来自 `git -C /home/highgo/postgres show bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8:<file> | nl -ba`；重点源码和辅助定位统一放在第 3 节的阅读顺序里。

## 1. 本节在总主线中的位置

前面 B-tree 课程讲的是一个具体 AM 如何维护自己的 page-level 不变量。
本节讲 core 如何把这类 AM 接入 DDL 和 DML。
也就是：

```text
DDL: CREATE INDEX
  -> DefineIndex()
  -> index_create()
  -> index_build()
  -> indexRelation->rd_indam->ambuild()
  -> btbuild()
```

以及：

```text
DML: INSERT/UPDATE/COPY/logical apply
  -> ExecInsertIndexTuples()
  -> index_insert()
  -> indexRelation->rd_indam->aminsert()
  -> btinsert()
  -> _bt_doinsert()
```

这两条路都叫“往索引里放 entry”。
但它们不是同一个问题。
build path 处理的是一个新建或重建的 index relation。
它面对的是全表 heap scan、历史 tuple、HOT chain、排序、并行、bulk write 和 catalog 状态推进。
insert path 处理的是当前语句刚写入或更新的一个 heap tuple。
它面对的是 per-tuple expression/predicate、unique/exclusion 约束、事务等待、SSI 冲突和 page split。
IndexAM 抽象的价值不在于隐藏所有差异。
它的价值在于把差异放在可控边界内。
core 不知道 B-tree 怎么 split page。
core 也不要求 GIN、GiST、BRIN 以 B-tree 的方式 bulk load。
但 core 必须统一管理这些事实：
index relation 已经有 catalog entry。
heap relation 处于某种锁和 snapshot 阶段。
index 是否 ready for inserts。
index 是否 valid for planner。
每个 heap tuple 应该形成哪些 index datums。
unique/exclusion 失败时谁负责抛错或返回待 recheck。
资源在 command、transaction、parallel worker 和 ERROR 中如何释放。
所以本节不是“B-tree build 源码导览”。
本节的主线是：
同一个 IndexAM API 如何把两条不同生命周期的写入路径接到同一套 correctness 语义上。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：
IndexAM 把“index entry 的物理组织”留给 AM，把“什么时候需要维护 index、用什么 snapshot 扫 heap、何时允许 planner 使用、何时对 DML ready、谁负责 constraint verdict”留给 core；build path 通过 `ambuild` 批量装载一个关系，insert path 通过 `aminsert` 对单个 heap TID 做原子维护。
这个模型有四层。
第一层是 catalog 层。
`pg_class` 和 `pg_index` 记录 index relation、AM OID、opclass、predicate、expression、`indisready`、`indisvalid` 等持久状态。
第二层是 relcache/API 层。
`IndexAmRoutine` 把 AM 的能力和 callback 暴露给 core。
`Relation->rd_indam` 是运行期找到 `ambuild`、`aminsert`、`ambulkdelete` 等函数的入口。
第三层是 command/executor 层。
`DefineIndex()` 和 `index_create()` 推进 DDL 状态。
`ExecOpenIndices()` 和 `ExecInsertIndexTuples()` 推进 DML 状态。
第四层是 AM 内部层。
B-tree 的 `btbuild()` 使用 tuplesort 和 bulk write。
B-tree 的 `btinsert()` 使用 search、unique check、right move、page split 和 WAL。
这四层不能合并。
如果 core 直接理解每种 AM 的页面结构，扩展和维护会失控。
如果 AM 自己决定 catalog 可见性和 concurrent build 阶段，MVCC 与 planner 语义会失控。
如果 build 走逐条 insert，性能和 WAL/IO 放大不可接受。
如果 insert 走 bulk build 的排序思路，事务原子性和 per-tuple 约束反馈会消失。
本节的核心矛盾 就是：
一个足够稳定的通用 API，必须容纳两个完全不同的 runtime path。
它既不能把接口做成最低公分母，牺牲 AM 的性能。
也不能把 AM 的内部状态上泄到 executor 和 DDL。
PostgreSQL 的答案是 callback 合同加状态分层。
合同稳定。
状态分层。
路径分离。
正确性在边界上组合。

## 3. 核心文件分工与阅读顺序

推荐阅读顺序如下。
不要按文件名排序。
先看 API，再看 DDL，再看 DML，最后看 B-tree 两条 AM 回调。

| 顺序 | 文件 | 读什么 | 目标 |
| --- | --- | --- | --- |
| 1 | `src/include/access/amapi.h` | `ambuild_function`、`aminsert_function`、`IndexAmRoutine` | 先确认 core 和 AM 的合同 |
| 2 | `src/include/nodes/execnodes.h` | `IndexInfo` | 明确 transient state 和 catalog-derived state |
| 3 | `src/backend/commands/indexcmds.c` | `DefineIndex()`、concurrent phases、`WaitForOlderSnapshots()` | 看 DDL 如何决定锁、snapshot 和 catalog 阶段 |
| 4 | `src/backend/catalog/index.c` | `index_create()`、`BuildIndexInfo()`、`index_build()`、`validate_index()` | 看 catalog entry、build 调用和 validate pass |
| 5 | `src/backend/access/index/indexam.c` | `index_insert()`、`index_insert_cleanup()` | 看通用 insert wrapper 的边界 |
| 6 | `src/backend/executor/execIndexing.c` | `ExecOpenIndices()`、`ExecInsertIndexTuples()`、`ExecCheckIndexConstraints()` | 看 executor 如何枚举 index 并分派 unique 模式 |
| 7 | `src/backend/access/nbtree/nbtree.c` | `bthandler()`、`btinsert()` | 看 B-tree 如何注册回调 |
| 8 | `src/backend/access/nbtree/nbtsort.c` | `btbuild()`、`_bt_spools_heapscan()`、`_bt_leafbuild()`、`_bt_load()` | 看 bulk build 的真实实现 |
| 9 | `src/backend/access/nbtree/nbtinsert.c` | `_bt_doinsert()`、`_bt_check_unique()`、`_bt_findinsertloc()`、`_bt_insertonpg()` | 看 retail insert 的真实实现 |
| 10 | `src/backend/access/heap/heapam_handler.c` | `heapam_index_build_range_scan()`、`heapam_index_validate_scan()` | 看 table AM 如何给 build/validate 提供 heap tuple |

`amapi.h:112-128` 定义两个最重要的写入口。
`ambuild(heapRelation, indexRelation, indexInfo)` 构建一个新 index。
`aminsert(indexRelation, values, isnull, heap_tid, heapRelation, checkUnique, indexUnchanged, indexInfo)` 插入一个 index tuple。
`amapi.h:233-326` 的 `IndexAmRoutine` 是能力位和 callback 表。
`ambuild`、`ambuildempty`、`aminsert`、`aminsertcleanup` 都在这个表里。
`nbtree.c:117-176` 的 `bthandler()` 返回静态 `IndexAmRoutine`。
B-tree 把 `.ambuild = btbuild`、`.ambuildempty = btbuildempty`、`.aminsert = btinsert`、`.aminsertcleanup = NULL`。
这说明 B-tree retail insert 没有额外 command-level cleanup 回调。
但别误解为所有 AM 都不需要 cleanup。
`indexam.c:241-249` 会在 AM 提供 `aminsertcleanup` 时调用它。
`indexcmds.c:545-556` 是 DDL 的入口 `DefineIndex()`。
它选择 lock mode、检查 AM 能力、形成 `IndexInfo`，然后调用 `index_create()`。
`index.c:730-750` 是 `index_create()` 参数表。
注意注释明确说 `indexInfo` 是 same info executor uses to insert into the index。
这不是说 build 和 insert 路径相同。
这是说两条路径共享 index definition 的运行期表示。
`index.c:3020-3100` 是 `index_build()`。
它做 security context、progress 初始化、parallel worker 数量规划，然后调用 `indexRelation->rd_indam->ambuild()`。
`execIndexing.c:310-457` 是 DML insert 的主入口。
它对每个 index 做 partial predicate、`FormIndexDatum()`、unique mode 选择，然后调用 `index_insert()`。
`indexam.c:213-234` 的 `index_insert()` 是薄 wrapper。
它做 relation/procedure 检查、SSI conflict check，然后直接调用 `aminsert`。
`nbtsort.c:298-355` 是 B-tree build。
`nbtinsert.c:104-270` 是 B-tree retail insert。
阅读时要牢记：
`btbuild()` 和 `_bt_doinsert()` 不是同一条路径的上下游。
它们是同一个 AM 合同的两个入口。

## 4. 关键数据结构与状态

### `IndexAmRoutine`

`IndexAmRoutine` 是 backend-local 的静态 callback 表。
core 不复制也不释放它。
`amapi.h:230-231` 注释明确说 AM 通常静态分配，core never copies nor frees。
它的能力位影响 DDL 语义。
`amcanunique` 决定 `CREATE UNIQUE INDEX` 是否允许。
`amcaninclude` 决定 `INCLUDE` 是否允许。
`amcanmulticol` 决定多列索引是否允许。
`amcanbuildparallel` 决定 `index_build()` 是否考虑 parallel CREATE INDEX。
`ampredlocks` 决定 `index_insert()` 是否需要 core 代做 predicate lock conflict check。
这些能力位不是 optimization hint。
它们是 core 和 AM 的功能合同。
如果 AM 宣称 `amcanunique`，它的 `aminsert` 就必须在 `UNIQUE_CHECK_YES` 下给出正确 verdict。
如果 AM 宣称 `amcanbuildparallel`，它的 `ambuild` 就必须能处理 parallel worker 的 snapshot、lock 和 resource 边界。

### `IndexInfo`

`IndexInfo` 定义在 `execnodes.h:176-241`。
它是 backend-local 状态。
它不是 catalog tuple。
它通常从 `pg_index` 和 relcache 构造，也会携带 command/executor 期间的临时字段。
关键字段组合如下。
`ii_NumIndexAttrs` 和 `ii_NumIndexKeyAttrs` 区分总列数和 key 列数。
`ii_IndexAttrNumbers[]` 记录 heap 属性号，0 表示 expression。
`ii_Expressions` 和 `ii_Predicate` 来自 index definition。
`ii_ExpressionsState` 和 `ii_PredicateState` 是 executor/build 期间准备出来的执行状态。
`ii_Unique` 和 `ii_NullsNotDistinct` 描述 unique 语义。
`ii_ReadyForInserts` 来自 `pg_index.indisready`。
executor 会跳过 not ready index。
`ii_Concurrent` 告诉 build path 当前是否是 concurrent build。
heap table AM 会据此选择 snapshot 语义。
`ii_BrokenHotChain` 是 build 过程中发现潜在 broken HOT chain 后回传给 `index_build()` 的状态。
`ii_ParallelWorkers` 是 `index_build()` 规划出的 worker 数量。
`ii_AmCache` 是 AM 私有缓存区。
`ii_Context` 记录持有该 `IndexInfo` 的 memory context。
这些字段不能单独解释。
例如 `ii_Unique = true` 不等于每次 insert 都抛错检查。
`ExecInsertIndexTuples()` 还要结合 `indimmediate`、`EIIT_NO_DUPE_ERROR`、deferred constraint 和 speculative insertion 选择 `UNIQUE_CHECK_*`。
再比如 `ii_ReadyForInserts = false` 不等于 index relation 不存在。
concurrent build 第一阶段 catalog entry 已经可见，但 index 还不对 DML ready。

### `IndexUniqueCheck`

`IndexUniqueCheck` 定义在 `genam.h:123-129`。
`UNIQUE_CHECK_NO` 表示不做唯一性检查。
`UNIQUE_CHECK_YES` 表示 insertion time 立即 enforce。
`UNIQUE_CHECK_PARTIAL` 表示只探测可能冲突，不阻塞，不抛错，返回是否已知唯一。
`UNIQUE_CHECK_EXISTING` 表示 deferred recheck 的伪插入，不重复插入 index tuple。
这是 executor 和 AM 之间很重要的责任切分。
executor 决定模式。
AM 执行模式。
如果 AM 在 `UNIQUE_CHECK_PARTIAL` 下阻塞或抛错，就破坏 deferred/speculative 合同。
如果 AM 在 `UNIQUE_CHECK_YES` 下只做 best effort，就破坏 unique constraint。

### `BTBuildState` / `BTSpool` / `BTShared`

B-tree build 的核心状态在 `nbtsort.c`。
`BTSpool` 保存 tuplesort、heap relation、index relation、是否 unique、是否 nulls not distinct。
它是 backend-local。
`BTBuildState` 是 build callback 的工作状态。
它有 primary `spool`，unique index 时可能还有 `spool2`。
`nbtsort.c:217-219` 注释说明 `spool2` 只在 unique index 需要。
dead tuples 被放到 `spool2`，以避免参与唯一性检查。
`BTShared` 是 parallel build 的 DSM shared state。
它保存 heap/index OID、unique/concurrent 标志、condition variable、spinlock、tuple counts、worker done count、broken HOT chain 状态。
`BTLeader` 是 leader backend 的 parallel context 状态。
它持有 `ParallelContext`、shared tuplesort 指针、snapshot、WAL/buffer usage 累计区。
这些状态的生命周期不同。
`BTSpool` 由 `btbuild()` 创建并在 `_bt_spooldestroy()` 释放。
`BTShared` 在 DSM 中，由 parallel context 管理。
`BTLeader` 在 leader backend 中，由 `_bt_end_parallel()` 关闭 parallel context 后结束。

### `BTInsertState`

`BTInsertStateData` 定义在 nbtree 头文件中，本节只关心语义。
它保存待插入 tuple、MAXALIGN 后的 size、scan key、当前 buffer、binary search bounds、posting list offset。
retail insert 的状态随时间推进。
先由 `_bt_doinsert()` 初始化。
`_bt_search_insert()` 找到并锁住 leaf 后填入 `buf`。
`_bt_check_unique()` 可能在该 page 上保存 bounds。
`_bt_findinsertloc()` 复用 bounds，必要时向右移动并改写 `buf`。
`_bt_insertonpg()` 最终写页、split、WAL、释放 buffer。
这个状态是 backend-local。
但它持有的 `Buffer` 是 shared buffer 的 pin/lock。
因此 cleanup 不能只靠 MemoryContext。
错误路径必须确保 pin/lock 经由 resource owner 和 buffer manager 收回。

### catalog flags

`pg_index.indisready` 和 `pg_index.indisvalid` 是 build 路径的关键持久状态。
`indisready` 主要回答：
新 DML 是否应该维护这个 index。
`indisvalid` 主要回答：
planner 是否可以把这个 index 当作查询可用路径。
普通 `CREATE INDEX` 在单事务内建好后，catalog 对外可见时通常已经 ready/valid。
`CREATE INDEX CONCURRENTLY` 必须分阶段推进。
第一阶段 catalog entry 可见，但 not ready and not valid。
第二阶段 build 后设置 ready。
第三阶段 validate 后等待旧 snapshot，再设置 valid。
这两个字段不是锁。
它们是 relcache/catalog 语义。
别把 `indisready` 理解成 index 内部页面已经完整。
也别把 `indisvalid` 理解成 DML 是否维护 index。

### tuple identity

build 和 insert 最终都要给 AM 一个 heap TID。
对 B-tree 来说，heap TID 还是 heapkeyspace 下的隐含 tiebreaker。
但 build path 遇到 HOT tuple 时可能用 root TID。
`heapam_index_build_range_scan()` 会把 heap-only tuple 的 TID 转成 HOT chain root TID。
`heapam_index_validate_scan()` 也用 root TID 和当前 index 中的 TID list 做 merge。
这就是为什么 `heap_tid` 不是一个简单物理字段。
它是“AM entry 指向 heap 可见性链路的身份”。

## 5. 主流程源码 walkthrough

### 5.1 普通 `CREATE INDEX` 的 build path

入口是 `DefineIndex()`。
`indexcmds.c:619-622` 先决定是否真正 concurrent。
临时表即使用户写了 `CONCURRENTLY`，也会被强制走非 concurrent。
`indexcmds.c:685-686` 选择 table lock。
普通 build 使用 `ShareLock`。
concurrent build 使用 `ShareUpdateExclusiveLock`。
这个 lock 选择影响 table AM heap scan 的并发假设。
然后 `DefineIndex()` 检查 relation kind。
普通 heap relation、materialized view、partitioned table 可以创建 index。
其他 relkind 报错。
接着查 `pg_am`，拿到 AM handler。
`indexcmds.c:868` 调用 `GetIndexAmRoutine()`。
这里开始 AM 能力位参与 DDL。
`indexcmds.c:873-877` 检查 unique support。
`indexcmds.c:878-882` 检查 included columns。
`indexcmds.c:883-887` 检查 multicolumn。
`indexcmds.c:888-892` 检查 exclusion constraint 是否有 `amgettuple`。
这些检查发生在真正创建 catalog entry 之前。
然后 `DefineIndex()` 构造 `IndexInfo`。
`indexcmds.c:924-934` 调用 `makeIndexInfo()`。
普通 build 下 `ii_ReadyForInserts = true`。
concurrent build 下 `ii_ReadyForInserts = false` 且 `ii_Concurrent = true`。
再往后，`ComputeIndexAttrs()` 填充列、opclass、collation、expression 等定义。
partitioned table 的 unique/exclusion 还有额外限制。
`indexcmds.c:960-965` 的注释说明，如果没有 global indexes，分区表 unique 必须包含 partition key。
否则相同 key 可以落到不同 partition 而无法被局部 index 约束。
准备好参数后，`DefineIndex()` 调用 `index_create()`。
`indexcmds.c:1221-1234` 设置 flags。
普通 build 不设置 `INDEX_CREATE_SKIP_BUILD`。
concurrent、skip_build、partitioned 都会设置 skip build。
`index.c:730-750` 是 `index_create()`。
它创建的是 cataloged index relation。
`index.c:981-995` 调用 `heap_create()` 创建 index relation 的 relcache entry 和物理存储。
`index.c:1006` 对新 index relation 拿 `AccessExclusiveLock`。
即使其他事务还看不到它，这也避免类似 CLUSTER 场景中的 lock manager 死锁风险警告。
`index.c:1021-1024` 插入 `pg_class` tuple。
`index.c:1050-1056` 插入 `pg_index` tuple。
这里会设置 ready/valid 状态。
普通 build 传入 `!concurrent && !invalid` 作为 valid，`!concurrent` 作为 ready。
`index.c:1062` 对 heap relation 发 relcache invalidation。
原因是 heap 的 index list 语义已经变化。
后面记录依赖、constraint、opclass、collation、expression、predicate 等。
`index.c:1233` 做 `CommandCounterIncrement()`。
这一步让本事务能看到刚插入的 catalog rows。
如果不是 bootstrap、不是 skip build、不是 partitioned index，`index.c:1283-1284` 调用 `index_build()`。
`index_build()` 开始前，index relation 的 catalog entry 已经存在，物理文件已经创建但为空。
`index.c:3004-3010` 注释直接说明这个前提。
`index.c:3020-3026` 是 `index_build()` 签名。
它不关闭 heap/index relation。
调用者负责关闭。
`index.c:3047-3051` 在 AM 支持 parallel build 时规划 worker 数。
`index.c:3065-3074` 切换到 table owner，并进入 security restricted operation。
这很重要，因为 index expression、predicate、support function 可能运行用户代码。
`index.c:3076-3094` 初始化 progress。
然后核心一步：
`index.c:3099-3100` 调用 `indexRelation->rd_indam->ambuild(heapRelation, indexRelation, indexInfo)`。
对于 B-tree，这就是 `btbuild()`。
`btbuild()` 在 `nbtsort.c:298-355`。
它先初始化 `BTBuildState`。
`nbtsort.c:323-325` 检查 index relation 是否已经有 block。
如果不为空，报错。
这说明 `btbuild()` 期待“新空 index relation”。
它不是 generic repair routine。
然后 `nbtsort.c:327` 调用 `_bt_spools_heapscan()`。
这个函数准备 tuplesort，并通过 table AM 扫 heap。
`nbtsort.c:433-437` 创建 primary tuplesort。
它用 `maintenance_work_mem`，不是 `work_mem`。
unique index 时，`nbtsort.c:444-476` 创建 secondary spool。
secondary spool 用于 dead tuples，且不做 unique sort。
随后 `nbtsort.c:480-486` 扫 heap。
没有 parallel leader 时调用 `table_index_build_scan()`。
有 parallel build 时调用 `_bt_parallel_heapscan()`，等待 worker 完成。
`table_index_build_scan()` 是 table AM wrapper。
`tableam.h:1818-1843` 注释说明它扫描 table，找出应当进入 index 的 tuple，并通过 callback 交给 AM。
heap 的实现是 `heapam_index_build_range_scan()`。
非 concurrent 普通 build 使用 `SnapshotAny`，并自行调用 `HeapTupleSatisfiesVacuum()`。
原因是普通 build 必须索引 `RECENTLY_DEAD` tuple，以保护已经存在的旧 snapshot。
`heapam_handler.c:1205-1210` 直接说明这一点。
concurrent build 或 bootstrap 使用 MVCC snapshot。
`heapam_handler.c:1363-1582` 是普通 build 的 tuple liveness 分类。
`HEAPTUPLE_LIVE` 会 index 并计入 reltuples。
`HEAPTUPLE_RECENTLY_DEAD` 可能也要 index，但不参与 unique checking。
HOT-updated recently-dead tuple 会让 `ii_BrokenHotChain = true`。
`HEAPTUPLE_INSERT_IN_PROGRESS` 和 `HEAPTUPLE_DELETE_IN_PROGRESS` 在 unique check 场景下可能需要等待并 recheck。
这说明 build path 的 uniqueness 不只是排序时发现 duplicate。
它还依赖 heap liveness 分类和事务等待。
每个需要进入 index 的 tuple 会调用 `_bt_build_callback()`。
`nbtsort.c:583-606` 中，如果 tuple alive 或没有 spool2，放入 primary spool。
否则放入 `spool2`，并标记 `havedead`。
然后 `btbuild()` 调用 `_bt_leafbuild()`。
`nbtsort.c:554-563` 先执行 one or two sorts。
`nbtsort.c:565-576` 初始化 write state，然后调用 `_bt_load()`。
`_bt_load()` 是 bulk load。
`nbtsort.c:1153` 通过 `smgr_bulk_start_rel()` 开始 bulk write。
如果有 `spool2`，`nbtsort.c:1158-1265` 把 primary spool 和 dead tuple spool merge。
merge 时相同 key 再按 heap TID 排序。
`nbtsort.c:1230-1233` 注释说明 heap TID 是隐含最后 key，保证物理唯一。
如果没有 spool2 且允许 dedup，`nbtsort.c:1267-1358` 会在 build 时形成 posting list。
如果不 merge 也不 dedup，`nbtsort.c:1360-1374` 逐个从 tuplesort 输出加载。
加载 page 的核心函数是 `_bt_buildadd()`。
它按 fillfactor 把 leaf page 填到目标空闲空间。
`nbtsort.c:17-24` 注释说明 leaf 默认按用户 fillfactor，upper page 固定 70%。
这是 build path 和 retail insert 的重要差异。
build path 不希望新 index 刚建好后任何插入都导致级联 split。
`_bt_buildadd()` 在 page 满时结束当前 page，生成 high key、rightlink、parent downlink。
`nbtsort.c:949-969` 把旧页 low key 加到 parent，并把 old page high key 复制成 new page low key。
`nbtsort.c:972-981` 设置 sibling links。
所有层级结束后，`_bt_uppershutdown()` 收尾。
`nbtsort.c:1093-1098` 把最上层 page 标成 root。
`nbtsort.c:1117-1118` 处理 rightmost page 并写出。
`nbtsort.c:1122-1131` 最后构造 metapage，让它指向 root。
注释说这通过写入有效 magic number 让 index 进入 valid state。
最后 `smgr_bulk_finish()` 结束 bulk write。
`btbuild()` 返回 `IndexBuildResult`。
`index_build()` 收到 stats 后处理 unlogged index init fork。
`index.c:3110-3115` 对 unlogged index 创建 init fork，并调用 `ambuildempty()`。
B-tree 的 `btbuildempty()` 在 `nbtree.c:182-197`，只给 init fork 写 metapage。
如果 build 过程中发现 broken HOT chain，普通非 concurrent build 会设置 `pg_index.indcheckxmin`。
`index.c:3146-3167` 是这个状态更新。
它表示旧 snapshot 在越过 event horizon 前不能安全使用该 index。
然后 `index.c:3176-3182` 更新 heap 和 index 的 `pg_class` stats。
`index.c:3185` 做 command counter increment。
如果是 exclusion constraint，`index.c:3188-3194` 还会第二次扫描 heap 做 constraint verification。
最后恢复 GUC、userid 和 security context。
普通 build 到这里完成。
catalog commit 后，其他 backend 能看到 ready and valid index。

### 5.2 `CREATE INDEX CONCURRENTLY` 的 build/validate path

concurrent build 的第一阶段仍然从 `DefineIndex()` 开始。
但 flags 会设置 `INDEX_CREATE_SKIP_BUILD` 和 `INDEX_CREATE_CONCURRENT`。
`index_create()` 只创建 catalog entry 和空 relation，不立即 `index_build()`。
`indexcmds.c:1631-1641` 说明必须先 commit，让 catalog entry 对其他事务可见。
新 index 会被标成 not `indisready` and not `indisvalid`。
这样其他 backend 能看见 index definition，但不会对它插入，也不会用它执行查询。
在 commit 前，`indexcmds.c:1652` 获取 session-level lock。
它防止 table/index 在整个 concurrent build 期间被 drop。
第二阶段开始后，`indexcmds.c:1695` 等待旧写事务。
这一步确保之后打开 table 的 writer 会看到新的 index list。
但此时 index 仍 not ready for inserts。
这段时间 index definition 只用于阻止不兼容 HOT update。
`indexcmds.c:1715-1719` 设置 snapshot，然后调用 `index_concurrently_build()`。
`index.c:1502-1543` 重新打开 heap/index，重新 `BuildIndexInfo()`。
因为前一个事务已经 commit，原来的 `IndexInfo` 和 relcache state 都不能跨事务复用。
`index.c:1538-1540` 断言 not ready for inserts，并设置 `ii_Concurrent = true`。
然后调用 `index_build()`。
这里的 `btbuild()` 仍然是 bulk build。
但 heap scan 使用 MVCC snapshot，只索引该 snapshot 可见的 tuple。
build 完成后，这个事务 commit，ready 状态对其他事务可见。
第三阶段，`indexcmds.c:1740-1742` 再等一次旧事务。
现在新打开 table 的 writer 已经会维护该 index。
然后 `indexcmds.c:1759-1765` 取得 reference snapshot，调用 `validate_index()`。
`validate_index()` 在 `index.c:3371-3498`。
它先用 `index_bulk_delete()` 扫描当前 index，把已有 TID 收进 tuplesort。
`index.c:3442-3445` 用 int8 编码 TID 排序。
`index.c:3449-3450` 调用 `index_bulk_delete()`，但 callback 永远返回 false。
`validate_index_callback()` 在 `index.c:3504-3511`，只收集 TID，不删除。
然后 `index.c:3473-3477` 调用 `table_index_validate_scan()`。
heap 的实现是 `heapam_index_validate_scan()`。
它从 heap 按 block 顺序扫描 snapshot 可见 tuple。
同时和排序后的 index TID list 做 merge。
发现 heap 中存在但 index 中缺失的 root TID，就调用 `index_insert()` 补 entry。
`heapam_handler.c:1921-1930` 的注释强调，即使当前 tuple 已经 committed dead，也不能简单跳过 uniqueness check。
因为 HOT 下这个插入是对整个 HOT chain 的 proxy uniqueness check。
`heapam_handler.c:1933-1941` 调用 `index_insert()`。
unique index 用 `UNIQUE_CHECK_YES`。
这说明 concurrent validate 是 build path 中临时回到 retail insert API 的地方。
但它不是初始 bulk build。
它只补缺失 TID，并依赖 AM 的 normal insert/unique correctness。
validate 后，`indexcmds.c:1774-1777` 保存 reference snapshot 的 xmin，然后 unregister snapshot。
必须先释放自己的 snapshot，再等其他旧 snapshot。
否则 concurrent index builds 之间可能互相等待。
`indexcmds.c:1803-1806` 调用 `WaitForOlderSnapshots(limitXmin, true)`。
`indexcmds.c:400-436` 说明它等待可能持有更旧 snapshot 的事务。
等完后，`indexcmds.c:1817` 设置 `indisvalid`。
`indexcmds.c:1829` 对 heap relation 发 relcache invalidation，以便 cached plan 重新考虑新 index。
最后释放 session-level lock。
concurrent build 的重要结论是：
bulk build 只构建 snapshot 可见集合。
validate 补齐 reference snapshot 可见但缺失的 tuple。
最终 valid 前还要等待可能看到更旧可见性的事务消失。
三个动作缺一不可。

### 5.3 DML `INSERT` 的 insert path

DML path 从 heap tuple 已经插入开始。
`execIndexing.c:7-9` 注释明确说 `ExecInsertIndexTuples()` 在 heap insert 之后调用。
入口函数是 `ExecInsertIndexTuples()`。
`execIndexing.c:310-317` 是签名。
它拿到 `ResultRelInfo`、`EState`、flags、slot、arbiterIndexes 和 speculative conflict 输出指针。
tuple TID 来自 `slot->tts_tid`。
`execIndexing.c:329` 断言这个 TID 有效。
先从 `ResultRelInfo` 取 index relations 和 `IndexInfo` 数组。
这些数组由 `ExecOpenIndices()` 准备。
`execIndexing.c:161-231` 中，executor 读取 heap relation 的 index OID list，对每个 index 调用 `index_open(RowExclusiveLock)`，再 `BuildIndexInfo()`。
如果 speculative insertion 且是 unique 非 exclusion index，还调用 `BuildSpeculativeIndexInfo()`。
真正插入时，`ExecInsertIndexTuples()` 遍历每个 index。
`execIndexing.c:368-370` 如果 `ii_ReadyForInserts` 为 false，就跳过。
这就是 concurrent build 的 `indisready` 对 DML 的作用。
`execIndexing.c:376-377` 处理 summarizing-only update。
这服务某些 table AM 更新优化。
接着处理 partial index predicate。
`execIndexing.c:380-397` 准备并执行 predicate。
predicate 不满足，就不为这个 heap tuple 维护该 partial index。
`execIndexing.c:404-408` 调用 `FormIndexDatum()`。
`FormIndexDatum()` 在 `index.c:2748-2802`。
普通列从 slot 取值。
expression column 通过 executor expression state 求值。
它只填 `values[]` 和 `isnull[]`。
不会形成 `IndexTuple`。
原因写在 `index.c:2742-2744`：
index AM 可能想在存储前改写数据。
然后选择 unique check 模式。
`execIndexing.c:429-436` 是关键分派。
非 unique 用 `UNIQUE_CHECK_NO`。
`EIIT_NO_DUPE_ERROR` 适用的 arbiter index 用 `UNIQUE_CHECK_PARTIAL`。
immediate unique 用 `UNIQUE_CHECK_YES`。
deferrable unique 用 `UNIQUE_CHECK_PARTIAL`。
这一步体现 executor 和 AM 的边界。
executor 知道约束时机。
AM 知道如何在物理结构中原子检查。
如果是 UPDATE，executor 还可能传 `indexUnchanged` hint。
`execIndexing.c:443-447` 调用 `index_unchanged_by_update()`。
这个 hint 不改变 correctness。
它只帮助 B-tree 在页空间紧张时优先尝试 bottom-up deletion。
然后 `execIndexing.c:449-457` 调用 `index_insert()`。
`indexam.c:213-234` 做最小包装。
`RELATION_CHECKS` 会阻止访问正在被 reindex processing 的 index。
如果 AM 不自己处理 predicate locks，core 调用 `CheckForSerializableConflictIn()`。
然后直接调用 `rd_indam->aminsert()`。
B-tree 的 `aminsert` 是 `btinsert()`。
`nbtree.c:205-224` 先 `index_form_tuple()` 形成 `IndexTuple`，把 heap TID 填入 `t_tid`，再调用 `_bt_doinsert()`。
`_bt_doinsert()` 是 B-tree retail insert 主流程。
`nbtinsert.c:115-117` 形成 insertion scan key。
如果需要 uniqueness check 且 key 中没有 NULL，`nbtinsert.c:122-124` 暂时清空 `scantid`。
这是为了先找到“第一个可能有该用户 key 的 page”。
唯一性建立后才恢复 heap TID tiebreaker。
如果 unique key 中有 NULL 且不是 `NULLS NOT DISTINCT` 语义，`nbtinsert.c:127-140` 跳过 unique check。
然后初始化 `BTInsertState`。
`nbtinsert.c:163-170` 从 root 搜索并锁住 leaf page。
如果需要 unique check，`nbtinsert.c:213-214` 调用 `_bt_check_unique()`。
`_bt_check_unique()` 在持有目标 leaf write lock 的情况下扫描同 key entries。
它用 `SnapshotDirty` 回 heap 判断同 key tuple 的真实事务状态。
如果发现可能冲突且对方事务仍在进行，返回 `xwait`。
`_bt_doinsert()` 在 `nbtinsert.c:216-235` 释放 page buffer，等待事务或 speculative token，然后从 root 重新搜索。
这避免在等待事务结束时长期持有 B-tree page lock。
也避免等待后沿用过期 page/offset。
如果 unique check 通过，`nbtinsert.c:238-240` 恢复 `scantid`。
然后 `nbtinsert.c:261-265` 调用 `_bt_findinsertloc()` 和 `_bt_insertonpg()`。
`_bt_findinsertloc()` 找到最终插入页和 offset。
对 heapkeyspace index，唯一性检查时先省略了 heap TID，所以它可能还要向右移动到真正属于该 heap TID 的 sibling page。
如果 page 空间不足，`nbtinsert.c:913-920` 先尝试 simple deletion、bottom-up deletion 或 dedup。
最终 `_bt_insertonpg()` 做物理写入。
`nbtinsert.c:1218-1233` 如果 page 放不下，会调用 `_bt_split()`。
split 后递归插入 parent downlink。
如果不用 split，`nbtinsert.c:1297-1421` 在 critical section 中改 page、写 WAL、设置 LSN。
`nbtinsert.c:1332-1410` 按 leaf insert、posting list split、upper insert、meta update 选择 WAL record。
最后释放 buffer。
DML insert path 的结论是：
executor 只循环 index relations。
generic wrapper 只做少量安全检查和 SSI 入口。
B-tree AM 自己负责物理定位、unique wait/retry、page cleanup、split、WAL 和 buffer release。

### 5.4 `INSERT ... ON CONFLICT` 的特殊 insert path

`ON CONFLICT` 仍然走 `ExecInsertIndexTuples()` 和 `index_insert()`。
但它多了 speculative lifecycle。
`nodeModifyTable.c:1143-1154` 先做 non-conclusive pre-check。
`ExecCheckIndexConstraints()` 注释说它不加锁，不能保证随后不会冲突。
它只是避免大量取消 speculative tuple。
如果 pre-check 发现 committed conflict，executor 直接执行 DO UPDATE、DO SELECT 或 DO NOTHING。
如果没有冲突，`nodeModifyTable.c:1232` 获取 speculative insertion lock。
`nodeModifyTable.c:1235-1239` speculative 插入 heap tuple。
然后 `nodeModifyTable.c:1242-1245` 用 `EIIT_NO_DUPE_ERROR` 调用 `ExecInsertIndexTuples()`。
这会让 arbiter unique index 使用 `UNIQUE_CHECK_PARTIAL`。
B-tree 在可能冲突时不抛 `unique_violation`，而是返回 false。
executor 得到 `specConflict` 后，`nodeModifyTable.c:1248-1249` complete speculative tuple，决定保留还是 kill。
`nodeModifyTable.c:1258` 释放 speculative insertion lock。
如果有 conflict，`nodeModifyTable.c:1265-1268` 回到 pre-check。
这条路径说明 pre-check 不是 correctness 边界。
正确性边界仍然在 AM 的 atomic insert/unique check 和 speculative token wait/retry。

### 5.5 deferred unique recheck path

deferrable unique 在 initial insert 时用 `UNIQUE_CHECK_PARTIAL`。
AM 插入 index tuple，但只返回是否可能冲突。
executor 把可能冲突的 index OID 放入 recheck list。
真正约束检查由 deferred trigger 做。
`constraint.c:175-177` 调用 `index_insert(..., UNIQUE_CHECK_EXISTING, ...)`。
这不是物理插入。
`nbtinsert.c:89-91` 注释明确说 `UNIQUE_CHECK_EXISTING` 只做 duplicate check，不实际 insert。
`_bt_check_unique()` 在 recheck 时还要求找回自己的 tuple。
如果找不到，`nbtinsert.c:773-779` 报错，提示可能是非 immutable index expression。
这条路径把表达式 index 的稳定性问题也放进了 AM recheck 边界。

## 6. 生命周期 / ownership / cleanup

### DDL build lifecycle

`DefineIndex()` 持有 heap relation lock。
普通 build 使用 transaction-level lock。
concurrent build 还持有 session-level heap lock，跨 transaction 保护对象不被 drop。
`index_create()` 创建 index relation、catalog entries、dependencies 和可能的 constraint。
如果后续 ERROR，transaction abort 会回滚 catalog tuple。
物理文件的删除由 storage manager 和事务性 relation create/drop 机制兜底。
`index_build()` 不拥有传入的 heap/index relation。
`index.c:3016-3018` 注释明确说调用者打开并负责关闭。
`index_build()` 拥有 build 期间的 security context 和 GUC nest level。
正常路径末尾调用 `AtEOXact_GUC(false, save_nestlevel)` 并恢复 userid/security context。
如果 ERROR，PostgreSQL 的 error recovery 会通过事务 abort 和 GUC cleanup 收尾。
B-tree `btbuild()` 拥有 `BTSpool`、tuplesort state、parallel context 和 bulk write state。
正常路径中：
`_bt_spooldestroy()` 调用 `tuplesort_end()`。
`_bt_end_parallel()` 等待 workers、累计 WAL/buffer usage、unregister snapshot、destroy parallel context、exit parallel mode。
`_bt_load()` 调用 `smgr_bulk_finish()`。
这些不是 MemoryContext 可以替代的外部资源。
如果 ERROR 发生在 build 中，tuplesort、parallel context、bulk write、relation locks、buffers 依赖各自的 resource owner 和 error cleanup 机制。
课程要注意：
源码正常路径中看到的 `tuplesort_end()` 不是唯一兜底。
但正常路径必须调用它，避免把短生命周期资源拖到更大 context。

### DML insert lifecycle

`ExecOpenIndices()` 在 result relation 初始化时打开所有 index。
它对每个 index 取得 `RowExclusiveLock` 并构造 `IndexInfo`。
`ExecCloseIndices()` 遍历 index relations。
`execIndexing.c:257` 先调用 `index_insert_cleanup()`。
`execIndexing.c:260` 再 `index_close(RowExclusiveLock)`。
`execIndexing.c:267-270` 注释说不手动释放 `IndexInfo` 和数组，交给 `FreeExecutorState()`。
这说明 `IndexInfo` 是 executor lifecycle 内存。
index relation lock 和 relcache refcount 则由 relation close / transaction lock 管理。
`ExecInsertIndexTuples()` 使用 per-tuple expr context 计算 predicate/expression。
它会复用 `IndexInfo` 中的 prepared expression state。
这使 repeated insert 不必每条 tuple 重新 prepare expression。
AM insert 内部的 buffer pin/lock 不归 MemoryContext 管。
B-tree `_bt_doinsert()` 在每次等待前释放 leaf buffer。
`_bt_insertonpg()` 正常路径释放 target buffer、meta buffer、child buffer。
如果 ERROR 发生在 buffer lock 持有期间，buffer manager 和 resource owner 会在错误清理中释放 pin/lock。
但源码仍要尽量在调用可能访问 catalog 的错误报告前释放 buffer。
`_bt_check_unique()` 在构造 duplicate key error 前释放 nbuf 和 insertstate buf。
`nbtinsert.c:645-655` 注释说明 `BuildIndexValueDescription()` 可能访问 catalog，最坏情况下可能碰同一个 index 并造成死锁。

### `IndexInfo` ownership

`BuildIndexInfo()` 从 open index relation 构造 `IndexInfo`。
`index.c:2435-2489` 用 relcache 的 `rd_index`、expression、predicate、exclusion info 初始化它。
concurrent build 跨 transaction 时不能复用旧 `IndexInfo`。
`index.c:1532-1537` 明确重新 `BuildIndexInfo()`。
parallel worker 中也重新 `BuildIndexInfo()`。
`nbtsort.c:1927-1934` worker 打开 relation 后构建自己的 `IndexInfo`，再进入 `table_index_build_scan()`。
这说明 `IndexInfo` 指针不能跨 backend 或 transaction 当稳定对象传递。
它是当前 backend 当前 memory context 下的运行期视图。

### relcache invalidation

`index_create()` 在创建 `pg_index` 后对 heap relation 发 relcache invalidation。
这样其他 backend 会刷新 index list。
`index_update_stats()` 即使没有实际修改 pg_class，也会发 relcache inval。
`index.c:2818-2824` 解释了这个 side effect。
concurrent build 设置 valid 后也对 heap relation 发 inval。
`indexcmds.c:1822-1829` 说明这是为了让 cached plans 重新考虑新 index。
invalidation 不是 lock。
它不阻塞正在执行的语句。
它只让后续 relcache/planner 语义刷新。

## 7. 正确性机制层次

### API correctness

`IndexAmRoutine` 是第一层正确性。
DDL 根据能力位拒绝不支持的组合。
这避免在 runtime 半路发现 AM 不会处理 unique、include、multicolumn 或 exclusion。
但是能力位不能保证页面级正确性。
它只是合同入口。
真正 correctness 由 AM 实现、table AM scan、catalog 状态和 transaction 机制组合保证。

### catalog visibility correctness

普通 build 在一个事务内创建 catalog rows、build index、更新 stats。
外部 backend 在 commit 前看不到新 index。
所以普通 build 可以用更强 table lock 和 `SnapshotAny` 处理旧 tuple。
concurrent build 则必须把 catalog visibility 拆成多阶段。
not ready means DML 不维护。
ready but not valid means DML 维护，但 planner 不使用。
valid means planner 可以使用。
这个三阶段解决的是：
不能阻塞 writers 太久。
又不能让 planner 使用不完整 index。
还不能让 HOT update 在 index definition 可见前制造不兼容链。

### heap visibility correctness

build path 不直接相信 heap scan 看到的 tuple 都是 live。
非 concurrent build 用 `SnapshotAny`，再用 `HeapTupleSatisfiesVacuum()` 分类。
`RECENTLY_DEAD` 可能仍需索引，因为旧 snapshot 可能还需要它。
`INSERT_IN_PROGRESS` / `DELETE_IN_PROGRESS` 在 unique build 中可能等待。
concurrent build 使用 MVCC snapshot，只索引 snapshot 可见 tuple。
validate pass 再补缺失。
最终 valid 前等待 older snapshots。
这三者一起保证新 index 对将来 planner 可用时不会漏掉对某些 snapshot 仍重要的 tuple。

### HOT correctness

heap-only tuple 不一定用自己的 physical TID 建 index entry。
build path 会把 HOT child 映射到 root TID。
这保持 index entry 指向 HOT chain root 的语义。
如果 build 发现 recently-dead HOT chain 可能对旧 snapshot 不安全，就设置 `ii_BrokenHotChain`。
普通 build 之后可能设置 `indcheckxmin`。
concurrent build 不设置它，而是通过 valid 前等待相关 snapshot 来解决。
这说明 HOT correctness 横跨 heap AM、index build、catalog flag 和 snapshot horizon。

### unique correctness

DML unique correctness 在 AM。
`execIndexing.c:15-20` 明确说 unique AM insert 同时检查冲突，并负责原子性与等待。
B-tree 在 `_bt_check_unique()` 中持有第一个可能含该 key 的 leaf write lock。
这让并发相同 key 的 inserter 串行化。
但它不会在等待事务时持有 page lock。
遇到 in-progress tuple 时释放 buffer、等待、从 root 重查。
这样把短 page lock 和长 transaction wait 分离。
`UNIQUE_CHECK_PARTIAL` 则只返回 potential conflict。
deferred trigger 或 speculative loop 以后再处理。

### WAL / crash correctness

build path 使用 bulk smgr loading。
`nbtsort.c:26-27` 说明它 bypass buffer cache 并高效 WAL-log pages。
retail insert path 在 critical section 中修改 page、写 WAL、设置 page LSN。
`nbtinsert.c:1297-1421` 是 simple insert 的 WAL 边界。
page split 可能留下 incomplete split。
后续 inserter 在插入前会 finish split。
`_bt_finish_split()` 注释说明 crash 或 failure 可能留下 incomplete split，插入例程不会允许直接插到 incompletely split page。
这保证 crash/retry 后结构修改能继续收尾。

### lock/pin correctness

build path 的 table lock 和 snapshot 决定 heap tuple 集合。
retail insert 的 B-tree buffer write lock 决定局部页结构。
resource owner 管 pin/lock cleanup。
heavyweight lock 管 relation-level 和 transaction wait。
speculative insertion lock 管 `ON CONFLICT` 中“等 verdict 而非等整个事务”的边界。
这些机制不能互相替代。
buffer lock 不能表达 SQL unique verdict。
catalog invalidation 不能保护 page memory。
transaction wait 不能保证 B-tree page 不被并发 split。

### SSI correctness

`index_insert()` 在 AM 不处理 predicate locks 时调用 `CheckForSerializableConflictIn()`。
B-tree 宣称 `ampredlocks = true`。
B-tree insert 仍在合适 page 上调用 `CheckForSerializableConflictIn()`。
`nbtinsert.c:247-254` 说明插入 index tuple 时需要检查现有 predicate lock。
unique violation 前也会做一次 conflict-in check，避免 unique error 掩盖 serializable conflict。
这是隔离级别 correctness，不是物理结构 correctness。

## 8. 错误路径 / 异常路径 / fallback

### AM 不支持功能

`DefineIndex()` 在创建 catalog entry 前检查 AM 能力。
不支持 unique、include、multicolumn 或 exclusion 会直接 ERROR。
这是最早失败点。
好处是不会创建半成品 catalog object。
这类错误不进入 `index_create()`，也不会调用 `ambuild`。

### index relation 非空

B-tree `btbuild()` 要求 index relation 为空。
`nbtsort.c:323-325` 如果已有 block 就 ERROR。
这保护 `btbuild()` 的假设：
它会从 metapage 开始顺序构造整棵树。
它不是向已有树增量填充的 API。

### unlogged index init fork

`index_build()` build 完主 fork 后，如果 index 是 unlogged 且 init fork 不存在，会创建 init fork。
然后调用 `ambuildempty()`。
这是一条 persistence fallback。
unlogged relation 崩溃后需要 init fork 作为重置模板。
B-tree 的 `btbuildempty()` 只写一个空 metapage。
这和 normal `btbuild()` 的 full load 不同。

### broken HOT chain

普通 build 遇到潜在 broken HOT chain，不是直接失败。
heap AM 设置 `ii_BrokenHotChain`。
`index_build()` 在非 reindex、非 concurrent 场景设置 `indcheckxmin`。
这个 fallback 的含义是：
index 可以创建，但对太旧 snapshot 不安全。
等事务 horizon 推进后才安全使用。
这是典型的 correctness 延迟策略。

### concurrent build validate

concurrent build 的初始 build 一定可能漏掉并发写入期间的 tuple。
系统不把这当错误。
fallback 是 validate pass。
它先收集 index TID，再扫 heap，发现缺失就调用 `index_insert()`。
对 unique index，它仍使用 `UNIQUE_CHECK_YES`。
如果这一步发现真实 unique violation，就报错，concurrent build 失败。
失败后留下的 invalid index 需要用户或后续命令处理。

### 等待 in-progress tuple

heap build scan 在 unique 场景遇到 in-progress insert/delete，可能释放 buffer lock 后等待，再 recheck。
B-tree retail insert 遇到 conflicting in-progress tuple，释放 leaf page lock，等待 xid 或 speculative token，然后从 root 重查。
exclusion check 也会结束 index scan、等待、重试。
共同规律是：
不能在持有局部 index/page/buffer 内部锁时等待事务。
等待之后也不能复用旧位置。

### `UNIQUE_CHECK_PARTIAL`

deferred unique 和 speculative insert 不要求 insertion time 得到最终 verdict。
AM 必须允许插入，并返回 potential conflict。
这不是放松 correctness。
这是把 final verdict 推迟到 deferred trigger 或 speculative loop。
如果后续 recheck 失败，语句或事务再按约束语义处理。

### speculative livelock avoidance

`execIndexing.c:77-95` 解释 speculative insertion 的 livelock 风险。
两个 backend 同时 speculative 插入相同 key，如果都 back out 并立即 retry，可能反复冲突。
PostgreSQL 用 XID 顺序决定谁等待、谁 back out。
这在 `check_exclusion_or_unique_constraint()` 的 `CEOUC_LIVELOCK_PREVENTING_WAIT` 中体现。
unique AM 的 `_bt_check_unique()` 也通过 speculative token 让其他 backend 等 verdict。

### page full fallback

B-tree retail insert 目标 page 空间不足时，不马上 split。
`_bt_findinsertloc()` 会尝试 `_bt_delete_or_dedup_one_page()`。
这个函数先 simple deletion，再 bottom-up deletion，再 dedup。
如果都不够，才 split。
这说明 page split 是 slow path，不是唯一 path。
`indexUnchanged` hint 会提高 bottom-up deletion 的优先级。
它是性能 hint，不是 correctness 输入。

### incomplete split

crash 或 failure 可能让 split 处在 incomplete 状态。
后续 inserter 在遇到 incompletely split page 时必须先 finish split。
`_bt_stepright()` 和 `_bt_finish_split()` 都参与这个收尾。
这是结构修改的 redo/fallback 机制。
reader 依靠 high key/rightlink 通常仍能搜索。
writer 负责完成 parent downlink。

## 9. 成本、资源与跨模块传播

### build path 成本模型

普通 B-tree build 的主要成本是：
heap scan 成本随 heap pages 和 heap tuples 增长。
expression/predicate evaluation 成本随 index expression 复杂度和 tuple 数增长。
tuplesort 成本随 index tuples、key width、collation/comparison 成本和 `maintenance_work_mem` 增长。
bulk write 成本随 index pages、WAL volume、storage bandwidth 增长。
upper page build 成本通常远小于 leaf load，但 key 很宽或 tuple 很多时也会显现。
unique index build 多一个 `spool2` 可能性。
dead tuples 多时，需要 merge primary spool 和 dead tuple spool。
这增加排序/merge 成本，但避免 dead tuples 参与 unique check。
dedup build 可以减少重复 key 的 index tuple 数和 leaf page 数。
但 dedup 本身需要维护 pending posting list 状态。
是否收益取决于重复度、key size 和 heap TID 分布。
parallel build 的资源模型不是“worker 越多总内存越多无上限”。
`nbtsort.c:421-431` 注释强调整体效果是 `maintenance_work_mem` 仍作为 CREATE INDEX 的高水位。
worker sortmem 是 `maintenance_work_mem / participants`。
但 parallel build 仍会增加 CPU 并发、IO 并发、WAL/buffer usage 汇总和 DSM/worker 管理成本。
如果 DSM 创建失败或 worker 没启动，`_bt_begin_parallel()` 回退 serial build。
这是性能 fallback，不是 correctness fallback。

### insert path 成本模型

retail insert 每个 index 都要付出一份成本。
一个 INSERT 到有 N 个 ready index 的表，至少要形成 N 次 index datums，调用 N 次 `index_insert()`。
partial index 可以通过 predicate 跳过。
summarizing-only update 可以跳过非 summarizing indexes。
HOT update 可以避免普通 index insert。
不能 HOT 的 UPDATE 会插入新的 index entries。
B-tree insert 的主要成本是：
root-to-leaf search。
leaf page write lock。
unique check 的 equal-key scan 和 heap visibility check。
page cleanup/dedup/bottom-up deletion。
page split 和 parent insertion。
WAL insertion 和可能的 WAL flush 间接压力。
当 unique key 冲突指向 in-progress transaction 时，还会有 transaction wait。
等待时间和被等待事务持续时间相关，不和 B-tree page 大小成比例。
但等待前后的重查会增加 CPU 和 latch/lock 成本。

### resource propagation

build path 消耗 `maintenance_work_mem`、temporary files、DSM、worker slots、WAL bandwidth、bulk write IO。
这些压力会传播到：
temp tablespace 或临时文件目录。
WAL insert/flush。
checkpointer 和 bgwriter 的后续写回压力。
replication apply/streaming lag。
autovacuum 的 snapshot horizon 和 bloat 间接压力。
insert path 消耗 shared buffer、WAL、relation extension、index page locks、transaction lock waits。
多 index 表会把单条 heap write 放大成多条 index write。
unique/exclusion index 还会把 heap visibility 和 lock manager wait 引入 hot path。

### cross-module boundaries

commands 层决定 DDL flags、lock mode、concurrent phases。
catalog 层创建 relation 和 `pg_index` 状态。
table AM 层决定 build scan 中哪些 heap tuple 应进入 index。
executor 层决定 DML 时维护哪些 index、如何计算 expression/predicate、unique check 模式是什么。
index AM 层决定物理写入、bulk load、page split、WAL record。
storage/WAL 层提供持久化与 crash recovery。
这些模块的成本会互相传播。
例如 `CREATE INDEX CONCURRENTLY` 为了降低 writer blocking，增加了多事务阶段、两次等待和 validate scan。
例如 B-tree retail insert 为了降低 page split，增加了 bottom-up deletion 和 dedup slow path。
例如 unique constraint 为了用户语义，把 heap visibility check 引入 index insert。

### background processes

本节没有一个后台进程直接推进 IndexAM build/insert 的 correctness state。
build 和 insert 都由前台 backend 或 parallel worker 执行。
但后台进程会承接资源后果。
walwriter 可能影响 WAL flush latency。
checkpointer/bgwriter 可能承接 build 或 insert 产生的 dirty page 写回。
autovacuum 会影响 OldestXmin、dead tuple、HOT chain 和未来 bloat。
startup process 在 crash recovery 时重放 B-tree WAL，并可能恢复 incomplete split 可继续完成的状态。
archiver/replication 会放大 WAL volume 的外部可见成本。

## 10. 观测与诊断入口

### `pg_stat_progress_create_index`

`CREATE INDEX` 和 `CREATE INDEX CONCURRENTLY` 最直接看 `pg_stat_progress_create_index`。
能看到 command、phase、blocks done/total、tuples done/total、partitions done/total 等粒度。
B-tree build 还会更新 subphase。
`nbtsort.c:390-392` table scan。
`nbtsort.c:555-562` sort phases。
`nbtsort.c:574-576` leaf load。
concurrent build 还能看到 wait phases。
`indexcmds.c:1671-1674` phase wait 1。
`indexcmds.c:1740-1741` phase wait 2。
`indexcmds.c:1804-1806` phase wait 3。
这些指标说明“卡在哪个阶段”。
它们不能直接说明卡在哪个 tuple、哪个 key 或哪个 page。

### wait events

build path 可能等待 relation locks、older snapshots、parallel workers、IO、WAL。
`_bt_parallel_heapscan()` 使用 condition variable，wait event 是 `WAIT_EVENT_PARALLEL_CREATE_INDEX_SCAN`。
concurrent build 等待旧事务时可以从 `pg_stat_activity` 和 lock wait 推断。
retail insert unique conflict 可能等待 transactionid 或 speculative insertion。
`pg_locks` 能看到 transactionid lock 等待。
`pg_stat_activity.wait_event_type` 能显示 Lock、IO、LWLock 等大类。
但 wait event 只说明当前等待点。
它不等于完整性能归因。

### `EXPLAIN`

`EXPLAIN (ANALYZE, BUFFERS, WAL)` 对普通 DML 能看到 heap/index 写入造成的 buffer 和 WAL 量。
它不能直接列出每个 index AM 的内部 slow path。
例如 B-tree bottom-up deletion、dedup 和 split 不会逐项显示。
但 WAL 增多、shared dirtied/written 增多、execution time 增长可以提示 insert path 成本。

### catalog state

`pg_index` 能观察 `indisready`、`indisvalid`、`indcheckxmin`。
concurrent build 期间这些字段可以帮助定位阶段。
但 catalog view 有 MVCC 可见性。
不同 session 在不同 transaction snapshot 下看到的状态可能不同。
relcache invalidation 也不是瞬时同步所有已执行语句。

### page-level observation

`pageinspect` 可以观察 B-tree metapage、page opaque、items、high key、posting list 等。
它适合验证 build 后页面密度、rightlink、root level。
但它不能安全地在生产高并发下证明完整因果。
page 状态是瞬时的。
insert path 中的等待、unique recheck、speculative token 需要日志、gdb 或 injection point 更适合。

### gdb / breakpoint

源码跟读时常用断点：
`DefineIndex`
`index_create`
`index_build`
`btbuild`
`_bt_spools_heapscan`
`_bt_load`
`ExecInsertIndexTuples`
`index_insert`
`btinsert`
`_bt_doinsert`
`_bt_check_unique`
`_bt_insertonpg`
观察重点不是函数是否被调用。
而是：
`IndexInfo` 中 `ii_ReadyForInserts`、`ii_Concurrent`、`ii_Unique`、`ii_BrokenHotChain` 如何变化。
`checkUnique` 是哪个枚举值。
build scan 使用的是 `SnapshotAny` 还是 MVCC snapshot。
`BTBuildState.spool2` 是否存在。
`_bt_check_unique()` 是否返回 `xwait`。
`_bt_insertonpg()` 是否进入 split path。

### 能看见 / 只能推断 / 看不见

能直接看见：
progress phase、lock wait、catalog flags、relation size、WAL counters、EXPLAIN WAL/buffers。
可以通过工具观察：
B-tree page layout、metapage、部分 wait stack、gdb 中的 `IndexInfo` 和 `checkUnique`。
只能推断：
某次 insert 是否因为 bottom-up deletion 避免了 split。
某次 build sort 是否因为 key comparison 成本而慢。
某个 unique conflict 是否主要耗在 transaction wait 还是 repeated recheck。
几乎不可见：
每个 index tuple 在 build sort 内部的比较次数。
AM 私有缓存的完整生命周期。
错误路径中 resource owner 的每个 cleanup 动作。
诊断时要把这些层次分开。
不要把 `pg_stat_progress_create_index` 当成完整因果图。
也不要把 pageinspect 的某个 page snapshot 当成整个 build 历史。

## 11. 常见误区

误区一：
`ambuild` 会逐条调用 `aminsert`。
B-tree 不是这样。
`btbuild()` 走 heap scan、tuplesort、bulk load。
只有 concurrent validate 补缺失 tuple 时才回到 `index_insert()`。
误区二：
`IndexInfo` 是 catalog state。
它不是。
它是 backend-local runtime representation。
它从 catalog/relcache 构造，也携带 executor/build 临时字段。
误区三：
`indisready` 和 `indisvalid` 是同一个“可用”概念。
不是。
ready 主要影响 DML 是否维护。
valid 主要影响 planner 是否使用。
concurrent build 正是靠二者分离工作。
误区四：
`CREATE INDEX CONCURRENTLY` 只是普通 build 加弱锁。
不是。
它有 catalog visible but not ready、build、set ready、validate、wait old snapshots、set valid 多阶段。
误区五：
unique index build 只要排序后发现重复就报错。
不完整。
heap liveness、recently dead tuple、HOT chain、in-progress transaction、validate pass 都会影响 unique verdict。
误区六：
retail insert 只是在 leaf page 上插一个 tuple。
不完整。
它可能做 predicate lock、unique heap check、transaction wait、bottom-up deletion、dedup、page split、parent insertion、metapage update 和 WAL。
误区七：
`indexUnchanged` hint 可以改变索引语义。
不能。
它只是 B-tree 优化信号，用于减少版本 churn 导致的 page split。
误区八：
看到 `pg_stat_progress_create_index` 的 tuple count 就能知道 index tuple 数。
不一定。
AM 可以拒绝、合并、dedup 或用不同方式存储 tuple。
table AM 返回 heap tuple count。
AM 自己返回 index tuple stats。

## 12. 课堂实验

### 实验一：跟普通 build 的 callback 边界

目标：
确认普通 `CREATE INDEX` 不是逐条调用 `btinsert()`，而是调用 `btbuild()`。
步骤：
1. 在 PostgreSQL debug build 上准备一个大表。
2. 对 `btbuild()`、`_bt_spools_heapscan()`、`_bt_build_callback()`、`_bt_load()`、`btinsert()` 下断点。
3. 执行 `CREATE INDEX idx_t_a ON t(a);`。
4. 观察 `btbuild()` 进入，`_bt_build_callback()` 多次进入，`btinsert()` 不在初始 build 中进入。
5. 在 `_bt_spools_heapscan()` 中看 `buildstate.spool2` 是否为 NULL。
6. 对 unique index 重复实验，观察 `spool2` 初始化。
需要回答：
`_bt_build_callback()` 收到的是 `values/isnull/TID/tupleIsAlive`，不是已经形成好的 B-tree page item。
AM 可以选择自己的 build strategy。

### 实验二：观察 concurrent build 的 catalog 阶段

目标：
确认 `indisready` 和 `indisvalid` 分阶段变化。
步骤：
1. 创建大表，让 `CREATE INDEX CONCURRENTLY` 有足够时间运行。
2. 在一个 session 执行 `CREATE INDEX CONCURRENTLY idx_t_a ON t(a);`。
3. 在另一个 session 轮询 `pg_index`，查看目标 index 的 `indisready`、`indisvalid`、`indislive`。
4. 同时查询 `pg_stat_progress_create_index` 的 phase。
5. 用一个长事务持有旧 snapshot，观察 wait phase 是否延长。
需要回答：
为什么 ready 之后 planner 仍不能使用？
为什么 valid 前还要等待 older snapshots？
为什么不同 session 看到 catalog 状态可能受 transaction snapshot 影响？

### 实验三：观察 unique insert 的 wait/retry

目标：
确认 DML unique insert 的冲突等待发生在 AM insert path 内。
步骤：
1. 建表 `create table t(a int unique);`。
2. session A: `begin; insert into t values (1);` 不提交。
3. session B: `insert into t values (1);`。
4. 在 session B 用 `pg_stat_activity` 和 `pg_locks` 观察 transactionid wait。
5. 在源码调试中给 `_bt_check_unique()` 和 `_bt_doinsert()` 的 wait 分支下断点。
6. 让 session A commit 或 rollback，观察 session B 重查后的结果。
需要回答：
为什么等待前必须释放 B-tree leaf buffer？
为什么等待后从 root 重新 search？
为什么 `index_insert()` wrapper 本身没有实现 unique wait？

## 13. 讨论题

1. 为什么 `index_build()` 不直接扫描 heap 后逐条调用 `index_insert()`？请从排序、WAL、buffer cache、page split 和 unique dead tuple 角度回答。
2. `IndexInfo.ii_ReadyForInserts`、`pg_index.indisready`、`pg_index.indisvalid` 分别服务哪个调用者？如果把它们合并会破坏什么场景？
3. concurrent build 的 validate pass 为什么先收集 index TID 再扫 heap merge，而不是直接对每个 heap tuple 做 index lookup？
4. 为什么 unique constraint 的 correctness 不能只看 index leaf page 上是否已有相同 key？
5. `UNIQUE_CHECK_PARTIAL` 为什么必须允许插入 physical index tuple？它把最终责任交给了谁？
6. B-tree build 的 `spool2` 为什么只在 unique index 中需要？dead tuples 为什么仍可能需要进入最终 index？
7. 如果一个 AM 宣称 `amcanbuildparallel`，它必须额外处理哪些资源和 snapshot 边界？
8. 观测一次 insert 变慢时，如何区分 index 数量放大、unique wait、page split、WAL flush 和 expression evaluation 成本？

## 14. 本节小结

本节唯一主问题是：
一个统一的 IndexAM 抽象如何同时承接批量 build 和逐条 insert。
核心答案是路径分离、状态分层、合同稳定。
build path 从 `DefineIndex()` 到 `index_create()`，再到 `index_build()` 和 AM 的 `ambuild()`。
B-tree 的 `btbuild()` 使用 heap/table AM scan、tuplesort、bulk smgr load 和 page-level sequential construction。
retail insert path 从 executor 的 `ExecInsertIndexTuples()` 到 generic `index_insert()`，再到 AM 的 `aminsert()`。
B-tree 的 `btinsert()` 使用 `_bt_doinsert()`，包含 search、unique check、wait/retry、insert location、cleanup/dedup/split 和 WAL。
`IndexInfo` 是两条路径的共享 runtime definition。
但它不是 catalog tuple，也不是跨 backend 稳定状态。
`IndexAmRoutine` 是 AM 与 core 的 callback 合同。
但它只定义边界，不替 AM 保证页面结构 correctness。
普通 build、concurrent build、validate pass 的 snapshot 语义不同。
非 concurrent build 用 `SnapshotAny` 和 heap liveness 分类保护旧 snapshot。
concurrent build 用 MVCC snapshot、ready/valid 阶段和 older snapshot wait 保护并发写入。
validate pass 用 `index_insert()` 补缺失 tuple，并重新依赖 AM unique correctness。
ownership 不能只看 MemoryContext。
build path 有 tuplesort、DSM、parallel context、bulk write、relation locks 和 snapshot。
insert path 有 relation refs、per-tuple expr context、buffer pin/lock、transaction wait 和 WAL critical section。
错误路径的共同规律是：
局部 page/buffer lock 不能跨长事务等待。
等待后必须重查。
catalog invalidation 只刷新语义，不阻塞并发。
deferred/speculative 路径推迟 verdict，但不降低 correctness。
可观测入口包括 `pg_stat_progress_create_index`、`pg_index` flags、wait events、`pg_locks`、`EXPLAIN (BUFFERS, WAL)`、pageinspect 和 gdb。
这些入口粒度不同。
能看到 phase 不等于能看到 page split。
能看到 lock wait 不等于能解释全部延迟。
从本节带走的可迁移规律是：
稳定内核抽象往往不是把所有实现差异抹平。
它是把不可合并的生命周期拆成不同 callback，再用 catalog state、snapshot、lock、WAL 和 cleanup 合同把它们重新组合成用户可见的一致语义。
