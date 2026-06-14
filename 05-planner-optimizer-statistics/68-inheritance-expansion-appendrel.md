# PostgreSQL inheritance / partition expansion 与 appendrel Var translation
## 课程定位
前置知识：已经理解 `PlannerInfo`、`RelOptInfo`、base relation pathlist、partition bound/catalog model，以及 planner-time partition pruning 的基本路径。 本节唯一主问题：
```text
inheritance/partition expansion 如何构造 appendrel、child rel 和 Var translation？
```
本节核心矛盾：
```text
SQL 语义把父表看成一个逻辑 relation；
planner 又必须为每个真正可扫描的 child relation 使用自己的 relid、列布局、锁、统计信息、path 和 row identity。
```
一句话运行模型：
```text
planner 延迟到 restriction、EC、lateral 和 rowmark 信息都建立后，
把带 inh 标记的 parent RTE 展开成一组 child RTE；
每个 child RTE 对应一个 RELOPT_OTHER_MEMBER_REL 的 RelOptInfo；
parent 到 child 的列语义通过 AppendRelInfo.translated_vars 和 parent_colnos 保存；
后续 size/path/createplan/setrefs 都依赖这份映射。
```
学完后应能判断：
- `rte->inh` 只是触发展开的标志，不是 appendrel 本体。
- 传统继承和分区表都会产生 appendrel，但展开顺序、是否扫描 parent、是否递归 flatten 不同。
- 为什么 partition pruning 发生在 child RTE / child RelOptInfo 创建之前。
- 为什么 `AppendRelInfo.translated_vars` 是 Var translation 的核心状态，而不是 EXPLAIN 里的 `Append` 节点。
- 为什么 parent qual、join qual、targetlist、updated columns、row identity 都必须经过同一套 parent-to-child 映射。
- 如何从 `EXPLAIN`、断点、`debug_print_plan`、`pg_locks` 和源码入口判断一个 child 是没被展开、被 constraint exclusion 标成 dummy，还是生成 path 后被成本竞争剪掉。
本课基于本地 `/home/nail/postgres` 源码，分支 `master`，短提交 `0e1f1ed157e`。
## 1. 本节在总主线中的位置
05 目录前面已经讲过 planner 入口、`PlannerInfo`、base relation path、partition bound/catalog model 和 planner-time partition pruning。 那些课程回答了几个前置问题。 第一，parser/rewrite 之后，一个 table reference 如何成为 `RangeTblEntry`。 第二，partitioned table 的 bound、partition key 和 `PartitionDesc` 如何进入 relcache 和 planner。
第三，planner-time pruning 如何计算 `live_parts`，避免为不可能命中的 partition 建 child path。 本节接着问一个更靠近 planner 内部形态的问题：
```text
当父表不能直接代表所有实际扫描对象时，
planner 如何把一个 parent relation 改写成一组 child relation，
又如何保持 parent-level SQL 语义不丢失？
```
这个问题不是只属于分区表。 传统 inheritance、partitioned table、flattened `UNION ALL` 都会进入 appendrel 世界。 它们的共同点是：
```text
上层 query 仍然引用 parent 输出列；
下层 path search 必须在 child relation 上发生。
```
本节只追 appendrel expansion 和 Var translation。 不展开 partition pruning 的完整算法。 不展开 executor tuple routing。 不展开 partition-wise join 的 joinrel 搜索。 这些主题都使用 appendrel 结果，但不是本节的主矛盾。 本节的主线是：
```text
RTE.inh
  -> add_other_rels_to_query()
  -> expand_inherited_rtentry()
  -> expand_single_inheritance_child()
  -> make_append_rel_info()
  -> build_simple_rel(child)
  -> set_append_rel_size()
  -> set_append_rel_pathlist()
  -> set_plan_references()
```
读源码时始终抓住三个对象：
```text
child RangeTblEntry
child RelOptInfo
AppendRelInfo
```
三者缺一不可。 只有 child RTE，没有 child RelOptInfo，后续不能生成 path。 只有 child RelOptInfo，没有 `AppendRelInfo`，parent qual 和 targetlist 无法安全翻译。 只有 `AppendRelInfo`，没有 child RTE，最终 plan 无法引用实际 relation。
## 2. 核心矛盾与一句话运行模型
一个普通查询看起来很简单：
```sql
SELECT a, b
FROM parent
WHERE a = 10;
```
如果 `parent` 是普通表，planner 只需要一个 base `RelOptInfo`。 但如果 `parent` 是 inheritance parent 或 partitioned table，真实扫描对象可能是：
```text
parent 本身
child_1
child_2
child_3
...
```
传统继承里，parent 也可以有自己的 heap 数据。 分区表里，partitioned parent 本身没有 table AM 数据要扫描。 child 的列顺序也不一定和 parent 一样。 传统继承允许 child 有额外列。 历史 DDL 可能造成 parent 和 child 的 inherited column attno 不一致。 dropped column 还会留下 tuple descriptor 洞。
如果 planner 简单复用 parent `Var`：
```text
Var(varno = parentRTI, varattno = 1)
```
那 child path 在执行时会读错 relation 或读错列。 如果 planner 完全把 parent query 重写成多个独立 query：
```text
SELECT child_1.a, child_1.b ...
UNION ALL
SELECT child_2.a, child_2.b ...
```
又会丢掉很多上层 planner 状态。 例如 equivalence class、rowmark、lateral dependency、target relation、permission、RLS、partition pruning、constraint exclusion 和 append path 成本都要重新协调。 PostgreSQL 的折中是 appendrel abstraction：
```text
parent relation 仍保留为一个 logical append parent；
每个 child relation 单独建 RTE 和 otherrel RelOptInfo；
AppendRelInfo 保存 parent column 到 child column 的翻译；
allpaths 用 child path 拼出 parent append path；
setrefs 把最终需要的 AppendRelInfo 放进 PlannedStmt。
```
这个 abstraction 的关键不是 `Append` plan node。 `Append` plan node 只是后续 path/createplan 阶段可能生成的物理计划节点。 appendrel 更早出现。 它是 planner 内部对“一个逻辑 relation 由多个 member relation 组成”的表达。 因此本节的判断模型是：
```text
appendrel = parent RTE + child RTEs + child RelOptInfos + AppendRelInfos + 后续 path 聚合规则
```
## 3. 核心文件分工与阅读顺序
| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/optimizer/plan/planmain.c` | `query_planner()` 为什么等 restriction、EC、lateral 等状态就绪后才调用 `add_other_rels_to_query()`。 |
| 2 | `src/backend/optimizer/plan/initsplan.c` | `add_base_rels_to_query()` 先建 parent baserel；`add_other_rels_to_query()` 再扩展 appendrel children。 |
| 3 | `src/backend/optimizer/prep/prepjointree.c` | `preprocess_relation_rtes()` 根据 `relhassubclass` 预处理 `rte->inh`，以及 `UNION ALL` flattening 会设置 subquery `inh`。 |
| 4 | `src/backend/optimizer/util/plancat.c` | `get_relation_info()` 和 `set_relation_partition_info()` 把 partition metadata 放入 parent `RelOptInfo`。 |
| 5 | `src/backend/optimizer/util/inherit.c` | 本节主入口：`expand_inherited_rtentry()`、`expand_partitioned_rtentry()`、`expand_single_inheritance_child()`、`apply_child_basequals()`。 |
| 6 | `src/backend/optimizer/util/appendinfo.c` | `make_append_rel_info()`、`make_inh_translation_list()`、`adjust_appendrel_attrs()`、`adjust_inherited_attnums_multilevel()`。 |
| 7 | `src/backend/optimizer/util/relnode.c` | `setup_simple_rel_arrays()`、`expand_planner_arrays()`、`build_simple_rel()` 和 child rel 的 lifecycle。 |
| 8 | `src/backend/optimizer/path/allpaths.c` | `set_append_rel_size()`、`set_append_rel_pathlist()`、`add_paths_to_append_rel()` 如何消费 children。 |
| 9 | `src/backend/optimizer/prep/prepunion.c` | flattened `UNION ALL` 也使用 appendrel，但 child RTE 和 `AppendRelInfo` 较早创建。 |
| 10 | `src/backend/optimizer/plan/setrefs.c` | `set_plan_references()` 把最终 `AppendRelInfo` 放入 `PlannerGlobal.appendRelations`。 |
| 11 | `src/include/nodes/pathnodes.h` | `PlannerInfo`、`RelOptInfo`、`AppendRelInfo`、`RowIdentityVarInfo` 的字段语义。 |
| 12 | `src/include/optimizer/prep.h` | prep 阶段公开入口，特别是 `get_plan_rowmark()` 与 set operation planning 的边界。 |
| 13 | `src/include/optimizer/appendinfo.h` / `inherit.h` / `planmain.h` | 入口函数声明和跨文件 contract。 |
推荐阅读顺序不是从 `AppendPath` 开始。 先读 `AppendRelInfo` 的注释。 再读 `expand_inherited_rtentry()`。 然后读 `make_inh_translation_list()`。 最后读 `set_append_rel_size()` 和 `set_append_rel_pathlist()`。 这样你会先看清“映射如何建立”，再看“路径如何消费映射”。 两个名字要区分：
| 名字 | 阶段 | 语义 |
| --- | --- | --- |
| appendrel | planner 内部 abstraction | 一个 parent RTE 对应多个 child RTE，靠 `AppendRelInfo` 翻译。 |
| `AppendPath` / `Append` | path / plan 节点 | 把多个 child path / subplan 串接输出。 |
`AppendRelInfo` 可以存在于生成 `AppendPath` 之前。 一个 child 也可能被 constraint exclusion 标成 dummy，最终不进入 `AppendPath.subpaths`。 所以诊断时不要把“没有出现在 `Append` 子计划中”直接等同于“没有 appendrel expansion”。
## 4. 可复现运行现象
先用可见现象建立边界。
### 4.1 传统继承会扫描 parent 本身
示例：
```sql
CREATE TABLE ih_parent(a int, b text);
CREATE TABLE ih_child(a int, b text, extra text) INHERITS (ih_parent);
INSERT INTO ih_parent VALUES (1, 'p');
INSERT INTO ih_child(a, b, extra) VALUES (2, 'c', 'x');
EXPLAIN SELECT * FROM ih_parent;
```
典型公开影子是一个 `Append`。 它包含 parent scan 和 child scan。 源码解释： `expand_inherited_rtentry()` 对传统继承调用 `find_all_inheritors()`。 返回列表第一个 OID 是 parent 自己。 循环里 parent 也会经过 `expand_single_inheritance_child()`。 此时 `oldrelation == newrelation`，`make_inh_translation_list()` 为 parent member 生成同 relid 的 simple Var。
这就是为什么传统 inheritance parent 既是 logical parent，也是 append member。
### 4.2 分区表不会为 partitioned parent 建扫描 RTE
示例：
```sql
CREATE TABLE p_parent(a int, b text) PARTITION BY RANGE (a);
CREATE TABLE p1 PARTITION OF p_parent FOR VALUES FROM (0) TO (100);
CREATE TABLE p2 PARTITION OF p_parent FOR VALUES FROM (100) TO (200);
EXPLAIN SELECT * FROM p_parent;
```
公开影子仍可能是 `Append`。 但 partitioned parent 自身不会作为 child scan 出现。 源码解释： `expand_partitioned_rtentry()` 只遍历 `PartitionDesc.oids`。 注释明确说不需要为 partitioned table 本身创建 child RTE，因为它不会被扫描。
### 4.3 planner-time pruning 会阻止 child RTE 创建
示例：
```sql
EXPLAIN SELECT * FROM p_parent WHERE a = 42;
```
如果 range bound 足够明确，通常只有 `p1` 进入计划。 这个 child 的缺失发生得很早。 `expand_partitioned_rtentry()` 先调用：
```text
relinfo->live_parts = prune_append_rel_partitions(relinfo)
```
然后只为 `live_parts` 中的 partition index 调 `expand_single_inheritance_child()`。 被 planner-time pruning 裁掉的 partition 没有 child RTE、没有 child `RelOptInfo`，也没有对应的 `AppendRelInfo`。
### 4.4 constraint exclusion 和 pruning 的公开影子相似
传统继承 child 也可以靠 check constraint 被排除。 示例：
```sql
CREATE TABLE ih_2026(a int CHECK (a >= 2026 AND a < 2027)) INHERITS (ih_parent);
EXPLAIN SELECT * FROM ih_parent WHERE a = 2030;
```
公开影子可能看不到 `ih_2026`。 但源码路径不同。 传统继承先创建 child RTE 和 child `RelOptInfo`。 之后 `set_append_rel_size()` 调 `relation_excluded_by_constraints()`，把 child 标成 dummy。 分区表 planner-time pruning 则可能根本不创建 child RTE。
### 4.5 child 列顺序不同但查询仍能正确
传统继承可以在历史 DDL 后出现 parent/child inherited column attno 不一致。 公开影子是：
```sql
SELECT parent_col FROM parent;
```
仍能在 child 上读到语义相同的列。 源码解释： `make_inh_translation_list()` 不按 attno 盲目映射。 它按 parent attribute name 找 child attribute，并检查 type 和 collation。 `translated_vars` 的第 N 个元素才表示 parent 第 N 个用户列在 child 中的表达式。
### 4.6 `UNION ALL` appendrel 的 translation 可以是表达式
示例：
```sql
EXPLAIN
SELECT * FROM (
  SELECT a, b FROM t1
  UNION ALL
  SELECT x + 1, y FROM t2
) u;
```
`UNION ALL` flattening 也会产生 appendrel。 但 `translated_vars` 不一定是 simple Var。 `pathnodes.h` 注释说 inheritance 情况下 list elements 总是 simple Vars，`UNION ALL` 情况下可以是任意表达式。 这解释了为什么 `adjust_appendrel_attrs()` 对 whole-row Var、nullingrels、returningtype 有额外限制。
## 5. 关键数据结构与状态边界
本节只讲影响主问题的字段组合。 不要把 raw field 当语义。 字段必须结合创建时机、owner、内存上下文和后续消费者解释。
### 5.1 `RangeTblEntry.inh`
`rte->inh` 是 parser/rewrite/prep 传给 planner 的触发标记。 它表示“这个 RTE 应该按 inheritance/appendrel 规则处理”。 它不表示 child 已经存在。 它也不表示最终一定有 `Append` plan node。 `addRangeTableEntry()` 创建 relation RTE 时会保存 `inh` 参数。 `preprocess_relation_rtes()` 会对 relation RTE 打开 relcache。
如果 `rte->inh` 为 true，它会用 `relation->rd_rel->relhassubclass` 更新该标志。 这里有一个历史边界。 如果 `relhassubclass` 仍为 true 但 child 已经不存在，后面不能简单把 `inh` 清掉。 源码注释说 planner 后续 decisions 已经依赖 `rte->inh`，所以要按完整 inheritance 路径处理。 对 partitioned table，`rte->relkind` 会是 `RELKIND_PARTITIONED_TABLE`。
对 flattened `UNION ALL`，`RTE_SUBQUERY` 也可能被设置 `inh`。 这个用法是对原始含义的扩展。
### 5.2 `PlannerInfo` 的 appendrel 相关字段
`PlannerInfo` 是一次 query level 的 planner-local 状态。 相关字段包括：
| 字段 | 语义 |
| --- | --- |
| `simple_rel_array` | RT index 到 `RelOptInfo *` 的数组。parent baserel 和 child otherrel 都放这里。 |
| `simple_rte_array` | RT index 到 `RangeTblEntry *` 的数组，比 `rt_fetch()` 少一些 indirection。 |
| `append_rel_list` | 当前 query level 的 `AppendRelInfo` flat list。 |
| `append_rel_array` | 以 child RT index 查 `AppendRelInfo *` 的数组。 |
| `rowMarks` | parent 和 child 的 `PlanRowMark` 列表。 |
| `all_result_relids` | UPDATE/DELETE/MERGE 中所有 target relation RT index，包含中间 partitioned rel。 |
| `leaf_result_relids` | 真正需要作为 leaf result relation 的 RT index。 |
| `part_schemes` | planner-local partition scheme cache。 |
`setup_simple_rel_arrays()` 在主 planning 早期按当前 rangetable 分配数组。 当 inheritance expansion 新增 child RTE 时，`expand_planner_arrays()` 会扩容这些数组。 注意：
```text
append_rel_array 是按 child_relid 索引；
append_rel_list 是按 parent-child pair 顺序扫描；
两者是同一批 AppendRelInfo 的不同访问方式。
```
### 5.3 `RelOptInfo` 中 parent 与 child 的区别
parent baserel：
```text
reloptkind = RELOPT_BASEREL
relid      = parent RT index
rte->inh   = true
```
child otherrel：
```text
reloptkind = RELOPT_OTHER_MEMBER_REL
relid      = child RT index
parent     = immediate parent RelOptInfo
top_parent = topmost parent RelOptInfo
```
`build_simple_rel(root, childRTindex, parent)` 通过 `parent != NULL` 判断 child relation。 child rel 会继承 parent 的部分 planner 状态。 例如 `nulling_relids`、lateral 信息和 top parent relids。 这不是简单复制。 这些字段回答的是“这个 child 在 join tree 中处在 parent 的哪个语义位置”。
对 partitioned parent，`RelOptInfo` 还会有：
| 字段 | 语义 |
| --- | --- |
| `part_scheme` | partition key 的 planner-local 表达。 |
| `boundinfo` | relcache `PartitionDesc` 中的 canonical boundinfo。 |
| `nparts` | 直接 partitions 数量。 |
| `part_rels` | 按 partition index 保存 live child `RelOptInfo *`，被裁剪的是 NULL。 |
| `live_parts` | planner-time pruning 后仍需要展开的 partition indexes。 |
| `all_partrels` | 当前 partitioned rel 下展开出的 child relids 集合。 |
`part_rels` 是 partition index 空间。 `append_rel_array` 是 RT index 空间。 二者不能混用。
### 5.4 `AppendRelInfo`
`AppendRelInfo` 是本节最核心的数据结构。 它描述一个 immediate parent RTE 和一个 child RTE 的关系。 关键字段：
| 字段 | 语义 |
| --- | --- |
| `parent_relid` | append parent 的 RT index。 |
| `child_relid` | append child 的 RT index。 |
| `parent_reltype` | inheritance parent composite type OID，用于 whole-row Var。 |
| `child_reltype` | inheritance child composite type OID，用于 whole-row Var。 |
| `translated_vars` | parent 第 N 个用户列在 child 上对应的 Var 或 expression。 |
| `num_child_cols` | `parent_colnos` 数组长度。 |
| `parent_colnos` | child 第 N 个用户列反向对应 parent 的第几个用户列，0 表示没有。 |
| `parent_reloid` | inheritance parent OID，用于 dropped column 或错误消息。 |
不变量：
```text
同一个 child_relid 最多只有一个 AppendRelInfo；
同一个 parent_relid 可以有多个 AppendRelInfo；
partition hierarchy 中每一层都是 immediate parent -> immediate child；
传统 inheritance 对 unpartitioned parent 会在 RTE expansion 时 flatten。
```
`append_rel_list` 注释还强调：
```text
对于 partitioned table 的 AppendRelInfo，
按 PartitionDesc 中更早出现的 partition 顺序放入 append_rel_list。
```
这个顺序对后续 partition-wise 逻辑和可解释性都有意义。
### 5.5 `translated_vars`
`translated_vars` 是 parent-to-child Var translation 的正向映射。 它的第 N 个元素对应 parent 的第 N 个用户列。 系统列不在里面。 whole-row Var 也不在里面。 dropped parent column 对应 `NULL`。 继承表场景下，元素总是 simple `Var`：
```text
parent attno N
  -> child Var(varno = childRTI, varattno = child inherited attno)
