# PostgreSQL partition-wise join / aggregate
## 课程定位
前置知识：已经理解 `RelOptInfo`、`Path`、`AppendPath`、`EquivalenceClass`、分区 bound 模型、planner-time pruning 和普通 join/aggregate path 生成。
本节唯一主问题：
```text
partition-wise join / aggregate 何时合法，为什么需要对齐 partition scheme 和 equivalence class？
```
本节核心矛盾：
```text
分区表让 planner 可以把一个大 join 或 aggregate 拆成多个小问题；
但拆分只有在“不会产生跨分区匹配或跨分区分组”的语义证明成立时才合法。
```
一句话运行模型：
```text
planner 先把 partitioned rel 的 key、bound 和 child RelOptInfo 放进 RelOptInfo；
partition-wise join 只在两侧 PartitionScheme 指针相同且每个 partition key 都有等价连接证明时，把 parent joinrel 拆成 child joinrel；
partition-wise aggregate 只在 GROUP BY 覆盖 partition key 时做 full aggregate，否则最多做 per-partition partial aggregate 再 finalize。
```
学完后应能判断：
- `enable_partitionwise_join` 打开但计划没有 partition-wise join 时，应该检查 GUC、whole-row Var、partition scheme、partition key equality、outer join strictness、child path 和 bound merge 哪一层失败。
- `enable_partitionwise_aggregate` 打开但只出现普通 Aggregate 时，应该检查 input rel 是否 partitioned、GROUP BY 是否覆盖全部 partition key、collation 是否匹配、partial aggregate 是否可用。
- 为什么“两个表分区数一样”不是合法条件。
- 为什么“SQL 里有 `a.id = b.id`”也不一定足够。
- 为什么 `EquivalenceClass` 不只是排序工具，也参与证明 partition key 等价。
- 为什么 planner 宁愿放弃 partition-wise 机会，也不能让一个分区的行去错过另一个分区的匹配。
本课基于本地 `/home/nail/postgres` 源码，分支 `master`，短提交 `0e1f1ed157e`。
## 1. 本节在总主线中的位置
前两节已经建立两个基础。
第 65 节讲分区空间如何从 catalog 和 relcache 变成 `PartitionDesc`、`PartitionBoundInfo` 和 planner-local `PartitionScheme`。
第 66 节讲 planner-time partition pruning 如何把 parent rel 的 child 集合缩小成 `live_parts`。
本节继续往后走。
分区已经展开。
child `RelOptInfo` 已经有 pathlist。
现在 planner 面对的问题不是“哪些分区可以跳过”，而是：
```text
能不能把 parent 层的 join 或 aggregate 变成每个 partition 上独立完成的小计划？
```
这个问题比 pruning 更危险。
pruning 的错误会漏扫分区。
partition-wise join 的错误会漏掉跨分区 join 结果。
partition-wise aggregate 的错误会把同一 group 分散到多个 partial result 中而不再合并。
所以本节的主线不是“怎样更快”。
主线是：
```text
planner 如何证明局部计算等价于全局计算？
```
partition-wise join 和 partition-wise aggregate 都围绕这个证明。
join 侧的证明依赖两件事。
第一，两边用同一个 `PartitionScheme`，表示 key 维度、策略、opfamily、collation、support function 是同一套坐标系。
第二，每个对应 partition key 都被 join clause 或 `EquivalenceClass` 证明相等。
aggregate 侧的证明也依赖两件事。
第一，输入 rel 是 partitioned rel，并且 planner 仍知道它的 key expression。
第二，`GROUP BY` 覆盖所有 partition key，且 collation 匹配；否则只能 partial aggregate 后全局 finalize。
读源码时不要从 `enable_partitionwise_join` 这个 GUC 开始期待一个开关式优化。
它只是允许 planner 考虑这条路径。
真正的合法性来自 `RelOptInfo`、`PartitionScheme`、`PartitionBoundInfo`、`EquivalenceClass` 和 `GroupPathExtraData` 的组合。
本节主要连接这些课程：
- `29-reloptinfo-search-node.md`：`RelOptInfo` 是 planner 的关系状态容器。
- `37-joinrel-dynamic-programming.md`：joinrel 按 `relids` 子问题逐层构造。
- `39-join-path-generation.md`：child joinrel 仍然要生成普通 join path。
- `42-upperrel-stage-pipeline.md`：aggregate 属于 upper planning。
- `43-upper-aggregate-paths.md`：HashAggregate / GroupAggregate / partial aggregate 的候选生成。
- `65-partition-bound-catalog-model.md`：partition key 和 bound 的状态来源。
- `66-partition-pruning-planner.md`：`live_parts` 决定哪些 child 还存在于 planner 搜索空间。
本节不展开 executor `ExecAppend`、`nodeAgg.c` transition state 或 FDW partition-wise pushdown。
它们只在解释 plan shape 和诊断边界时出现。
## 2. 核心矛盾与一句话运行模型
最直观的 partition-wise join 例子是两个按同一列 range 分区的表：
```sql
SELECT *
FROM orders o
JOIN shipments s ON o.order_id = s.order_id;
```
如果两张表都按 `order_id` 使用同样的 range 分区，且所有分区边界对齐，那么 `orders_2026q1` 只需要 join `shipments_2026q1`。
这看起来像一个简单优化。
但内核不能只看名字或分区数。
合法条件必须覆盖这几个事实：
- 两边的 partition key 真的是同一语义，不只是列名相同。
- equality operator 属于 partition key 的 opfamily。
- collation 不改变比较语义。
- hash/list/range 的 bound 能形成一对一或最多一对一的匹配。
- outer join 的 nullable side 不会让后续 partition key 证明失效。
- child rel 的 targetlist、joininfo 和 EC child member 已经完成 parent-to-child 翻译。
核心 tension 是：
```text
局部化计算减少搜索和执行成本
  vs
SQL 语义允许等价、NULL、outer join、表达式 key、不同 bound 和不同 collation 破坏局部化假设
```
PostgreSQL 的选择很保守。
它只在 planner 能证明“同一 logical group 或 join match 不会跨越当前 partition pair”时才拆分。
证明失败不会报错。
planner 回退到普通 join 或普通 aggregate。
这和 partition pruning 的行为相似：优化失败不能改变结果。
一句话运行模型可以分成 join 和 aggregate 两条线。
join 线：
```text
set_relation_partition_info()
  -> find_partition_scheme()
  -> set_append_rel_size()
  -> build_join_rel()
  -> build_joinrel_partition_info()
  -> have_partkey_equi_join()
  -> try_partitionwise_join()
  -> build_child_join_rel()
  -> populate_joinrel_with_paths()
  -> generate_partitionwise_join_paths()
  -> add_paths_to_append_rel()
```
aggregate 线：
```text
create_grouping_paths()
  -> create_ordinary_grouping_paths()
  -> group_by_has_partkey()
  -> create_partial_grouping_paths()
  -> create_partitionwise_grouping_paths()
  -> make_grouping_rel(child)
  -> create_ordinary_grouping_paths(child)
  -> add_paths_to_append_rel()
```
两条线的共同点是：
```text
parent 语义先被证明可拆分；
child 语义通过 AppendRelInfo 翻译；
每个 child 仍然生成普通 path；
最后用 Append / MergeAppend 把 child result 拼回 parent relation。
```
关键点：partition-wise 不是一种新的 join algorithm 或 aggregate algorithm。
它是一种 plan decomposition。
每个 child 上仍然可能是 Hash Join、Merge Join、Nested Loop、HashAggregate 或 GroupAggregate。
## 3. 核心文件分工与阅读顺序
| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/nodes/pathnodes.h` | `RelOptInfo` 分区字段、`EquivalenceClass` child member、`PartitionwiseAggregateType`。 |
| 2 | `src/backend/optimizer/util/plancat.c` | `set_relation_partition_info()` 和 `find_partition_scheme()` 如何建立 planner-local partition scheme。 |
| 3 | `src/backend/optimizer/path/allpaths.c` | `set_append_rel_size()` 设置 `consider_partitionwise_join`；`standard_join_search()` 的 partition-wise path 收尾；`generate_partitionwise_join_paths()` 汇总 child join paths。 |
| 4 | `src/backend/optimizer/util/relnode.c` | `build_join_rel()`、`build_joinrel_partition_info()`、`have_partkey_equi_join()`、`set_joinrel_partition_key_exprs()`、`build_child_join_rel()`。 |
| 5 | `src/backend/optimizer/path/joinrels.c` | `try_partitionwise_join()` 建 child joinrel、翻译 `SpecialJoinInfo` 和 restrictlist、调用普通 join path 生成。 |
| 6 | `src/backend/optimizer/path/joinpath.c` | child joinrel 仍然通过 `add_paths_to_joinrel()` 生成物理算法候选。 |
| 7 | `src/backend/partitioning/partbounds.c` | `partition_bounds_equal()`、`partition_bounds_merge()`、list/range bound matching、hash bound 限制。 |
| 8 | `src/backend/optimizer/path/equivclass.c` | child `EquivalenceMember`、`exprs_known_equal()`、derived clause lookup。 |
| 9 | `src/backend/optimizer/path/pathkeys.c` | partition order pathkeys、`build_partition_pathkeys()` 和 `partitions_are_ordered()` 的边界。 |
| 10 | `src/backend/optimizer/plan/planner.c` | `create_ordinary_grouping_paths()`、`create_partitionwise_grouping_paths()`、`group_by_has_partkey()`。 |
| 11 | `src/backend/optimizer/util/appendinfo.c` | `adjust_appendrel_attrs()` / multilevel 翻译 parent 表达式到 child 表达式。 |
| 12 | `src/backend/optimizer/geqo/geqo_eval.c` | GEQO 路径下也要在生成 joinrel 后补 partition-wise paths。 |
推荐阅读顺序：
```text
RelOptInfo 分区字段
  -> find_partition_scheme()
  -> set_append_rel_size()
  -> build_joinrel_partition_info()
  -> have_partkey_equi_join()
  -> try_partitionwise_join()
  -> generate_partitionwise_join_paths()
  -> create_partitionwise_grouping_paths()
