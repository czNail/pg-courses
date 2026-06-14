# PostgreSQL planner-time partition pruning 与 Append 子路径裁剪
## 课程定位
前置知识：已经理解 `RelOptInfo`、`RestrictInfo`、`Path`、`AppendPath`、参数化路径以及 base relation pathlist 生成。
本节唯一主问题：
```text
planner-time partition pruning 如何根据约束、参数和表达式裁剪 Append 子路径？
```
核心矛盾：分区表的查询应该尽早丢掉不可能命中的分区，以避免为每个 child 建 `RelOptInfo`、估算 rows、生成 path 和持锁；但 planner 只能使用在规划期语义稳定的信息，不能把依赖运行期参数、stable 函数或外层 `PARAM_EXEC` 的判断提前固化成计划形状。
学完后应能判断：一个分区没有出现在 `Append` 子路径里，是 planner-time pruning 已经没有展开它，还是 constraint exclusion 把 child 标成 dummy，还是 path 阶段生成后被 `add_path()` 竞争剪掉，或是 executor initial / per-scan pruning 在运行期跳过了它。
本课基于本地 `/home/nail/postgres` 源码，分支 `master`，短提交 `0e1f1ed157e`。
## 1. 本节在总主线中的位置
前一组 base relation path 课程讲的是普通 rel 如何生成 scan path。
分区表先要回答一个更早的问题：
```text
这个 partitioned rel 的哪些 child 还有资格进入后续 path 搜索？
```
如果这个问题回答得太晚，planner 会为大量无关分区做无用工作。
如果回答得太早，又可能把运行期参数才能决定的分区永久删掉。
本节只讲 planner 侧的分区裁剪。
运行期 executor 如何用 `PartitionPruneInfo` 跳过 subplan，只在本节作为边界出现。
partition-wise join / aggregate 的合法性判断留给下一节。
inheritance expansion 和 `AppendRelInfo` 的完整翻译模型留给后续 `68`。
这一节读源码时只盯住一条状态链：
```text
parent RelOptInfo 的 partition metadata
  -> baserestrictinfo 中可用于 pruning 的 qual
  -> PartitionPruneStep
  -> live_parts bitmap
  -> child RTE / child RelOptInfo / AppendRelInfo
  -> live child path
  -> AppendPath.subpaths
```
任何不能解释这条链上状态变化的函数，先不要展开。
## 2. 核心矛盾与一句话运行模型
一句话运行模型：
```text
plancat.c 从 relcache 复制 partition scheme、boundinfo、partition key expr；
inherit.c 在展开 partitioned RTE 时调用 prune_append_rel_partitions()；
partprune.c 把 parent baserestrictinfo 转成 planner 可用的 PartitionPruneStep；
get_matching_partitions() 计算 live_parts bitmap；
inherit.c 只为 live_parts 创建 child RTE、AppendRelInfo 和 child RelOptInfo；
allpaths.c 只在这些 live child 上生成 child paths，再把它们收集成 AppendPath.subpaths。
```
这里的核心 tension 有三层。
第一层是搜索空间：
```text
尽早裁剪分区，减少 child rel 和 child path 数量
```
对立面是语义安全：
```text
只有规划期确定的表达式才能影响计划结构
```
第二层是表达式求值：
```text
immutable 常量折叠可以成为 Const
```
对立面是运行期依赖：
```text
stable 函数、普通 Param、PARAM_EXEC 不能随意提前当作常量
```
第三层是 Append 子路径映射：
```text
planner-time pruning 直接改变 child RTE 和 subpath 集合
```
对立面是 executor pruning：
```text
initial / per-scan pruning 只能在已有 subplans 中选择跳过哪些
```
所以本节的关键词不是“分区裁剪会让查询更快”。
更准确的模型是：
```text
partition pruning 是把一个逻辑分区空间压缩成一个更小的 planner 搜索空间；
planner-time 版本只能使用不会在执行期改变的事实；
运行期版本保留 plan shape，通过 subplan mapping 选择跳过。
```
## 3. 核心文件分工与阅读顺序
| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/optimizer/util/plancat.c` | `set_relation_partition_info()` 把 relcache 的 partition key、bound 和 constraint 复制到 `RelOptInfo`。 |
| 2 | `src/backend/optimizer/util/inherit.c` | `expand_partitioned_rtentry()` 调 `prune_append_rel_partitions()`，并只为 `live_parts` 建 child RTE / `AppendRelInfo` / child `RelOptInfo`。 |
| 3 | `src/backend/partitioning/partprune.c` | `prune_append_rel_partitions()`、`gen_partprune_steps()`、`match_clause_to_partition_key()` 和 `get_matching_partitions()` 的主逻辑。 |
| 4 | `src/include/partitioning/partprune.h` | `PartitionPruneContext` 描述一次 pruning 计算需要的 partition metadata、support function 和表达式执行上下文。 |
| 5 | `src/include/nodes/plannodes.h` | `PartitionPruneInfo`、`PartitionedRelPruneInfo`、`PartitionPruneStepOp`、`PartitionPruneStepCombine` 的 executor 契约。 |
| 6 | `src/include/nodes/pathnodes.h` | `RelOptInfo.live_parts`、`part_rels`、`partexprs`、`AppendRelInfo` 和 `PlannerGlobal.partPruneInfos`。 |
| 7 | `src/backend/optimizer/path/allpaths.c` | `set_append_rel_size()`、`set_append_rel_pathlist()`、`add_paths_to_append_rel()` 如何消费已经展开的 live child。 |
| 8 | `src/backend/optimizer/prep/prepunion.c` | `UNION ALL` 也会产生 append-like subpaths，用来对比分区表 pruning 和普通 append path 收集的边界。 |
推荐阅读顺序不是从 `partprune.c` 顶部线性读到底。
先读 `RelOptInfo` 分区字段。
再读 `expand_partitioned_rtentry()`。
然后读 `prune_append_rel_partitions()`。
最后读 `make_partition_pruneinfo()`，理解哪些 qual 被留给 executor。
两个名字容易混淆：
| 名字 | 阶段 | 结果 |
| --- | --- | --- |
| `prune_append_rel_partitions()` | planner-time expansion | 返回 `live_parts`，影响 child 是否被展开。 |
| `make_partition_pruneinfo()` | create plan / executor contract | 构造 `PartitionPruneInfo`，让 executor 在已有 subplans 中跳过。 |
本节主流程以前者为主。
后者只用来解释参数和表达式为什么不能都在 planner-time 固化。
## 4. 可复现运行现象
先用几个现象建立诊断边界。
### 4.1 常量命中单个 range 分区
示例 SQL：
```sql
CREATE TABLE mlog(ts date, payload text) PARTITION BY RANGE (ts);
CREATE TABLE mlog_2026_01 PARTITION OF mlog
  FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE mlog_2026_02 PARTITION OF mlog
  FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