```
`UNION ALL` appendrel 场景下，元素可以是任意表达式：
```text
parent output column N
  -> child expression
```
这就是为什么 `adjust_appendrel_attrs()` 在处理 nullingrels 和 returningtype 时要检查翻译结果是不是 `Var`。
### 5.6 `parent_colnos`
`parent_colnos` 是 reverse translation。 它按 child column number 索引。 值为 1-based parent column number。 值为 0 表示 child column 没有对应 parent column。 传统继承 child 可以有额外列。 dropped child column 也会得到 0。 `expand_single_inheritance_child()` 构造 child alias/eref 时会用它。
如果 child column 对应 parent column，就复用 parent 查询中为该列指定的别名。 如果是 child 自己的新列，就使用 child 真实列名。 这影响 `EXPLAIN` 和 ruleutils 输出的列名。
### 5.7 `PartitionDesc` 与 `live_parts`
partitioned table expansion 不直接扫 `pg_inherits`。 它通过 `PartitionDirectoryLookup(root->glob->partition_directory, parentrel)` 得到 `PartitionDesc`。 `PartitionDesc.oids[i]` 是 partition index 到 child relation OID 的映射。 `relinfo->live_parts` 是 `Bitmapset`。
其中成员是 partition index，不是 RT index。 流程是：
```text
PartitionDesc.oids
  -> prune_append_rel_partitions()
  -> live_parts partition indexes
  -> try_table_open(childOID)
  -> child RT index
  -> AppendRelInfo child_relid