```
不要从 `joinpath.c` 开始。
`joinpath.c` 只解释 child joinrel 里面有哪些物理算法。
本节的核心合法性判断在 `plancat.c`、`relnode.c`、`joinrels.c` 和 `planner.c`。
也不要只读 `partbounds.c`。
bound matching 解决的是“哪些 child pair 对齐”。
它不证明 join key 等价。
join key 等价来自 explicit join clause 和 `EquivalenceClass`。
## 4. 可复现运行现象
### 4.1 分区完全对齐，join 可以拆成 child joins
示例 SQL：
```sql
SET enable_partitionwise_join = on;

CREATE TABLE o(id int, v text) PARTITION BY RANGE (id);
CREATE TABLE o1 PARTITION OF o FOR VALUES FROM (0) TO (100);
CREATE TABLE o2 PARTITION OF o FOR VALUES FROM (100) TO (200);

CREATE TABLE s(id int, v text) PARTITION BY RANGE (id);
CREATE TABLE s1 PARTITION OF s FOR VALUES FROM (0) TO (100);
CREATE TABLE s2 PARTITION OF s FOR VALUES FROM (100) TO (200);

EXPLAIN SELECT * FROM o JOIN s USING (id);
```
常见公开影子：
```text
Append
  -> Hash Join on o1/s1
  -> Hash Join on o2/s2
```
也可能是 `Merge Join`、`Nested Loop` 或带 `Parallel Append` 的形态。
重点不是算法名称。
重点是 parent 层 join 被拆成多个 child join。
源码解释：
`find_partition_scheme()` 让两张表复用同一个 `PartitionScheme`。
`have_partkey_equi_join()` 看到 `o.id = s.id`，并确认 equality operator 属于 partition key opfamily。
`try_partitionwise_join()` 建立 `o1 join s1` 和 `o2 join s2` 的 child joinrel。
`generate_partitionwise_join_paths()` 把 child joinrel 的 cheapest path 收集成 parent `AppendPath`。
### 4.2 分区数量相同但 scheme 不同，不能拆
示例 SQL：
```sql
CREATE TABLE a(id int, v text) PARTITION BY RANGE (id);
CREATE TABLE a1 PARTITION OF a FOR VALUES FROM (0) TO (100);
CREATE TABLE a2 PARTITION OF a FOR VALUES FROM (100) TO (200);

CREATE TABLE b(id int, v text) PARTITION BY RANGE ((id + 0));
CREATE TABLE b1 PARTITION OF b FOR VALUES FROM (0) TO (100);
CREATE TABLE b2 PARTITION OF b FOR VALUES FROM (100) TO (200);