EXPLAIN SELECT * FROM mlog WHERE ts = DATE '2026-02-14';
```
预期公开影子：`Append` 可能消失，或者 `Append` 只有一个 child。
源码解释：`DATE '2026-02-14'` 已是 `Const`，`match_clause_to_partition_key()` 能把 `ts = Const` 匹配到 partition key，`get_matching_range_bounds()` 最终只返回 2 月分区。
关键点：这个分区没有出现在 child path 中，不是成本竞争输了。
它在 `expand_partitioned_rtentry()` 阶段就没有被展开。
### 4.2 stable 函数不能 planner-time 固化
示例 SQL：
```sql
EXPLAIN SELECT * FROM mlog WHERE ts = CURRENT_DATE;
```
公开影子：generic plan 中通常仍有多个分区 subplans，执行时可能显示 `Subplans Removed`。
源码解释：`CURRENT_DATE` 不是普通 immutable 常量。
`PARTTARGET_PLANNER` 路径只接受匹配分区键后的 `expr` 是 `Const`，并且比较操作符是 immutable。
stable 表达式可以用于 executor startup pruning，但不能随意改变 planner-time plan shape。
关键点：`stable` 不等于 planner-time immutable。
### 4.3 prepared statement 参数的两种命运
示例 SQL：
```sql
PREPARE q(date) AS SELECT * FROM mlog WHERE ts = $1;
EXPLAIN EXECUTE q(DATE '2026-02-14');
```
公开影子取决于 custom / generic plan。
custom plan 如果 `PARAM_EXTERN` 有 bound value 并被 `PARAM_FLAG_CONST` 标记，`eval_const_expressions()` 可以把参数替换成 `Const`。
这种情况下 planner-time pruning 可能直接裁掉无关 child。
generic plan 没有具体参数值，通常会保留 subplans，并依赖 initial pruning。
关键点：用户看到的“参数也能裁剪”不一定发生在同一个阶段。
### 4.4 correlated nested loop 中的 `PARAM_EXEC`
典型形态是外层行驱动内层分区表：
```sql
EXPLAIN
SELECT *
FROM keys k
WHERE EXISTS (
  SELECT 1 FROM mlog p WHERE p.ts = k.ts
);
```
公开影子：可能出现 runtime per-scan pruning，而不是 planner-time pruning。
源码解释：外层值在规划期未知，进入 executor 后才通过 `PARAM_EXEC` slot 传入内层。
`match_clause_to_partition_key()` 检测到 `PARAM_EXEC` 后，只有 `PARTTARGET_EXEC` 可以使用它生成 per-scan pruning step。
关键点：如果把 `PARAM_EXEC` 提前用于 planner-time pruning，会在不同 outer row 下产生错误结果。
### 4.5 分区被约束排除，不等于 partition pruning
普通继承表和分区表 child 都可能被 `relation_excluded_by_constraints()` 标成 dummy。
公开影子也是 child 不见了。
但源码路径不同。
planner-time partition pruning 的主入口是 `prune_append_rel_partitions()`。
constraint exclusion 的主入口在 `allpaths.c` 的 `set_rel_size()` 和 `set_append_rel_size()`。
诊断时要看 child 是否根本没有 RTE，还是 RTE 存在但 `IS_DUMMY_REL(childrel)`。
## 5. 关键数据结构与状态边界
本节只讲影响主问题的字段组合。
不要把 raw field 当作语义。
字段必须和创建时机、访问者、内存上下文、表达式稳定性一起解释。
### 5.1 `RelOptInfo` 的分区字段
| 字段 | 语义 |
| --- | --- |
| `part_scheme` | 分区策略、key 数、opfamily、collation、support function 的 planner-local 拷贝。 |
| `nparts` | 当前 partitioned rel 的直接 partition 数。 |
| `boundinfo` | 分区边界，来自 relcache `PartitionDesc`。 |
| `partition_qual` | 当前 rel 作为分区时继承来的 partition constraint。 |
| `part_rels` | 按 partition index 保存 child `RelOptInfo *`，被 planner-time pruning 裁掉的槽位是 `NULL`。 |
| `live_parts` | `Bitmapset`，成员是 `part_rels[]` / `PartitionDesc` 的 partition index。 |
| `partexprs` | 每个 partition key 对应的表达式列表，base rel 通常每个 key 只有一个表达式。 |
| `nullable_partexprs` | joinrel 下可为空 partition key 表达式，本节 base rel 主线中通常为空数组。 |
这些字段都是 planner-local。
它们不跨 backend 共享。
它们不进入 relcache 本体。
它们的生命周期属于一次 `planner()` 调用。
### 5.2 `PartitionScheme`
`PartitionScheme` 是 planner 对 `PartitionKey` 的轻量拷贝。
`find_partition_scheme()` 会在 `root->part_schemes` 中复用等价 scheme。
它保存：
| 字段 | 作用 |
| --- | --- |
| `strategy` | list、range 或 hash。 |
| `partnatts` | 分区 key 数。 |
| `partopfamily` | 判断操作符是否属于分区 opfamily。 |
| `partopcintype` | opclass 输入类型，用来找 cross-type support function。 |
| `partcollation` | clause collation 必须匹配。 |
| `partsupfunc` | bound lookup 时使用的比较或 hash support function。 |
planner-time pruning 不是直接比较 SQL 文本。
它比较的是：
```text
partition key expression
  + clause operator
  + opfamily strategy
  + collation
  + strictness
  + volatility
  + comparison support function