```
RT index 是新增 RTE 后才产生的。 因此 planner-time pruning 必须发生在 child RTE 创建之前。
### 5.8 `PlanRowMark`
`FOR UPDATE` / `FOR SHARE` 查询需要 rowmark。 parent 被 rowmark 后，`expand_inherited_rtentry()` 会把 parent `PlanRowMark.isParent` 设为 true。 之后 `expand_single_inheritance_child()` 为每个 child 建 child rowmark。 child rowmark 字段含义：
| 字段 | 语义 |
| --- | --- |
| `rti` | child RT index。 |
| `prti` | top parent RT index。 |
| `rowmarkId` | 与 top parent 共用 rowmark id。 |
| `markType` | 根据 child RTE relkind 重新选择。 |
| `isParent` | partitioned child table 为 true，executor 忽略它作为扫描目标。 |
top parent 的 `allMarkTypes` 会累积所有 descendant 的 rowmark type。 如果 descendant 需要额外 junk column，`expand_inherited_rtentry()` 会补 `ctid`、whole-row 或 `tableoid` 到 processed targetlist。 这是 rowmark 和 appendrel expansion 之间最容易被漏掉的耦合。
### 5.9 `RTEPermissionInfo`
父 RTE 有 permission info。 child RTE 通常不做独立权限检查。 `expand_single_inheritance_child()` 会设置：
```text
childrte->perminfoindex = 0
```
child `RelOptInfo.userid` 对普通 inheritance/partition otherrel 使用 parent 的 userid。 这样 inherited query 的权限语义是针对 parent 的。 `translated_vars` 还会用于列级权限和 updated columns 的映射。 `translate_col_privs()` 把 parent bitmapset 中的列号翻译成 child column number。
whole-row 权限不会简单变成 child whole-row。 它会展开成所有 inherited columns。 这是为了避免 child 的额外列导致权限要求过严。
### 5.10 `AppendPath` 与 `AppendRelInfo` 的边界
`AppendRelInfo` 描述语义映射。 `AppendPath` 描述执行路径组合。 `set_append_rel_pathlist()` 会遍历 `root->append_rel_list`，找出当前 parent 的 child rel。 它对每个非 dummy child 调 `set_rel_pathlist()`。 之后 `add_paths_to_append_rel()` 收集 child 的 cheapest path、partial path、pathkeys 和 parameterization。
如果最终只有一个 live child，物理计划不一定保留明显的 `Append` 节点。 但 appendrel expansion 已经发生。 诊断时要分清 planner internal state 和 final plan shape。
## 6. 主流程源码 walkthrough
本节主流程以一条 SELECT 查询为轴。 UPDATE/DELETE/MERGE 的额外 row identity 会在同一条链上补充说明。
### 6.1 从 relation RTE 到 parent baserel
parse analysis 创建 `RTE_RELATION`。 `addRangeTableEntry()` 记录：
```text
rte->relid
rte->inh
rte->relkind
rte->rellockmode
```
锁模式也在这里确定。 普通 SELECT 是 `AccessShareLock`。 `FOR UPDATE/SHARE` 会让 `isLockedRefname()` 选择 `RowShareLock`。 进入 planner 后，`preprocess_relation_rtes()` 扫 rangetable。 它打开 relation，但不重新加锁，因为 relation 已经由 parser/rewrite 锁住。
如果 `rte->inh` 为 true，它检查 `relation->rd_rel->relhassubclass`。 无 child 时可以清掉 `inh`。 可能存在 false positive，所以后面仍要支持“标记有 child，但实际列表只有 parent”的路径。
### 6.2 `query_planner()` 延迟展开 appendrel
`query_planner()` 先调用：
```text
add_base_rels_to_query()
```
它只沿 jointree 创建 base `RelOptInfo`。 appendrel member relations 此时还不创建。 随后 planner 做一批 parent-level 准备：
```text
build_base_rel_tlists()
find_lateral_references()
deconstruct_jointree()
generate_base_implied_equalities()
create_lateral_join_info()
extract_restriction_or_clauses()
```
最后才调用：
```text
add_other_rels_to_query()
```
源码注释给出原因：
```text
delay this to the end so that we have as much information as possible
```
这句话是理解本节的关键。 child expansion 必须看到 parent 的 restriction clauses。 否则 partition pruning 和 child qual translation 会太早。 child 也需要 parent 的 lateral 信息和 join tree 语义位置。 否则 parameterized path 和 join legality 可能不正确。
### 6.3 `add_other_rels_to_query()`
`add_other_rels_to_query()` 遍历 `simple_rel_array`。 它只处理 `RELOPT_BASEREL`。 已经创建的 child otherrel 会被跳过。 伪流程：
```text
for rti in simple_rel_array:
  rel = simple_rel_array[rti]
  rte = simple_rte_array[rti]
  if rel is NULL:
    continue
  if rel->reloptkind != RELOPT_BASEREL:
    continue
  if rte->inh:
    expand_inherited_rtentry(root, rel, rte, rti)