SET enable_partitionwise_join = on;
EXPLAIN SELECT * FROM a JOIN b ON a.id = b.id;
```
表面上两边都有两个 range 分区。
但 `PartitionScheme` 还要比较策略、key 数、opfamily、opcintype、collation 和 support function。
更重要的是，partition key expression 的匹配也要落到 `partexprs` 上。
如果 key expression 不是同一个 planner 表达式，后续 `have_partkey_equi_join()` 很可能无法证明对应 key 等价。
诊断时不要把“边界看起来一样”当作源码证明。
源码证明用的是 planner-local expression tree 和 opfamily/collation。
### 4.3 有 join 条件但不是 partition key equality，不能拆
示例 SQL：
```sql
SET enable_partitionwise_join = on;
EXPLAIN SELECT * FROM o JOIN s ON o.id + 1 = s.id;
```
这个 join 是合法 SQL。
它也可能用普通 Hash Join。
但它不能证明 `o` 的某个 partition 只会匹配 `s` 的同一 partition。
`have_partkey_equi_join()` 会尝试把 join clause 两边表达式匹配到两侧的 partition keys。
`o.id + 1` 不等于 `o` 的 partition key `id`。
因此 partition-wise join 不合法。
### 4.4 GROUP BY 覆盖 partition key，可以 full partition-wise aggregate
示例 SQL：
```sql
SET enable_partitionwise_aggregate = on;
EXPLAIN SELECT id, count(*) FROM o GROUP BY id;
```
如果 `o` 按 `id` 分区，`GROUP BY id` 覆盖全部 partition key。
同一个 `id` 值只可能落在一个 partition 中。
planner 可以在每个 child 上做完整 aggregate，然后 append 结果。
源码入口是 `group_by_has_partkey()`。
它对 `parse->groupClause` 对应的表达式和 `input_rel->partexprs` 做 `equal()` 检查，并确认 collation 不冲突。
这里用 `parse->groupClause` 而不是 `processed_groupClause`。
源码注释说明即使某些 grouping key 被证明冗余，只要原始 group clause 覆盖 partition key，full partition-wise aggregate 仍然可以成立。
### 4.5 GROUP BY 不覆盖 partition key，只能 partial 或普通 aggregate
示例 SQL：
```sql
SET enable_partitionwise_aggregate = on;
EXPLAIN SELECT count(*) FROM o;
```
如果没有 `GROUP BY id`，不同 partition 的行属于同一个全局 group。
每个 child 上做 `count(*)` 后必须全局相加。
这不是 full partition-wise aggregate。
如果 aggregate 支持 partial，planner 可以在 child 上做 partial aggregate，再 append，再 finalize。
如果 aggregate 不支持 partial，planner 只能走普通 aggregate。
### 4.6 collate 或 opfamily 不匹配，不能当成同一坐标系
示例方向：
```sql
-- text key 上使用不同 collation 或不同 opclass 的分区定义
-- 再用看似相同的 equality join 或 GROUP BY
```
这类例子依赖环境中 collation 和 opclass。
诊断重点是：
`find_partition_scheme()` 比较 `partcollation`。
`have_partkey_equi_join()` 比较 join clause 的 `inputcollid` 和 partition key collation。
`group_by_has_partkey()` 比较 grouping expression collation 和 partition collation。
如果这些不一致，两个值的“相等”或“排序相邻”就不是同一套规则。
planner 必须拒绝拆分。
### 4.7 hash partition 的限制更保守
`partition_bounds_merge()` 对 hash partition 的注释很直接。
当前 merge bound 路径不支持 hash bound 的一般合并。
如果两边 bound 完全相等，`compute_partition_bounds()` 可以走 `partition_bounds_equal()` 的快路径。
否则 hash partition 的 merged-bound 情况会返回 `NULL`。
这不是 executor 不会 hash。
这是 partition-wise decomposition 无法保证一对一 child pair。
## 5. 关键数据结构与状态边界
本节所有状态都是 planner-local 或 relcache 派生状态。
它们不跨 backend 共享。
它们不需要 WAL。
它们的正确性来自 catalog snapshot、relcache/partition directory、表达式翻译、EC 推导和 conservative fallback。
### 5.1 `PartitionScheme`
`PartitionScheme` 是 planner 对 relcache `PartitionKey` 的拷贝和复用。
`find_partition_scheme()` 在 `root->part_schemes` 中查找已有 scheme。
匹配条件包括：
| 字段 | 为什么重要 |
| --- | --- |
| `strategy` | list/range/hash 的 bound 语义不同。 |
| `partnatts` | 多列分区必须逐列证明。 |
| `partopfamily` | equality、ordering、hash 语义来自 opfamily。 |
| `partopcintype` | support function 的输入类型。 |
| `partcollation` | text-like key 的比较语义。 |
| `parttyplen` / `parttypbyval` | bound datum copy 的内存语义。 |
| `partsupfunc` | partition bound lookup/merge 使用的比较或 hash support function。 |
源码后续直接用指针相等判断 scheme 是否相同：
```text
outer_rel->part_scheme == inner_rel->part_scheme
```
这不是偶然优化。
因为 `find_partition_scheme()` 已经把结构相同的 scheme 合并成同一个 planner-local 对象。
因此 pointer equality 是“同一套分区坐标系”的压缩表示。
### 5.2 `RelOptInfo` 的 partition 字段
和本节直接相关的字段：
| 字段 | join/aggregate 中的语义 |
| --- | --- |
| `part_scheme` | 当前 rel 的分区坐标系。 |
| `nparts` | partition 数；joinrel 初始为 `-1`，`0` 可表示放弃 partition-wise。 |
| `boundinfo` | 当前 rel 的 canonical bound；joinrel 可能来自 input bound 或 merge bound。 |
| `partbounds_merged` | joinrel 的 bound 是否由两侧 bound merge 得到。 |
| `part_rels` | partition index 到 child `RelOptInfo *` 的数组。 |
| `live_parts` | planner-time pruning 后仍参与的 partition index。 |
| `all_partrels` | 所有 child relids，用于 child pair 反查。 |
| `partexprs` | 非 nullable partition key expression 列表。 |
| `nullable_partexprs` | outer/full join 后只能在 strict operator 下使用的 key expression。 |
| `consider_partitionwise_join` | 当前 rel 是否允许参与 partition-wise join 搜索。 |
`nparts = 0` 在 joinrel 上尤其重要。
它可能表示“这个 joinrel 被当作 unpartitioned 处理”。
这通常是 fallback，不是 SQL 语义失败。
例如 child 被 pruning 后无法构造必要 child joinrel，源码会把 `joinrel->nparts = 0`，让后续路径把它当普通 joinrel。
### 5.3 `consider_partitionwise_join`
这个字段不是“已经能 partition-wise join”。
它表示“可以参与 partition-wise join 判断”。
base partitioned rel 上，`set_append_rel_size()` 设置它的条件是：
```text
enable_partitionwise_join
  AND reloptkind == RELOPT_BASEREL
  AND RTE relkind == RELKIND_PARTITIONED_TABLE
  AND 没有 whole-row Var 需求
```
whole-row Var 会让 child targetlist 翻译和 rowtype 语义变复杂。
因此 base rel 直接不进入 partition-wise join 考虑。
child rel 上，`set_append_rel_size()` 会把 parent 的标志传给 child。
源码注释说这里“abuse” 这个 flag。
它也被用来表示 child rel 已经完成 targetlist 和 EC child member 设置，足以作为 per-partition 输入。
### 5.4 `EquivalenceClass`
`EquivalenceClass` 记录 planner 已知的等价表达式。
它不是只给排序用。
本节至少有三种用法。
第一，join search 使用 `has_eclass_joins` 判断是否可能从 EC 生成 join clause。
第二，`have_partkey_equi_join()` 在 explicit restrictlist 不足时，调用 `exprs_known_equal()` 查询两个 partition key expression 是否已知相等。
第三，child rel 需要 child `EquivalenceMember`，否则 child pathkeys、MergeAppend 和 child join clause 推导会缺信息。
`pathnodes.h` 里有一个重要边界：
child member 不直接放进 `ec_members`。
它们放在 `ec_childmembers` 数组中，并通过 `EquivalenceMemberIterator` 按 child relids 访问。
这避免了分区很多时让每个 EC 的普通 member list 膨胀。
### 5.5 `partexprs` 与 `nullable_partexprs`
partitioned base rel 通常每个 key 在 `partexprs[i]` 中只有一个表达式。
joinrel 更复杂。
`set_joinrel_partition_key_exprs()` 会按 join type 分类：
| JoinType | joinrel 如何看待 partition key |
| --- | --- |
| INNER | outer 和 inner 的 key 都可作为非 nullable key。 |
| SEMI / ANTI | 只有 outer key 继续有意义，inner key 不出现在输出中。 |
| LEFT | outer key 是非 nullable key；inner key 只能放进 nullable key。 |
| FULL | 两边 key 都 nullable，并额外加入一些 `CoalesceExpr` 方便匹配 `JOIN USING` 输出。 |
这个分类解释了为什么 outer join 不能粗暴地说“join 后仍按两边 key 分区”。
nullable side 产生的 NULL 行不满足原来的 partition constraint。
但如果后续 join 使用 strict equality，NULL 行不会匹配任何对端行。
因此源码允许在 strict operator 下使用 `nullable_partexprs` 做进一步 partition-wise join 证明。
### 5.6 `PartitionBoundInfo`
`PartitionBoundInfo` 回答“partition index 对应哪段 key space”。
partition-wise join 需要两侧 child pair 对齐。
最便宜的情况是：
```text
rel1->nparts == rel2->nparts
AND partition_bounds_equal(...)
```
这时 joinrel 可以复用 input bound。
如果不是完全相等，range/list 可以尝试 `partition_bounds_merge()`。
这个函数要求每个 partition 最多匹配或重叠另一侧一个 partition。
如果一侧一个 partition 对应另一侧多个 partition，当前实现返回 `NULL`。
原因很直接：当前 child joinrel 数组表达的是一组一对一 join segment。
它不能表达“一侧 child 同时参与多个 join segment”而不复制路径和语义状态。
### 5.7 `GroupPathExtraData.patype`
aggregate 侧使用 `PartitionwiseAggregateType`：
| 值 | 语义 |
| --- | --- |
| `PARTITIONWISE_AGGREGATE_NONE` | 不使用 partition-wise aggregation。 |
| `PARTITIONWISE_AGGREGATE_FULL` | 每个 child 完整聚合，append 后无需再次 finalize 同一 group。 |
| `PARTITIONWISE_AGGREGATE_PARTIAL` | 每个 child partial aggregate，append 后还需要 finalize/global aggregate。 |
`create_grouping_paths()` 在 GUC 打开且没有 grouping sets 时，把 `extra.patype` 先设成 FULL 的理论可能。
`create_ordinary_grouping_paths()` 再根据 input rel 是否 partitioned、GROUP BY 是否包含 partition key、partial aggregate 是否支持，降级为 FULL、PARTIAL 或 NONE。
### 5.8 `AppendRelInfo`
partition-wise join 和 aggregate 都要把 parent 表达式翻译到 child。
相关入口包括：
```text
adjust_appendrel_attrs()
adjust_appendrel_attrs_multilevel()
adjust_child_relids()
find_appinfos_by_relids()
```
这些函数保证 child joinrel 使用 child varno、child relids 和 child target。
没有这层翻译，parent 的 `RestrictInfo`、`SpecialJoinInfo`、`PathTarget` 和 HAVING qual 不能直接挂到 child rel 上。
## 6. 主流程源码 walkthrough：partition-wise join
本节 join 主流程从一个 parent joinrel 的创建开始。
### 6.1 base partitioned rel 获取 partition metadata
`get_relation_info()` 中遇到 partitioned table，会调用 `set_relation_partition_info()`。
关键状态变化：
```text
root->glob->partition_directory
  -> PartitionDirectoryLookup()
  -> rel->part_scheme
  -> rel->boundinfo
  -> rel->nparts
  -> rel->partexprs
  -> rel->partition_qual