```
### 5.3 `PartitionPruneStep`
`PartitionPruneStep` 是把 qual 压缩成 pruning VM 的一步。
当前主要有两类：
| 类型 | 作用 |
| --- | --- |
| `PartitionPruneStepOp` | 一个或多个 partition key 上的操作符约束，例如 `key = Const`、`key < Const`、`key IS NULL`。 |
| `PartitionPruneStepCombine` | 把多个 step 的结果做 `UNION` 或 `INTERSECT`，对应 OR / AND。 |
`step_id` 是当前 pruning context 内的顺序编号。
`perform_pruning_combine_step()` 要求被引用的 source step 先执行。
所以 step list 不只是集合。
它也是一个小型 DAG 的线性执行序。
### 5.4 `GeneratePruningStepsContext`
`GeneratePruningStepsContext` 是 step 生成期的工作状态。
关键字段：
| 字段 | 语义 |
| --- | --- |
| `target` | `PARTTARGET_PLANNER`、`PARTTARGET_INITIAL` 或 `PARTTARGET_EXEC`。 |
| `steps` | 已生成的 `PartitionPruneStep` 列表。 |
| `has_mutable_op` | 找到过 stable 但非 immutable 的比较操作符。 |
| `has_mutable_arg` | 找到过可用于 startup pruning 的非 Const 表达式，不含 exec param。 |
| `has_exec_param` | 找到过 `PARAM_EXEC`。 |
| `contradictory` | clauses 被证明自相矛盾，可以返回空集合。 |
`has_mutable_arg` 这个名字容易误读。
这里不是说 volatile 函数也能用。
`match_clause_to_partition_key()` 明确拒绝 volatile expression。
这里表示表达式不是 planner-time Const，但在 executor startup 时可能稳定可求值。
### 5.5 `PartitionPruneContext`
`PartitionPruneContext` 是实际执行 pruning step 时的上下文。
planner-time 调用中：
| 字段 | planner-time 值 |
| --- | --- |
| `strategy` | 来自 `rel->part_scheme->strategy`。 |
| `partnatts` | 来自 `rel->part_scheme->partnatts`。 |
| `nparts` | 来自 `rel->nparts`。 |
| `boundinfo` | 来自 `rel->boundinfo`。 |
| `partsupfunc` | 来自 `rel->part_scheme->partsupfunc`。 |
| `stepcmpfuncs` | 按 step 和 key 分配，用来缓存比较函数。 |
| `ppccontext` | `CurrentMemoryContext`。 |
| `planstate` | `NULL`。 |
| `exprcontext` | `NULL`。 |
| `exprstates` | `NULL`。 |
`planstate == NULL` 是重要边界。
planner-time pruning 不执行任意表达式状态。
它只能消费已经是 `Const` 的值。
### 5.6 `AppendRelInfo`
`AppendRelInfo` 描述 parent RTE 和 child RTE 的列映射。
关键字段：
| 字段 | 作用 |
| --- | --- |
| `parent_relid` | parent RT index。 |
| `child_relid` | child RT index。 |
| `translated_vars` | parent 列到 child 列或表达式的映射。 |
| `parent_colnos` | child 列反查 parent 列。 |
| `parent_reloid` | inheritance 场景下用于错误消息和身份。 |
planner-time pruning 先决定哪些 partition index 是 live。
只有 live partition 才会通过 `expand_single_inheritance_child()` 建 child RTE 和 `AppendRelInfo`。
这意味着被 plan-time 裁掉的分区不会参与后续 Var translation、child qual 复制和 child path generation。
### 5.7 `PartitionPruneInfo`
`PartitionPruneInfo` 属于运行期 pruning 契约。
它不是 planner-time pruning 的返回值。
`make_partition_pruneinfo()` 会把它放进 `root->partPruneInfos`。
关键字段：
| 字段 | 作用 |
| --- | --- |
| `relids` | 对应父 plan node 的 apprelids。 |
| `prune_infos` | 每个可运行期裁剪的 partition hierarchy 的 `PartitionedRelPruneInfo` 列表。 |
| `other_subplans` | 不属于这些 hierarchy 或不能裁剪的 subplan index，executor 不能裁掉。 |
`PartitionedRelPruneInfo` 继续保存：
| 字段 | 作用 |
| --- | --- |
| `present_parts` | plan 中还存在的 partition index。 |
| `subplan_map` | partition index 到 parent plan subplan index。 |
| `subpart_map` | partition index 到下一级 prune info index。 |
| `leafpart_rti_map` | leaf partition 的 RT index。 |
| `initial_pruning_steps` | executor startup 可用的 pruning steps。 |
| `exec_pruning_steps` | 每次 scan 可用的 pruning steps。 |
| `execparamids` | per-scan pruning 实际依赖的 `PARAM_EXEC` IDs。 |
这组 mapping 说明一个事实：
```text
executor pruning 不是删除 plan node；
它是在已经存在的 subplan index 空间里选择 active subplans。
```
## 6. 主流程源码 walkthrough
本节主流程从一个 partitioned table RTE 进入 planner 开始。
### 6.1 `plancat.c` 填充 partition metadata
入口：
```text
get_relation_info()
  -> set_relation_partition_info()
     -> PartitionDirectoryLookup()
     -> find_partition_scheme()
     -> set_baserel_partition_key_exprs()
     -> set_baserel_partition_constraint()
```
`set_relation_partition_info()` 把 relcache 中的 partition descriptor 复制或挂到 `RelOptInfo`。
`rel->part_scheme` 来自 `find_partition_scheme()`。
`rel->boundinfo` 指向 `PartitionDesc` 的 bound info。
`rel->nparts` 是直接 partition 数。
`set_baserel_partition_key_exprs()` 把 partition key 转成当前 relid 下的表达式。
普通列分区键变成 `Var(varno = rel->relid, varattno = attno)`。
表达式分区键会复制 relcache 中的 expression，再把 varno 改成当前 relid。
`set_baserel_partition_constraint()` 把当前 rel 作为 partition 时的 constraint 放到 `rel->partition_qual`。
这些状态准备好后，`partprune.c` 才能把 query qual 和分区边界联系起来。
### 6.2 `inherit.c` 在展开分区树前裁剪
主入口：
```text
expand_partitioned_rtentry()
  -> PartitionDirectoryLookup()
  -> relinfo->live_parts = prune_append_rel_partitions(relinfo)
  -> expand_planner_arrays(root, num_live_parts)
  -> palloc0(relinfo->part_rels)
  -> for each i in live_parts:
       try_table_open(partdesc->oids[i])
       expand_single_inheritance_child()
       build_simple_rel()
       relinfo->part_rels[i] = childrelinfo
       recurse if child is partitioned
```
这里的时间点很关键。
`prune_append_rel_partitions()` 在 child RTE 创建之前执行。
因此 plan-time 被裁掉的 partition 不会有 child RT index。
也不会有 child `RelOptInfo`。
也不会出现在 `root->append_rel_list`。
也不会进入 `set_append_rel_size()`。
如果 `live_parts` 为空，`part_rels` 仍会分配为 `nparts` 长度的数组，但所有槽位为空。
如果某个 live partition 在并发 DDL 后 `try_table_open()` 失败，源码会把它从 `live_parts` 删除并继续。
这是异常路径，不是普通 pruning 判断。
### 6.3 `prune_append_rel_partitions()` 的 planner-time 判断
入口：
```text
prune_append_rel_partitions(RelOptInfo *rel)
  -> clauses = rel->baserestrictinfo
  -> if nparts == 0 return NULL
  -> if !enable_partition_pruning or clauses == NIL return all partitions
  -> gen_partprune_steps(rel, clauses, PARTTARGET_PLANNER, &gcontext)
  -> if contradictory return NULL
  -> if steps == NIL return all partitions
  -> build PartitionPruneContext with planstate = NULL
  -> get_matching_partitions(&context, pruning_steps)
