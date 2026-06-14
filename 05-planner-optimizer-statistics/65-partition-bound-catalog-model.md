# PostgreSQL partition bound / catalog model
## 课程定位
前置知识：已经理解 `ProcessUtility()` 如何执行 DDL，知道 catalog 是事务性 heap 表，也理解 relcache 是 backend-local 的长寿命对象。
本节唯一主问题：
```text
分区表的 bound、partition key、partition descriptor 和 relcache 如何表达分区空间？
```
本节核心矛盾：
```text
catalog 必须用事务性、可回滚、可持久化的元组描述分区定义；
planner 和 executor 又需要一个 canonical、可搜索、可缓存的内存模型来快速判断分区空间。
```
一句话运行模型：
```text
父表的 partition key 存在 pg_partitioned_table；
每个子分区的 bound 存在 pg_class.relpartbound；
父子边存在 pg_inherits；
relcache 懒构造 PartitionKey 和 PartitionDesc；
planner/executor 通过 PartitionDirectory 在一次查询内稳定引用这些 descriptor。
```
学完后应能判断：
- 哪些信息属于持久 catalog，哪些只是 relcache 派生状态。
- `PartitionBoundSpec`、`PartitionBoundInfo` 和 `PartitionDesc` 分别回答什么问题。
- 为什么 default partition 不是普通 bound 值，而是整个分区空间的补集。
- 为什么 relcache invalidation 只传播“语义过期”，不会替其它 backend 改内存指针。
- 为什么 planner pruning、executor tuple routing 和 DDL overlap check 都依赖同一套 bound canonicalization。
- 如何从 catalog、`EXPLAIN`、锁等待、断点和源码入口定位分区空间表达错误。
本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e90741e12a3715909e1b2d71ff9344`。
## 1. 本节在总主线中的位置
05 目录前面的课程已经讲过 planner 的 base relation path、Append、DDL catalog update 和 relcache invalidation。
分区表把这些主题重新拧在一起。
创建一个 partitioned table 是 DDL。
打开一个 partitioned table 要走 relcache。
优化一个查询要把 parent 展开成 child relation，并生成 Append 或 MergeAppend。
执行 `INSERT` 要根据 tuple 的 key 找到 leaf partition。
本节不讲 planner-time pruning 的完整算法。
那是下一节 `66-partition-pruning-planner.md` 的主题。
本节也不讲 partition-wise join / aggregate。
那需要等价类、joinrel、partition scheme 对齐和 child join 搜索。
本节只回答一个更底层的问题：
```text
PostgreSQL 先把分区空间表达成什么状态，后续模块才有可能 pruning、routing 或 partition-wise planning？
```
这条主线从 DDL 写 catalog 开始，到 relcache 构造内存模型结束。
后续 planner 和 executor 都只是这个模型的消费者。
如果这个模型错了，pruning 再聪明也只能在错误空间里优化。
如果这个模型太慢，分区数增加时 planner、executor 和 DDL 都会被拖慢。
所以本节的核心不是“分区表有哪些语法”。
核心是：
```text
持久化 catalog 模型如何变成 backend-local 的 partition space model？
```
## 2. 核心矛盾与一句话运行模型
分区空间看起来可以用一个简单列表表达：
```text
parent
  -> partition p2024m01: [2024-01-01, 2024-02-01)
  -> partition p2024m02: [2024-02-01, 2024-03-01)