```
`PartitionDirectory` 保证一次 planner 调用中重复查询同一 relation 的 partition descriptor 时使用一致缓存。
`find_partition_scheme()` 则把 relcache partition key 拷贝成 planner-local `PartitionScheme`。
这一步是后续 pointer equality 的基础。
### 6.2 appendrel size 阶段设置 child 状态
`set_append_rel_size()` 遍历 partitioned rel 的 child。
它做的不只是 rows 估算。
和本节有关的状态包括：
```text
childrel->joininfo
childrel->reltarget
child EquivalenceMember
childrel->consider_partitionwise_join
childrel->consider_parallel
child rows/path size
```
如果 parent 有 `has_eclass_joins` 或 useful pathkeys，源码会调用 `add_child_rel_equivalences()`。
这一步让 parent EC 中的表达式能够映射到 child。
没有 child EC，后续 child pathkeys 和 child join clause 推导会缺信息。
`consider_partitionwise_join` 也在这里传给 child。
源码注释指出：即使 child 后续被证明 dummy，这个 flag 也表示它已经完成了足够的 setup。
### 6.3 parent joinrel 创建
标准 DP 搜索中，`make_join_rel()` 做 join legality 后调用 `build_join_rel()`。
`build_join_rel()` 为新的 `relids` 集合创建或复用 joinrel。
新建 joinrel 时，分区相关字段先是空或未定：
```text
part_scheme = NULL
nparts = -1
boundinfo = NULL
part_rels = NULL
live_parts = NULL
partexprs = NULL
nullable_partexprs = NULL
consider_partitionwise_join = false
```
随后它构造 join targetlist、restrictlist、joininfo、rows、parallel 标志和 hook。
最后调用：
```text
build_joinrel_partition_info(root, joinrel, outer_rel, inner_rel, sjinfo, restrictlist)
```
这一步才判断 joinrel 是否可被看作 partitioned joinrel。
### 6.4 `build_joinrel_partition_info()` 的合法性门
核心判断可以压缩成：
```text
PGS_CONSIDER_PARTITIONWISE enabled
AND outer_rel->part_scheme != NULL
AND inner_rel->part_scheme != NULL
AND outer_rel->consider_partitionwise_join
AND inner_rel->consider_partitionwise_join
AND outer_rel->part_scheme == inner_rel->part_scheme
AND have_partkey_equi_join(...)
```
前几项是结构和开关。
最关键的是后两项。
`part_scheme` 指针相同表示两边使用同一套 partition key 坐标系。
`have_partkey_equi_join()` 表示 join condition 能保证对应 key 相等。
两者缺一不可。
如果只对齐 scheme 但没有 equality join，`o.id < s.id` 仍然可能跨 partition 匹配。
如果只有 equality join 但 scheme 不同，`a.id = b.id` 也不能说明 `a` 的第 3 个 partition 只匹配 `b` 的第 3 个 partition。
### 6.5 `have_partkey_equi_join()` 先读 explicit restrictlist
函数先遍历 join restrictlist。
它过滤这些情况：
- outer join 中 pushed-down clause 不能作为本层 outer join 的证明。
- `rinfo->can_join` 为 false 的 clause 不能用。
- 不是 mergejoinable equality 也不是 hashjoinable equality 的 clause 不能用。
- clause 两边必须分别来自 rel1 和 rel2。
- 表达式必须匹配各自 partition key。
- 两边匹配到的 key ordinal 必须相同。
- collation 必须和 partition key collation 一致。
- operator 必须属于 partition key 的 opfamily。
源码用 `pk_known_equal[PARTITION_MAX_KEYS]` 跟踪每个 key 位置是否已经证明相等。
只有 `num_equal_pks == part_scheme->partnatts` 才返回 true。
多列分区时，证明第一列相等不够。
所有 partition key 都必须被覆盖。
### 6.6 strict operator 与 nullable partition key
如果 join operator 是 strict，`have_partkey_equi_join()` 会允许匹配 `nullable_partexprs`。
原因是 strict operator 遇到 NULL 不返回 true。
outer join 产生的 NULL-extended row 不会和另一侧任意 partition 形成 join match。
因此用 strict equality 做后续 partition-wise join 是安全的。
如果 operator 不 strict，就不能这样推理。
这不是实现细节。
这是 outer join + partition-wise join 的正确性边界。
### 6.7 `have_partkey_equi_join()` 再查 EC
explicit restrictlist 不足时，函数继续查 `EquivalenceClass`。
它为每个 partition key 找 btree opfamily。
hash partition key 会先从 hash opfamily 找 equality operator，再找到一个 mergejoin opfamily。
然后它遍历 `rel1->partexprs[ipk]` 和 `rel2->partexprs[ipk]`，调用：
```text
exprs_known_equal(root, expr1, expr2, btree_opfamily)
```
这解释了本节主问题里的 `EquivalenceClass`。
SQL 中的 equality 可能不是当前 join pair 的直接 restrictlist。
它可能来自传递闭包：
```sql
a.k = c.k AND c.k = b.k
```
也可能因 join order 变化而在另一个阶段被推导出来。
如果 partition-wise join 只看当前 explicit clause，会错过合法机会。
如果不受 EC join domain 和 collation/opfamily 限制地看 equality，又会产生错误证明。
### 6.8 joinrel 写入 partition key expression
通过合法性判断后，`build_joinrel_partition_info()` 写入：
```text
joinrel->part_scheme = outer_rel->part_scheme;
set_joinrel_partition_key_exprs(joinrel, outer_rel, inner_rel, sjinfo->jointype);
joinrel->consider_partitionwise_join = true;
```
注意它还没有设置 `joinrel->boundinfo`、`joinrel->nparts` 或 `joinrel->part_rels`。
源码注释明确说这些在 `try_partitionwise_join()` 中计算。
这是一个二阶段模型：
```text
build_joinrel_partition_info()
  证明 joinrel 可以被视为 partitioned
try_partitionwise_join()
  实际创建 child joinrel 和 bound mapping