```
这段代码把三类结果区分得很清楚。
结果一：无 partition。
返回 `NULL`，表示空集合。
结果二：禁用 pruning 或没有 qual。
返回 `0..nparts-1` 的全集 bitmap。
结果三：qual 自相矛盾。
返回 `NULL`，表示没有 live partition。
结果四：有 qual 但没有可用于 planner-time pruning 的 step。
返回全集 bitmap。
结果五：有可用 step。
调用 `get_matching_partitions()` 计算 survivor。
这也是诊断时的第一层分叉。
看到所有分区都在计划里，不一定说明 pruning 没被调用。
可能是被调用后没有 planner-time usable step。
### 6.4 `gen_partprune_steps()` 添加默认分区保护
`gen_partprune_steps()` 先初始化 `GeneratePruningStepsContext`。
如果当前 partitioned table 自己也是一个 partition，并且有 default partition，它会把 `rel->partition_qual` 拼进 clauses。
目的不是让普通路径更复杂。
而是处理 default partition 不能仅靠普通 bound lookup 精确排除的边界。
之后进入 `gen_partprune_steps_internal()`。
这部分说明 partition pruning 不是只看用户 WHERE。
父分区约束也可能参与推理。
### 6.5 `gen_partprune_steps_internal()` 拆 BoolExpr
核心逻辑：
```text
foreach clause:
  unwrap RestrictInfo
  Const false/null => contradictory
  OR => recursively generate steps for each arm, combine by UNION
  AND => recursively generate steps, combine by INTERSECT
  other => try matching each partition key
```
OR 的特殊点是 dummy combine step。
如果 OR 某个 arm 不包含可裁剪条件，不能简单忽略它。
例如：
```sql
WHERE ts = DATE '2026-02-14' OR payload LIKE 'x%'
```
第二个 arm 不能帮助裁剪。
整个 OR 也不能只保留 2 月分区。
源码用一个 `source_stepids == NIL` 的 combine step 表示“这个 arm 不可裁剪，因此返回全部 partitions”。
再与其它 arm 做 union。
这保证 OR 不会错误裁剪。
AND 的处理相反。
多个可用条件可以取交集。
如果任意 AND arm 自相矛盾，整个表达式矛盾。
### 6.6 `match_clause_to_partition_key()` 匹配分区键
匹配的核心输入是：
```text
clause
partition key expression
partition key opfamily
partition key collation
target phase
```
它支持几类形态。
第一类是 boolean partition key。
例如 `partkey IS TRUE`、`partkey IS FALSE`、`partkey IS UNKNOWN`。
`IS NOT TRUE` 和 `IS NOT FALSE` 会转成 bool test 加 `IS NULL` 的 OR。
第二类是普通二元 `OpExpr`。
要求一侧等于 partition key。
如果 partition key 在右侧，需要找到 commutator 把操作符交换到左侧。
第三类是 `ScalarArrayOpExpr`。
例如 `key = ANY (ARRAY[...])`。
常量数组会被拆成多个 element clause。
非 constant array 在 planner-time 不能使用，但可能进入 runtime pruning。
第四类是 nullness test。
`IS NULL` 和 `IS NOT NULL` 直接影响 null-accepting partition 或 default partition 的处理。
匹配失败有多个返回值，不是简单 true/false。
| 返回值 | 语义 |
| --- | --- |
| `PARTCLAUSE_NOMATCH` | 当前 key 不匹配，可以试下一个 key。 |
| `PARTCLAUSE_MATCH_CLAUSE` | 得到一个 `PartClauseInfo`。 |
| `PARTCLAUSE_MATCH_NULLNESS` | 得到 nullness 信息。 |
| `PARTCLAUSE_MATCH_STEPS` | 已递归生成 steps。 |
| `PARTCLAUSE_MATCH_CONTRADICT` | clause 自相矛盾。 |
| `PARTCLAUSE_UNSUPPORTED` | 形态或属性不支持 pruning。 |
这个多状态返回值很重要。
它避免把“不是这个 key”误判成“这个 clause 永远不能 pruning”。
### 6.7 planner-time expression 边界
`OpExpr` 匹配到 partition key 后，源码检查另一侧 `expr`。
planner-time 规则非常保守：
```text
if target == PARTTARGET_PLANNER:
    expr 必须已经是 Const
    operator 必须 immutable
```
如果 `expr` 不是 `Const`：
1. planner-time 直接返回 unsupported。
2. runtime target 会继续检查是否包含 Var。
3. 包含 Var 一律不能用于 pruning。
4. 包含 volatile function 一律不能用于 pruning。
5. 包含 `PARAM_EXEC` 只能用于 `PARTTARGET_EXEC`。
6. 不含 `PARAM_EXEC` 的非 Const 可标记为 `has_mutable_arg`，给 initial pruning 使用。
这里要区分三种“参数”。
| 形态 | planner-time 可能性 | 说明 |
| --- | --- | --- |
| literal Const | 可以 | 最直接的 planner-time pruning。 |
| `PARAM_EXTERN` 被 custom plan 替换成 Const | 可以 | `planner.c` 注释说明 bound param 标记 `PARAM_FLAG_CONST` 时可替换。 |
| generic plan 中的 `PARAM_EXTERN` | 通常不可以 | 可留给 executor startup pruning。 |
| `PARAM_EXEC` | 不可以 | 只能在 per-scan pruning 中使用。 |
`stable` 函数也要分清。
`eval_const_expressions()` 普通规划路径只安全折叠 immutable 函数。
`estimate_expression_value()` 为估算可以更冒险地考虑 stable，但那是估算，不是改写 plan shape 的 pruning 依据。
### 6.8 生成 op steps
`gen_prune_steps_from_opexps()` 会根据 partition strategy 组合 key clauses。
range partition 需要 prefix。
例如 `(a, b)` range partition 下：
```sql
WHERE a = 10 AND b < 20
```
可以形成 `(a, b)` 的 prefix lookup。
但只有 `b < 20` 没有 `a`，通常不能精确 range pruning。
hash partition 需要能计算 hash 的 key。
list partition 可以处理 equality，也对 `<>` 有特殊支持。
源码会为重复或组合的 key clause 生成多个 `PartitionPruneStepOp`。
多个 step 最后用 `INTERSECT` 组合。
### 6.9 `get_matching_partitions()` 执行 pruning VM
执行逻辑：
```text
allocate results[num_steps]
for step in pruning_steps:
  if StepOp:
    results[step_id] = perform_pruning_base_step()
  if StepCombine:
    results[step_id] = perform_pruning_combine_step()
final_result = results[last_step_id]
convert bound_offsets to partition indexes
include null partition if scan_null
include default partition if scan_default
return Bitmapset(partition indexes)
```
`perform_pruning_base_step()` 会把 step 中的 expression 转成 datum。
planner-time 下 expression 已经是 `Const`。
runtime 下会通过 `ExprState` 和 `ExprContext` 求值。
得到 datum 后，再按策略分派：
| 策略 | 函数 |
| --- | --- |
| hash | `get_matching_hash_bounds()` |
| list | `get_matching_list_bounds()` |
| range | `get_matching_range_bounds()` |
返回的中间结果不是直接 partition index。
它先是 bound offset、`scan_default`、`scan_null`。
最后才根据 `boundinfo->indexes` 映射到 partition index。
这解释了 default partition 的特殊性。
一些查询范围落在没有显式分区覆盖的 gap 上时，default partition 仍可能需要扫描。
### 6.10 child expansion 与 AppendPath.subpaths
`live_parts` 只控制分区表 child 的展开。
后续 `allpaths.c` 仍然按 append relation 流程工作：
```text
set_append_rel_size()
  -> iterate root->append_rel_list for this parent
  -> copy joininfo and targetlist to child
  -> set_rel_size(child)
  -> accumulate parent rows/width from live child