```
这保证 appendrel parent 是 base relation。 child relation 会在 expansion 过程中变成 other member relation。
### 6.4 `expand_inherited_rtentry()` 的总分支
`expand_inherited_rtentry()` 首先区分两类 RTE：
```text
RTE_SUBQUERY
  -> expand_appendrel_subquery()

RTE_RELATION
  -> inheritance / partition expansion
```
`RTE_SUBQUERY` 代表 flattened `UNION ALL`。 对应 child RTE 和 `AppendRelInfo` 已经在 pull-up / set operation preprocessing 中生成。 这里只需要为 child subquery 建 `RelOptInfo`。 本节主线看 `RTE_RELATION`。 函数打开 parent relation：
```text
oldrelation = table_open(parentOID, NoLock)
lockmode = rte->rellockmode
```
parent 已经有锁。 child 是第一次进入 query pipeline，所以后面需要拿同等级锁。 如果 parent 有 rowmark，函数先保存原 `isParent` 和 `allMarkTypes`。 然后把 parent rowmark 标为 parent：
```text
oldrc->isParent = true
```
接下来按 parent relkind 分支：
```text
RELKIND_PARTITIONED_TABLE
  -> expand_partitioned_rtentry()

ordinary table
  -> traditional inheritance path
```
源码注释强调 partitioned table 不允许同时有传统 inheritance children。 所以两条路径互斥。
### 6.5 传统 inheritance 展开
传统路径调用：
```text
inhOIDs = find_all_inheritors(parentOID, lockmode, NULL)
```
这个函数会扫描 inheritance set，并获取 child locks。 返回列表应该包含 parent 自己，并且 parent OID 是第一个。 随后：
```text
expand_planner_arrays(root, list_length(inhOIDs))
```
因为每个 inheritance member 都可能新增一个 RTE。 对每个 OID：
```text
if childOID != parentOID:
  newrelation = table_open(childOID, NoLock)
else:
  newrelation = oldrelation

if child is temp table of another backend:
  table_close(newrelation, lockmode)
  continue

expand_single_inheritance_child()
build_simple_rel(childRTindex, rel)
table_close(newrelation, NoLock)
```
parent 自己也会经过 `expand_single_inheritance_child()`。 这就是传统 inheritance 的特殊性。 原始 parent RTE 代表 whole inheritance set。 新建的 parent-member RTE 代表 parent heap 本身。 二者是不同 RT index。
### 6.6 partitioned table 展开
partitioned path 调用：
```text
partdesc = PartitionDirectoryLookup(root->glob->partition_directory, parentrel)
```
`PartitionDirectory` 是 planner invocation 里的查询级 cache。 它由 `plancat.c` 的 partition info 路径创建，planner shutdown 时销毁。 如果 `partdesc->nparts == 0`，函数直接返回。 然后执行 planner-time pruning：
```text
relinfo->live_parts = prune_append_rel_partitions(relinfo)
```
`live_parts` 是 `PartitionDesc` index。 然后只按 live partition 数扩容 planner arrays。 `relinfo->part_rels` 被分配为 `nparts` 大小。 被 pruning 裁掉的槽位保留 NULL。 循环每个 live partition index：
```text
childOID = partdesc->oids[i]
childrel = try_table_open(childOID, lockmode)
```
这里使用 `try_table_open()` 是异常路径的一部分。 如果 partition 最近被 detached 并 drop，打开可能失败。 源码选择把它当成已经被 pruned：
```text
relinfo->live_parts = bms_del_member(relinfo->live_parts, i)
continue
```
如果发现其它 backend 的 temp partition，则 `elog(ERROR)`。 正常情况下继续：
```text
expand_single_inheritance_child()
childrelinfo = build_simple_rel(root, childRTindex, relinfo)
relinfo->part_rels[i] = childrelinfo
relinfo->all_partrels = bms_add_members(...)
```
如果 child 自己也是 partitioned table，则递归。 递归前会把 updated columns 从 parent 映射到 child：
```text
child_updatedCols =
  translate_col_privs(parent_updatedCols, appinfo->translated_vars)
```
这说明多层 partition hierarchy 不是一次 flatten 到 leaf。 每一层都有自己的 immediate parent-child `AppendRelInfo`。
### 6.7 `expand_single_inheritance_child()`
这个函数同时创建三类状态：
```text
child RangeTblEntry
AppendRelInfo
PlanRowMark / result relation bookkeeping
```
它先复制 parent RTE 的大部分 scalar fields：
```text
childrte = makeNode(RangeTblEntry)
memcpy(childrte, parentrte, sizeof(RangeTblEntry))
```
然后替换 child 相关字段：
```text
childrte->relid = childOID
childrte->relkind = childrel->rd_rel->relkind
```
如果 child 本身是 partitioned table：
```text
childrte->inh = true
```
否则：
```text
childrte->inh = false
```
child RTE 的 `securityQuals` 被清空。 源码注释说明，这是为了让 inherited RLS 像 regular permissions checks 一样工作。 parent securityQuals 后续会和其它 base restriction clauses 一起传播到 child。 child RTE 不单独做 permission checking：
```text
childrte->perminfoindex = 0
```
接着把 child RTE 加入 `parse->rtable`。 新的 RT index 就是当前 rtable 长度。 这一步之后，child 才有可供 plan 引用的 relid。
### 6.8 `make_append_rel_info()`
`expand_single_inheritance_child()` 调用：
```text
appinfo = make_append_rel_info(parentrel, childrel,
                               parentRTindex, childRTindex)
```
`make_append_rel_info()` 做四件事：
```text
appinfo->parent_relid = parentRTindex
appinfo->child_relid = childRTindex
appinfo->parent_reltype = parentrel->rd_rel->reltype
appinfo->child_reltype = childrel->rd_rel->reltype
make_inh_translation_list(parentrel, childrel, childRTindex, appinfo)
appinfo->parent_reloid = RelationGetRelid(parentrel)
```
然后 caller 把它接入 planner：
```text
root->append_rel_list = lappend(root->append_rel_list, appinfo)
root->append_rel_array[childRTindex] = appinfo
```
list 服务 parent 扫描。 array 服务 child relid 快速查找。
### 6.9 `make_inh_translation_list()`
这个函数构造 `translated_vars` 和 `parent_colnos`。 它先读 parent 和 child tuple descriptor。 然后初始化 reverse translation：
```text
appinfo->num_child_cols = newnatts
appinfo->parent_colnos = palloc0(newnatts * sizeof(AttrNumber))
```
主循环按 parent attributes 走。 如果 parent attr 已 dropped：
```text
vars = lappend(vars, NULL)
continue
```
如果 parent relation 和 child relation 是同一个 relation：
```text
vars = lappend(vars, makeVar(childRTindex, old_attno + 1, ...))
pcolnos[old_attno] = old_attno + 1
continue
```
这就是传统 inheritance parent-member 的映射。 如果不是同一个 relation，它先尝试 child tuple descriptor 中“上一个匹配列之后的列”。 简单 inheritance 场景中这通常命中。 如果不命中，就用 syscache 按 attribute name 查：
```text
SearchSysCacheAttName(new_relid, attname)
```
找不到会 `elog(ERROR)`。 找到后检查 type 和 typmod。 不匹配会报 `ERRCODE_INVALID_COLUMN_DEFINITION`。 再检查 collation。 最终生成 child Var：
```text
makeVar(childRTindex,
        new_attno + 1,
        atttypid,
        atttypmod,
        attcollation,
        0)