```
### 6.9 `try_partitionwise_join()` 建 child joinrel
`make_join_rel()` 在普通 `populate_joinrel_with_paths()` 后调用 `try_partitionwise_join()`。
它先检查：
```text
joinrel->part_scheme != NULL
joinrel->nparts != 0
rel1 和 rel2 都是 partitioned rel
joinrel、rel1、rel2 都允许 partitionwise join
joinrel->part_scheme == rel1->part_scheme == rel2->part_scheme
```
然后调用 `compute_partition_bounds()`。
如果两边 bound 完全一致，joinrel 复用 input bound，按相同 partition index 配对。
如果 bound 不完全一致，range/list 会尝试 `partition_bounds_merge()`。
如果 merge 失败，`joinrel->nparts = 0`，这条 partition-wise 路径放弃。
### 6.10 child segment 的空输入处理
对每个 partition segment，`try_partitionwise_join()` 取出 child_rel1 和 child_rel2。
然后按 join type 判断空输入是否可以忽略：
| join type | segment 可忽略条件 |
| --- | --- |
| INNER / SEMI | 任一输入 empty。 |
| LEFT / ANTI | left 输入 empty。 |
| FULL | 两侧都 empty。 |
如果 child 被 pruning 成 `NULL`，但又不能根据 join type 忽略，源码放弃 partition-wise join：
```text
joinrel->nparts = 0
return
```
这是正确性保守路径。
因为 planner 无法为缺失 child 生成对应 child join path。
### 6.11 parent 状态翻译到 child
对可处理的 child pair，源码构造：
```text
child_sjinfo = build_child_join_sjinfo(...)
appinfos = find_appinfos_by_relids(...)
child_restrictlist = adjust_appendrel_attrs(parent_restrictlist)
child_joinrel = build_child_join_rel(...)
```
`build_child_join_rel()` 创建 `RELOPT_OTHER_JOINREL`。
它把 parent joinrel 的 target、joininfo、partition key expression、rows 估算相关状态翻译成 child 版本。
`AppendRelInfo` 是这一步的关键。
父表 varno 不应该泄漏到 child joinrel。
### 6.12 child joinrel 生成普通 path
child joinrel 建好后，源码继续调用：
```text
make_grouped_join_rel(...)
populate_joinrel_with_paths(...)
```
`populate_joinrel_with_paths()` 最终仍然进入普通 join path generation。
因此每个 child segment 可以选择自己的 Hash Join、Merge Join 或 Nested Loop。
partition-wise join 不要求所有 child 用同一种算法。
不过最终 parent `AppendPath` 只能消费每个 child 已经生成好的 path。
### 6.13 parent joinrel 汇总 child paths
`standard_join_search()` 每一层 `join_search_one_level()` 完成后，对该层 joinrel 调用：
```text
generate_partitionwise_join_paths(root, rel)
generate_useful_gather_paths(root, rel, false)
set_cheapest(rel)
```
为什么不在 `try_partitionwise_join()` 里立刻汇总？
`allpaths.c` 注释给出原因：
同一个 child joinrel 的 path 可能在 `join_search_one_level()` 中被多次追加。
如果过早用 child path 构造 parent append path，后续 `add_path()` 可能删除被引用的 child path。
所以 parent append path 必须等 child join paths 全部稳定后再创建。
`generate_partitionwise_join_paths()` 递归处理 partitioned child joinrel。
如果某个 child 没有任何 path，parent joinrel 会被标成 unpartitioned：
```text
rel->nparts = 0
return
```
如果所有 child 都 dummy，parent joinrel 也 `mark_dummy_rel()`。
否则它收集 non-dummy child joinrel，调用：
```text
add_paths_to_append_rel(root, rel, live_children)
```
这一步让 parent rel 获得由 child join paths 拼成的 Append/MergeAppend/Parallel Append 候选。
## 7. 主流程源码 walkthrough：partition-wise aggregate
aggregate 的主流程在 upper planning 阶段。
它不走 `joinrels.c`。
它消费的是 scan/join 阶段已经形成的 input `RelOptInfo`。
### 7.1 理论开关
`create_grouping_paths()` 先判断普通 grouping 能力。
它设置：
```text
GROUPING_CAN_USE_SORT
GROUPING_CAN_USE_HASH
GROUPING_CAN_PARTIAL_AGG
```
然后处理 partition-wise aggregate 开关：
```text
if (enable_partitionwise_aggregate && !parse->groupingSets)
    extra.patype = PARTITIONWISE_AGGREGATE_FULL;
else
    extra.patype = PARTITIONWISE_AGGREGATE_NONE;
```
这里的 FULL 只是理论初值。
真正是否 full，要到 `create_ordinary_grouping_paths()` 看 input rel 和 GROUP BY。
grouping sets 暂不支持 partition-wise aggregate。
### 7.2 `create_ordinary_grouping_paths()` 决定 full / partial / none
关键判断：
```text
extra->patype != NONE
AND IS_PARTITIONED_REL(input_rel)
```
如果 input rel 不是 partitioned rel，直接没有 partition-wise aggregate。
如果 input rel 是 partitioned rel，源码继续判断：
```text
if parent wants FULL and group_by_has_partkey(...)
    patype = FULL
else if partial aggregate is supported
    patype = PARTIAL
else
    patype = NONE
```
所以 full partition-wise aggregate 的条件比 partial 更强。
partial 只要求 aggregate 支持 partial mode。
full 还要求同一 group 不跨 partition。
### 7.3 `group_by_has_partkey()`
这个函数回答：
```text
GROUP BY 是否包含 input_rel 的每个 partition key expression，并且 collation 匹配？
```
它先从 `groupClause` 和 `targetList` 取出 grouping expressions。
然后对每个 partition key ordinal：
```text
for each partexpr in input_rel->partexprs[cnt]
  for each groupexpr in groupexprs
    remove one RelabelType
    if equal(groupexpr, partexpr)
       check collation
       found = true