set_append_rel_pathlist()
  -> set_rel_pathlist(child)
  -> collect non-dummy childrels
  -> add_paths_to_append_rel(root, rel, live_childrels)
add_paths_to_append_rel()
  -> collect cheapest child paths
  -> create_append_path(... unparameterized ...)
  -> create partial / parallel append paths
  -> generate ordered append paths
  -> generate parameterized Append paths when every child can supply matching required_outer
```
所以 `AppendPath.subpaths` 是多层过滤后的结果。
过滤一：planner-time partition pruning 没有展开 pruned child。
过滤二：constraint exclusion 或 child size 阶段把 child 标成 dummy。
过滤三：child 自己 pathlist 生成失败或不合法。
过滤四：append parent 只为某种 path 形态收集合格 child subpath。
过滤五：`add_path()` 可能剪掉某个 AppendPath 候选。
本节主问题只覆盖第一层，但诊断必须能区分后四层。
## 7. initial / exec pruning 与 planner-time 的边界
`make_partition_pruneinfo()` 是理解参数边界的关键函数。
它不改变 `live_parts`。
它为 executor 构造在已有 subplans 上 pruning 的信息。
主流程：
```text
make_partition_pruneinfo(root, parentrel, subpaths, prunequal)
  -> scan subpaths
  -> map partition child relid to subplan index
  -> group relevant parent partitioned rels
  -> make_partitionedrel_pruneinfo()
  -> build PartitionPruneInfo
  -> append to root->partPruneInfos
  -> return pruneinfo index