```
并设置 reverse mapping：
```text
pcolnos[new_attno] = old_attno + 1
```
最后：
```text
appinfo->translated_vars = vars
```
这里有三个重要不变量。 第一，parent dropped column 不能被翻译成 child column。 第二，parent/child inherited column 必须按 name、type、typmod、collation 匹配。 第三，translation 中的 Var 使用 child RT index。 因此后续只要把 parent Var 替换为 `translated_vars[N]`，表达式就落到了 child relation 上。
### 6.10 child alias 和 `parent_colnos`
`expand_single_inheritance_child()` 构造 child `alias` 和 `eref`。 它遍历 child tuple descriptor。 如果 child column dropped，列名用空字符串。 如果 `appinfo->parent_colnos[cattno]` 指向 parent column，则复用 parent RTE 的 query-assigned column name。
否则用 child 真实列名。 这样 ruleutils / EXPLAIN 在展示 child-table columns 时不至于失去 parent query 的别名语义。 这一步不改变执行语义。 但它影响可观察 SQL 表达。
### 6.11 child RTE 和 appinfo 放入数组
函数要求 caller 已经扩容数组：
```text
Assert(childRTindex < root->simple_rel_array_size)
```
然后设置：
```text
root->simple_rte_array[childRTindex] = childrte
root->append_rel_array[childRTindex] = appinfo
```
此时 child `RelOptInfo` 还没有创建。 随后 caller 调：
```text
build_simple_rel(root, childRTindex, parentRelOptInfo)
```
这个顺序很重要。 `build_simple_rel()` 需要从 `append_rel_array[childRTindex]` 取 `AppendRelInfo`。 如果先建 child rel 再填 appinfo，child base quals 无法翻译。
### 6.12 `build_simple_rel()` 创建 child otherrel
`build_simple_rel()` 看到 `parent != NULL`，设置：
```text
rel->reloptkind = RELOPT_OTHER_MEMBER_REL
rel->parent = parent
rel->top_parent = parent->top_parent ? parent->top_parent : parent
```
它也传播 parent 的 lateral 和 nulling 信息。 对 relation RTE，它调用：
```text
get_relation_info(root, rte->relid, rte->inh, rel)
```
如果 child 也是 partitioned table，`rte->inh` 为 true。 `get_relation_info()` 会为它设置 partition info。 随后 child 的核心一步是：
```text
apply_child_basequals(root, parent, rel, rte, appinfo)
```
这会把 parent `baserestrictinfo` 翻译到 child。 如果翻译后某个 qual 变成 constant false 或 NULL，则立即：
```text
mark_dummy_rel(rel)
```
源码注释强调必须立即标 dummy。 因为递归 `expand_partitioned_rtentry()` 时需要正确看到 child 是否已经无扫描意义。
### 6.13 `adjust_appendrel_attrs()` 翻译表达式
`apply_child_basequals()` 对每个 parent `RestrictInfo` 调：
```text
adjust_appendrel_attrs(root, rinfo->clause, 1, &appinfo)
```
`adjust_appendrel_attrs()` 是通用表达式 mutator。 看到 `Var` 时，如果 `var->varno` 等于 `appinfo->parent_relid`，就替换成 child 表达式。 普通用户列：
```text
newnode = copyObject(list_nth(appinfo->translated_vars,
                              var->varattno - 1))