```
所有 key 都 found 才返回 true。
注意这里只看 `partexprs`。
full aggregate 不能依赖 nullable partition key。
否则 outer join 产生的 NULL-extended row 可能破坏“同一 group 位于同一 partition”的证明。
### 7.4 partial grouping relation
如果一般 partial aggregate 可用，源码会调用 `create_partial_grouping_paths()`。
这个函数可能返回 `partially_grouped_rel`。
当 `patype == PARTITIONWISE_AGGREGATE_PARTIAL` 时，调用者会强制创建它。
原因是后续需要把 child partial aggregate append 到这个 upperrel 上。
`create_partial_grouping_paths()` 的注释说明：
这个 upperrel 中的所有 path 都已经做了 partial aggregate，但还需要 FinalizeAggregate。
### 7.5 递归到 child
`create_partitionwise_grouping_paths()` 遍历：
```text
input_rel->live_parts
```
对每个 child：
```text
child_input_rel = input_rel->part_rels[i]
child_target = copy_pathtarget(parent target)
child_extra = copy extra
appinfos = find_appinfos_by_relids(child relids)
adjust_appendrel_attrs(target/having/targetList)
child_grouped_rel = make_grouping_rel(...)
create_ordinary_grouping_paths(child)
```
这和 join 侧很像。
parent 的 target、HAVING 和 targetList 都必须翻译到 child。
然后 child 使用普通 aggregate path generation。
### 7.6 append child aggregate results
如果 child partial grouping 都成功，源码调用：
```text
add_paths_to_append_rel(root, partially_grouped_rel, partially_grouped_live_children)
```
如果是 full partition-wise aggregate，则还会调用：
```text
add_paths_to_append_rel(root, grouped_rel, grouped_live_children)
```
这里的 full 和 partial 对应不同 parent rel。
full child result 可以直接作为 `grouped_rel` 的 result。
partial child result 必须进入 `partially_grouped_rel`，后续再 finalize。
### 7.7 aggregate 与 join 的差异
partition-wise join 是 scan/join 阶段的 decomposition。
它需要证明 partition pair 不跨界匹配。
partition-wise aggregate 是 upper 阶段的 decomposition。
它需要证明 group 不跨 partition，或者保留 finalize 阶段。
join 侧依赖 `EquivalenceClass` 很重。
aggregate 侧主要依赖 GROUP BY expression 和 partition key expression 的结构匹配。
join 侧需要 child joinrel。
aggregate 侧需要 child grouped upperrel。
两者最终都回到 `add_paths_to_append_rel()`。
## 8. 生命周期 / ownership / cleanup
### 8.1 谁创建
`PartitionScheme` 由 `find_partition_scheme()` 在 planner memory context 中创建。
`RelOptInfo` 由 `build_simple_rel()`、`build_join_rel()`、`build_child_join_rel()`、`make_grouping_rel()` 等创建。
child `EquivalenceMember` 由 `add_child_rel_equivalences()` 或相关 EC helper 创建。
child join `SpecialJoinInfo` 由 `build_child_join_sjinfo()` 临时创建。
child restrictlist、target、HAVING、targetList 通过 `adjust_appendrel_attrs()` 创建翻译副本。
### 8.2 谁持有
`PlannerInfo` 持有这次 planning 的全局状态。
`root->simple_rel_array` 按 RT index 找 base/other rel。
`root->join_rel_list` 和 `root->join_rel_hash` 持有 joinrel。
`root->join_rel_level[k]` 是 DP 当前层的时间轴。
`RelOptInfo.part_rels` 按 partition index 持有 child rel 指针。
`EquivalenceClass` 持有 parent member 和 child member 索引。
`grouped_rel` / `partially_grouped_rel` 作为 upperrel 持有 aggregate pathlist。
### 8.3 谁释放
planner 中这些对象通常属于 planner context。
一次 query planning 结束后由 memory context 整体释放。
没有 ResourceOwner refcount。
没有 buffer pin。
没有 WAL cleanup。
这不意味着可以随便泄漏。
`try_partitionwise_join()` 中 per-child 的 `appinfos`、`child_relids` 和 child `SpecialJoinInfo` 会在循环尾显式释放。
源码注释说明，当有几千个 partition 时，循环内临时对象会积累显著内存。
因此它主动 `pfree(appinfos)`、`bms_free(child_relids)`、`free_child_join_sjinfo()`。
### 8.4 ERROR / abort 时怎么办
planning 阶段 ERROR 会走 PostgreSQL 标准 error unwinding。
planner memory context 会被上层清理。
本节没有事务级状态需要回滚。
catalog snapshot 和 relcache 状态由外层 planner/parse/relcache 机制管理。
如果 planning 因 OOM、stack depth 或 catalog lookup failure ERROR，已经创建的 planner-local `RelOptInfo` 不会进入 executor。
### 8.5 长期对象如何失效
长期对象是 catalog/relcache 中的 partition key 和 partition descriptor。
本节使用的是 planner-local 拷贝和 `PartitionDirectory` lookup。
DDL 改变分区定义时，relcache invalidation 让后续 backend 重建 partition descriptor。
已经开始的一次 planner 调用使用自己的 snapshot 和 memory context。
这节课不把 invalidation 当作主线，但要记住：
`PartitionScheme` 不是全局缓存对象。
它只在一次 planner 调用中代表一组等价 partition key properties。
## 9. 正确性机制层次
partition-wise join / aggregate 的正确性不是一个开关保证的。
它是多层证明叠加。
| 层次 | 机制 | 保证 | 不能保证 |
| --- | --- | --- | --- |
| catalog / relcache | `PartitionKey`、`PartitionDesc`、`PartitionBoundInfo` | 分区空间 canonical 表达 | join key 一定等价 |
| planner-local scheme | `PartitionScheme` pointer reuse | 两个 rel 使用同一 key 坐标系 | bound 一定一对一 |
| pruning state | `live_parts` | 哪些 child 仍参与 planner 搜索 | child pair 一定存在 |
| EC / restrictlist | `have_partkey_equi_join()` | 每个 partition key 有 equality 证明 | aggregate group 不跨 partition |
| bound matching | `partition_bounds_equal()` / `partition_bounds_merge()` | child segment 能配对 | 每个 child path 一定可生成 |
| child translation | `AppendRelInfo` | parent expression 能落到 child varno | 成本一定更低 |
| aggregate group proof | `group_by_has_partkey()` | full aggregate 不跨 partition | partial aggregate 一定收益 |
| fallback | `nparts = 0`、普通 path | 证明失败时保持正确 | 保留 partition-wise 形态 |
### 9.1 为什么需要对齐 PartitionScheme
`PartitionScheme` 对齐是“同一坐标系”证明。
对于 range partition：
比较函数、opfamily 和 collation 决定边界排序。
对于 list partition：
equality 和 collation 决定一个值属于哪个 list set。
对于 hash partition：
hash support function 和 opfamily 决定 hash remainder。
如果这些不同，即使 SQL 看起来都是 `id`，partition index 也不代表同一空间。
planner 不能用 partition ordinal 进行 child pairing。
### 9.2 为什么还需要 EquivalenceClass
scheme 对齐只说明坐标系相同。
它没有说明 join predicate 会按这个坐标系匹配。
例如：
```sql
SELECT * FROM a JOIN b ON a.k BETWEEN b.k - 10 AND b.k + 10;
```
两边按同一个 key 分区也不能局部 join。
因为一个 `a.k` 可能匹配相邻 partition 的 `b.k`。
必须有 equality 证明，才能说一个 key value 只可能落在对应 partition pair 中。
EC 的作用是补足 explicit restrictlist 不直接表达的 equality。
planner 的 join order 会改变当前 pair 看到的 restrictlist。
如果 `a.k = c.k` 和 `c.k = b.k` 已经形成 EC，那么 `a.k` 与 `b.k` 也可被证明等价。
### 9.3 outer join 的 NULL 边界
outer join 会制造 NULL-extended rows。
这些 rows 不满足 nullable side 原 partition constraint。
所以 joinrel 不能简单继承两边所有 partition key。
`set_joinrel_partition_key_exprs()` 把 key 分成 non-nullable 和 nullable。
`match_expr_to_partition_keys()` 只有在 strict operator 下才查 nullable key。
这个边界防止 planner 用会匹配 NULL 的 operator 做错误推理。
### 9.4 aggregate 的 full 与 partial 边界
full partition-wise aggregate 的不变量：
```text
每个 GROUP BY group 的所有输入行都来自同一个 partition。
```
这需要 GROUP BY 覆盖所有 partition keys。
partial partition-wise aggregate 的不变量：
```text
每个 partition 可以先产出 partial transition state；
全局 finalize 阶段仍会把跨 partition 的同一 group 合并。
```
因此 partial 对 GROUP BY 覆盖 key 的要求更低，但要求 aggregate 支持 partial aggregation。
### 9.5 为什么 fallback 必须安静
partition-wise planning 是优化。
它失败时不能改变语义。
所以源码大量使用 “return without doing anything” 或 `nparts = 0`。
这让后续阶段按普通 join/aggregate 继续。
不要把“没有 partition-wise plan”看作错误。
它通常只是某个证明条件不成立。
## 10. 错误路径 / 异常路径 / fallback
### 10.1 GUC 关闭
`enable_partitionwise_join` 默认 false。
`enable_partitionwise_aggregate` 默认 false。
`guc_parameters.dat` 标记它们是 `PGC_USERSET`，并带 `GUC_EXPLAIN`。
join GUC 打开后，`planner()` 把 `PGS_CONSIDER_PARTITIONWISE` 写入 `glob->default_pgs_mask`。
如果关闭，`build_joinrel_partition_info()` 直接不做任何 partition-wise 判断。
aggregate GUC 关闭后，`extra.patype = NONE`。
### 10.2 whole-row Var
base rel 如果需要 whole-row Var，`set_append_rel_size()` 不设置 `consider_partitionwise_join`。
典型 SQL 形态包括直接引用整行：
```sql
SELECT o FROM o JOIN s USING (id);
```
具体是否触发要看 targetlist 和 attr_needed。
诊断时可以在 `set_append_rel_size()` 打断点查看 `rel->attr_needed[InvalidAttrNumber - rel->min_attr]`。
### 10.3 scheme 不同
`outer_rel->part_scheme != inner_rel->part_scheme` 时，joinrel 不会被标成 partitioned。
这种失败没有 WARNING。
它是正常 fallback。
常见原因：
- 分区策略不同。
- key 数不同。
- key 类型或 opclass 不同。
- collation 不同。
- expression key 不同。
### 10.4 equality 证明不足
`have_partkey_equi_join()` 返回 false 时，joinrel 不会进行 partition-wise join。
常见原因：
- join condition 不是 equality。
- equality operator 不属于 partition opfamily。
- 多列 partition 只连接了部分 key。
- expression 不等于 partition key expression。
- outer join pushed-down clause 不能作为本层证明。
- EC 没有形成跨两侧 key 的等价关系。
### 10.5 bound merge 失败
两边 scheme 相同、key equality 成立，也不代表 bound 一定能配对。
`compute_partition_bounds()` 先尝试 exact equal。
exact equal 不成立时，list/range 尝试 merge。
如果一个 partition overlap 多个 partition，当前实现放弃。
hash 分区的非 exact merge 当前也放弃。
fallback 是 `joinrel->nparts = 0`。
### 10.6 child 被 pruning 后无法生成必要 segment
如果 child relation 被 planner-time pruning 成 `NULL`，但 join type 不能证明该 segment 对结果无贡献，`try_partitionwise_join()` 放弃整个 parent partition-wise join。
这避免出现缺 child path 的 Append。
### 10.7 child pathlist 为空
`generate_partitionwise_join_paths()` 要求每个 non-dummy child joinrel 有 path。
如果某个 child pathlist 为空，它也放弃 parent partition-wise shape：
```text
rel->nparts = 0
return
```
### 10.8 aggregate 不支持 partial
GROUP BY 不覆盖 partition key 时，最多只能 partial partition-wise aggregate。
如果 `can_partial_agg()` 不成立，`GROUPING_CAN_PARTIAL_AGG` 不会设置。
这时 `patype = NONE`。
常见原因包括 aggregate 没有 combine function、包含不支持 partial 的语义，或 grouping sets。
### 10.9 stack depth
`try_partitionwise_join()` 和 `generate_partitionwise_join_paths()` 都调用 `check_stack_depth()`。
深层 partition hierarchy 或递归 partition-wise join 可能触发 stack depth 防护。
这是 ERROR，不是普通 fallback。
## 11. 成本、资源与跨模块传播
partition-wise join / aggregate 的收益来自局部化。
成本也来自局部化。
### 11.1 planning time 与 planner memory
普通 joinrel 只需要一个 parent joinrel；partition-wise join 还会为每个匹配 partition segment 建 child joinrel。
如果有 `P` 个分区、`J` 个可拆 joinrel，child joinrel 和 child path generation 大致随 `P * J` 放大。
每个 child joinrel 仍会经历 `add_paths_to_joinrel()`、costing、parameterized path 判断和 `add_path()` 剪枝。
分区很多时，EC child member、derived equality clause、bound merge 和 child target 翻译也会成为 planner memory 压力来源。
这解释了为什么两个 GUC 默认关闭：它们可能让执行更快，也可能先让 planning 明显变重。
### 11.2 bound、pathkeys 与执行资源
range/list bound merge 要按 canonical bound 顺序比较，成本随 partition 数、key 维度、default partition 和 interleaved list partition 复杂度增长。
hash partition 只有 exact bound 场景容易处理，非 exact merge 当前更保守。
如果上层需要 ordering，`add_paths_to_append_rel()` 还会结合 `build_partition_pathkeys()` 和 `partitions_are_ordered()` 判断 Append/MergeAppend 机会。
partition-wise aggregate 可能降低单个 hash table 峰值，但 partial aggregate 会产生 partial states，并在 finalize 阶段再次聚合。
所以不能只看 child 局部成本；要同时看 append、finalize、parallel 和上层 sort/group 需求。
### 11.3 跨模块传播
| 模块 | 传播边界 |
| --- | --- |
| relcache / partition directory | 提供 `PartitionKey`、`PartitionDesc`、`boundinfo`。 |
| inheritance expansion | 创建 child RTE、`AppendRelInfo`、child `RelOptInfo`。 |
| equivalence class | 证明 join key equality，维护 child member。 |
| join search | 决定 parent joinrel 何时创建，child joinrel 何时补 path。 |
| aggregate upperrel | 决定 full/partial partition-wise aggregate。 |
| pathkeys | 判断 partition order 是否可提供有用 ordering。 |
| createplan / executor | 消费最终 Append/Agg/Join path，但不重新证明合法性。 |
## 12. 观测与诊断入口
`EXPLAIN` 能看到最终 plan shape，但看不到大多数合法性中间状态。
partition-wise join 的公开影子通常是 parent `Append` 下挂多个 child join：
```text
Append
  -> Hash Join
       -> Seq Scan on child_a_1
       -> Seq Scan on child_b_1
  -> Hash Join
       -> Seq Scan on child_a_2
       -> Seq Scan on child_b_2