```
`subpaths` 是已经生成出来的 child scan path 列表。
`relid_subplan_map` 是从 partition child relid 到 subpath index 的临时数组。
这里的“subplan”命名有历史味道。
在 `make_partition_pruneinfo()` 阶段它仍然对应 subpath 位置。
之后 createplan / setrefs 会把它落实成 executor 能使用的 subplan index。
### 7.1 startup pruning
`PARTTARGET_INITIAL` 的含义是 executor startup。
它可以使用不含 `PARAM_EXEC` 的可求值表达式。
例如 generic prepared statement 的 `PARAM_EXTERN`。
它不能使用 outer tuple 变化带来的值。
如果 `context.has_mutable_op` 或 `context.has_mutable_arg` 为 true，源码保留 `initial_pruning_steps`。
如果没有这些信息，plan-time pruning 已经做了所有能做的，startup pruning 没必要重复。
### 7.2 per-scan pruning
`PARTTARGET_EXEC` 的含义是每次 plan node scan 时都可能重新裁剪。
它用于含 `PARAM_EXEC` 的 pruning 表达式。
`pull_exec_paramids()` 只收集 `paramkind == PARAM_EXEC` 的 Param。
`get_partkey_exec_paramids()` 再从实际 steps 中提取真正用到的 Param IDs。
如果没有实际用到 exec param，`exec_pruning_steps` 会被置空。
这避免 executor 在每次 rescan 时做没有必要的 pruning work。
### 7.3 三阶段边界总结
| 阶段 | 输入值可用性 | 结果形态 | 典型例子 |
| --- | --- | --- | --- |
| planner-time | 已是 `Const`，operator immutable | 不展开 child，不生成 subpath | literal 常量，自定义计划中被替换成 Const 的参数。 |
| executor startup | 执行开始时可求值，不依赖 outer tuple | 已有 subplans 中标记 inactive | generic plan 的 external parameter，stable expression。 |
| executor per-scan | 每次 rescan 可能变化的 `PARAM_EXEC` | 每次 scan 重新决定 active subplans | nested loop inner 依赖外层 key。 |
这张表是本节最重要的诊断工具。
不要用“是否能裁剪”概括三者。
正确问题是：
```text
裁剪发生在哪个生命周期点，改变的是 plan shape、startup active set，还是每次 scan 的 active set？
```
## 8. 生命周期 / ownership / cleanup
### 8.1 谁创建
`plancat.c` 创建或填充 `RelOptInfo` 的分区元数据。
`inherit.c` 调用 `prune_append_rel_partitions()` 创建 `live_parts` bitmap。
`partprune.c` 创建临时 `PartitionPruneStep`、`PartitionPruneContext` 和结果 bitmap。
`inherit.c` 只为 live partition 创建 child RTE、`AppendRelInfo` 和 child `RelOptInfo`。
`make_partition_pruneinfo()` 创建运行期 pruning 所需的 `PartitionPruneInfo`。
### 8.2 谁持有
planner-local 状态由 `PlannerInfo` 和当前 planning memory context 持有。
`RelOptInfo.live_parts` 挂在 parent rel 上。
`RelOptInfo.part_rels` 按 partition index 指向 child rel。
`root->append_rel_list` 持有 `AppendRelInfo` 平铺列表。
`root->partPruneInfos` 暂存运行期 pruning 信息。
`PlannerGlobal.partPruneInfos` 进入最终 `PlannedStmt`。
### 8.3 谁释放
planner-time 临时状态通常随 planner memory context reset 释放。
未被选择的 `Path`、`PartitionPruneStep` 工作状态、临时 bitmap 都没有单独 refcount。
最终 plan 中需要的 `PartitionPruneInfo` 会被复制或保留到 plan context。
executor 侧再在 plan state 初始化时建立自己的 pruning state。
本节不涉及 `ResourceOwner`。
它没有 buffer pin、relation refcount 或 WAL record 需要在事务 abort 时逐项释放。
### 8.4 ERROR / abort 时谁兜底
planner 阶段如果 ERROR，当前 query planning 的 memory context 被释放。
已经打开的 relation 通常由 planner / relcache 路径按调用约定关闭或由错误恢复清理。
分区 child 在 `expand_partitioned_rtentry()` 中 `try_table_open()` 后会 `table_close(childrel, NoLock)`。
锁保留到事务结束，避免计划期间使用的 rel 被并发破坏。
如果规划失败，事务 abort 或语句级错误清理会释放锁。
分区 pruning 自己没有自定义 cleanup hook。
它依赖内存上下文和上层 relation open/close 协议。
### 8.5 长期对象如何失效
partition bound、partition key、partition constraint 来自 relcache 和 catalog。
计划依赖相关 relation OID、syscache invalidation 和 plan cache invalidation。
DDL 改变分区结构时，旧 cached plan 需要失效或重建。
planner-time pruning 不能把 catalog state 当成永久事实。
它只能把“本次规划 snapshot 下的分区结构”固化进 plan。
## 9. 正确性机制层次
分区裁剪的正确性不是单一机制保证。
它由多层边界叠加。
| 层次 | 机制 | 保证 | 不保证 |
| --- | --- | --- | --- |
| 语义边界 | `PARTTARGET_PLANNER` 只接受 Const 和 immutable operator | plan shape 不依赖执行期变化 | 不保证所有可裁剪分区都提前裁掉。 |
| 表达式安全 | 拒绝 Vars、volatile functions、错误阶段的 Params | 不用未知值做提前裁剪 | 不替代普通 qual 执行。 |
| opfamily | 操作符必须属于 partition opfamily，或 list `<>` 特例 | bound 比较和 partition key 语义一致 | 不保证用户自定义 operator 的业务语义正确。 |
| strictness | `op_strict()` | null 输入不会误命中普通 bound | 不等于所有 null 逻辑都简单。 |
| collation | `PartCollMatchesExprColl()` | 文本比较顺序和分区边界一致 | 不修复 collation 变更造成的外部问题。 |
| default partition | `scan_default` | gap 或无法归属值不会被错误丢弃 | 不保证 default 一定能被精确裁掉。 |
| catalog / relcache | relation lock、relcache invalidation、plan invalidation | 分区结构变更不会静默复用错误计划 | 不提供执行期动态重规划。 |
| executor pruning | `PartitionPruneInfo` mappings | 在保留 plan shape 下安全跳过 subplans | 不降低规划阶段 child path 生成成本。 |
两个常见错误推理需要避免。
第一，不是所有 `WHERE partition_key op expr` 都能 planner-time prune。
`expr` 必须在规划期成为 `Const`。
第二，partition pruning 不替代 executor qual。
即使一个分区被保留，scan node 仍要执行原始 qual。
Pruning 只是证明某些分区不可能有匹配行。
它不是证明保留分区内所有行都匹配。
## 10. 错误路径 / 异常路径 / fallback
### 10.1 `enable_partition_pruning = off`
`prune_append_rel_partitions()` 直接返回全集。
后续仍可能发生 constraint exclusion。
但分区 pruning 的主路径被关闭。
这适合做实验对照。
不要把关闭后仍消失的 child 都归因于 partition pruning。
### 10.2 没有 usable pruning steps
有 qual 不等于有 step。
常见原因：
| 原因 | 例子 |
| --- | --- |
| clause 不匹配 partition key | `payload = 'x'`。 |
| collation 不匹配 | 文本 key 使用不同 collation 的比较。 |
| operator 不在 opfamily | 自定义 operator 没有加入 partition opfamily。 |
| expr 不是 planner-time Const | `ts = CURRENT_DATE`。 |
| expr 含 Var | `p.ts = other.ts`。 |
| expr 含 volatile function | `ts = random_date()`。 |
| range partition 缺少前缀 key | `(a,b)` 分区只有 `b = 1`。 |
fallback 是返回全集。
这保持 correctness，但丢失规划期开销收益。
### 10.3 contradictory clauses
`Const false`、`Const NULL`、`IS NULL` 与 strict operator 冲突、`IS NULL` 与 `IS NOT NULL` 冲突，都可能让 `context.contradictory = true`。
planner-time 主路径会返回空集合。
parent appendrel 可能被后续标成 dummy。
运行期 pruning info 构造中如果发现 contradiction，源码注释说这本不应发生。
它会选择禁用 run-time pruning，返回 `NIL`。
这是保守 fallback。
### 10.4 default partition
default partition 是最容易误判的异常路径。
range 或 hash bound lookup 可能遇到 `boundinfo->indexes[i] < 0`。
这表示某些值没有显式 partition 覆盖。
如果存在 default partition，就必须设置 `scan_default`。
因此某个看似精确的范围查询仍可能保留 default partition。
除非额外约束能证明 default 不可能命中。
### 10.5 detached / dropped partition
`expand_partitioned_rtentry()` 中 `try_table_open(childOID, lockmode)` 可能返回 `NULL`。
注释说明这是分区最近被 detach 后又 drop 的情况。
源码行为是把该 partition 从 `live_parts` 删除并继续。
这不是 pruning 证明。
这是并发 DDL 下的容错路径。
### 10.6 stack depth
分区树可能很深。
`expand_partitioned_rtentry()` 和 `set_append_rel_size()` 都有 `check_stack_depth()`。
深层分区树的异常不是“裁剪失败”。
它可能直接触发 stack depth 保护。
## 11. 成本、资源与跨模块传播
### 11.1 成本变量
planner-time pruning 的收益主要随分区数扩张。
关键变量：
| 变量 | 放大路径 |
| --- | --- |
| `nparts` | bound lookup、bitmap、child expansion 和 path generation 的上限。 |
| live partition 数 | child RTE、child `RelOptInfo`、child pathlist 的实际数量。 |
| partition key 数 | stepcmpfuncs 大小约为 `partnatts * num_steps`。 |
| pruning step 数 | OR / IN / ScalarArrayOpExpr 会增加 step 数。 |
| child index 数 | 每个 live child 后续 `create_index_paths()` 还会扫描索引列表。 |
| child stats 访问 | live child 越多，plancat 和 stats 读取越多。 |
| parameterized path 种类 | `add_paths_to_append_rel()` 要为所有 child 匹配 required_outer。 |
计划时间通常不是 pruning lookup 本身最贵。
真正昂贵的是没有裁掉时的后续放大：
```text
child RTE
  -> child RelOptInfo
  -> child size estimate
  -> child index metadata
  -> child pathlist
  -> append ordered / parameterized / parallel path combinations