```
system columns 不需要特殊翻译。 因为 system attribute numbers 在所有 heap relation 中一致。 whole-row Var 走特殊路径。 如果 parent/child 都有 named rowtype，并且 rowtype 不同，构造：
```text
ConvertRowtypeExpr(child whole-row Var -> parent rowtype)
```
如果是 `UNION ALL` 这种无 named rowtype appendrel，则构造 `RowExpr`。 这解释了为什么 whole-row reference 是 appendrel translation 中的复杂分支。 它不是简单把 `varno` 改成 child。
### 6.14 child base quals 的安全层级
`apply_child_basequals()` 翻译每个 parent `RestrictInfo` 后，会调用：
```text
eval_const_expressions(root, childqual)
```
如果变成 constant true，就丢弃。 如果变成 constant false 或 NULL，就返回 false。 如果得到 AND clause，则拆成隐式 AND 列表。 每个 child qual 重新构造 `RestrictInfo`。 同时保留 parent qual 的安全属性：
```text
is_pushed_down
has_clone
is_clone
security_level
```
child RTE 自己的 `securityQuals` 通常为空。 只有 `UNION ALL` appendrel 子查询可能有 child-specific securityQuals。 这种情况下函数会把它们拉入 child `baserestrictinfo`。
### 6.15 `set_append_rel_size()`
path 阶段进入 `set_rel_size()` 时，如果 parent RTE 是 appendrel，就走：
```text
set_append_rel_size(root, rel, rti, rte)
```
它遍历 `root->append_rel_list`，只处理 `appinfo->parent_relid == parentRTindex` 的 child。 child `RelOptInfo` 已经在 `add_other_rels_to_query()` 阶段创建。 这一步做几件事：
```text
relation_excluded_by_constraints()
translate parent joininfo
translate parent reltarget
add child EC entries
set_rel_consider_parallel()
set_rel_size(child)
accumulate parent rows/tuples/width
```
注意 `baserestrictinfo` 不在这里重新翻译。 它已经在 `build_simple_rel()` 里通过 `apply_child_basequals()` 完成。 `set_append_rel_size()` 翻译的是 join quals 和 targetlist。 child reltarget 可能包含任意表达式。 源码注释提醒，appendrel child 的 targetlist 不一定只有 Var 和 PlaceHolderVar。
这是因为 `UNION ALL` translation 可能产生表达式。
### 6.16 parent size 如何聚合
appendrel parent 的 size 不是 parent heap 的 size。 `set_append_rel_size()` 用 live child 的估算聚合：
```text
parent_tuples += childrel->tuples
parent_rows   += childrel->rows
parent_size   += childrel->reltarget->width * childrel->rows
```
width 按 child rows 加权平均。 每个 parent attribute 的 width 也按 child 表达式估算。 如果 child expression 不是 simple Var，或没有 child attr width，就 fallback 到 datatype average width。 如果没有 live children，parent appendrel 被标成 dummy。 这说明 appendrel parent 的 rows/width 是 child search 的派生结果。
不是 catalog 中 parent 自己的统计信息。
### 6.17 `set_append_rel_pathlist()`
下一阶段生成 path：
```text
set_append_rel_pathlist(root, rel, rti, rte)
```
它同样遍历 `append_rel_list`。 对每个 child：
```text
set_rel_pathlist(root, childrel, childRTindex, childRTE)
```
如果 child 是 dummy，跳过。 非 dummy child 放入 `live_childrels`。 最后：
```text
add_paths_to_append_rel(root, rel, live_childrels)
```
`add_paths_to_append_rel()` 收集：
```text
unparameterized cheapest paths
startup paths
partial paths
parallel append inputs
pathkeys
required_outer sets
```
然后为 parent appendrel 添加 `AppendPath`、`MergeAppendPath` 或相关 partial append paths。
### 6.18 `set_plan_references()` 的最终边界
planning 后期，`set_plan_references()` 会处理 `root->append_rel_list`。 源码注释说：
```text
append_rel_list holds AppendRelInfos for all append rels in this query level
```
它会调整 RT indexes，并把 surviving `AppendRelInfo` 加到：
```text
root->glob->appendRelations
```
最后 `planner.c` 把它写入：
```text
PlannedStmt.appendRelations
```
因此 `AppendRelInfo` 不是只在 expansion 函数里短暂存在。 它会成为 planned statement 的一部分，供后续阶段理解 inheritance/partition target relation 和 row identity 等映射。
## 7. 生命周期 / ownership / cleanup
### 7.1 谁创建
`RangeTblEntry` parent 由 parse/rewrite 阶段创建。 child `RangeTblEntry` 由 `expand_single_inheritance_child()` 在 planner 中创建。 child `RelOptInfo` 由 `build_simple_rel()` 创建。 `AppendRelInfo` 由 `make_append_rel_info()` 创建。
`translated_vars` 和 `parent_colnos` 在 `make_inh_translation_list()` 中创建。 `PartitionDirectory` 在 planner global 中按需创建。
### 7.2 谁持有
这些对象大多属于 planner 当前 memory context。 `root->parse->rtable` 持有新增 child RTE。 `root->simple_rte_array` 持有 child RTE 快速索引。 `root->simple_rel_array` 持有 child `RelOptInfo`。 `root->append_rel_list` 和 `root->append_rel_array` 持有 `AppendRelInfo`。
最终需要进入 plan 的 `AppendRelInfo` 会进入 `PlannerGlobal.appendRelations`，再进入 `PlannedStmt.appendRelations`。 `Relation` relcache 指针不是这些对象的 owner。 它们只在 expansion 过程中提供 tuple descriptor、relkind、rowtype、partition descriptor、权限和统计信息。
### 7.3 谁释放
planner-local 对象由 planner memory context 统一释放。 `PartitionDirectory` 有显式销毁路径：
```text
DestroyPartitionDirectory(glob->partition_directory)
```
它在 planner shutdown 前被调用。 打开的 relation 用 `table_close()` 关闭。 但 child relation locks 会保留到事务结束或 portal 生命周期所需边界。 传统继承中：
```text
table_close(newrelation, NoLock)
```
表示关闭 relcache reference，但不释放已经持有的 lock。 partitioned path 中 child 也在创建完 child state 后关闭 relation。
### 7.4 ERROR / abort 谁兜底
planner 中途 ERROR 时，MemoryContext 会释放 planner-local palloc 对象。 relation locks 由事务 abort / ResourceOwner 清理。 syscache tuple 由显式 `ReleaseSysCache()` 释放。 `make_inh_translation_list()` 里 `SearchSysCacheAttName()` 成功后立即释放 tuple。
如果 ERROR 发生在创建了部分 child RTE 后，整个 planning 调用 abort。 不会把半成品 `PlannedStmt` 交给 executor。
## 8. 正确性机制层次
appendrel expansion 的 correctness 不是一个机制单独保证的。 它由多层边界叠加而成。
| 层次 | 机制 | 保证 | 不能保证 |
| --- | --- | --- | --- |
| relation lock | parent 已锁，child expansion 时拿同等级 lock | planning 期间 child schema 不被不兼容 DDL 改坏 | 不保证 child 会有 path 或会被扫描 |
| relcache / catalog | tuple descriptor、relkind、partition descriptor | 读到当前 transaction 可见的 relation metadata | 不跨 backend 共享指针 |
| `rte->inh` | 是否进入 appendrel expansion | 触发处理 inheritance / partition / UNION ALL appendrel | 不代表 child 已展开 |
| `AppendRelInfo` | parent-child 映射 | parent Var、qual、target、attnums 可翻译到 child | 不决定成本和 path 选择 |
| `translated_vars` | 正向列映射 | parent 用户列对应 child Var/表达式 | 不包含 system columns 和 whole-row Var |
| `parent_colnos` | 反向列映射 | child column 可回溯 parent column | child extra columns 可能没有 parent |
| planner-time pruning | `live_parts` | 未命中 partition 不创建 child RTE | 只使用规划期安全事实 |
| constraint exclusion | child dummy rel | child RTE 已存在但后续不扫描 | 不能减少 expansion 阶段的 child 数量 |
| setrefs | final RT indexes 和 appendrels | PlannedStmt 使用最终映射 | 不重新验证 catalog |
### 8.1 锁和 schema 稳定性
parent relation 在 parser/rewrite 阶段已经持有锁。 child relation 是 expansion 第一次接触，所以需要按 parent `rellockmode` 加锁。 传统 inheritance 使用 `find_all_inheritors(parentOID, lockmode, NULL)`。 partitioned path 使用 `try_table_open(childOID, lockmode)`。
这样 tuple descriptor 和 relkind 在 planning 期间不会被不兼容 DDL 改掉。
### 8.2 type / typmod / collation 检查
Var translation 不是只按列名。 `make_inh_translation_list()` 对 inherited column 检查：
```text
name
type OID
typmod
collation
```
如果 type 或 collation 不匹配，直接 ERROR。 这是防止 parent qual 被翻译到语义不同 child column 的核心机制。
### 8.3 permissions 和 RLS 边界
child RTE 不独立做 permission check。 权限检查以 parent RTE 为主。 列级权限和 updated columns 通过 `translate_col_privs()` 映射。 child RTE `securityQuals` 清空，parent security quals 后续作为 restriction clauses 传播。 这个设计让 inherited RLS 的语义和常规 permission check 保持一致。
不要把 child RTE 的 `perminfoindex = 0` 理解成“不需要安全检查”。 正确理解是：
```text
安全语义仍来自 parent；
child 只承担物理扫描和列映射角色。
```
### 8.4 多层 partition hierarchy
多层 partition 不一次性把 top parent 直接映射到 leaf。 每一层都有 immediate parent-child `AppendRelInfo`。 `adjust_appendrel_attrs_multilevel()` 和 `adjust_inherited_attnums_multilevel()` 负责多层递归。 如果 child 不是给定 parent 的 descendant，源码会 `elog(ERROR)`。
这避免了跨层误用 translation。
## 9. 错误路径 / 异常路径 / fallback
### 9.1 parent 标记有 subclass，但实际没有 child
`preprocess_relation_rtes()` 可能因为 `relhassubclass` false positive 保留 `rte->inh`。 传统 inheritance path 仍会调用 `find_all_inheritors()`。 返回列表至少包含 parent 自己。 源码不再试图在 `expand_inherited_rtentry()` 里清 `inh`。 原因是前面已经有 decisions 依赖它。
结果是按 inheritance path 生成 parent-member child。 这是 correctness 优先于局部简化。
### 9.2 partition 被 detach/drop 竞争
`expand_partitioned_rtentry()` 用 `try_table_open()` 打开 partition。 如果 partition 最近 detach 后被 drop，打开失败。 源码选择：
```text
delete i from relinfo->live_parts
continue
```
调用者看到的效果类似该 partition 被 pruning。 这是 partition expansion 的重要 fallback。 它避免 planner 因为 stale partition descriptor 直接失败。 前提是这个 partition 对当前语义已经不应再被访问。
### 9.3 其它 backend 的 temp child
传统 inheritance 可能遇到其它 backend 的 temp child。 源码注释说因为 buffering issues 不能安全访问。 处理方式是 silently ignore。 partitioned path 理论上定义阶段已经禁止其它 session temp partition。 但源码仍做 paranoia check。 如果发现，直接 ERROR：
```text
temporary relation from another session found as partition
```
传统 inheritance 和 partitioned table 在这里行为不同。
### 9.4 column name 找不到
`make_inh_translation_list()` 按 parent inherited column name 查 child。 如果找不到：
```text
elog(ERROR, "could not find inherited attribute ...")
```
这通常代表 catalog / inheritance metadata 不一致，或某个 DDL 路径破坏了继承不变量。 planner 不能 fallback 到按 attno 猜测。 按 attno 猜测会把 parent Var 指到错误 child column。
### 9.5 type 或 collation 不匹配
如果 child 同名 inherited column 的 type、typmod 或 collation 不匹配，planner ERROR。 这是结构性错误。 不能通过 cast 自动修复。 因为 parent qual、index condition、sort/group semantics 都依赖精确类型和 collation。
### 9.6 dropped parent column 被引用
`translated_vars` 中 dropped parent column 是 `NULL`。 如果后续试图翻译这个 parent column，`adjust_appendrel_attrs()` 会 ERROR。 错误消息使用 `parent_reloid` 找 parent relation name。 这说明 `parent_reloid` 不是路径搜索用字段。 它主要服务错误诊断。
### 9.7 child qual 化简成 false
`apply_child_basequals()` 翻译 parent qual 后可能得到 constant false 或 NULL。 例如 parent qual 对某个 child partition constraint 逻辑上矛盾。 函数返回 false。 `build_simple_rel()` 立即 `mark_dummy_rel(rel)`。 这不是 ERROR。 这是 child-level pruning / exclusion 的正常退化路径。
### 9.8 whole-row Var 的 conversion
whole-row Var 不能简单替换 attno。 如果 child rowtype 与 parent rowtype 不同，需要 `ConvertRowtypeExpr`。 如果是 `UNION ALL` appendrel 且没有 named rowtype，则需要 `RowExpr`。 如果同时携带不适合非 Var translation 的 `nullingrels` 或 non-default returningtype，源码会 ERROR。
这是为了避免表达式结构在 outer join 或 DML returning 语义下被错误替换。
### 9.9 stack depth
`expand_partitioned_rtentry()` 和 `set_append_rel_size()` 都有 `check_stack_depth()`。 深 inheritance / partition hierarchy 会递归。 stack depth check 是防止 pathological schema 让 backend 栈溢出的边界。
## 10. 成本、资源与跨模块传播
appendrel expansion 的成本主要随 child 数扩张。 要把它拆成几个阶段看。
| 阶段 | 主要成本 | 随什么扩张 |
| --- | --- | --- |
| partition pruning 前 | parent restriction 分析、PartitionDesc lookup | partitioned parent 数、partition key 复杂度 |
| child expansion | child lock、relcache open、RTE / AppendRelInfo / RelOptInfo palloc | live child 数、层级深度 |
| Var translation | expression_tree_mutator 复制 parent quals、target、joininfo | qual 数、target 宽度、child 数 |
| child size | child stats、constraint exclusion、index list、FDW hooks | live child 数、index 数、统计对象数 |
| child path | 每个 child path search | live child 数、每 child index 数、parameterization 数 |
| append path aggregation | 收集 pathkeys、required_outer、partial paths | child path 数、排序需求、并行路径数 |
### 10.1 planner-time pruning 的成本价值
分区表的关键优化是先算 `live_parts`。 被裁掉的 partition 不创建 child RTE。 这节省：
```text
lock
relcache open
AppendRelInfo
child RelOptInfo
qual translation
child path search
```
constraint exclusion 发生得更晚。 它通常已经付出了 expansion 和 child rel creation 成本。 所以同样公开影子是 child 不在 final plan，成本差别可能很大。
### 10.2 `translated_vars` 的复制成本
`adjust_appendrel_attrs()` 会复制表达式树。 如果 parent 有复杂 targetlist、很多 join quals 或 security quals，成本按 child 数放大。 公式近似是：
```text
O(number_of_children * size_of_parent_expressions)
```
这是 appendrel planning 在大分区数场景下的常见 CPU 来源。 它通常不会出现在 `pg_stat_*` 里。 需要 `perf`、flamegraph 或断点计数看。
### 10.3 lock 和 relcache 压力
每个 live child 都需要 relation lock 和 relcache open。 大量 partitions 会放大：
```text
lock table entries
relcache/syscache lookups
catcache memory
planner local palloc
```
这类成本发生在 planning，不是 executor scan。 `EXPLAIN` 默认看到的是 plan 形状，不会直接告诉你 planner 花了多少时间在 child expansion。 需要 `EXPLAIN (ANALYZE, TIMING)` 的 planning time、server log 或 profiling 辅助。
### 10.4 append path 聚合成本
`add_paths_to_append_rel()` 不只是把 cheapest path list 拼起来。 它还要收集所有 child 可用的 pathkeys 和 parameterizations。 child path 越多，append parent 需要考虑的组合越多。 `MergeAppend`、parallel append、parameterized append path 都会增加搜索空间。 这解释了为什么大分区数查询即使每个 child 都很小，planner 仍可能变慢。
### 10.5 跨模块传播
appendrel expansion 连接多个模块：
| 模块 | 连接点 |
| --- | --- |
| parser/rewrite | parent RTE、lock mode、permission info、rowmark seed。 |
| relcache/catalog | child tuple descriptor、partition descriptor、relkind、rowtype。 |
| partition pruning | `live_parts` 决定哪些 partition 进入 expansion。 |
| restrictions / EC | parent quals 和 EC 需要按 child translation 复制。 |
| path search | child otherrel 生成 scan path，parent appendrel 聚合 path。 |
| DML planning | target relation descendants、row identity、updated columns translation。 |
| setrefs / PlannedStmt | final RT indexes 和 `appendRelations` 固化。 |
这个机制不涉及 WAL。 它也不涉及 shared memory 并发状态。 正确性主要来自锁、relcache snapshot、planner-local ownership 和表达式翻译不变量。
## 11. 观测与诊断入口
### 11.1 能直接看到的状态
`EXPLAIN` 能看到 final plan shape。 例如：
```sql
EXPLAIN (VERBOSE, COSTS OFF)
SELECT * FROM p_parent WHERE a = 42;
```
可以看到哪些 child scan 出现在最终 plan 中。 `VERBOSE` 有时能显示 child output list，帮助判断 targetlist 是否已经落到 child relid。 `EXPLAIN (ANALYZE)` 能看到 planning time。 如果 child 数很多，planning time 上升但 execution time 很低，appendrel expansion 和 path search 是可疑方向。
`pg_locks` 能看到当前事务持有的 relation locks。 对长时间 planning 或使用 explicit transaction 的实验，可以观察 child relation locks 是否被持有。
### 11.2 只能间接推断的状态
`AppendRelInfo`、`translated_vars`、`part_rels`、`live_parts` 都不是 SQL 可直接查询的状态。 只能通过：
```text
gdb breakpoint
debug_print_plan
debug_print_rewritten
EXPLAIN shape
planner profiling
```
间接推断。 被 pruning 裁掉的 partition 和被 constraint exclusion 标成 dummy 的 child，在 final EXPLAIN 中都可能不可见。 区分方法是断点或日志：
```text
expand_single_inheritance_child()
build_simple_rel()
relation_excluded_by_constraints()
set_append_rel_pathlist()
```
如果没有进入 `expand_single_inheritance_child()`，说明 child 没被展开。 如果进入了但 later dummy，说明 expansion 已经发生。
### 11.3 gdb 断点建议
源码跟读可以放这些断点：
```gdb
break expand_inherited_rtentry
break expand_partitioned_rtentry
break expand_single_inheritance_child
break make_append_rel_info
break make_inh_translation_list
break adjust_appendrel_attrs
break apply_child_basequals
break set_append_rel_size
break set_append_rel_pathlist
```
关键变量：
```gdb
print rti
print parentRTindex
print childRTindex
print parentrte->inh
print childrte->inh
print appinfo->parent_relid
print appinfo->child_relid
print list_length(appinfo->translated_vars)
print relinfo->live_parts
```
多层 partition 时重点看：
```text
appinfo->parent_relid 是否是 immediate parent
childrelinfo->parent 是否指向 immediate parent RelOptInfo
top_parent 是否保持最顶层 parent
```
### 11.4 `debug_print_plan` 的边界
`debug_print_plan` 能打印计划树。 它不等于 planner 内部 appendrel 状态全量 dump。 最终计划可能已经经过 setrefs、subplan 删除、Append 简化等处理。 所以它适合确认 final shape。 不适合证明某个 child 从未被展开。
### 11.5 profiling 入口
大分区数 planning 慢时，常见热点包括：
```text
expand_partitioned_rtentry
make_inh_translation_list
adjust_appendrel_attrs_mutator
set_append_rel_size
set_rel_pathlist
add_paths_to_append_rel
```
如果热点在 `adjust_appendrel_attrs_mutator()`，通常说明 parent 表达式复杂度乘以 child 数。 如果热点在 relcache/syscache，通常说明 child 数和 index/statistics 元数据过多。 如果热点在 partition pruning，则要回到上一节的 pruning step 和 bound lookup。
## 12. 课堂实验
### 实验 1：传统 inheritance parent 为什么出现两次身份
目标：观察 original parent RTE 和 parent-member child RTE 的区别。 步骤：
```sql
DROP TABLE IF EXISTS ih_child, ih_parent CASCADE;
CREATE TABLE ih_parent(a int, b text);
CREATE TABLE ih_child(a int, b text, extra text) INHERITS (ih_parent);
EXPLAIN (VERBOSE, COSTS OFF)
SELECT a, b FROM ih_parent WHERE a >= 0;
```
断点：
```gdb
break expand_inherited_rtentry
break expand_single_inheritance_child
break make_inh_translation_list
```
观察：
```text
find_all_inheritors() 返回 parent OID first。
parent-member child 的 oldrelation == newrelation。
parent-member child 仍然有新的 childRTindex。
AppendRelInfo.parent_relid 和 child_relid 不相等。
```
问题：
```text
为什么不能直接让 original parent RTE 同时代表 whole set 和 parent heap？
```
提示： whole set rows/width/path 聚合语义和单个 parent heap scan 语义不同。
### 实验 2：partition pruning 发生在 child RTE 创建前
目标：区分 planner-time pruning 和 child dummy。 步骤：
```sql
DROP TABLE IF EXISTS p_parent CASCADE;
CREATE TABLE p_parent(a int, b text) PARTITION BY RANGE (a);
CREATE TABLE p1 PARTITION OF p_parent FOR VALUES FROM (0) TO (100);
CREATE TABLE p2 PARTITION OF p_parent FOR VALUES FROM (100) TO (200);
EXPLAIN (VERBOSE, COSTS OFF)
SELECT * FROM p_parent WHERE a = 42;
```
断点：
```gdb
break expand_partitioned_rtentry
break expand_single_inheritance_child
break build_simple_rel
```
观察：
```text
relinfo->live_parts 只包含命中的 partition index。
只有 live partition 会进入 expand_single_inheritance_child()。
被裁掉的 partition 没有 child RT index。
```
变体：
```sql
PREPARE q(int) AS SELECT * FROM p_parent WHERE a = $1;
EXPLAIN EXECUTE q(42);
```
比较 custom plan 与 generic plan 下 child expansion 的差异。
### 实验 3：列顺序与 Var translation
目标：观察 `translated_vars` 不按 attno 盲目映射。 可以用传统 inheritance 和 DDL 构造列顺序差异。 思路：
```sql
DROP TABLE IF EXISTS ih_a, ih_b CASCADE;
CREATE TABLE ih_a(a int);
ALTER TABLE ih_a ADD COLUMN b text;
CREATE TABLE ih_b(extra text) INHERITS (ih_a);
EXPLAIN (VERBOSE, COSTS OFF)
SELECT a, b FROM ih_a;
```
断点：
```gdb
break make_inh_translation_list
```
观察：
```text
parent old_attno 按 parent descriptor 走。
child new_attno 通过 name 匹配。
translated_vars 中 Var.varattno 是 child column number。
parent_colnos 反向记录 child column 对应 parent column。
```
不同版本或 DDL 顺序可能导致列布局差异不明显。 如果无法稳定复现，直接在断点里检查普通 child 的 `translated_vars` 也能完成实验目标。
### 实验 4：child qual 化简成 dummy
目标：观察 expansion 已发生但 child 后续被标 dummy。 步骤：
```sql
DROP TABLE IF EXISTS ih_c1, ih_c2, ih_p CASCADE;
CREATE TABLE ih_p(a int);
CREATE TABLE ih_c1(CHECK (a >= 0 AND a < 10)) INHERITS (ih_p);
CREATE TABLE ih_c2(CHECK (a >= 10 AND a < 20)) INHERITS (ih_p);
SET constraint_exclusion = on;
EXPLAIN (VERBOSE, COSTS OFF)
SELECT * FROM ih_p WHERE a = 15;
```
断点：
```gdb
break expand_single_inheritance_child
break apply_child_basequals
break relation_excluded_by_constraints
break set_dummy_rel_pathlist
```
观察：
```text
传统 inheritance child RTE 已经创建。
后续 constraint exclusion 或 child qual 逻辑让某些 child 变 dummy。
final plan 不显示 dummy child。
```
## 13. 常见误区
误区一：把 `rte->inh` 当成“已经展开”。 `rte->inh` 只是触发条件。 真正展开要看 child RTE、child `RelOptInfo` 和 `AppendRelInfo`。 误区二：看到 final plan 没有某个 partition，就认为 planner 从未接触它。 可能是 planner-time pruning 没展开。 也可能是 constraint exclusion 标 dummy。
也可能是 child path 生成后没有进入 cheapest append path。 误区三：认为 `translated_vars` 只需要改 `varno`。 传统 inheritance 要处理 dropped columns、child extra columns、name match、type/typmod/collation。 whole-row Var 还需要 rowtype conversion。
`UNION ALL` translation 可能是表达式。 误区四：认为 partitioned table 和 traditional inheritance expansion 完全一样。 partitioned parent 不扫描自身。 partitioned expansion 按 `PartitionDesc` 和 `live_parts` 走，并且多层 hierarchy level-by-level 递归。
传统 inheritance 对 unpartitioned tree 更接近 flatten，并包含 parent heap member。 误区五：认为 child RTE 不做 permission check 就没有权限语义。 权限语义在 parent 上。 child RTE 主要承担物理扫描身份。 列级权限和 updated columns 通过 `translated_vars` 映射。 误区六：把 `AppendRelInfo` 当成 executor path node。
`AppendRelInfo` 是 planner/setrefs 的语义映射结构。 `AppendPath` 和 `Append` 才是 path/plan 层的执行组合节点。
## 14. 讨论题
1. 为什么 `add_other_rels_to_query()` 要延迟到 restriction clauses、EC 和 lateral 信息建立后，而不是在 `add_base_rels_to_query()` 里立即展开？
2. 传统 inheritance 中为什么 parent heap 需要一个新的 child RTE，而不能复用 original parent RTE？
3. 为什么 partitioned table expansion 要先调用 `prune_append_rel_partitions()`，再创建 child RTE？
4. `translated_vars` 为什么不能只按 parent attno 到 child attno 一一对应？
5. child RTE 的 `perminfoindex = 0` 为什么不等于跳过权限检查？
6. `AppendRelInfo.parent_colnos` 主要服务哪些反向映射场景？
7. 如果一个 child 没有出现在 `EXPLAIN` 的 `Append` 子计划里，你会如何判断它是没展开、dummy，还是 path 竞争失败？
8. 多层 partition hierarchy 为什么采用 immediate parent-child translation，而不是 top parent 直接映射到 leaf？
## 15. 源码练习
### 15.1 给 expansion 加最小日志
练习目标：看清 RT index 的产生顺序。 建议在本地源码临时加日志，不提交：
```c
elog(LOG, "appendrel parent %u child %u parentRTI %d childRTI %d",
     RelationGetRelid(parentrel),
     RelationGetRelid(childrel),
     parentRTindex,
     childRTindex);