```
partition-wise aggregate 的公开影子通常是 child aggregate 再 append，或者 partial 后 finalize：
```text
Append
  -> HashAggregate on child_1
  -> HashAggregate on child_2

Finalize Aggregate
  -> Append
       -> Partial Aggregate on child_1
       -> Partial Aggregate on child_2
```
具体节点可能因 cost 变成 GroupAggregate、Parallel Append、Gather、MergeAppend、Nested Loop 或 Merge Join。
EXPLAIN 看不到 `PartitionScheme` 是否指针相同、哪个 key 没证明、`pk_known_equal[]`、`partbounds_merged`、`joinrel->nparts` 何时变成 `0`、child EC member 是否建立，也看不到 `group_by_has_partkey()` 因 expression mismatch 还是 collation mismatch 失败。
这些需要断点、临时日志或源码计数器。
推荐断点：
```text
set_relation_partition_info
find_partition_scheme
set_append_rel_size
build_joinrel_partition_info
have_partkey_equi_join
try_partitionwise_join
partition_bounds_merge
build_child_join_rel
generate_partitionwise_join_paths
create_grouping_paths
create_ordinary_grouping_paths
group_by_has_partkey
create_partitionwise_grouping_paths
create_partial_grouping_paths
```
断点打印建议：
```text
rel->relids
rel->part_scheme
rel->nparts
rel->live_parts
rel->consider_partitionwise_join
joinrel->partbounds_merged
part_scheme->strategy
part_scheme->partnatts
restrictlist length
extra->patype
```
看到没有 partition-wise join 时，按顺序问：
1. `enable_partitionwise_join` 是否真的打开？
2. base rel 是否是 partitioned table？
3. base rel 是否需要 whole-row Var？
4. 两侧 `part_scheme` 指针是否相同？
5. 每个 partition key 是否都有 equality 证明？
6. equality operator、opfamily、collation 是否匹配？
7. bound 是否 exact equal 或可 merge？
8. child 是否被 pruning 到无法形成必要 segment？
9. child joinrel 是否有 path？
10. parent append path 是否生成后被成本竞争淘汰？
看到没有 partition-wise aggregate 时，按顺序问：
1. `enable_partitionwise_aggregate` 是否打开？
2. query 是否有 grouping sets？
3. input rel 是否 partitioned？
4. GROUP BY 是否覆盖所有 partition key？
5. grouping expression 是否和 `partexprs` 结构相等？
6. collation 是否匹配？
7. 如果不能 full，partial aggregate 是否可用？
8. child aggregate paths 是否都生成？
9. append partial/full paths 是否生成后输给普通 aggregate？
`pg_stat_statements` 可显示 planning time 和 execution time 变化，但不能告诉你 partition-wise 合法性哪层失败。
`EXPLAIN (ANALYZE, BUFFERS)` 能验证最终扫描了哪些 child，`pg_stat_io` 能辅助看 IO 变化。
planner 内部放大通常要用 `perf`、火焰图、`debug_print_plan`、gdb 或临时日志。
## 13. 常见误区
- 分区数一样不等于 partition-wise join 合法；必须 scheme 对齐、key equality 成立、bound 能配对。
- `enable_partitionwise_join=on` 不是强制使用；它只允许 planner 考虑，合法性失败或成本竞争失败都可能没有该形态。
- EC 不只是排序优化；`have_partkey_equi_join()` 会用它补足 equality 传递证明。
- partition-wise aggregate 不一定更快；partial aggregate 可能产生大量 partial groups、append 和 finalize 成本。
- EXPLAIN 没有 partition-wise 不等于 pruning 没生效；pruning 和 partition-wise decomposition 是两层不同判断。
- outer join 下 nullable side 的 key 不能无条件继续传递，只能在 strict operator 边界内使用。
- full aggregate 与 partial aggregate 不是成本差异；前者证明 group 不跨 partition，后者保留全局 finalize。
- child joinrel 不是 executor 自动产生的，而是 planner 在 `try_partitionwise_join()` 中显式创建的 `RELOPT_OTHER_JOINREL`。
## 14. 课堂实验
### 实验 1：从 EXPLAIN 观察 partition-wise join 合法性
目标：看到 scheme + equality 成立时 parent join 被拆成 child joins。
步骤：
```sql
DROP TABLE IF EXISTS o CASCADE;
DROP TABLE IF EXISTS s CASCADE;

CREATE TABLE o(id int, payload text) PARTITION BY RANGE (id);
CREATE TABLE o_0_100 PARTITION OF o FOR VALUES FROM (0) TO (100);
CREATE TABLE o_100_200 PARTITION OF o FOR VALUES FROM (100) TO (200);