```
### 11.2 planner-time pruning 的资源收益
planner-time pruning 可以减少：
1. child relation open。
2. `AppendRelInfo` 数量。
3. child `RelOptInfo` 数量。
4. child targetlist 和 qual translation。
5. child pathlist 构造。
6. parent rows / width 聚合工作。
7. AppendPath subpaths 数量。
8. createplan 递归深度。
9. executor plan node 数量。
initial pruning 不能减少前 7 项。
它只能减少 executor startup 后实际扫描的 subplans。
per-scan pruning 还会在 rescan 中付出额外判断成本。
### 11.3 和 `allpaths.c` 的边界
`allpaths.c` 不负责从 partition bound 推断 live partitions。
它只消费已经展开出来的 child rel。
`set_append_rel_size()` 仍会检查 constraint exclusion。
`set_append_rel_pathlist()` 仍会给 child 生成 path。
`add_paths_to_append_rel()` 仍会收集 cheapest、startup、partial、parallel、ordered 和 parameterized subpaths。
所以如果想定位分区太多导致规划慢，要先问：
```text
prune_append_rel_partitions() 返回了多少 live_parts？
```
再问：
```text
每个 live child 在 path generation 中放大了多少候选？
```
### 11.4 和 prepared statement / plan cache 的边界
plan cache 会在 custom 和 generic plan 之间选择。
custom plan 可能因为 bound params 被当作 Const 而发生 planner-time pruning。
generic plan 不能依赖某次执行的参数值改变 plan shape。
因此同一条 `PREPARE` 语句前几次和后几次的计划可能不同。
这不是 partition pruning 不稳定。
这是 plan cache 在规划成本和执行成本之间做选择。
诊断时要记录：
```sql
SHOW plan_cache_mode;
EXPLAIN EXECUTE q(...);
EXPLAIN (ANALYZE, VERBOSE) EXECUTE q(...);
```
必要时用 `force_custom_plan` 和 `force_generic_plan` 对照。
### 11.5 和 relcache / invalidation 的边界
partition bound 来自 relcache `PartitionDesc`。
planner 会把本次规划依赖的 relation OID 和 invalidation 信息放进计划依赖。
分区 DDL 后旧计划应失效。
这部分正确性依赖 relcache invalidation 和 plan cache revalidation。
partition pruning 自己不监听 DDL。
它只消费当前 planner 看到的 catalog state。
### 11.6 和 executor 的边界
planner-time pruning 改变 plan tree 大小。
executor pruning 改变 active subplans。
两者都使用 `PartitionPruneStep` 概念，但生命周期和输入值不同。
不要用 executor 的 `Subplans Removed` 反推出 planner-time pruning 的全部行为。
被 planner-time 删除的 subplan 根本不会出现在 executor 的 subplan 列表里。
## 12. 观测与诊断入口
### 12.1 能直接观测什么
`EXPLAIN` 可以看到最终 plan tree。
如果 `Append` 下只有少量 child，说明某些 child 没有进入最终 plan。
但它不能直接告诉你这些 child 是在哪里消失的。
`EXPLAIN (ANALYZE)` 可能显示 `Subplans Removed`。
这通常反映 executor startup pruning。
per-scan pruning 的效果可能表现为某些 subplan `never executed` 或 loops 很少。
但不同版本和计划形态的显示细节需要以本地结果为准。
### 12.2 只能推断什么
planner-time `live_parts` 不直接暴露在 SQL 视图中。
`PartitionPruneStep` 列表也不在普通 EXPLAIN 中完整显示。
要确认必须用断点或临时日志。
推荐断点：
```text
src/backend/optimizer/util/inherit.c: expand_partitioned_rtentry
src/backend/partitioning/partprune.c: prune_append_rel_partitions
src/backend/partitioning/partprune.c: match_clause_to_partition_key
src/backend/partitioning/partprune.c: get_matching_partitions
src/backend/optimizer/path/allpaths.c: add_paths_to_append_rel
src/backend/partitioning/partprune.c: make_partition_pruneinfo
```
建议在断点处打印：
```text
rel->relid
rel->nparts
bms_num_members(rel->live_parts)
gcontext.target
gcontext.has_mutable_arg
gcontext.has_exec_param
list_length(gcontext.steps)
list_length(live_childrels)
list_length(subpaths)
```
### 12.3 诊断流程
面对“为什么分区没裁掉”，按顺序问：
1. 父表是不是 partitioned rel，`rel->part_scheme` 是否非空？
2. `enable_partition_pruning` 是否开启？
3. qual 是否进入 `rel->baserestrictinfo`？
4. qual 是否匹配 `rel->partexprs` 中的 partition key？
5. operator 是否在分区 opfamily 中？
6. collation 是否匹配？
7. operator 是否 strict？
8. planner-time 下另一侧是否已经是 `Const`？
9. 是否存在 volatile function 或 Var？
10. 是否是 `PARAM_EXEC`，只能留给 per-scan？
11. range 多列分区是否缺少前缀 key？
12. default partition 是否必须保留？
13. child 是否已展开但被 constraint exclusion 设为 dummy？
14. child path 是否生成后被 append path 收集阶段排除？
这个顺序能避免一上来改 cost 参数。
partition pruning 失败大多数不是 cost 问题。
它首先是语义匹配和生命周期问题。
### 12.4 用源码计数器验证
临时加日志时，优先放在 `prune_append_rel_partitions()` 返回前。
观察三类数：
```text
nparts
list_length(pruning_steps)
bms_num_members(result)
```
再在 `expand_partitioned_rtentry()` 中观察 `try_table_open()` 次数。
如果 `result` 已经很小，但 `try_table_open()` 仍很多，说明理解的 relation 层级不对，可能是在子分区层递归展开。
如果 `result` 是全集，再看 `match_clause_to_partition_key()` 的返回值。
不要先看 `AppendPath.total_cost`。
### 12.5 SQL 对照实验的观测粒度
`EXPLAIN` 粒度是单 query 的计划形状。
`pg_stat_statements` 粒度是归一化语句的累计统计，不能告诉你某次是否 custom plan。
`pg_locks` 可以看到分区 relation lock，但不区分这些锁来自 pruning 前展开还是其它路径。
`perf` 可以看到规划时间热点，比如 `set_append_rel_size()`、`create_index_paths()` 或 `get_relation_info()`。
`gdb` 才能直接看 `live_parts` 和 pruning steps。
## 13. 常见误区
### 13.1 把 partition pruning 当成 cost-based 选择
planner-time pruning 是语义证明。
它不是因为某个分区 cost 高才丢掉。
如果不能证明分区不可能命中，就必须保留。
### 13.2 以为 stable 函数可以 planner-time prune
stable 只表示单个 statement 内结果稳定。
它不等于可固化进 cached plan shape。
planner-time pruning 使用的是更严格的 immutable / Const 边界。
### 13.3 把 prepared statement 的一次结果推广到所有执行
custom plan 和 generic plan 可能走不同 pruning 阶段。
同样的 SQL 文本可能前几次 planner-time prune，后面 generic plan 只做 runtime prune。
诊断必须记录 `plan_cache_mode` 和是否 generic。
### 13.4 把 `Subplans Removed` 理解为 planner-time pruning
`Subplans Removed` 是 executor 侧公开影子。
planner-time 被删除的 subplan 不会出现在 plan 中。
所以它也不会被计入 `Subplans Removed`。
### 13.5 忽略 default partition
default partition 是 correctness fallback。
只要查询范围可能落入未覆盖 bound，default 就不能被随意裁掉。
这会让“明明范围很窄”的查询仍保留 default partition。
### 13.6 忽略多列 range 分区的 prefix 规则
range partition bound 是按 key 顺序比较。
没有前缀 key 时，后续 key 的条件通常不能单独定位 bound 区间。
`WHERE b = 1` 对 `(a,b)` range partition 的裁剪能力有限。
### 13.7 把 child 不见了都归因于 partition pruning
child 可能没展开。
也可能展开后被 constraint exclusion 标 dummy。
也可能有 path 但没有被 append parent 收集。
也可能 append path 候选被 `add_path()` 剪掉。
需要按阶段定位。
## 14. 课堂实验
### 实验 1：常量和 stable 表达式对照
目标：观察 planner-time 和 startup pruning 的边界。
步骤：
```sql
DROP TABLE IF EXISTS mlog CASCADE;
CREATE TABLE mlog(ts date, payload text) PARTITION BY RANGE (ts);
CREATE TABLE mlog_2026_01 PARTITION OF mlog
  FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE mlog_2026_02 PARTITION OF mlog
  FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