```
但内核里不能只存这样的字符串。
原因有五个。
第一，分区 key 可能是列，也可能是表达式。
`PARTITION BY RANGE (date_trunc('month', ts))` 需要保存表达式、类型、collation、opclass 和 support function。
第二，bound 必须能事务性修改和回滚。
`CREATE TABLE ... PARTITION OF`、`ATTACH PARTITION`、`DETACH PARTITION` 都是 catalog 更新，不是改一个全局内存 map。
第三，不同策略的 bound 搜索形态完全不同。
LIST 要按值查找，NULL 单独处理，DEFAULT 是补集。
RANGE 要处理 lower/upper、MINVALUE/MAXVALUE、gap、相邻边界去重。
HASH 要按 modulus/remainder 建立 remainder 到 partition index 的映射。
第四，分区描述符会被 planner、executor、trigger、COPY、logical replication 和 DDL 反复使用。
每次都扫 `pg_class`、`pg_inherits` 和 `pg_partitioned_table` 会把常见路径拖成 catalog lookup 热点。
第五，DDL 可以改变分区集合。
backend-local 指针不能跨进程共享。
因此变更只能通过 invalidation 传播过期事实，然后由各 backend 在安全边界重建。
PostgreSQL 的分层选择是：
| 层次 | 状态 | 主要职责 |
| --- | --- | --- |
| parse tree | `PartitionSpec` / `PartitionBoundSpec` | 表达用户语法和 transform 后的 bound 节点。 |
| catalog | `pg_partitioned_table` / `pg_class.relpartbound` / `pg_inherits` | 持久化 parent key、child bound 和父子边。 |
| relcache key | `PartitionKeyData` | 把 key 的列/表达式、opclass、collation、support function 做成可执行元数据。 |
| relcache desc | `PartitionDescData` + `PartitionBoundInfoData` | 把所有 child bound canonicalize 成可搜索的分区空间。 |
| query-local directory | `PartitionDirectory` | 在一次 planner 或 executor 生命周期内复用同一个 descriptor 指针。 |
| consumer | planner pruning、executor routing、DDL validation | 根据 canonical bound 判断 live partitions、目标 partition 或 overlap。 |
这个分层回答不同问题。
`pg_partitioned_table` 回答“父表按什么 key 划分空间”。
`pg_class.relpartbound` 回答“这个子表占据父表 key space 的哪一块”。
`pg_inherits` 回答“父子拓扑关系是什么，是否处于 detach pending”。
`PartitionKey` 回答“如何比较或 hash 一个 key 值”。
`PartitionBoundInfo` 回答“给定 key 值或新 bound，如何映射到 canonical partition index”。
`PartitionDesc` 回答“canonical partition index 对应哪个 relation OID，是否 leaf，以及是否存在 detached partition”。
`PartitionDirectory` 回答“本次查询中对同一个 relation OID 应该复用哪个 descriptor”。
不要把这些状态压成一个“分区列表”。
每一层都有自己的生命周期、可见性和错误路径。
## 3. 核心文件分工与阅读顺序
| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/catalog/pg_partitioned_table.h` | 父表 partition key 的持久 catalog 行，包含策略、key 列、opclass、collation、表达式和 default partition OID。 |
| 2 | `src/include/catalog/pg_class.h` | 子分区的 `relispartition` 与 `relpartbound`，以及 parent relkind `RELKIND_PARTITIONED_TABLE`。 |
| 3 | `src/include/catalog/pg_inherits.h` | 父子边、`inhseqno` 和 `inhdetachpending`。 |
| 4 | `src/include/nodes/parsenodes.h` | `PartitionSpec`、`PartitionBoundSpec`、`PartitionRangeDatum` 的 parse/DDL 节点形态。 |
| 5 | `src/backend/commands/tablecmds.c` | `DefineRelation()`、`ATExecAttachPartition()` 的 DDL 主流程、lock order、default partition 检查。 |
| 6 | `src/backend/catalog/partition.c` | `StorePartitionKey()`、default partition OID 更新、partition key dependency。 |
| 7 | `src/backend/catalog/heap.c` | `StorePartitionBound()` 写 `pg_class.relpartbound`、设置 `relispartition`、触发 parent/default relcache invalidation。 |
| 8 | `src/backend/parser/parse_utilcmd.c` | `transformPartitionBound()` 把 raw bound 转成类型化 `Const` / range datum。 |
| 9 | `src/include/utils/partcache.h` | `PartitionKeyData` 的字段语义。 |
| 10 | `src/backend/utils/cache/partcache.c` | `RelationGetPartitionKey()`、`RelationBuildPartitionKey()`、`RelationGetPartitionQual()`。 |
| 11 | `src/include/partitioning/partdesc.h` | `PartitionDescData` 和 `PartitionDirectory` 对外 contract。 |
| 12 | `src/include/partitioning/partbounds.h` | `PartitionBoundInfoData` 的 canonical bound 表达。 |
| 13 | `src/backend/partitioning/partdesc.c` | `RelationGetPartitionDesc()`、`RelationBuildPartitionDesc()`、snapshot-aware detached partition 处理。 |
| 14 | `src/backend/partitioning/partbounds.c` | `partition_bounds_create()`、list/range/hash bound canonicalization、overlap 检查。 |
| 15 | `src/backend/utils/cache/relcache.c` | relcache rebuild 时 partition key/descriptor 如何保留、失效和延迟释放。 |
| 16 | `src/backend/optimizer/util/plancat.c` | planner 如何把 `PartitionDesc` 写入 `RelOptInfo`。 |
| 17 | `src/backend/optimizer/util/inherit.c` | partitioned RTE 展开 child relation。 |
| 18 | `src/backend/executor/execPartition.c` | executor tuple routing 如何用 `PartitionDesc` 找 leaf partition。 |
推荐阅读顺序不是从 grammar 开始背语法。
更有效的顺序是：
```text
pg_partitioned_table / pg_class / pg_inherits
  -> DefineRelation() 写 key 和 bound
  -> RelationGetPartitionKey()
  -> RelationGetPartitionDesc()
  -> partition_bounds_create()
  -> relcache rebuild / invalidation
  -> planner/executor consumers
```
当前源码中，partition descriptor 的实现文件是 `src/backend/partitioning/partdesc.c`。
`src/backend/utils/cache/partcache.c` 仍然是必读文件，但它负责 partition key 和 partition qual，不负责构造 `PartitionDesc`。
这点容易被旧笔记或文件名误导。
## 4. 关键数据结构与状态
### 4.1. `pg_partitioned_table`
`pg_partitioned_table` 每个 partitioned parent 一行。
关键字段组合是：
| 字段 | 语义 |
| --- | --- |
| `partrelid` | partitioned parent relation OID。 |
| `partstrat` | `l`、`r`、`h` 三类 partition strategy。 |
| `partnatts` | partition key 维度数。 |
| `partdefid` | default partition OID，没有则 `InvalidOid`。 |
| `partattrs` | 每个 key 对应的 attribute number；`0` 表示该 key 是表达式。 |
| `partclass` | 每个 key 使用的 operator class。 |
| `partcollation` | 每个 key 的 collation。 |
| `partexprs` | `partattrs` 中为 `0` 的 key 对应的表达式树。 |
raw field 不是语义。
`partattrs = 0` 不能单独解释。
它必须和 `partexprs` 的第几个表达式、该表达式的类型、opclass、collation 和 support function 一起解释。
`partdefid` 也不是“一个范围上界”。
default partition 是剩余空间。
它的 partition constraint 依赖其他所有 partition 的 bound。
这也是为什么新增普通 partition 时，源码要 invalidation default partition 的 relcache。
### 4.2. `pg_class.relpartbound`
子分区的 bound 存在 `pg_class` 的 varlen 字段 `relpartbound` 中。
它不是固定 C struct 字段的一部分。
relcache 中 `rd_rel` 只保存 `pg_class` fixed-size 部分。
要读取 `relpartbound`，源码必须通过 syscache attr 或直接扫 `pg_class` tuple。
`StorePartitionBound()` 做三件关键事。
第一，把 `PartitionBoundSpec` 序列化成 node string，写入 `pg_class.relpartbound`。
第二，把子表的 `relispartition` 设置为 true。
第三，必要时更新 parent 的 `pg_partitioned_table.partdefid`，并 invalidation parent/default partition 的 relcache。
这说明 child bound 是持久 catalog 状态。
它不是 parent relcache 里的私有列表。
parent 的 `PartitionDesc` 只是从多个 child 的 `relpartbound` 派生出来的缓存。
### 4.3. `pg_inherits`
分区表复用 inheritance catalog 表达父子关系。
`pg_inherits` 关键字段是：
| 字段 | 语义 |
| --- | --- |
| `inhrelid` | child relation OID。 |
| `inhparent` | parent relation OID。 |
| `inhseqno` | 多父继承的序号；partition path 通常只允许一个直接 parent。 |
| `inhdetachpending` | DETACH CONCURRENTLY 过程中的 pending 标记。 |
分区空间不是只靠 `pg_inherits` 表达。
`pg_inherits` 只告诉你“谁是 child”。
它不告诉你 child 占据哪一段 key space。
完整分区空间需要同时读取：
```text
parent key: pg_partitioned_table
parent-child edge: pg_inherits
child bound: pg_class.relpartbound
```
缺任何一层都不能安全 routing。
### 4.4. `PartitionBoundSpec`
`PartitionBoundSpec` 是 bound 的 node 表达。
它既出现在 parse/transform 阶段，也被序列化到 `pg_class.relpartbound`。
关键字段是：
| 字段 | 语义 |
| --- | --- |
| `strategy` | 当前 bound 属于 list、range 还是 hash。 |
| `is_default` | 是否 default partition。 |
| `modulus` / `remainder` | hash partition 的 bound。 |
| `listdatums` | list partition 的 Const 列表，NULL 特殊处理。 |
| `lowerdatums` / `upperdatums` | range partition 的 lower/upper bound。 |
`PartitionRangeDatum` 还会区分三类值：
```text
MINVALUE
VALUE
MAXVALUE
```
range bound 中一旦某个 key 使用 `MINVALUE` 或 `MAXVALUE`，后续 key 必须同样是无限值。
`validateInfiniteBounds()` 负责这个边界。
### 4.5. `PartitionKeyData`
`PartitionKeyData` 是 relcache 派生状态。
它来自 `pg_partitioned_table`，但不是 catalog tuple 的简单拷贝。
关键字段组合是：
| 字段 | 语义 |
| --- | --- |
| `strategy` / `partnatts` | 分区策略和 key 维度。 |
| `partattrs` / `partexprs` | key 来源是列还是表达式。 |
| `partopfamily` / `partopcintype` | 比较或 hash 需要的 opfamily 与输入类型。 |
| `partsupfunc` | range/list 使用 btree compare support，hash 使用 hash extended support。 |
| `partcollation` | 比较函数调用时使用的 collation。 |
| `parttypid` / `parttyplen` / `parttypbyval` | datum copy、比较、bound copy 需要的类型信息。 |
`RelationGetPartitionKey()` 会返回 relcache 内部指针。
调用者必须保持 relation open。
源码注释强调 partition key 创建后不允许变化。
因此 relcache rebuild open entry 时可以保留旧 `rd_partkey`。
这与 `PartitionDesc` 不同。
partition key 是 parent 的空间坐标系。
partition descriptor 是 child set 和 bound 的当前视图。
key 相对稳定，desc 会随着 ATTACH/DETACH/CREATE PARTITION 变化。
### 4.6. `PartitionBoundInfoData`
`PartitionBoundInfoData` 是 canonical bound 表达。
它不是 catalog tuple。
它通常挂在 `PartitionDesc` 里，也可以在 planner 中用于 virtual partitioned joinrel。
核心字段是：
| 字段 | 语义 |
| --- | --- |
| `strategy` | list/range/hash。 |
| `ndatums` | `datums` 数组长度。 |
| `datums` | 排序后的 bound datum 数组。 |
| `kind` | range bound 中每个 datum 是 MINVALUE、VALUE 还是 MAXVALUE；list/hash 为 NULL。 |
| `nindexes` | `indexes` 数组长度，策略相关。 |
| `indexes` | 从 bound datum 或 hash remainder 映射到 canonical partition index。 |
| `null_index` | LIST 中接收 NULL 的 partition index。 |
| `default_index` | default partition index。 |
| `interleaved_parts` | LIST 中可能 interleaved 的 partition 位图，主要服务 planner 判断。 |
三种 strategy 的 `indexes` 语义不同。
LIST 中，`indexes[i]` 对应 `datums[i]` 这个 list value。
RANGE 中，`indexes[i]` 对应第 `i` 个 distinct range bound 的上界，`-1` 表示 gap；最后还有一个额外 `-1` 表示最大 bound 之后。
HASH 中，`nindexes` 等于 greatest modulus，`indexes[remainder]` 指向接收该 remainder 的 partition。
这就是 canonicalization 的意义。
后续代码不再把每个 child 的原始 SQL bound 从头解释一遍。
它只看排序后的 datums 和 indexes。
### 4.7. `PartitionDescData`
`PartitionDescData` 把 bound space 和 relation OID 接起来。
关键字段是：
| 字段 | 语义 |
| --- | --- |
| `nparts` | 当前 descriptor 包含的 partition 数。 |
| `detached_exist` | descriptor 是否看到了 detached/pending detached partition。 |
| `oids` | canonical partition index 到 relation OID 的数组。 |
| `is_leaf` | canonical partition index 是否 leaf partition。 |
| `boundinfo` | canonical bound space。 |
| `last_found_*` | executor tuple routing 的小缓存。 |
`oids` 的顺序不是 `pg_inherits` 扫描顺序。
`RelationBuildPartitionDesc()` 先读取 child OID 和 boundspec，再调用 `partition_bounds_create()` 得到 `mapping`。
随后用 mapping 把原始 child 列表重排成 canonical bound order。
这点是诊断分区问题时最容易漏掉的。
看到 `pg_inherits` 的顺序，不等于 `PartitionDesc.oids` 的顺序。
### 4.8. `RelationData` 中的分区字段
`RelationData` 持有分区相关 lazy cache：
| 字段 | 管理者 |
| --- | --- |
| `rd_partkey` / `rd_partkeycxt` | `RelationGetPartitionKey()`。 |
| `rd_partdesc` / `rd_pdcxt` | `RelationGetPartitionDesc(rel, false)` 或可复用 all-partitions descriptor。 |
| `rd_partdesc_nodetached` / `rd_pddcxt` | `omit_detached=true` 且 snapshot 条件允许时的 descriptor。 |
| `rd_partdesc_nodetached_xmin` | 判断 no-detached descriptor 是否可被当前 snapshot 复用。 |
| `rd_partcheck` / `rd_partcheckvalid` / `rd_partcheckcxt` | `RelationGetPartitionQual()` 生成的 partition constraint。 |
这些都是 backend-local 指针。
其它 backend 不能直接访问，也不能直接释放。
DDL 后的 shared invalidation 只让本 backend 之后处理消息并重建。
### 4.9. `PartitionDirectory`
`PartitionDirectory` 是查询生命周期里的 descriptor cache。
planner 在 `PlannerGlobal` 里有 `partition_directory`。
executor 在 `EState` 里有 `es_partition_directory`。
它的作用不是替代 relcache。
它保证同一次 planner 或 executor 生命周期中，对同一个 relation OID 反复 lookup 时拿到同一个 `PartitionDesc`。
`PartitionDirectoryLookup()` 还会对 relation 增加 refcount。
这样 directory 持有的 `PartitionDesc` 指针不会因为 relcache entry refcount 降为零而被销毁。
`DestroyPartitionDirectory()` 再把这些 refcount 降回去。
这个设计暴露了一个重要事实：
```text
PartitionDesc 是 relcache 内部指针，但它被外借给 query-local structures。
```
因此 relcache rebuild 时不能粗暴释放旧 descriptor。
## 5. 主流程源码 walkthrough
本节主流程以创建并使用一个 range partitioned table 为轴。
它不是单个函数。
它跨 DDL、catalog、relcache、planner 和 executor。
### 5.1. parser 生成 partition 节点
语法阶段生成 `PartitionSpec` 和 `PartitionBoundSpec`。
`PARTITION BY RANGE (logdate)` 进入 parent 的 `PartitionSpec`。
`FOR VALUES FROM (...) TO (...)` 进入 child 的 `PartitionBoundSpec`。
此时很多值还可能是 raw parse node。
它们还没有被 coercion 到 partition key 类型。
### 5.2. `DefineRelation()` 创建 relation shell
`tablecmds.c` 中 `DefineRelation()` 先调用 `heap_create_with_catalog()`。
这一步写入基础 `pg_class`、`pg_attribute`、`pg_type` 等 catalog。
随后执行第一次 `CommandCounterIncrement()`。
源码注释说明：必须让新建 relation tuple 对后续 open 可见。
之后 `relation_open(relationId, AccessExclusiveLock)` 打开新表。
如果这是 partitioned parent，后续会写 `pg_partitioned_table`。
如果这是 partition child，后续会写 `pg_class.relpartbound` 和 `pg_inherits`。
### 5.3. 写 parent partition key
对 parent table，`DefineRelation()` 会在处理 raw defaults/generated expressions 后处理 `stmt->partspec`。
关键调用链是：
```text
DefineRelation()
  -> transformPartitionSpec()
  -> ComputePartitionAttrs()
  -> StorePartitionKey()
  -> CommandCounterIncrement()
```
`StorePartitionKey()` 位于 `src/backend/catalog/partition.c`。
它写入 `pg_partitioned_table`。
同时为 opclass、collation、partition expression 记录 dependency。
如果 partition key 是普通列，源码还会把该列对 table 记录 `DEPENDENCY_INTERNAL`。
原因是不能单独 drop 作为 partition key 的列。
如果 key 是表达式，`recordDependencyOnSingleRelExpr()` 会记录表达式引用的函数、collation、列等对象。
最后 `StorePartitionKey()` 调用 `CacheInvalidateRelcache(rel)`。
这迫使下一次 CCI 后同 backend 重新构造使用新 catalog entry 的 relcache。
### 5.4. transform child bound
对 `CREATE TABLE child PARTITION OF parent FOR VALUES ...`，`DefineRelation()` 打开 parent。
它确认 parent 的 `relkind` 是 `RELKIND_PARTITIONED_TABLE`。
如果存在 default partition，它会以 `AccessExclusiveLock` 打开 default partition。
这个锁并不是为了保护新 child 可见性。
新 child 在提交前其他 backend 看不到。
锁的重点是 default partition 的 constraint 会因为新增 partition 改变。
其他 backend 不能在旧 constraint 下继续依赖 default partition。
随后源码调用：
```text
transformPartitionBound()
  -> transformPartitionBoundValue()
  -> transformPartitionRangeBounds()
  -> validateInfiniteBounds()
```
这一步把 bound 值转换为 partition key 对应类型的 `Const`。
LIST 会去重重复 datum。
RANGE 会检查 FROM/TO 的 key 数量是否等于 `partnatts`。
HASH 会检查 modulus 大于 0，remainder 小于 modulus。
Hash partition 不允许 default partition。
这是 transform 阶段就会报错的边界。
### 5.5. 检查新 bound 是否重叠
`check_new_partition_bound()` 在 `partbounds.c`。
它先拿 parent 的 `PartitionKey` 和 `PartitionDesc`。
然后按策略检查。
LIST 检查 list value 是否已经被已有 partition 接收。
RANGE 检查 lower/upper 是否与已有 range 重叠。
HASH 检查 modulus/remainder 规则和 remainder 覆盖是否冲突。
Hash 的额外规则是：
```text
every modulus must be a factor of the next larger modulus
```
这个规则保证 hash remainder space 可以用 greatest modulus 的 indexes 数组表达。
如果不强制这个规则，`rowHash % greatest_modulus` 就无法唯一映射到所有不同 modulus 的 partition。
Default partition 的检查不同。
它不与普通 bound 重叠。
它代表“其他所有 partition 不接收的值”。
因此新增 default partition 时只需要检查是否已经存在 default partition。
### 5.6. 检查 default partition 内容
新增普通 partition 时，如果 parent 已经有 default partition，default partition 的补集约束会变窄。
源码调用：
```text
check_default_partition_contents(parent, defaultRel, bound)
```
它先用新 partition bound 生成 constraint，再取反得到 default partition 的新约束。
如果 default partition 已有 CHECK 约束能推出新约束，则可以避免扫描。
否则需要扫描 default partition 及其 subpartition。
发现已有行应该属于新 partition 时，DDL 报错。
这条路径解释了一个可观察现象：
```text
给有数据的 default partition 新增普通 partition，可能因为 default 中已有冲突行而失败。
```
这不是 planner pruning 问题。
这是 DDL correct-by-construction 的边界。
### 5.7. 写 child bound
`StorePartitionBound()` 位于 `src/backend/catalog/heap.c`。
它更新 child 的 `pg_class` tuple。
动作包括：
- `nodeToString(bound)` 写入 `relpartbound`。
- 设置 `relispartition = true`。
- 对普通 table 清理可能残留的 `relhassubclass`。
- 如果 bound 是 default，更新 parent 的 `pg_partitioned_table.partdefid`。
- 执行 `CommandCounterIncrement()`。
- invalidation default partition 和 parent relcache。
注意这里先写 child 的 `relpartbound`，然后 `StoreCatalogInheritance()` 写 `pg_inherits`。
这是 `CREATE TABLE ... PARTITION OF` 路径里的实际顺序。
`ATTACH PARTITION` 路径则先 `CreateInheritance()`，再 `StorePartitionBound()`。
阅读源码时不要把两条路径理想化成完全相同的顺序。
它们服务不同前提：一个是新建 child，一个是已有表附加为 partition。
### 5.8. 构造 partition key
后续任何代码打开 parent 并请求 partition key 时，进入：
```text
RelationGetPartitionKey()
  -> RelationBuildPartitionKey()
  -> SearchSysCache1(PARTRELID)
  -> 读取 pg_partitioned_table
  -> 查 pg_opclass
  -> get_opfamily_proc()
  -> fmgr_info_cxt()
  -> reparent context to CacheMemoryContext
```
`RelationBuildPartitionKey()` 不是只读 catalog 字段。
它会做这些派生：
- 将 `partattrs` 复制成 `key->partattrs`。
- 将 `partexprs` 从 node string 还原成 expression tree。
- 对 partition expression 做 `eval_const_expressions()` 和 `fix_opfuncids()`。
- 根据 strategy 选择 support function 编号。
- LIST/RANGE 用 btree compare support。
- HASH 用 hash extended support。
- 收集每个 key 的类型长度、byval、align 信息。
构造中用的 memory context 先挂在 `CurTransactionContext`。
全部成功后再 reparent 到 `CacheMemoryContext`。
这避免构造中 ERROR 把半成品永久留在长寿命 context。
### 5.9. 构造 partition descriptor
parent 的 child set 和 bound space 通过：
```text
RelationGetPartitionDesc(rel, omit_detached)
  -> RelationBuildPartitionDesc()
  -> find_inheritance_children_extended()
  -> 逐个 child 读取 pg_class.relpartbound
  -> partition_bounds_create()
  -> partition_bounds_copy()
  -> mapping 重排 OID 和 is_leaf
  -> 写入 rd_partdesc 或 rd_partdesc_nodetached
```
`find_inheritance_children_extended()` 用单个 snapshot 获取 child OID 列表。
这保证返回值对应某个明确 catalog 时间点。
之后每个 child 的 `relpartbound` 优先从 syscache 读取。
如果 syscache 没有，源码会直接扫 `pg_class`。
这个 fallback 不是性能优化，而是并发 DDL 正确性边界。
源码注释列出两个问题。
第一，并发 ATTACH 可能让 syscache 暂时看不到新的 `relpartbound`。
第二，`DETACH CONCURRENTLY` 是两阶段过程，可能先标 detach pending，再清理 `relpartbound`。
如果组合到不一致视图，`RelationBuildPartitionDesc()` 会 `AcceptInvalidationMessages()` 后重试一次。
只重试一次是为了避免 catalog corruption 下无限循环。
### 5.10. canonicalize bound
`partition_bounds_create()` 根据 key strategy 分派：
```text
partition_bounds_create()
  -> create_list_bounds()
  -> create_range_bounds()
  -> create_hash_bounds()
```
LIST 路径会把所有非 NULL list values 收集起来，按 partition key 的 compare function 排序。
NULL 不放入 `datums`，而是记录到 `null_index`。
DEFAULT 不放入 `datums`，而是记录到 `default_index`。
RANGE 路径会收集每个 partition 的 lower 和 upper bound。
排序后只保留 distinct bound datum。
lower bound 对应的 gap 在 `indexes` 中记为 `-1`。
upper bound 对应 partition index。
最后多一个 `-1` 表示最大 bound 之后没有普通 range partition。
HASH 路径把 modulus/remainder 排序并填充 remainder array。
`nindexes` 是 greatest modulus。
`indexes[rowHash % nindexes]` 直接得到 partition index 或 `-1`。
这一步的输出是 canonical space。
DDL overlap check、executor routing、planner bound comparison 都建立在这个 canonical space 上。
### 5.11. planner 消费 descriptor
planner 在 `plancat.c` 的 `set_relation_partition_info()` 中使用 descriptor。
调用链是：
```text
set_relation_partition_info()
  -> CreatePartitionDirectory(CurrentMemoryContext, true)
  -> PartitionDirectoryLookup()
  -> find_partition_scheme()
  -> rel->boundinfo = partdesc->boundinfo
  -> rel->nparts = partdesc->nparts
  -> set_baserel_partition_key_exprs()
  -> set_baserel_partition_constraint()
```
这里 `omit_detached=true`。
planner 需要避免把对当前 snapshot 不应可见的 detached partition 纳入搜索。
同时 `PartitionDirectory` 保证一次 planning 中同一个 relation 的 descriptor 一致。
`inherit.c` 的 `expand_partitioned_rtentry()` 再通过 directory lookup 拿 descriptor。
它先调用 pruning 得到 `live_parts`，再为 surviving child 初始化 planner arrays。
下一节会展开 pruning。
本节只记住：pruning 的输入就是本节建立的 bound space。
### 5.12. executor 消费 descriptor
`INSERT`、`COPY FROM`、logical apply 等路径要把 tuple route 到 leaf partition。
`execPartition.c` 中 `ExecFindPartition()` 会初始化 partition dispatch。
关键状态包括：
```text
PartitionDispatchData:
  reldesc
  key
  keystate
  partdesc
  indexes[]
```
`get_partition_for_tuple()` 根据 strategy 查找 partition index。
HASH 路径最直接：
```text
rowHash = compute_partition_hash_value(...)
return boundinfo->indexes[rowHash % boundinfo->nindexes]
```
LIST 和 RANGE 路径会做 binary search。
`PartitionDesc` 中的 `last_found_*` 字段用于优化连续落到同一个 partition 的场景。
例如按当前时间写入 range partition，很多 tuple 都会落到最新月份。
连续命中阈值后，executor 可以先检查 last found partition 是否仍匹配，避免每行 binary search。
DEFAULT 和 NULL partition 不使用这个缓存。
原因是 default 没有具体 bound offset 可验证，NULL 路径本身已经非常便宜。
## 6. 生命周期 / ownership / cleanup
### 6.1. catalog 状态由事务持有
`pg_partitioned_table`、`pg_class.relpartbound` 和 `pg_inherits` 都是 catalog heap tuple。
它们的创建、更新、删除遵守普通 MVCC 和 WAL。
事务 abort 时，不需要手写“反向删除 partition key”。
普通 catalog tuple 版本回滚即可让未提交变更不可见。
但是同一事务内部仍然需要 `CommandCounterIncrement()`。
CCI 让后续步骤能打开刚创建的 relation，读取刚写入的 partition key 或 bound。
这也是 `DefineRelation()` 中多次 CCI 的原因。
### 6.2. relcache key 由 `RelationData` 持有
`RelationBuildPartitionKey()` 为 `rd_partkey` 创建单独 memory context。
构造中挂在 `CurTransactionContext`。
成功后 reparent 到 `CacheMemoryContext`。
`RelationDestroyRelation()` 会删除 `rd_partkeycxt`。
open relation rebuild 时，`relcache.c` 会尽量保留 `rd_partkey`。
源码理由是 partition key 不允许在 partitioned relation 创建后改变。
因此只要旧 key 已构造，保留它比重建更安全，也避免外借指针移动。
### 6.3. relcache descriptor 由 `rd_pdcxt` / `rd_pddcxt` 持有
`RelationBuildPartitionDesc()` 也用独立 memory context。
成功前挂在 `CurTransactionContext`。
完成后 reparent 到 `CacheMemoryContext`。
all-partitions descriptor 存在 `rd_partdesc` / `rd_pdcxt`。
omit-detached descriptor 存在 `rd_partdesc_nodetached` / `rd_pddcxt`。
如果构造失败，transaction context 会在 ERROR cleanup 中释放半成品。
这避免 partition descriptor 构造中 catalog lookup、node parse、bound copy 失败后污染长寿命 relcache context。
### 6.4. descriptor 指针外借后的延迟释放
`RelationGetPartitionDesc()` 返回 relcache 内部指针。
这和很多 relcache API 返回 copy 的习惯不同。
因此源码必须保护旧 descriptor。
`partdesc.c` 在写入新 descriptor 前，会把旧 `rd_pdcxt` 或 `rd_pddcxt` reparent 到新 context 下面。
`relcache.c` 在 rebuild open relation 时，也会保留旧 descriptor context。
注释明确指出这是因为可能存在 `PartitionDirectory` 指向旧 descriptor。
只有当 relation refcount 降到零，`RelationClose` 或 `RelationClearRelation` 才能清理这些旧 context。
所以不要把 invalidation 理解为“立即 free 旧 descriptor”。
正确模型是：
```text
invalidation 让语义过期；
refcount 和 directory 保护指针安全；
memory context 在安全边界释放旧 descriptor。
```
### 6.5. `PartitionDirectory` 的生命周期
planner 创建 directory 时使用当前 planner context。
executor 创建 directory 时使用 `estate->es_query_cxt`。
`PartitionDirectoryLookup()` 第一次看到 relation OID 时：
- 调用 `RelationIncrementReferenceCount(rel)`。
- 保存 `rel` 指针。
- 保存 `RelationGetPartitionDesc()` 返回的 descriptor。
directory 销毁时调用 `RelationDecrementReferenceCount()`。
它不直接释放 descriptor。
它只释放自己持有的 relation refcount。
具体 descriptor 是否释放，仍由 relcache entry 和 memory context 决定。
### 6.6. partition constraint cache
child partition 的 partition constraint 通过 `RelationGetPartitionQual()` 生成。
它读取 child 的 `relpartbound`，用 parent key 生成约束表达式。
如果 parent 自己也是 partition，还会递归合并 parent partition qual。
最后用 `map_partition_varattnos()` 把 parent attno 映射到 child attno。
`rd_partcheck` 返回给 caller 时是 working copy。
缓存的版本存在 `rd_partcheckcxt`。
default partition 的 constraint 依赖其他 partition 的 bounds。
新增或删除 partition 时，源码必须 invalidation default partition relcache。
这不是可选优化。
否则旧 default constraint 会让 query 或 DDL 验证使用过期空间补集。
## 7. 正确性机制层次
分区空间正确性不是一个机制保证的。
它由 catalog MVCC、DDL locks、CCI、dependency、invalidation、snapshot 和 relcache ownership 叠加而成。
### 7.1. MVCC 保证 catalog 事务性
parent key、child bound 和 parent-child edge 都是 catalog tuple。
未提交 DDL 对其他事务不可见。
abort 后 catalog 不留下 committed 语义。
这保证分区定义不会因为 DDL 中途 ERROR 而全局半更新。
MVCC 不保证当前 backend 的缓存自动刷新。
当前 backend 仍需要 CCI 和 local invalidation。
### 7.2. Heavyweight lock 保证 DDL/DML 边界
创建 partition child 时，parent 已经持有足够强的锁。
`ATTACH PARTITION` 会锁待 attach 表。
新增普通 partition 时，如果存在 default partition，还要以强锁锁住 default partition。
源码注释强调锁顺序：
```text
lock parent
  -> lock default partition
  -> lock partition being added or removed
```
这个顺序用来避免并发 attach/detach 与 default partition constraint 更新之间死锁。
lock 不能替代 invalidation。
lock 能阻止并发语义冲突。
它不会帮其他 backend 更新已经缓存的 `rd_partdesc`。
### 7.3. CCI 保证同事务阶段化可见性
`DefineRelation()` 创建 relation shell 后 CCI。
添加 generated/default expression 后 CCI。
写 partition key 后 CCI。
写 partition bound 后 `StorePartitionBound()` 内部也 CCI。
这些 CCI 的共同目的不是提交事务。
它们只让同一事务后续命令或后续步骤看到前一个 command 的 catalog tuple。
漏掉 CCI 会导致后续 `relation_open()`、`RelationGetPartitionKey()` 或 index/trigger cloning 读不到刚写入的元数据。
过早 CCI 又会扩大半成品 catalog 对本 backend 的可见窗口。
### 7.4. overlap check 保证空间不重叠
`check_new_partition_bound()` 是 DDL correctness 的核心。
它在写 catalog 前使用已有 `PartitionDesc` 检查新 bound。
LIST 不允许同一个非 NULL value 被多个 partition 接收。
RANGE 不允许区间重叠。
HASH 不允许 remainder 覆盖冲突，并强制 modulus factor rule。
Default partition 特判。
它不与普通 partition overlap。
它只要求 parent 当前没有 default partition。
### 7.5. default partition constraint 保证补集语义
default partition 的真实语义是：
```text
not(any ordinary partition accepts tuple)
```
因此普通 partition 集合变化时，default 的 partition constraint 也变化。
新增 partition 时，要检查 default partition 中是否已有行属于新 partition。
如果有，DDL 必须失败，除非用户先移动数据。
这保证 tuple routing 后不会出现“同一行既属于 default 又属于新增 partition”的 committed 状态。
### 7.6. invalidation 保证缓存语义过期
`StorePartitionKey()` invalidation parent relcache。
`StorePartitionBound()` invalidation parent relcache。
如果 default partition 存在，`StorePartitionBound()` 还 invalidation default partition relcache。
原因分别是：
- parent 的 `rd_partkey` / `rd_partdesc` 需要从新 catalog 重建。
- default partition 的 `rd_partcheck` 依赖其他 partitions 的 bounds。
shared invalidation 不传播新的 `PartitionDesc`。
它只传播 relid 级过期事实。
receiver 根据本地 refcount、open state 和 relation lock 决定删除、重建或延迟标 invalid。
### 7.7. snapshot 保护 detach 语义
`RelationGetPartitionDesc(rel, omit_detached)` 有 snapshot 相关行为。
如果不要求 omit detached，cached `rd_partdesc` 可以包含 detached/pending detached partitions。
如果要求 omit detached，并且存在 active snapshot，源码可能使用 `rd_partdesc_nodetached`。
但复用前必须检查保存的 `pg_inherits.xmin` 是否不在当前 active snapshot 中。
这说明 detached partition 可见性不是简单的布尔状态。
它与当前 snapshot 是否应该看见 detach 标记有关。
planner 和 executor 选择 `omit_detached` 的策略也不同。
planner 通常希望省略当前 snapshot 下不可见的 detached partition。
executor tuple routing 在事务隔离级别下要避免把可用 partition 过早排除。
### 7.8. dependency 保证生命周期约束
partition key 用到的列、opclass、collation 和表达式引用对象都有 dependency。
作为 partition key 的普通列对 table 是 internal dependency。
这意味着不能单独 drop 该列。
partition expression 中引用的函数或列也进入 dependency 图。
dependency 不负责 tuple routing。
它负责对象生命周期。
如果你能 drop partition key 依赖的函数或列，后续 relcache 构造 `PartitionKey` 就可能无法解释 catalog 中的 key。
## 8. 错误路径 / 异常路径 / fallback
### 8.1. 无效 bound specification
`transformPartitionBound()` 会拦截明显不匹配的 bound。
父表是 RANGE，却写 LIST bound，会报 invalid bound specification。
HASH partition 写 default partition，会报 hash-partitioned table may not have a default partition。
HASH modulus 小于等于 0 或 remainder 不小于 modulus，也会在 transform 阶段失败。
这些错误发生在写 catalog 前。
因此不需要额外 cleanup partition catalog。
### 8.2. 新 bound 与已有 partition 冲突
`check_new_partition_bound()` 发现 overlap 后报错。
典型场景：
- LIST 重复 value。
- RANGE 区间相交。
- HASH modulus/remainder 覆盖已有 remainder。
- 新 default partition 与已有 default partition 冲突。
这时 relation shell 可能已经创建。
事务 abort 会回滚 catalog tuple。
物理文件 cleanup 依赖 storage 层 pending delete 机制。
不要把这类 ERROR 想成 tablecmds 手写删除每一条 catalog。
### 8.3. default partition 中已有冲突数据
新增普通 partition 时，如果 default partition 已有行本应落入新 partition，`check_default_partition_contents()` 报错。
这条路径可能扫描 default partition。
如果 existing constraints 能推出新 default constraint，则避免扫描。
因此同一个 DDL 在不同 schema 下成本可能差很多。
它不是只写几行 catalog。
### 8.4. relpartbound 读取 fallback
`RelationBuildPartitionDesc()` 读取 child `relpartbound` 时先走 syscache。
如果拿不到，会直接扫 `pg_class`。
这个 fallback 处理并发 ATTACH 和 syscache/invalidation 的窗口。
它不是普通性能 fallback。
如果直接扫也没有，并且还没重试过，源码会 `AcceptInvalidationMessages()` 后 `goto retry`。
如果重试后仍没有 `relpartbound`，报 `missing relpartbound`。
这通常意味着 catalog 不一致或并发 detach/drop 场景已经超出可修复窗口。
### 8.5. DETACH CONCURRENTLY 的两阶段不一致窗口
`DETACH CONCURRENTLY` 不是一次性删除所有 catalog 语义。
它会先标记 detach pending，再后续清理 inheritance/bound 关系。
`find_inheritance_children_extended()` 能返回 detached 信息和 `detached_xmin`。
`RelationBuildPartitionDesc()` 需要基于 active snapshot 判断是否能省略这些 partitions。
如果读取 child list 和读取 child bound 之间看到不一致，源码重试一次。
这说明 relcache descriptor 构造不是简单无锁读 catalog。
它包含对并发 DDL 可见性的保守处理。
### 8.6. relcache rebuild 中 ERROR
open relation 收到 invalidation 时，`RelationRebuildRelation()` 要 in-place rebuild。
源码会先构造一个临时 `newrel`，再 swap 字段。
如果构造过程中 ERROR，旧 relation 仍标记为 invalid，但旧指针没有被释放。
下一次访问会再尝试 rebuild。
对 partition descriptor，old `rd_pdcxt` 还可能被 `PartitionDirectory` 持有。
所以 rebuild 不能直接 free。
### 8.7. no partition found
executor tuple routing 如果找不到 partition，会报错。
常见原因包括：
- 没有 default partition。
- LIST/RANGE/HASH bound 没覆盖该 tuple。
- 并发 detach 让当前 executor descriptor 不再包含某个 partition。
这个错误不能只从 SQL 文本判断。
需要看当时 executor 持有的 `PartitionDesc`、isolation level、`omit_detached` 选择和 relation locks。
## 9. 成本、资源与跨模块传播
### 9.1. DDL 成本随 partition 数扩张
新增 partition 不是 O(1) catalog insert。
常见成本包括：
- 读取 parent partition key。
- 构造或读取 parent `PartitionDesc`。
- `check_new_partition_bound()` 在已有 bound space 上检查 overlap。
- 如果有 default partition，可能扫描 default partition 数据。
- invalidation parent/default relcache。
- cloning parent index、trigger、foreign key。
对大量 partitions，瓶颈可能从 catalog write 转移到 relcache rebuild、default partition scan 或 lock wait。
### 9.2. descriptor 构造成本
`RelationBuildPartitionDesc()` 至少要读取所有 direct child OID。
然后读取每个 child 的 `relpartbound`。
再调用 `partition_bounds_create()` 排序和 canonicalize。
成本变量包括：
- direct partition 数 `nparts`。
- partition key 维度 `partnatts`。
- bound datum 的 pass-by-value/pass-by-reference copy 成本。
- compare support function 成本。
- 是否存在 detached partition。
- syscache 命中率和 fallback 到 catalog scan 的次数。
RANGE 要处理最多 `2 * nparts` 个 raw bounds。
LIST 要处理所有 list values，不只是 partition 数。
HASH 的 `nindexes` 等于 greatest modulus，可能大于 partition 数。
### 9.3. planner 成本传播
planner 把 `PartitionDesc.boundinfo` 写入 `RelOptInfo.boundinfo`。
后续 pruning、appendrel expansion、partitionwise join 都会消费它。
因此 descriptor 构造成本通常出现在 planning time。
分区数很多时，即使最终 pruning 掉大量 partitions，planner 也可能先付出构造 descriptor、展开必要元数据和处理 invalidation 的成本。
下一节 pruning 会继续讨论如何减少 child path 搜索。
本节只强调：pruning 的前提模型本身也有成本。
### 9.4. executor routing 成本
`get_partition_for_tuple()` 的成本取决于 strategy。
HASH 通常是 hash support function 加数组访问。
LIST/RANGE 是 binary search 加比较函数调用。
`last_found_*` 优化连续命中同一 partition 的场景。
这对按时间写入最新 range partition 很有效。
对均匀随机 key，缓存帮助有限，但额外成本也主要是记录和重置计数。
### 9.5. relcache churn
频繁 ATTACH/DETACH partition 会触发 parent relcache invalidation。
每个 backend 不会立即同步重建。
它们会在处理 invalidation 后，本地删除、标 invalid 或 rebuild。
如果 workload 中大量 backend 都频繁访问该 partitioned table，DDL 提交后的后续查询可能分散承担 rebuild 成本。
这个成本在 SQL 层不一定表现为单个 DDL 的耗时。
它可能表现为后续查询 planning latency 抖动。
### 9.6. 跨模块连接
| 模块 | 连接点 |
| --- | --- |
| parser/analyzer | raw partition syntax 要 transform 成类型化 bound value。 |
| catalog/DDL | key、bound、parent-child edge 都是事务性 catalog tuple。 |
| dependency | partition key 列、表达式、opclass、collation 进入对象生命周期图。 |
| relcache/syscache | key 和 descriptor 是 backend-local 派生缓存，依赖 invalidation。 |
| planner | `RelOptInfo.boundinfo`、`nparts`、`live_parts` 来源于 descriptor。 |
| executor | tuple routing 使用同一套 boundinfo 查找 leaf partition。 |
| lock manager | DDL、default partition constraint 更新和 concurrent detach 依赖 relation locks。 |
| WAL/storage | catalog 更新和新建 relation 通过普通 heap/WAL/storage cleanup 保证 crash safety。 |
## 10. 观测与诊断入口
### 10.1. 可直接观测的 catalog 状态
可以从 SQL 看三层持久状态：
```sql
select c.oid, c.relname, c.relkind, c.relispartition,
       pg_get_expr(c.relpartbound, c.oid) as bound
from pg_class c
where c.relname like 'p_%'
order by c.relname;
select partrelid::regclass, partstrat, partnatts,
       partdefid::regclass, partattrs, partclass, partcollation,
       pg_get_expr(partexprs, partrelid) as exprs
from pg_partitioned_table;
select inhparent::regclass, inhrelid::regclass, inhseqno, inhdetachpending
from pg_inherits
order by inhparent::regclass::text, inhseqno;
```
这些能看到持久 catalog。
它们看不到当前 backend 的 `PartitionDesc` 是否已经构造，也看不到 `PartitionDirectory` 是否持有旧 descriptor。
### 10.2. 看 partition constraint
`pg_get_partition_constraintdef()` 能显示 child partition constraint。
示例：
```sql
select c.oid::regclass,
       pg_get_partition_constraintdef(c.oid)
from pg_class c
where c.relispartition
order by 1;
```
对 default partition，要特别注意结果会随着其他 partitions 增减而变化。
如果新增普通 partition 后 default constraint 没有刷新，优先怀疑 relcache invalidation 或会话仍持有旧 snapshot/descriptor。
### 10.3. 看 planner 现象
`EXPLAIN` 可以看到是否生成 Append，以及 pruning 后剩余哪些 partitions。
最小入口：
```sql
explain (costs off)
select * from p_sales
where sold_at >= date '2024-02-01'
  and sold_at < date '2024-03-01';
```
如果 expected partition 没有被 pruning，先不要直接改 pruning 代码。
先确认：
- `pg_partitioned_table` 的 key 是否是你以为的列或表达式。
- `relpartbound` 是否是预期 bound。
- partition key opclass/collation 是否匹配表达式比较。
- parent relcache 是否被 invalidation 后重建。
### 10.4. 看 lock 等待
新增 partition 涉及 parent、default partition 和 child locks。
并发 attach/detach 或 default partition 相关 DDL 时，可看：
```sql
select pid, locktype, relation::regclass, mode, granted
from pg_locks
where relation in ('p_sales'::regclass, 'p_sales_default'::regclass)
order by pid, granted;
```
`pg_locks` 能解释阻塞。
它不能解释 relcache 是否已经重建。
也不能显示 `PartitionBoundInfo.indexes` 的内部映射。
### 10.5. gdb 断点
源码跟读时推荐断点：
```text
RelationBuildPartitionKey
RelationBuildPartitionDesc
partition_bounds_create
create_list_bounds
create_range_bounds
create_hash_bounds
check_new_partition_bound
StorePartitionBound
ExecFindPartition
get_partition_for_tuple
```
观察重点：
- `key->strategy`、`key->partnatts`、`key->partattrs`。
- `partdesc->nparts`、`partdesc->oids[]`。
- `boundinfo->ndatums`、`boundinfo->nindexes`、`null_index`、`default_index`。
- `mapping[i]` 如何把 catalog child 顺序映射成 canonical order。
- `omit_detached` 和 `rd_partdesc_nodetached_xmin`。
### 10.6. 能看到、只能推断、几乎不可见
| 类别 | 状态 |
| --- | --- |
| 能直接观测 | `pg_partitioned_table`、`pg_class.relpartbound`、`pg_inherits`、`pg_locks`、`EXPLAIN` 中的 Append/partition pruning 结果。 |
| 需要推断 | relcache 是否刚 rebuild、某个 backend 何时处理 shared invalidation、default partition constraint 是否来自新 descriptor。 |
| 几乎不可见 | 单个 `PartitionDesc` 的 memory context 生命周期、`PartitionDirectory` 持有的旧 descriptor 指针、`PartitionBoundInfo.indexes` 的完整内部数组。 |
真实诊断通常要把 catalog 查询、`EXPLAIN`、锁、server log、gdb 和源码一起用。
不要把 `pg_class` 一行解释成完整 runtime truth。
## 11. 常见误区
### 11.1. 把 `pg_inherits` 当成完整分区定义
`pg_inherits` 只表达父子关系。
分区 key 在 `pg_partitioned_table`。
bound 在 child 的 `pg_class.relpartbound`。
只看 `pg_inherits`，无法判断 overlap、default、NULL partition 或 hash remainder。
### 11.2. 把 `PartitionBoundSpec` 当成执行时查找结构
`PartitionBoundSpec` 是 node 形态。
它适合 catalog 序列化和 DDL transform。
executor 不会每行解释 `PartitionBoundSpec`。
执行时使用的是 canonical `PartitionBoundInfo`。
### 11.3. 以为 default partition 是一个普通最大范围
default partition 是补集。
它的语义依赖其他所有 non-default partitions。
因此新增或删除普通 partition 会改变 default partition constraint。
这也是 default partition 相关 DDL 需要额外锁和 invalidation 的原因。
### 11.4. 以为 relcache invalidation 会同步修改所有 backend
invalidation 不会把新的 descriptor 推送给其他 backend。
它只让 receiver 知道某个 relid 的本地缓存语义过期。
receiver 后续在本地安全边界删除或重建。
这就是 DDL 提交后，后续查询可能承担 relcache rebuild 成本的原因。
### 11.5. 以为 partition key 和 partition descriptor 生命周期相同
partition key 创建后不允许改变。
relcache rebuild open entry 时可以保留 `rd_partkey`。
partition descriptor 描述 child set 和 bounds。
ATTACH/DETACH/CREATE PARTITION 都会改变它。
两者都在 relcache，但 invalidation 和保留策略不同。
### 11.6. 以为分区顺序就是 catalog 扫描顺序
`PartitionDesc.oids[]` 是 canonical bound order。
它由 `partition_bounds_create()` 返回的 mapping 决定。
`pg_inherits` 的扫描顺序或 OID 顺序不等于 executor routing 使用的 partition index。
诊断 `boundinfo->indexes` 时必须同时看 `partdesc->oids[]`。
### 11.7. 把 planner pruning 错误直接归因给 pruning
pruning 的输入是 `PartitionKey`、`PartitionBoundInfo`、partition key expressions 和 restriction clauses。
如果 key expression、opclass、collation 或 bound canonicalization 与预期不一致，pruning 只是忠实消费错误或不同的输入。
先检查本节模型，再检查 pruning algorithm。
## 12. 课堂实验
### 实验 1：从 catalog 重建 mental model
创建一个 range partitioned table：
```sql
create table p_sales(id int, sold_at date, amount numeric)
partition by range (sold_at);
create table p_sales_2024_01 partition of p_sales
for values from ('2024-01-01') to ('2024-02-01');
create table p_sales_2024_02 partition of p_sales
for values from ('2024-02-01') to ('2024-03-01');
create table p_sales_default partition of p_sales default;
```
任务：
1. 查询 `pg_partitioned_table`，确认 parent key。
2. 查询 `pg_class.relpartbound`，确认每个 child bound。
3. 查询 `pg_inherits`，确认父子边。
4. 用 `pg_get_partition_constraintdef()` 看 default partition constraint。
源码跟读：
- 在 `StorePartitionKey()` 断点，看 parent key 如何写入 catalog。
- 在 `StorePartitionBound()` 断点，看 child bound 如何写入 `pg_class`。
- 在 `RelationBuildPartitionDesc()` 断点，看 `oids[]` 和 `boundinfo` 如何生成。
### 实验 2：观察 default partition 补集变化
先插入一行落入 default：
```sql
insert into p_sales values (1, date '2024-04-15', 10);
```
然后尝试新增覆盖该值的 partition：
```sql
create table p_sales_2024_04 partition of p_sales
for values from ('2024-04-01') to ('2024-05-01');
```
预期现象：
- 如果 default 中已有冲突行，DDL 报错。
- 报错来自 `check_default_partition_contents()`。
- 这不是 planner pruning 行为。
变体：
1. 删除或移动 default 中冲突行。
2. 再次创建 partition。
3. 比较 default partition constraint 的变化。
### 实验 3：LIST 的 NULL 和 DEFAULT
创建 list partition：
```sql
create table p_region(region text, payload int)
partition by list (region);
create table p_region_cn partition of p_region
for values in ('cn');
create table p_region_null partition of p_region
for values in (null);
create table p_region_default partition of p_region default;
```
任务：
- 在 `create_list_bounds()` 断点观察 `null_index`。
- 观察 `default_index` 不在普通 `datums` 中。
- 插入 `region is null`、`region = 'cn'`、`region = 'us'`，在 `get_partition_for_tuple()` 看命中路径。
结论：
NULL partition 和 default partition 是两个不同通道。
不能把 default 理解成“包含 NULL”。
### 实验 4：HASH modulus rule
创建 hash partition：
```sql
create table p_hash(id int)
partition by hash (id);
create table p_hash_0 partition of p_hash
for values with (modulus 4, remainder 0);
create table p_hash_1 partition of p_hash
for values with (modulus 4, remainder 1);
```
尝试添加不满足 factor rule 的 modulus 组合。
任务：
- 在 `check_new_partition_bound()` 看 `greatest_modulus`。
- 在 `create_hash_bounds()` 看 `boundinfo->nindexes`。
- 在 `get_partition_for_tuple()` 看 `rowHash % nindexes`。
结论：
hash bound 的 catalog 语法允许不同 modulus，但 canonical index array 要求它们形成可映射的 remainder space。
### 实验 5：relcache invalidation 与 descriptor rebuild
准备两个会话。
会话 A：
```sql
begin;
select count(*) from p_sales;
```
会话 B：
```sql
create table p_sales_2024_03 partition of p_sales
for values from ('2024-03-01') to ('2024-04-01');
```
任务：
- 在会话 A 继续查询前后，用 gdb 观察是否进入 `RelationBuildPartitionDesc()`。
- 在 B 的 `StorePartitionBound()` 断点观察 `CacheInvalidateRelcache(parent)`。
- 结合 transaction isolation 解释 descriptor 何时重建。
结论：
DDL 提交不会直接修改会话 A 的内存。
会话 A 在处理 invalidation 后本地重建或继续持有当前 snapshot 下合法 descriptor。
## 13. 讨论题
1. 为什么 parent key 放在 `pg_partitioned_table`，而 child bound 放在 `pg_class.relpartbound`？如果把所有 bound 都放在 parent 的一行里，会破坏哪些 DDL、MVCC 或 relcache 边界？
2. `PartitionKey` 和 `PartitionDesc` 都挂在 relcache 中，为什么前者可以在 open relation rebuild 中保留，后者不能简单保留？
3. Default partition 的 constraint 为什么依赖其他 partition 的 bound？新增普通 partition 时，如果不锁 default partition，会出现什么并发风险？
4. `PartitionBoundInfo.indexes` 在 list、range、hash 中语义不同。为什么 PostgreSQL 仍然使用同一个结构，而不是三套完全独立的 descriptor？
5. `PartitionDirectoryLookup()` 为什么要增加 relation refcount？如果只保存 `PartitionDesc *` 而不持有 relation，会有什么 use-after-free 风险？
6. `RelationBuildPartitionDesc()` 为什么在 syscache 读不到 `relpartbound` 时直接扫 `pg_class`，而不是只调用 `AcceptInvalidationMessages()`？
7. Hash partition 为什么要求 modulus factor rule？如果允许任意 modulus，`rowHash % greatest_modulus` 的 indexes array 会出现什么歧义？
8. 哪些状态可以用 SQL 直接看到，哪些必须通过 gdb 或源码推断？生产诊断时你会如何组合 `pg_class`、`EXPLAIN`、`pg_locks` 和断点？
## 14. 本节小结
本节唯一主问题是：
```text
分区表的 bound、partition key、partition descriptor 和 relcache 如何表达分区空间？
```
核心链路是：
```text
DDL parse nodes
  -> transform key/bound
  -> pg_partitioned_table / pg_class.relpartbound / pg_inherits
  -> RelationGetPartitionKey()
  -> RelationGetPartitionDesc()
  -> partition_bounds_create()
  -> PartitionDirectory
  -> planner pruning / executor routing / DDL validation
```
核心状态分四层。
Catalog 层持久化 parent key、child bound 和 inheritance edge。
Relcache 层把 key 和 bounds 派生成 `PartitionKey`、`PartitionDesc` 和 `PartitionBoundInfo`。
Query-local 层用 `PartitionDirectory` 固定一次 planner/executor 生命周期中的 descriptor 指针。
Consumer 层用同一套 canonical bound space 做 pruning、routing 和 overlap check。
ownership 的关键点是：
- catalog tuple 由事务和 MVCC 管。
- `rd_partkeycxt`、`rd_pdcxt`、`rd_pddcxt` 由 relcache memory context 管。
- `PartitionDirectory` 通过 relation refcount 保护外借 descriptor 指针。
- invalidation 只让本地缓存语义过期，不释放其他 backend 指针。
正确性来自分层组合。
MVCC 保证 DDL 提交/回滚。
CCI 保证同事务阶段化可见性。
relation lock 保证 DDL 与 DML 的并发边界。
overlap check 保证非 default 分区空间不重叠。
default partition scan/constraint 保证补集语义。
dependency 保证 partition key 依赖对象不能被破坏。
snapshot-aware descriptor 处理 concurrent detach。
异常路径中，transform 阶段会拦截 invalid bound。
overlap check 会拦截冲突 bound。
default partition 内容检查会拦截已有数据冲突。
descriptor 构造会在 syscache 不一致窗口 fallback 到 `pg_class` scan 并重试一次。
relcache rebuild 会保留旧指针直到 refcount 安全。
可观测入口包括 `pg_partitioned_table`、`pg_class.relpartbound`、`pg_inherits`、`pg_get_partition_constraintdef()`、`EXPLAIN`、`pg_locks` 和 gdb 断点。
不可直接观测的是单个 backend 的 `PartitionBoundInfo.indexes`、`PartitionDirectory` 指针持有和 relcache memory context 释放时刻。
从本节抽象出的可迁移规律是：
```text
持久 catalog 适合表达事务性事实；
hot path 需要 canonical in-memory model；
两者之间必须通过 ownership、snapshot 和 invalidation 明确边界。
```
这个规律同样适用于 index metadata、rewrite rule、RLS policy、extended statistics 和 plan cache。
诊断时不要只问“catalog 里有没有这一行”。
更完整的问题应该是：
```text
这个 catalog fact 是否已经对当前 command 可见？
它是否已经被 relcache 派生成当前 backend 的内存状态？
这个内存状态是否被当前 query-local owner 持有？
它是否已经收到 invalidation 但尚未在安全边界重建？
后续 planner/executor 消费的是哪一层状态？
```
这些判断仍然依赖 PostgreSQL 版本、partition 数量、DDL 并发、隔离级别、active snapshot、backend 数量和 workload 形态。
不要把单个字段、单个函数或单个 `EXPLAIN` 结果解释成完整因果。