CREATE TABLE s(id int, payload text) PARTITION BY RANGE (id);
CREATE TABLE s_0_100 PARTITION OF s FOR VALUES FROM (0) TO (100);
CREATE TABLE s_100_200 PARTITION OF s FOR VALUES FROM (100) TO (200);

INSERT INTO o SELECT g, 'o-' || g FROM generate_series(0, 199) g;
INSERT INTO s SELECT g, 's-' || g FROM generate_series(0, 199) g;
ANALYZE o;
ANALYZE s;

SET enable_partitionwise_join = off;
EXPLAIN SELECT * FROM o JOIN s USING (id);

SET enable_partitionwise_join = on;
EXPLAIN SELECT * FROM o JOIN s USING (id);
```
观察点：GUC off 时通常是 parent append 输入后的普通 join；GUC on 且合法时通常能看到每个 child pair 上的 join，算法名称不是重点。
源码回读：`set_append_rel_size()`、`build_joinrel_partition_info()` 和 `try_partitionwise_join()` 是否分别设置 flag、写入 scheme、创建 child joinrel。
### 实验 2：删除 key equality
目标：证明 scheme 对齐不够。
步骤：
```sql
SET enable_partitionwise_join = on;
EXPLAIN SELECT * FROM o JOIN s ON o.id < s.id;
```
观察点：两边仍然是同样的分区表，但 `o.id < s.id` 不是 equality，一个 `o` 分区可能匹配多个 `s` 分区。
断点：`have_partkey_equi_join()`，看 `num_equal_pks` 是否覆盖全部 key。
### 实验 3：多列分区只连接部分 key
目标：证明每个 partition key 都要被覆盖。
步骤方向：
```sql
CREATE TABLE x(a int, b int, v text) PARTITION BY RANGE (a, b);
CREATE TABLE y(a int, b int, v text) PARTITION BY RANGE (a, b);
SET enable_partitionwise_join = on;
EXPLAIN SELECT * FROM x JOIN y ON x.a = y.a;
EXPLAIN SELECT * FROM x JOIN y ON x.a = y.a AND x.b = y.b;
```
观察点：
- 只连接 `a` 不足以证明 `(a,b)` partition key 全部相等。
- 加上 `b` 后才可能合法。
源码回读：`pk_known_equal[]` 中每个 key ordinal 都要 true。
### 实验 4：full partition-wise aggregate
目标：观察 GROUP BY 覆盖 partition key 时 child 完整聚合。
步骤：
```sql
SET enable_partitionwise_aggregate = on;
EXPLAIN SELECT id, count(*) FROM o GROUP BY id;
```
观察点：plan 可能出现每个 child 下的 Aggregate，再 Append。
断点：`group_by_has_partkey()` 和 `create_partitionwise_grouping_paths()`，打印 `patype` 与 child 数。
### 实验 5：partial partition-wise aggregate
目标：观察 GROUP BY 不覆盖 partition key 时的 partial/finalize 边界。
步骤：
```sql
SET enable_partitionwise_aggregate = on;
EXPLAIN SELECT count(*) FROM o;
```
观察点：如果 aggregate 支持 partial，可能看到 child partial aggregate 和上层 finalize。
源码回读：`create_ordinary_grouping_paths()` 中 `patype` 是否从 FULL 降级为 PARTIAL，`create_partial_grouping_paths()` 是否创建 `partially_grouped_rel`。
### 实验 6：bound 不完全对齐
目标：观察 bound merge 的保守边界。
方向：一侧 range 分区是 `[0,100), [100,200)`，另一侧是 `[0,50), [50,100), [100,200)`，join key 仍然 equality。
预期：如果一个 partition 需要匹配另一侧多个 partition，当前 bound merge 会失败。
断点：`partition_bounds_merge()` / `merge_range_bounds()`，观察 `joinrel->nparts` 是否变成 `0`。
### 实验 7：源码插桩计数 child joinrel
目标：量化 partition 数对 planning 的放大。
插桩位置：
```text
try_partitionwise_join()
build_child_join_rel()
generate_partitionwise_join_paths()
```
建议打印 parent `relids`、`cnt_parts`、child `relids`、child pathlist length。
对比 10、100、1000 个 partition，关注 planning time、planner memory 和 child joinrel 数量。
### 实验 8：EC 传递等价
目标：观察没有直接 `a.k = b.k` 时，EC 是否仍能证明 key equality。
方向：三张同 scheme 分区表 `a,b,c`，条件是 `a.k = c.k AND c.k = b.k`。
断点：`have_partkey_equi_join()` 和 `exprs_known_equal()`，关注 explicit restrictlist 是否不足、EC 查询是否补上。
## 15. 讨论题
1. 为什么 `PartitionScheme` 用 pointer equality，而不是每次深度比较所有字段？
2. 如果两张表 range 分区边界相同，但 key collation 不同，partition-wise join 为什么不合法？
3. 多列 partition key 中只证明第一列 equality，会出现什么跨分区错误？
4. outer join 后 nullable side 的 partition key 为什么只能在 strict operator 下继续使用？
5. `GROUP BY` 不包含 partition key 时，full partition-wise aggregate 会错在哪里？
6. partial partition-wise aggregate 为什么仍然需要全局 finalize？
7. 为什么 `try_partitionwise_join()` 不在 child path 尚未稳定时创建 parent append path？
8. 如果一个 partition overlap 另一侧两个 partitions，当前实现为什么宁愿放弃？
9. EXPLAIN 没有 partition-wise plan 时，如何区分“合法性失败”和“成本竞争失败”？
10. 分区很多时，EC child member 的存储方式为什么会影响 optimizer 扩展性？
## 16. 本节小结
partition-wise join / aggregate 的核心不是一个执行技巧。
它是 planner 对“全局计算能否拆成 partition-local 计算”的语义证明。
join 侧的链路是：`PartitionScheme` 对齐，partition key equality 由 restrictlist 或 EC 证明，bound exact match 或可 merge，child joinrel 完成 `AppendRelInfo` 翻译并生成普通 join path，最后 parent 用 Append/MergeAppend 汇总。
aggregate 侧的链路是：input rel 必须仍被 planner 看作 partitioned；GROUP BY 覆盖 partition key 才能 full，否则只有 aggregate 支持 partial 时才能 per-partition partial，再 append 并 finalize。
`PartitionScheme` 回答“是否同一套分区坐标系”。
`EquivalenceClass` 和 join restrictlist 回答“join key 是否逐 key 相等”。
`PartitionBoundInfo` 回答“child segment 是否能安全配对”。
`AppendRelInfo` 回答“parent 语义如何翻译到 child”。
这些状态缺一层，planner 都不能安全拆分。
ownership 主要属于一次 planner 调用的 memory context；没有 WAL、pin、refcount 或 shared memory cleanup。
但 per-partition 临时对象会在热点循环中显式释放，避免几千分区时 planning memory 被放大。
异常路径的基本策略是保守 fallback：GUC 关闭、whole-row Var、scheme 不同、equality 证明不足、bound merge 失败、child path 缺失、partial aggregate 不支持，都会回到普通 path。
除 stack depth 或内部错误外，这些不是用户可见错误。
观测上，EXPLAIN 只能看到最终 plan shape，不能告诉你 `have_partkey_equi_join()` 哪一项失败，也看不到 `part_scheme` 指针、EC child member 或 `partbounds_merged`。
定位合法性边界要在 `build_joinrel_partition_info()`、`have_partkey_equi_join()`、`try_partitionwise_join()`、`group_by_has_partkey()` 和 `create_partitionwise_grouping_paths()` 处观察 planner-local 状态。
可迁移规律：把全局计算拆成局部计算前，必须先证明数据划分边界和语义等价边界重合；如果只能证明局部可预聚合，必须保留全局 finalize；证明失败时，优化器应该安静回退。
partition-wise join 是否更快依赖 partition 数、child rows、统计准确性、join algorithm、并行度和上层 ordering。
partition-wise aggregate 是否更快依赖 group cardinality、partial state 大小、work_mem、child 数量和 finalize 成本。
本节最终要带走的判断是：为什么某个 plan 可以被拆，为什么另一个看起来相似的 plan 不能被拆，以及拆分后的 planning 和 execution 成本会从哪里扩散。