CREATE TABLE mlog_default PARTITION OF mlog DEFAULT;
EXPLAIN SELECT * FROM mlog WHERE ts = DATE '2026-02-14';
EXPLAIN SELECT * FROM mlog WHERE ts = CURRENT_DATE;
```
源码观察：
1. 在 `prune_append_rel_partitions()` 断点，记录 `list_length(gcontext.steps)`。
2. 在 `match_clause_to_partition_key()` 断点，看 `expr` 是否 `Const`。
3. 在 `expand_partitioned_rtentry()` 断点，看 `bms_num_members(relinfo->live_parts)`。
预期判断：
常量查询可以 planner-time 裁掉无关分区。
`CURRENT_DATE` 通常不能 planner-time 固化。
如果 default partition 被保留，回到 `scan_default` 解释。
### 实验 2：custom plan 与 generic plan
目标：观察 `PARAM_EXTERN` 在不同 plan cache 策略下的阶段变化。
步骤：
```sql
PREPARE q(date) AS SELECT * FROM mlog WHERE ts = $1;
SET plan_cache_mode = force_custom_plan;
EXPLAIN EXECUTE q(DATE '2026-02-14');
SET plan_cache_mode = force_generic_plan;
EXPLAIN EXECUTE q(DATE '2026-02-14');
```
源码观察：
1. 在 `eval_const_expressions_mutator()` 观察 `PARAM_EXTERN` 是否被替换成 `Const`。
2. 在 `prune_append_rel_partitions()` 观察 planner-time steps。
3. 在 `make_partition_pruneinfo()` 观察是否生成 `initial_pruning_steps`。
预期判断：
custom plan 可能出现 planner-time pruning。
generic plan 更可能保留 subplans，并依赖 executor startup pruning。
### 实验 3：多列 range prefix
目标：验证 range 分区 key prefix 对 pruning 能力的影响。
步骤：
```sql
DROP TABLE IF EXISTS r2 CASCADE;
CREATE TABLE r2(a int, b int, payload text) PARTITION BY RANGE (a, b);
CREATE TABLE r2_p1 PARTITION OF r2 FOR VALUES FROM (0, 0) TO (10, 0);
CREATE TABLE r2_p2 PARTITION OF r2 FOR VALUES FROM (10, 0) TO (20, 0);
CREATE TABLE r2_p3 PARTITION OF r2 FOR VALUES FROM (20, 0) TO (30, 0);
EXPLAIN SELECT * FROM r2 WHERE a = 12 AND b = 3;
EXPLAIN SELECT * FROM r2 WHERE b = 3;
```
源码观察：
1. 在 `gen_prune_steps_from_opexps()` 观察 `keyclauses`。
2. 在 `perform_pruning_base_step()` 观察 `nvalues`。
3. 在 `get_matching_range_bounds()` 观察返回的 bound offsets。
预期判断：
带 `a` 的查询能形成 range prefix。
只有 `b` 的查询不能同等精确。
### 实验 4：区分 pruning 和 constraint exclusion
目标：确认 child 消失发生在哪个阶段。
步骤：
```sql
SET enable_partition_pruning = off;
EXPLAIN SELECT * FROM mlog WHERE ts = DATE '2026-02-14';
SET constraint_exclusion = on;
EXPLAIN SELECT * FROM mlog WHERE ts = DATE '2026-02-14';
```
源码观察：
1. `prune_append_rel_partitions()` 是否直接返回全集。
2. `relation_excluded_by_constraints()` 是否把 child 设为 dummy。
3. `set_append_rel_pathlist()` 收到多少 `live_childrels`。
预期判断：
关闭 partition pruning 不等于所有 child 必然进入最终 plan。
仍要看 constraint exclusion 和 child dummy path。
## 15. 讨论题
1. 为什么 `PARTTARGET_PLANNER` 只接受 `Const`，而不是直接执行 stable 函数？
2. custom plan 中参数被替换成 `Const` 后 planner-time pruning 是安全的；generic plan 中为什么不能这么做？
3. `PARAM_EXEC` 为什么只能用于 per-scan pruning，而不是 executor startup pruning？
4. OR 子句中有一个 arm 不能匹配 partition key 时，为什么不能忽略这个 arm？
5. default partition 为什么经常要保留？它和 range bound gap 的关系是什么？
6. `AppendRelInfo` 为什么只为 live partition 创建？如果先为所有分区创建再裁剪，会多付出哪些成本？
7. `PartitionPruneInfo.subplan_map` 为什么不能简单用 partition index 代替？
8. 如何区分一个分区是 planner-time pruning 没展开，还是展开后被 constraint exclusion 标 dummy？
## 16. 本节小结
planner-time partition pruning 的主链路是：
```text
set_relation_partition_info()
  -> expand_partitioned_rtentry()
  -> prune_append_rel_partitions()
  -> gen_partprune_steps(PARTTARGET_PLANNER)
  -> get_matching_partitions()
  -> live_parts
  -> only live child RTE / AppendRelInfo / RelOptInfo
  -> child path generation
  -> AppendPath.subpaths
```
核心状态是 `RelOptInfo` 上的 partition metadata 和 `live_parts`。
`live_parts` 的成员是 partition index，不是 RT index。
它决定哪些 partition child 会被展开。
`AppendRelInfo` 只为 live child 创建。
`AppendPath.subpaths` 是后续 path 阶段的结果，不是 pruning 函数直接返回的对象。
正确性边界是表达式生命周期。
planner-time 只能使用已经成为 `Const` 的比较值和 immutable operator。
generic external params、stable expressions 和 `PARAM_EXEC` 必须留给 executor initial 或 per-scan pruning。
异常路径中，禁用 pruning 或没有 usable step 会返回全集。
矛盾 qual 会返回空集合。
default partition 会因 gap 和 null/default 语义被保守保留。
运行期 pruning 构造失败时会选择禁用运行期 pruning，而不是冒险裁错。
成本上，planner-time pruning 的价值不只是少扫分区。
它更早减少 child RTE、`AppendRelInfo`、child `RelOptInfo`、child pathlist、append subpath 和 createplan 递归。
initial pruning 和 per-scan pruning不能回收这些规划期开销。
观测上，`EXPLAIN` 只能显示最终 plan shape。
`Subplans Removed` 是 executor 侧影子，不代表 planner-time pruning。
要直接确认，需要断点或日志观察 `live_parts`、`gcontext.steps` 和 `subpath` mapping。
可迁移的系统规律是：
```text
越早裁剪，越能降低搜索空间；
越早裁剪，越需要更强的语义不变性；
当值的生命周期晚于 plan shape，系统必须保留结构并把选择推迟到运行期。
```
这个规律不只适用于分区表。
它也适用于参数化路径、join order legality、predicate pushdown、partial index 匹配和 cached plan 选择。