```
放置位置：
```text
expand_single_inheritance_child()
  -> make_append_rel_info() 之后
```
观察传统 inheritance 和 partitioned table 的差异。
### 15.2 统计 Var translation 次数
练习目标：感受表达式复制成本。 在 `adjust_appendrel_attrs_mutator()` 处理 `Var` 的分支里加计数。 比较：
```sql
SELECT * FROM p_parent WHERE a = 42;
SELECT * FROM p_parent WHERE a BETWEEN 1 AND 100 AND b LIKE '%x%';
```
再增加 partition 数量。 观察计数如何随 child 数和 qual 复杂度放大。
### 15.3 断言 child RTE 不做 permission check
练习目标：理解 parent permission contract。 在 `expand_single_inheritance_child()` 后检查：
```text
childrte->perminfoindex == 0
```
再观察 `build_simple_rel()` 中 child `rel->userid` 如何从 parent 继承。 思考：
```text
如果为每个 child 独立做 permission check，会改变哪些用户可见语义？
```
## 16. 版本与实现边界
本课基于 PostgreSQL `master`，短提交 `0e1f1ed157e`。 几个结论属于稳定语义：
```text
parent-level SQL 语义需要 child-level physical planning；
AppendRelInfo 保存 parent-child translation；
child RelOptInfo 是 RELOPT_OTHER_MEMBER_REL；
partitioned parent 本身不扫描；
traditional inheritance parent heap 是 append member；
planner-time pruning 可以避免创建 child RTE；
constraint exclusion 通常发生在 child 已创建之后。
```
几个结论属于当前实现路径：
```text
具体函数拆分；
PartitionDirectory 的创建位置；
setrefs 对 appendRelations 的 flatten 细节；
row identity 变量的内部表示；
partition-wise join 对 partexprs 的消费路径。
```
读旧资料时要注意： 多层 partition hierarchy 的 expansion 不是把所有 leaf 一次性 flatten 到 top parent。 当前实现 level-by-level 创建 `AppendRelInfo` 和 child `RelOptInfo`。 传统 inheritance 的 unpartitioned path 仍然更接近 flatten。
## 17. 本节小结
本节唯一主问题是：
```text
inheritance/partition expansion 如何构造 appendrel、child rel 和 Var translation？
```
答案可以压缩成一条链：
```text
parent RTE 带 inh
  -> parent baserel 先建立 restriction / EC / lateral 等语义状态
  -> add_other_rels_to_query() 触发展开
  -> expand_inherited_rtentry() 区分 UNION ALL、传统继承和 partitioned table
  -> expand_single_inheritance_child() 创建 child RTE
  -> make_append_rel_info() 创建 parent-child 映射
  -> make_inh_translation_list() 构造 translated_vars / parent_colnos
  -> build_simple_rel() 创建 child otherrel 并翻译 base quals
  -> allpaths 为 child 生成 path 并聚合成 append parent path
  -> setrefs 把最终 AppendRelInfo 放入 PlannedStmt
```
核心状态是三件套：
```text
child RangeTblEntry
child RelOptInfo
AppendRelInfo
```
`translated_vars` 是 parent Var 到 child Var/表达式的正向映射。 `parent_colnos` 是 child column 到 parent column 的反向映射。 系统列和 whole-row Var 走特殊规则。 inheritance translation 通常是 simple Var。 `UNION ALL` translation 可以是任意表达式。 生命周期上，这些状态主要属于 planner memory context。
relation lock 和 relcache reference 单独管理。 ERROR 时 planner-local 内存由 context 清理，locks 由事务 abort / ResourceOwner 清理。 正确性来自 child locks、tuple descriptor 校验、type/typmod/collation 检查、permission 语义归属 parent、rowmark 和 row identity 的统一映射。
成本主要随 live child 数、表达式大小、index/statistics 元数据和 child path 数放大。 planner-time partition pruning 的价值在于它能在 child RTE 创建前减少搜索空间。 constraint exclusion 的公开结果可能类似，但成本边界更晚。 可迁移的系统规律是：
```text
当一个 logical object 需要映射到多个 physical object 时，
不要只复制对象；
必须同时保存 identity mapping、attribute translation、ownership/lifetime 和后续消费者的稳定 contract。
```
对 PostgreSQL planner 来说，这个 contract 就是 appendrel。
