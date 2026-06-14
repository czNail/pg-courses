# PostgreSQL partition routing 与 ModifyTable
## 课程定位
前置知识：已经读过 04 目录中 `ExecutorStart`、`TupleTableSlot`、`ExprContext`、`ModifyTable` 和 executor cleanup 的章节。
还应理解 PostgreSQL declarative partitioning 的 SQL 语义，以及 `INSERT`、`UPDATE`、`BEFORE/AFTER` trigger、RLS 和 `WITH CHECK OPTION` 的基本行为。
本节唯一主问题：
```text
INSERT/UPDATE 如何在 executor 中选择目标分区，partition constraint、tuple conversion 和 trigger 顺序如何协作？
```
核心矛盾：执行器必须按每一行的运行时值选择正确 leaf partition，并允许 trigger 改写 tuple；但分区层级、列顺序差异、default partition、RLS、约束、transition table、RETURNING 和跨分区 UPDATE 又要求状态长期可复用、可清理、可解释。
一句话目标：读完后，你应能从一个分区 DML 的异常或慢路径，判断问题落在路由搜索、tuple descriptor 转换、partition constraint、trigger 改写、row movement、FDW/table AM，还是观测口径上。
本课基于本地 `/home/nail/postgres` 源码。
源码基线：
```text
branch: master
commit: 0e1f1ed157e9
```
本课只讲 executor 侧 tuple routing。
planner 侧 partition pruning、Append 子计划生成、partition bound catalog 表达方式，只作为边界出现。
不要把本节理解成分区功能总览。
本节只围绕 `ModifyTable` 中一行 tuple 如何被路由、转换、检查和写入展开。
## 1. 本节在总主线中的位置
04 目录前面的课程已经建立了 executor 的几条主线。
`ExecutorStart()` 把 planner 产物固化成 `EState` 和 `PlanState`。
`ExecProcNode()` 让节点按需拉取 tuple。
`TupleTableSlot` 承载虚拟 tuple、物理 tuple、descriptor 和 pin 生命周期。
`ExprContext` 给表达式提供 per-tuple scratch 空间。
`ModifyTable` 把上游 tuple 流变成 INSERT / UPDATE / DELETE / MERGE 的副作用。
第 24 节已经把 `ModifyTable` 作为 DML 边界讲过。
这一节把其中的 partition routing 单独拉出来。
原因是分区路由不是 `ExecInsert()` 里的一个小 helper。
它把 executor 的几个难点压在同一行 tuple 上：
- 每行要根据运行时分区键选择 leaf partition。
- 选择过程可能穿过多级 partitioned table。
- 每一级可能有不同 tuple descriptor。
- 目标 leaf 的 `ResultRelInfo` 可能懒创建，也可能复用 `ModifyTableState` 中已有对象。
- `BEFORE ROW` trigger 能改写 tuple。
- 改写后的 tuple 还要重新满足目标 partition constraint。
- UPDATE 修改分区键时，语义会变成源分区 DELETE 加根表 INSERT。
- RETURNING、transition table 和 AFTER trigger 仍要保持 SQL 可见顺序。
这节课的主线可以压缩成：
```text
subplan tuple
  -> ModifyTable
  -> ExecPrepareTupleRouting()
  -> ExecFindPartition()
  -> root/child tuple conversion
  -> trigger / generated / WCO / constraint
  -> table AM or FDW insert/update/delete
  -> AFTER trigger / transition / RETURNING
```
这个顺序不是所有分支的完整代码路径。
它是诊断 partitioned INSERT/UPDATE 时最稳定的 mental model。
如果你在生产系统看到 `no partition of relation found for row`、`violates partition constraint`、`moving row to another partition during a BEFORE trigger is not supported`，应该能回到这条链路定位。
本节和相邻课程的边界如下。
第 09 节负责解释 junk attr 和 row identity。
本节只用它说明 UPDATE 如何找到源 tuple。
第 12 节和第 05 节负责 executor memory context 和 cleanup。
本节只用它说明 `PartitionTupleRouting`、slot、Relation 和 FDW callback 的 ownership。
第 24 节负责完整 DML 边界。
本节只深挖 `ModifyTable` 内 partitioned INSERT/UPDATE 的路由、转换和触发器顺序。
后续如果继续扩展，应把 runtime pruning 和 routing/pruning 边界拆成另一课。
## 2. 核心矛盾与一句话运行模型
一句话运行模型：
```text
ModifyTable 先把候选 tuple 放在 root table 的语义下，
再用 PartitionTupleRouting 在 executor 中按 partition key 找到 leaf ResultRelInfo，
必要时把 tuple 转成 leaf descriptor，
随后在 leaf 上执行 trigger、generated column、WCO、constraint、table AM/FDW 和 AFTER/RETURNING。
```
这里有一个容易误读的点。
“选择分区”不是一条 catalog lookup。
它是一个运行时状态机。
它要消费当前 tuple 的 Datum。
它要用 `ExprContext` 计算 partition key expression。
它要处理多级 partitioned table。
它要缓存已触达 leaf 的 `ResultRelInfo`。
它要在 descriptor 不同的层级之间转换 slot。
它还要保持 trigger 能看到正确 relation 和 `tableoid`。
核心 tension 可以写成：
```text
每行动态选择目标 leaf partition 的语义精确性
  vs
高频 INSERT/UPDATE hot path 不能为所有分区预建完整执行状态
```
PostgreSQL 的选择是懒初始化。
`ExecSetupPartitionTupleRouting()` 只建立 root dispatch。
真正 leaf partition 的 `ResultRelInfo` 在第一次路由到该 leaf 时才创建或复用。
这让单行 INSERT into partitioned table 的启动成本较低。
代价是 runtime 路径里有更多条件分支：
- 这个 leaf 是否已经触达。
- 是否能复用 `ModifyTableState` 中已有 `ResultRelInfo`。
- 是否要打开 leaf relation 和 indexes。
- 是否要为 root-to-child conversion 建 slot。
- 是否要为 FDW routing 调 `BeginForeignInsert`。
- 是否要为 WCO、RETURNING 和 ON CONFLICT 转换表达式。
本节的第二个 tension 是 trigger 顺序。
SQL 语义允许 `BEFORE ROW` trigger 修改即将写入的 row。
但是 partition routing 必须先知道要把 tuple 送到哪个 leaf。
PostgreSQL 对 INSERT 的运行时选择是：
- 先根据当前 root tuple 选择 leaf。
- 转成 leaf rowtype。
- 在 leaf 上执行 `BEFORE ROW INSERT` trigger。
- 如果 trigger 改写 clone partition tuple 导致不再满足该 leaf constraint，就报错。
- 不在 trigger 后重新路由到另一个 leaf。
这个设计避免了 trigger 中任意代码导致无限路由、跨分区 trigger 顺序重排和 transition table 语义混乱。
UPDATE 的选择不同。
普通 UPDATE 先在源 leaf 上做 `BEFORE ROW UPDATE` trigger。
随后检查新 tuple 是否仍满足源 leaf partition constraint。
如果不满足，并且语句是针对 root partitioned table 的 UPDATE，则进入跨分区 UPDATE。
跨分区 UPDATE 实际执行为：
```text
DELETE old tuple from source leaf
  -> convert new tuple to root rowtype
  -> INSERT into root partitioned table
  -> executor routing sends it to destination leaf
```
这不是纯粹的 heap update。
CTID 链不能跨 relation。
所以 executor 通过 delete+insert 模拟 row movement，并额外处理并发重试、foreign key AFTER trigger 和 RETURNING。
把这两个路径放在一起看，本节的核心不变量是：
```text
route chooses a ResultRelInfo;
conversion makes the slot match that ResultRelInfo;
trigger may change tuple contents;
partition constraint validates that the current tuple still belongs there;
table AM/FDW only sees a tuple whose descriptor matches the target relation.
```
任何诊断都要围绕这几个动词排序。
如果只看错误信息，很容易把“没有可匹配分区”和“已路由后不满足目标分区约束”混在一起。
如果只看 EXPLAIN，也看不到每行到底命中了哪个 leaf。
如果只看 `ResultRelInfo` 指针，也看不到它是 borrowed 还是 routing 过程中动态创建。
## 3. 核心文件分工与阅读顺序
阅读顺序按 runtime mental model 排列，不按文件名排序。
| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/executor/nodeModifyTable.c` | `ExecInitModifyTable()`、`ExecInsert()`、`ExecUpdateAct()`、`ExecCrossPartitionUpdate()`、`ExecPrepareTupleRouting()` 的主链路。 |
| 2 | `src/backend/executor/execPartition.c` | `PartitionTupleRouting`、`PartitionDispatch`、`ExecSetupPartitionTupleRouting()`、`ExecFindPartition()`、`ExecInitPartitionInfo()`、`ExecCleanupTupleRouting()`。 |
| 3 | `src/backend/executor/execMain.c` | `ExecPartitionCheck()`、`ExecPartitionCheckEmitError()`、`ExecConstraints()`、`ExecWithCheckOptions()`。 |
| 4 | `src/backend/commands/trigger.c` | `ExecBRInsertTriggers()`、`ExecBRUpdateTriggers()`、`ExecARInsertTriggers()`、`ExecARUpdateTriggers()` 和 trigger 改写后的约束边界。 |
| 5 | `src/backend/catalog/partition.c` | partition bound、partition descriptor 和 partition qual 的 catalog/relcache 边界。 |
| 6 | `src/include/executor/execPartition.h` | executor 侧 partition routing 对外接口和 pruning 结构边界。 |
| 7 | `src/include/nodes/execnodes.h` | `ModifyTableState`、`ResultRelInfo`、routing 相关字段和 trigger/returning slot。 |
| 8 | `src/backend/executor/execUtils.c` | `ExecGetRootToChildMap()`、`ExecGetChildToRootMap()` 和 trigger/returning slot 创建。 |
建议先读函数入口。
不要先从 `partition.c` 的 bound 数据结构读起。
原因是本节要解释 executor 怎么在一条 DML tuple 上协作，而不是解释分区元数据如何存储。
第一轮阅读只抓这条链：
```text
ExecInitModifyTable()
  -> maybe ExecSetupPartitionTupleRouting()
ExecModifyTable()
  -> ExecInsert()
     -> ExecPrepareTupleRouting()
        -> ExecFindPartition()
     -> ExecBRInsertTriggers()
     -> ExecPartitionCheck()
     -> table_tuple_insert() / FDW
  -> ExecUpdate()
     -> ExecUpdatePrologue()
     -> ExecUpdateAct()
        -> ExecPartitionCheck(..., false)
        -> maybe ExecCrossPartitionUpdate()
```
第二轮再补 ownership：
```text
PartitionTupleRouting.memcxt
  -> partition_dispatch_info[]
  -> nonleaf_partitions[]
  -> partitions[]
  -> is_borrowed_rel[]
  -> ExecCleanupTupleRouting()
```
第三轮再看异常路径：
```text
no partition
default partition constraint changed
BEFORE trigger moved row
cross-partition UPDATE conflict
ON CONFLICT DO UPDATE moves row
leaf partition UPDATE directly violates partition constraint
```
这种阅读顺序能避免一个常见误区。
`ExecFindPartition()` 不是最终写入函数。
它只返回当前 tuple 应写入哪个 `ResultRelInfo`。
后续 trigger、constraint、table AM/FDW 才决定最终是否写入。
## 4. 关键数据结构与状态
本节的数据结构都在 backend-local executor memory 中。
普通指针不能跨 backend 共享。
分区 catalog 信息来自 relcache 和 partition directory。
写入动作最终落到 table AM、index AM、trigger queue 和 WAL。
### 4.1 `ModifyTableState`
`ModifyTableState` 是本节的顶层 owner。
和 partition routing 直接相关的字段包括：
| 字段 | 语义 |
| --- | --- |
| `operation` | 当前 DML 操作：`CMD_INSERT`、`CMD_UPDATE`、`CMD_DELETE`、`CMD_MERGE`。 |
| `resultRelInfo` | planner/runtime 已知的目标 relation 数组。 |
| `rootResultRelInfo` | SQL 语句中提到的 root target relation，也是 tuple routing 的 root。 |
| `mt_resultOidAttno` | inherited UPDATE/DELETE/MERGE 用 `tableoid` junk attr 找到源 result rel。 |
| `mt_root_tuple_slot` | 跨分区 UPDATE 时把 child tuple 转回 root rowtype 的 slot。 |
| `mt_partition_tuple_routing` | executor 侧 tuple routing 状态。 |
| `mt_transition_capture` | transition table 捕获状态，routing 时可能要记录 root/child tuple。 |
| `mt_epqstate` | 并发更新后 EvalPlanQual recheck 状态。 |
`mt_partition_tuple_routing` 并不一定在 `ExecInitModifyTable()` 中创建。
partitioned INSERT 会在 init 阶段创建。
UPDATE/MERGE 只有真正发生 row movement 时，才在 `ExecCrossPartitionUpdate()` 中创建。
这是 lazy setup 的第一层。
如果 `FOR PORTION OF` 等路径需要提前插入 leftovers，也可能提前建立 routing 状态。
读源码时不要假设 `operation == CMD_UPDATE` 就一定没有 routing。
应该看 `mt_partition_tuple_routing` 是否为 NULL，以及是否有 `mt_root_tuple_slot`。
`rootResultRelInfo` 的语义也不能只看指针。
如果 `node->rootRelation > 0`，root 可能是单独创建的 `ResultRelInfo`。
如果语句只修改普通单表，root 可以直接指向 `resultRelInfo[0]`。
对分区 UPDATE，源 leaf 的 `ResultRelInfo` 会把 `ri_RootResultRelInfo` 指回 root。
这让下游代码在没有 `mtstate` 时也能找到语句原始目标。
### 4.2 `ResultRelInfo`
`ResultRelInfo` 是 leaf 写入动作的执行期对象。
它不是 relation catalog 的副本。
它聚合了当前语句在该 relation 上要用到的 executor 状态。
本节关注这些字段：
| 字段 | 语义 |
| --- | --- |
| `ri_RelationDesc` | 当前目标 relation 的 `Relation` descriptor。 |
| `ri_RootResultRelInfo` | 如果当前 relation 是 child/partition，指向语句 root target。 |
| `ri_PartitionTupleSlot` | root-to-child conversion 需要时保存转换后 tuple 的 slot。 |
| `ri_RootToChildMap` / `ri_ChildToRootMap` | root/child tuple descriptor 转换缓存。 |
| `ri_PartitionCheckExpr` | partition constraint 表达式状态，首次检查时懒创建。 |
| `ri_TrigDesc` | 该 relation 的 trigger 描述。 |
| `ri_WithCheckOptions` | RLS / view WCO 检查列表。 |
| `ri_projectReturning` | RETURNING projection。 |
| `ri_IndexRelationDescs` | 需要维护或冲突检查的 index descriptors。 |
| `ri_FdwRoutine` | foreign partition 的 FDW modify/insert 回调。 |
一个 `ResultRelInfo` 有两种来源。
第一种是 `ExecInitModifyTable()` 为 plan 中结果 relation 初始化。
第二种是 `ExecInitPartitionInfo()` 在 routing 首次触达某个 leaf 时动态创建。
动态创建的 leaf `ResultRelInfo` 会保存在 `PartitionTupleRouting.partitions[]`。
如果同一个 leaf 已经存在于 `ModifyTableState.resultRelInfo[]`，routing 可以 borrow 它。
`is_borrowed_rel[]` 记录 cleanup 责任。
borrowed 的 `ResultRelInfo` 由 `ExecEndPlan()` 等外层路径关闭。
routing 动态创建的 `ResultRelInfo` 由 `ExecCleanupTupleRouting()` 关闭 indexes 和 relation。
这就是为什么 raw pointer 不是 ownership。
同样指向 leaf partition 的 `ResultRelInfo *`，可能有不同关闭责任。
### 4.3 `PartitionTupleRouting`
`PartitionTupleRouting` 是 executor routing 的长期状态。
它的注释在 `execPartition.c` 顶部。
核心字段：
| 字段 | 语义 |
| --- | --- |
| `partition_root` | SQL 目标 partitioned table 的 `Relation`。 |
| `partition_dispatch_info` | 每个已触达 partitioned table 的 `PartitionDispatch` 数组。 |
| `nonleaf_partitions` | 非 leaf partition 的最小 `ResultRelInfo`，用于 constraint check。 |
| `partitions` | 每个已触达 leaf partition 的 `ResultRelInfo`。 |
| `is_borrowed_rel` | `partitions[]` 中对应 `ResultRelInfo` 的 ownership 标记。 |
| `num_dispatch` / `max_dispatch` | dispatch 数组当前长度与容量。 |
| `num_partitions` / `max_partitions` | leaf `ResultRelInfo` 数组当前长度与容量。 |
| `memcxt` | routing 子结构的分配上下文，通常是 `estate->es_query_cxt`。 |
这个对象解决两个问题。
第一，避免每行重新打开整个 partition tree。
第二，避免初始化从未触达的 leaf partition。
`ExecSetupPartitionTupleRouting()` 只做最小初始化：
```text
palloc0 PartitionTupleRouting
set partition_root
set memcxt
ExecInitPartitionDispatchInfo(root)
return proute
```
真正触达 leaf 时才会：
- 打开 leaf relation。
- 初始化 `ResultRelInfo`。
- 打开 indexes。
- 转换 WCO / RETURNING 表达式。
- 设置 root-to-child conversion slot。
- 调 FDW `BeginForeignInsert`。
- 把 leaf 放入 `proute->partitions[]`。
这解释了为什么第一次插入某个 partition 的 latency 可能比后续行高。
这不是 planner 估算错误。
它是 executor lazy initialization 成本。
### 4.4 `PartitionDispatch`
`PartitionDispatch` 表示一个 partitioned table 层级。
它被封装在 `PartitionTupleRouting` 中。
关键字段：
| 字段 | 语义 |
| --- | --- |
| `reldesc` | 当前 partitioned table 的 relation descriptor。 |
| `key` | 当前层级 partition key。 |
| `keystate` | partition key expression 的 `ExprState` 列表，懒创建。 |
| `partdesc` | relcache/partition directory 中的 partition descriptor。 |
| `tupslot` | parent-to-this-level descriptor 转换需要时的临时 slot。 |
| `tupmap` | parent rowtype 到当前 rowtype 的 attr map。 |
| `indexes[]` | partition index 到 `partitions[]` 或 `partition_dispatch_info[]` 的映射。 |
多级分区中，tuple 从 root 层往下走。
如果中间分区的列顺序或 rowtype 和直接 parent 不同，`ExecFindPartition()` 必须把 slot 转成当前层级的 descriptor。
否则 `FormPartitionKeyDatum()` 用 `slot_getattr()` 取到的列就可能错位。
这就是 `PartitionDispatch.tupmap` 和 `tupslot` 的意义。
它们不是为了最终写入 leaf。
它们是为了在每一级计算 partition key 时，让 `ecxt_scantuple` 和 descriptor 匹配。
`indexes[]` 是 routing hot path 的核心缓存。
如果某个 partition index 已经触达过，下一次可以直接取对应 `ResultRelInfo` 或下一层 `PartitionDispatch`。
如果为 -1，说明还没有为该 child 建过执行期状态。
### 4.5 tuple conversion map
本节涉及三类转换。
第一类是 parent-to-subpartition conversion。
它发生在 `ExecFindPartition()` 继续向下递归时。
来源是 `PartitionDispatch.tupmap`。
目标是正确计算下一层 partition key。
第二类是 root-to-child conversion。
它发生在 `ExecPrepareTupleRouting()` 找到 leaf 后。
来源是 `ExecGetRootToChildMap(partrel, estate)`。
目标是让即将写入的 slot 匹配 leaf relation descriptor。
第三类是 child-to-root conversion。
它发生在跨分区 UPDATE。
源 leaf 上的新 tuple 要先转回 root rowtype。
然后 `ExecInsert(context, rootResultRelInfo, slot, ...)` 从 root 开始重新路由。
这三类转换不能混用。
parent-to-subpartition 是路由中间态。
root-to-child 是写入目标态。
child-to-root 是 row movement 的回到 root 态。
`TupleConversionMap *` 返回 NULL 是有效语义。
它表示 descriptor 已兼容，不需要转换。
不能把 NULL 解释成“没有初始化”。
源码用 `ri_RootToChildMapValid` 和 `ri_ChildToRootMapValid` 区分“已经计算出不需要 map”和“还没计算”。
这是内核中常见的状态组合：
```text
map pointer + valid flag = semantic state
```
### 4.6 partition constraint
partition constraint 和普通 CHECK constraint 分开处理。
`ExecConstraints()` 明确不检查 partition constraint。
partition constraint 由 `ExecPartitionCheck()` 负责。
`ExecPartitionCheck()` 首次使用时会调用 `RelationGetPartitionQual()` 取得 qual，并在 `estate->es_query_cxt` 下准备表达式状态。
随后它用 `GetPerTupleExprContext(estate)`，把 `ecxt_scantuple` 指向当前 slot，再执行 `ExecCheck()`。
NULL 结果按成功处理。
这和普通 CHECK constraint 语义一致。
`emitError` 参数决定失败时返回 false 还是直接 ERROR。
INSERT 路径通常在最终 constraint 阶段用 `emitError=true`。
UPDATE 路径先用 `emitError=false` 判断是否需要 row movement。
跨分区 UPDATE 如果不能移动，或直接更新 leaf partition 时不允许移动，则用 `ExecPartitionCheckEmitError()` 报错。
`ExecPartitionCheckEmitError()` 还有一个重要细节。
如果 tuple 已经被路由并转成 child rowtype，它会尝试把 slot 反向转成 root rowtype，用来生成面向用户输入的错误描述。
这不是语义检查需要。
这是错误信息的可解释性需要。
### 4.7 trigger 与 transition table 状态
trigger 状态主要挂在 `ResultRelInfo` 上。
`ri_TrigDesc` 描述有哪些 trigger。
`ri_TrigFunctions` 和 `ri_TrigWhenExprs` 缓存函数和 WHEN 表达式。
trigger 的 old/new slot 按需从 `execUtils.c` 创建。
INSERT routing 与 trigger 的关键顺序是：
```text
route to leaf
convert to leaf slot
BEFORE ROW INSERT trigger on leaf
generated column
RLS WCO
constraints
partition constraint if needed
table AM/FDW insert
index insert
AFTER ROW INSERT trigger
view WCO
RETURNING
```
这个顺序解释了两个看似矛盾的现象。
`BEFORE ROW INSERT` trigger 可以修改普通列和分区键列。
但如果它把 tuple 改到不属于当前 leaf，PostgreSQL 不会重新路由，而是报错。
这是 `trigger.c` 中 `ExecBRInsertTriggers()` 对 clone trigger 和 `ExecPartitionCheck()` 的保护。
UPDATE trigger 的顺序不同。
`ExecUpdatePrologue()` 在源 leaf 上执行 `BEFORE ROW UPDATE` trigger。
然后 `ExecUpdateAct()` 计算 generated column、检查 partition constraint。
如果分区 constraint 失败，可能进入 `ExecCrossPartitionUpdate()`。
跨分区 UPDATE 内部的 INSERT 会在目标 leaf 上走 INSERT trigger 路径。
所以一次逻辑 UPDATE 可能对应源 leaf 的 UPDATE/DELETE trigger 和目标 leaf 的 INSERT trigger。
这也是 transition table 与 foreign key AFTER trigger 处理复杂的原因。
## 5. 主流程源码 walkthrough
这一节按时间推进。
先看初始化。
再看 INSERT。
再看 UPDATE 保持原分区。
最后看 UPDATE 迁移到另一个分区。
### 5.1 `ExecInitModifyTable()` 创建 root 与 result relations
`ExecInitModifyTable()` 首先确定 root target relation。
如果 `node->rootRelation > 0`，说明 root partitioned/inherited table 不在普通 result relation list 中。
它会单独 `makeNode(ResultRelInfo)` 并 `ExecInitResultRelation()`。
否则 root 就是唯一 result relation。
然后它初始化 `mtstate->resultRelInfo[]`。
child result relation 会把 `ri_RootResultRelInfo` 指向 root。
这是后续 conversion、error message、transition capture 和 trigger 使用 root 语义的基础。
接着初始化 subplan。
之后，对 UPDATE/DELETE/MERGE，它会从 subplan targetlist 找 row identity junk attr。
heap relation 通常需要 `ctid`。
foreign table 等可能使用 `wholerow`。
继承/分区 UPDATE 还会找 `tableoid` junk attr。
`tableoid` 用来把每条上游 tuple 映射回源 `ResultRelInfo`。
这一步和 partition routing 是互补关系。
UPDATE 的源 relation 来自 junk attr。
INSERT 的目标 relation 来自 partition routing。
对 partitioned INSERT，`ExecInitModifyTable()` 会立即建立：
```text
mtstate->mt_partition_tuple_routing =
  ExecSetupPartitionTupleRouting(estate, rootRel)
```
对 UPDATE/MERGE，init 阶段通常不建 routing。
只有当 `ExecUpdateAct()` 发现新 tuple 不再满足源 partition constraint 时，才进入 row movement 并创建 routing。
这个差异服务 hot path。
大多数 UPDATE 不改分区键。
为它们预建 tuple routing 会浪费。
### 5.2 `ExecInsert()` 先路由再触发 BEFORE INSERT
`ExecInsert()` 的第一段很关键。
如果 `mtstate->mt_partition_tuple_routing` 非 NULL，说明输入 `slot` 仍处在 root target 的语义下。
它调用：
```text
ExecPrepareTupleRouting(mtstate, estate, proute,
                        resultRelInfo, slot,
                        &partRelInfo)
```
返回值是匹配 leaf rowtype 的 slot。
同时 `resultRelInfo` 被替换为 leaf `ResultRelInfo`。
此后 `ExecInsert()` 的大部分逻辑都在 leaf 上执行。
这包括打开 leaf indexes、执行 leaf trigger、计算 leaf stored generated column、检查 leaf RLS/WCO、检查 leaf constraints、调用 table AM/FDW、插入 leaf indexes、触发 AFTER trigger 和 RETURNING。
`ExecMaterializeSlot(slot)` 紧接在 routing 后。
这不是语义装饰。
trigger、FDW、constraint 和 table AM 可能要求 slot 独立于上游临时表达式结果。
尤其是经过 conversion 的 virtual slot，不能让后续路径引用已经会被 reset 的 per-tuple 内存。
INSERT 的 BEFORE trigger 注释说明了顺序原因。
不能先检查 constraint 再触发 BEFORE trigger。
因为 trigger 可以改变待插入值。
同时 trigger 可以执行任意用户代码并产生副作用。
对于 `INSERT ... ON CONFLICT`，即使最终可能冲突，BEFORE trigger 仍对每次尝试插入执行。
所以 INSERT 的可观察顺序不是“先验证一切再触发用户代码”。
它是“先路由到一个候选 leaf，再让 leaf 的 BEFORE trigger 改写，然后检查写入条件”。
### 5.3 `ExecPrepareTupleRouting()` 连接选择与转换
`ExecPrepareTupleRouting()` 很短。
但它是本节最重要的桥。
它先调用 `ExecFindPartition()`。
如果找不到有效 leaf，错误会在 `ExecFindPartition()` 内抛出。
如果找到了 leaf，它处理 transition table 的一个优化：
没有 leaf BEFORE INSERT trigger 时，可以保存原始 root tuple，避免 root-child-root 的往返转换。
然后它调用：
```text
map = ExecGetRootToChildMap(partrel, estate)
```
如果 map 非 NULL，就把 root slot 转成 leaf slot：
```text
slot = execute_attr_map_slot(map->attrMap,
                             slot,
                             partrel->ri_PartitionTupleSlot)
```
最后返回 leaf slot，并把 leaf `ResultRelInfo` 传给调用者。
这里有三个诊断点。
第一，`partrel` 是最终 leaf。
第二，返回 slot 的 descriptor 可能已经不是 root descriptor。
第三，transition capture 可能仍记着未转换的原始 slot。
这解释了为什么单看 gdb 中 `slot->tts_tupleDescriptor` 不够。
你还要问这个 slot 是 root tuple、leaf tuple、transition table 原始 tuple，还是 error message 临时反向转换 tuple。
### 5.4 `ExecFindPartition()` 从 root 向 leaf 下降
`ExecFindPartition()` 是路由搜索主循环。
它先切到 per-tuple memory context。
原因是 partition key expression evaluation 和错误描述可能产生短生命周期内存。
它保存当前 `ecxt_scantuple`，结束时恢复。
这避免影响外层表达式求值状态。
如果 root target 本身也是某个上层 partition 的 partition，它先检查 root 的 partition constraint。
这是防止从一个不属于该 root 的 tuple 开始路由。
主循环从 `proute->partition_dispatch_info[0]` 开始。
每一层做四件事：
1. 把 `ecxt->ecxt_scantuple` 指向当前层 slot。
2. 用 `FormPartitionKeyDatum()` 提取 partition key Datums。
3. 用 `get_partition_for_tuple()` 找该层匹配的 child index。
4. 判断 child 是 leaf 还是 sub-partitioned table。
如果找不到 child，抛出：
```text
no partition of relation "%s" found for row
```
这个错误发生在“无法从当前层 bound 找到匹配 child”。
它不等同于“已进入某 leaf 但违反 leaf constraint”。
leaf 情况下，`ExecFindPartition()` 先看 `dispatch->indexes[partidx]`。
如果 >= 0，说明这个 leaf 的 `ResultRelInfo` 已经建过。
直接从 `proute->partitions[]` 取。
如果是 -1，说明第一次触达。
它先尝试 `ExecLookupResultRelByOid()` 复用 `ModifyTableState` 中已有 `ResultRelInfo`。
如果能复用，调用 `ExecInitRoutingInfo()` 补充 routing 所需状态。
如果不能复用，调用 `ExecInitPartitionInfo()` 动态创建。
非 leaf 情况下，也先看 `indexes[]`。
已建过就直接进入下一层 `PartitionDispatch`。
没建过就调用 `ExecInitPartitionDispatchInfo()`。
然后如果下一层 descriptor 和当前层不同，通过 `dispatch->tupmap` 把 slot 转成下一层 rowtype。
注意这里的 conversion 是为了继续找分区。
它不是最终 leaf 写入 conversion。
最后，如果当前 child 是 default partition，`ExecFindPartition()` 会执行 partition constraint check。
原因是 default partition 的有效范围可能因并发添加 partition 而变化。
仅靠 bound 选择 default 不足以证明 tuple 仍属于 default。
这一步覆盖 leaf 和 non-leaf default。
对 leaf default，如果当前 slot 还是 root rowtype，代码会用 root-to-child map 临时转换后再检查。
### 5.5 partition bound lookup 的成本模型
`get_partition_for_tuple()` 根据策略分派。
HASH partition 计算 hash 后直接用 remainder 找 index。
LIST/RANGE partition 通常需要 bound search。
源码中有一个局部缓存策略。
如果连续多次找到同一个 LIST/RANGE partition，超过 `PARTITION_CACHED_FIND_THRESHOLD` 后，会先检查当前 values 是否仍属于 last found partition。
命中则跳过 binary search。
这个优化解释了一个常见现象。
按时间递增插入 RANGE partitioned table 时，许多行连续命中同一个当前分区。
路由搜索成本会比“每行随机分区键”更低。
但 default partition 和 NULL partition 不走同样缓存。
多级分区的成本还要乘以层级。
每层都要提取 key、计算表达式、查 bound、可能转换 slot。
所以分区数、层级数、partition key expression 成本、descriptor 差异和 leaf 首次触达次数都会影响 INSERT CPU。
EXPLAIN 不会把这些成本拆成单独节点。
通常只能通过 profiler、gdb 计数或源码插桩看到。
### 5.6 `ExecInitPartitionInfo()` 首次触达 leaf
如果某个 leaf 第一次被 routing 触达，`ExecInitPartitionInfo()` 会在 `proute->memcxt` 下做较多工作。
它打开 leaf relation：
```text
table_open(partOid, RowExclusiveLock)
```
然后 `InitResultRelInfo()` 创建 leaf `ResultRelInfo`。
接着 `CheckValidResultRel()` 验证 leaf 可以作为 INSERT target。
这点对 UPDATE row movement 也适用。
因为跨分区 UPDATE 的目标 leaf 是通过 DELETE+INSERT 写入。
如果 leaf 有 index，可能打开 indexes。
`ON CONFLICT` 需要额外初始化 arbiter index 信息。
如果有 WCO 或 RETURNING，planner 未必为所有 runtime leaf 都预建表达式。
动态 leaf 会用属性名映射，把 root/first result relation 的表达式转换到 leaf attno。
然后 `ExecInitRoutingInfo()` 设置 root-to-child map、conversion slot、FDW routing callback 和 partitions array。
这就是“首次触达 leaf 比后续慢”的具体来源。
它包含 relation open、index open、expression init、FDW begin callback 和内存数组扩容。
后续同 leaf 只走 cached `indexes[]`。
### 5.7 INSERT 后续检查顺序
路由和 conversion 之后，`ExecInsert()` 进入 leaf 写入。
顺序可以压缩成：
```text
open indexes if needed
BEFORE ROW INSERT
INSTEAD OF / FDW / table path
generated stored columns
RLS WITH CHECK
ExecConstraints()
ExecPartitionCheck()
ON CONFLICT or table_tuple_insert()
ExecInsertIndexTuples()
AFTER ROW INSERT
view WITH CHECK
RETURNING
```
不同分支会跳过或调整其中一部分。
FDW path 会调用 `ExecForeignInsert()`。
FDW batch insert 会先把 slot 缓存在 `ResultRelInfo` 的 batch slots 中，稍后 flush。
普通 table path 会设置 `slot->tts_tableOid`。
这让 generated column、constraint、trigger 和 RETURNING 中引用 `tableoid` 时得到 leaf relation OID。
partition constraint 的条件值得注意。
对已通过 tuple routing 到达 leaf 的 INSERT，如果 leaf 没有 BEFORE ROW INSERT trigger，通常不需要再次检查 leaf partition constraint。
因为 routing 已经证明 tuple 属于 leaf。
如果 leaf 有 BEFORE trigger，它可能改写分区键。
因此需要在 trigger 后再次检查。
如果语句直接 INSERT/UPDATE 某个 leaf partition，则 `ri_RootResultRelInfo` 为空或不同路径，constraint check 也按直接目标处理。
### 5.8 `ExecBRInsertTriggers()` 不重新路由
`ExecBRInsertTriggers()` 在 leaf 上运行。
它把 slot 转成 heap tuple 传给 trigger。
trigger 返回 NULL 表示“do nothing”。
trigger 返回不同 tuple 表示改写 NEW row。
对于 partition clone trigger，如果改写后 tuple 不再满足当前 leaf partition constraint，源码报：
```text
moving row to another partition during a BEFORE FOR EACH ROW trigger is not supported
```
这个错误不是找不到分区。
它是在已经选定 leaf 后，trigger 尝试把 row 移出 leaf。
为什么不重新路由？
因为 BEFORE trigger 属于当前 leaf 的 trigger 集合。
如果重新路由到另一个 leaf，就会引出新 leaf 的 BEFORE trigger 是否也要执行、已执行 trigger 的副作用是否可撤销、transition table 如何记录、trigger 递归是否无限等问题。
PostgreSQL 选择把这个行为显式禁止。
所以，INSERT 的 routing 不是“直到所有 trigger 稳定后再决定分区”。
它是“先决定候选 leaf，再禁止 leaf BEFORE trigger 把 row 移出 leaf”。
### 5.9 UPDATE 源 relation 来自 row identity
UPDATE 的上游 subplan 输出包含 row identity junk attr。
对分区/继承 UPDATE，通常还有 `tableoid`。
`ExecModifyTable()` 每拉到一行，会根据 `tableoid` 找到源 `ResultRelInfo`。
然后用 `ctid` 或 `wholerow` 定位旧 tuple。
`ExecGetUpdateNewTuple()` 生成新 tuple slot。
这个 slot 的 descriptor 是源 result relation。
也就是说，UPDATE 一开始不是从 root routing。
它从“旧 row 所在 leaf”开始。
这点和 INSERT 不同。
INSERT 没有旧 row identity。
它从 root tuple 值路由到 leaf。
UPDATE 有旧 row identity。
它先在源 leaf 上尝试 update，再根据 partition constraint 判断是否需要 row movement。
### 5.10 `ExecUpdatePrologue()` 先执行源 leaf BEFORE UPDATE
`ExecUpdatePrologue()` 做三件事。
第一，materialize 新 slot。
第二，按需打开 indexes。
第三，执行 `BEFORE ROW UPDATE` triggers。
如果 trigger 返回 NULL，UPDATE 变成 no-op。
如果 trigger 改写 NEW tuple，后续检查使用改写后的 slot。
`ExecBRUpdateTriggers()` 还会处理并发可见性。
它可能通过 `GetTupleForTrigger()` 获取旧 tuple。
READ COMMITTED 下如果目标 tuple 已被并发更新，可能形成 EPQ candidate。
随后 `ExecGetUpdateNewTuple()` 会基于新版本重建待更新 tuple。
这说明 UPDATE 的 partition constraint 检查不只是计算一个表达式。
它站在 trigger、EPQ 和 slot materialization 之后。
### 5.11 `ExecUpdateAct()` 判断是否跨分区
`ExecUpdateAct()` 先设置 `updateCxt->crossPartUpdate = false`。
然后进入 `lreplace` 标签。
这里先计算 stored generated columns，并 materialize slot。
接着：
```text
partition_constraint_failed =
  relispartition &&
  !ExecPartitionCheck(resultRelInfo, slot, estate, false)
```
如果 partition constraint 没失败，继续 RLS UPDATE WCO、普通 constraints、`table_tuple_update()`。
这是同分区 UPDATE。
如果 partition constraint 失败，可能表示分区键改了。
此时不立即报错。
如果语句是对 partitioned root 执行的 UPDATE，它会尝试 `ExecCrossPartitionUpdate()`。
如果直接对 leaf partition 执行 UPDATE，`ExecCrossPartitionUpdate()` 会用 `ExecPartitionCheckEmitError()` 报 partition constraint violation。
这里的语义很重要。
只有从 root partitioned table 发起的 UPDATE 才有 row movement 的上下文。
直接 UPDATE 某个 leaf partition，用户语义是修改该 leaf。
改到不属于该 leaf 的 row 会违反该 leaf 的 partition constraint。
### 5.12 `ExecCrossPartitionUpdate()` 是 DELETE + root INSERT
跨分区 UPDATE 的实现主线：
```text
if ON CONFLICT DO UPDATE would move row: ERROR
if updating leaf directly: partition constraint ERROR
ensure mt_partition_tuple_routing exists
ensure mt_root_tuple_slot exists
ExecDelete(source leaf, changingPart=true, processReturning=false)
if delete did not happen: skip insert or retry
convert child tuple to root rowtype if needed
ExecInsert(rootResultRelInfo, root slot)
reset transition capture original_insert_tuple
return success
```
它先禁止一个特殊场景。
`INSERT ... ON CONFLICT DO UPDATE` 如果 UPDATE 结果要移动到另一个 partition，会报 feature not supported。
这是为了避免 speculative insertion、冲突 tuple、row movement 和 uniqueness inference 组合出难以维护的语义。
然后它确保 routing 状态存在。
UPDATE/MERGE 的 routing 可能懒创建。
创建时必须切到 `estate->es_query_cxt`。
因为这些状态要活到 query end。
同时创建 `mt_root_tuple_slot`。
源 leaf 的 tuple 在重新插入前必须转回 root rowtype。
删除阶段调用 `ExecDelete()`。
它设置 `changingPart=true`，并跳过 DELETE 的 RETURNING。
因为跨分区 UPDATE 的 RETURNING 应该来自后续 INSERT/UPDATE 语义处理，而不是把内部 DELETE 暴露出去。
如果 DELETE 没发生，不能继续 INSERT。
否则一个旧 row 可能变成两个 row。
如果 DELETE 失败是因为并发 UPDATE，UPDATE 路径可能拿到 retry slot 并跳回 `lreplace`。
这是对 CTID 链不能跨 relation 的补偿。
普通 heap update 可以沿同 relation 的 tuple chain 做 EPQ。
跨分区 update 的 delete+insert 没有跨 relation tuple chain。
所以这里以有限方式模拟并发语义。
最后，源 leaf slot 通过 `ExecGetChildToRootMap()` 转成 root slot。
再调用 `ExecInsert()` 从 root 开始。
这会重新执行 INSERT routing。
目标 leaf 的 BEFORE INSERT trigger、generated column、WCO、constraint、table AM/FDW、AFTER INSERT trigger 都按 INSERT 规则执行。
### 5.13 跨分区 UPDATE 的 AFTER trigger 与 foreign key
跨分区 UPDATE 内部做了 DELETE 和 INSERT。
但从 SQL 用户视角，它仍是一条 UPDATE。
如果 root partitioned table 被 foreign key 引用，源 leaf 的普通 AFTER UPDATE trigger 不足以表达 root 层 UPDATE。
源码中 `ExecCrossPartitionUpdateForeignKey()` 会排队 root 层 update event。
`ExecARUpdateTriggers()` 的接口也显式支持 `src_partinfo`、`dst_partinfo` 和 `is_crosspart_update`。
这说明 trigger 子系统不是简单跟着物理 table AM 调用走。
它要把物理 delete+insert 映射回 SQL 语义需要的事件。
诊断跨分区 UPDATE 的 trigger 行为时，要区分：
- 源 leaf 上为了移动而发生的 DELETE。
- 目标 leaf 上重新 INSERT。
- root relation 上为了 FK/AFTER UPDATE 语义排队的事件。
- RETURNING 使用的 new tuple。
### 5.14 RETURNING 与 `tableoid`
RETURNING projection 挂在 `ResultRelInfo` 上。
动态创建 leaf 时，如果需要 RETURNING，`ExecInitPartitionInfo()` 会把 planner 生成的 RETURNING list 用属性名映射到 leaf attno，并调用 `ExecBuildProjectionInfo()`。
普通 INSERT/UPDATE path 会在 AFTER trigger 之后处理 RETURNING。
`slot->tts_tableOid` 会在 generated/constraint/RETURNING 可能引用前设置。
跨分区 UPDATE 的 RETURNING 要特别小心。
内部 DELETE 不处理 RETURNING。
后续 INSERT 的结果 slot 才作为 UPDATE row movement 的可见结果。
这就是 `ExecCrossPartitionUpdate()` 中 `processReturning=false` 和 `context->cpUpdateReturningSlot` 相关状态存在的原因。
如果你用 `RETURNING tableoid, *` 观察跨分区 UPDATE，看到的是目标 leaf 的 `tableoid`。
这不是额外查询。
这是 executor 在 slot 上维护的 `tts_tableOid` 语义。
## 6. 生命周期 / ownership / cleanup
`PartitionTupleRouting` 通常在 `estate->es_query_cxt` 下创建。
它的 lifetime 是一条 executor invocation。
`ExecutorEnd()` 期间，`ExecEndModifyTable()` 会调用 `ExecCleanupTupleRouting()`。
然后如果有 `mt_root_tuple_slot`，调用 `ExecDropSingleTupleTableSlot()`。
最后外层 `ExecEndPlan()`、`ExecResetTupleTable()`、`FreeExecutorState()` 继续清理 executor 状态。
`PartitionTupleRouting` 内部对象的 ownership 要分开看。
root partitioned table 的 `Relation` 不是 routing cleanup 关闭。
因为它是 query target relation，由外层 result relation cleanup 管。
子 partitioned table 在 `ExecInitPartitionDispatchInfo()` 中打开。
cleanup 时从 `i = 1` 开始关闭 `partition_dispatch_info[i]->reldesc`。
leaf partitions 分两种。
borrowed leaf `ResultRelInfo` 来自 `ModifyTableState.resultRelInfo[]`。
它们不由 routing cleanup 关闭。
动态创建 leaf `ResultRelInfo` 来自 `ExecInitPartitionInfo()`。
它们由 `ExecCleanupTupleRouting()` 关闭 indexes 和 relation。
FDW routing 也有生命周期。
`ExecInitRoutingInfo()` 如果看到 `BeginForeignInsert`，会让 FDW 初始化 insert routing 状态。
cleanup 中如果看到 `EndForeignInsert`，会让 FDW shutdown。
这和普通 foreign modify 的 `BeginForeignModify` / `EndForeignModify` 是不同入口。
slot cleanup 也分层。
`ri_PartitionTupleSlot` 如果通过 `table_slot_create(partrel, &estate->es_tupleTable)` 创建，会进入 EState tuple table，由 executor tuple table reset 管。
`PartitionDispatch.tupslot` 通过 `MakeSingleTupleTableSlot()` 创建，不在 EState tuple table 中。
它由 `ExecCleanupTupleRouting()` 中 `ExecDropSingleTupleTableSlot()` 释放。
`mt_root_tuple_slot` 通过 `table_slot_create(rootRel, NULL)` 创建，也需要显式 drop。
这些差异说明 MemoryContext 只管内存。
slot 可能持有 TupleDesc refcount、buffer pin 或 slot ops 资源。
必须走 slot cleanup 协议。
ERROR 路径下，当前 statement 会通过 PostgreSQL 的 error cleanup 释放 memory context、resource owner、locks 和 registered resources。
但正常 `ExecutorEnd()` 的顺序仍然重要。
因为它让 FDW callback、index close、slot drop、Relation close 按协议运行。
不要把“错误会 abort”理解成可以省略正常 cleanup。
## 7. 正确性机制层次
本节正确性不是一个机制保证的。
它来自多个边界叠加。
| 层次 | 机制 | 保证 | 不能保证 |
| --- | --- | --- | --- |
| relation lock | root、subpartition、leaf 上的 `RowExclusiveLock` | 写入期间 relation 结构不会以不兼容方式消失 | 不决定 tuple 属于哪个 partition |
| relcache/partition directory | `PartitionDirectoryLookup()`、`PartitionDesc` | 读取当前可用 partition descriptor | 不替代 partition constraint recheck |
| routing search | `FormPartitionKeyDatum()` + `get_partition_for_tuple()` | 根据当前 slot 值选择 child index | 不保证 trigger 后仍属于该 leaf |
| tuple conversion | attr map + slot descriptor | 让表达式和 table AM 看到正确 rowtype | 不保证业务约束成立 |
| partition constraint | `ExecPartitionCheck()` | 验证 tuple 属于当前 relation 的 partition qual | 不负责普通 CHECK/NOT NULL |
| ordinary constraints | `ExecConstraints()` | NOT NULL、CHECK 等 relation constraints | 明确不检查 partition constraint |
| trigger ordering | BEFORE/AFTER trigger 函数和 queue | 满足 SQL trigger 时序和 transition 语义 | 不允许 INSERT BEFORE trigger 任意跨 leaf reroute |
| MVCC/EPQ | snapshot、tuple lock、EvalPlanQual | 处理并发 UPDATE/DELETE 可见性 | CTID 链不能跨 relation |
| WAL/table AM | `table_tuple_insert/update/delete()` | 持久化和 crash safety | 不理解 root partition routing 语义 |
root-to-child conversion 是正确性边界。
没有它，leaf constraint、generated column、trigger 和 table AM 都可能按错误 attno 解释 Datum。
child-to-root conversion 也是正确性边界。
跨分区 UPDATE 必须重新从 root 开始路由。
否则一个源 leaf 的 descriptor 可能直接喂给 root routing 或目标 leaf insert。
default partition check 是另一个正确性边界。
default 的含义是“没有匹配其他 bound”。
如果并发添加 partition，default constraint 可能变化。
所以 routing 到 default 后还要 check constraint。
这不是重复校验。
它是 default partition 语义的一部分。
trigger 后 partition constraint 是 SQL 顺序边界。
BEFORE trigger 先于 constraint。
因此 constraint 必须看 trigger 改写后的 tuple。
但 INSERT 不允许 trigger 把 tuple 搬到另一个 leaf。
UPDATE 允许通过 row movement 搬，但搬运由 UPDATE 的 constraint failure path 触发，而不是 INSERT trigger 中无限重路由。
## 8. 错误路径 / 异常路径 / fallback
### 8.1 找不到 partition
发生位置：`ExecFindPartition()`。
触发条件：当前 partitioned table 没有 partition，或 `get_partition_for_tuple()` 返回 -1。
典型错误：
```text
no partition of relation "..." found for row
```
这个错误说明 bound lookup 没有找到 child。
它通常来自分区范围缺口、NULL key 未定义、DEFAULT partition 不存在，或 read committed 下并发 detach 让 partition 不再可用。
诊断时先打印失败层级的 relation name 和 partition key values。
不要先看 leaf constraint。
因为根本没有 leaf。
### 8.2 partition constraint violation
发生位置：`ExecPartitionCheck()` 或 `ExecPartitionCheckEmitError()`。
典型错误：
```text
new row for relation "..." violates partition constraint
```
这个错误说明当前 relation 已经确定。
tuple 不满足该 relation 的 partition qual。
常见场景：
- 直接 UPDATE leaf partition，把分区键改出 leaf 范围。
- INSERT routing 后 BEFORE trigger 改写分区键。
- default partition constraint 因新增 partition 变得更窄。
- error message 需要把 child rowtype 转回 root rowtype。
和 8.1 的区别是：这里有目标 relation。
### 8.3 INSERT BEFORE trigger 试图移动 row
发生位置：`ExecBRInsertTriggers()`。
错误：
```text
moving row to another partition during a BEFORE FOR EACH ROW trigger is not supported
```
这个路径只在 trigger 返回修改后的 tuple 且该 tuple 不再满足当前 leaf constraint 时触发。
它不是自动 fallback 到重新路由。
这是有意限制。
修复通常应把分区键改写放在 INSERT 前的 SQL 表达式、应用层，或改成对 root 的 UPDATE row movement 模型。
### 8.4 跨分区 UPDATE 并发冲突
跨分区 UPDATE 先 DELETE 源 leaf，再 INSERT root。
如果 DELETE 没发生，不能继续 INSERT。
否则会复制 row。
并发 UPDATE 情况下，`ExecDelete()` 可能返回 EPQ slot。
非 MERGE UPDATE 会用该 slot 重新投影 new tuple，并回到 `lreplace`。
MERGE 有自己的 retry 逻辑。
这个 fallback 的边界是有限的。
它不能创造跨 relation CTID chain。
它只能避免明显的 duplicate/resurrection，并尽量模拟普通 UPDATE 的 READ COMMITTED 行为。
### 8.5 `ON CONFLICT DO UPDATE` 不能移动 partition
如果 `INSERT ... ON CONFLICT DO UPDATE` 的 UPDATE 结果需要进入不同 partition，`ExecCrossPartitionUpdate()` 报 feature not supported。
错误信息是：
```text
invalid ON UPDATE specification
```
detail 会说明结果 tuple 会出现在不同 partition。
这个限制来自 speculative insertion、conflict arbiter、源 tuple 和目标 tuple relation 语义叠加。
它不是简单缺少一行 reroute 代码。
### 8.6 FDW 和 batch insert fallback
foreign partitions 走 FDW callback。
routing 初始化可能调用 `BeginForeignInsert`。
写入可能走 `ExecForeignInsert()` 或 batch insert。
如果 FDW 不支持 batching，batch size 为 1。
FDW 返回 NULL 可以表示 do nothing。
transition table 对 foreign child 有额外限制。
例如收集 transition tuples from child foreign tables 可能报 unsupported。
诊断 foreign partition 时不要只看 heap table path。
## 9. 成本、资源与跨模块传播
partition routing 成本主要在 CPU 和 executor 本地内存。
它通常不是 wait event。
因此慢 SQL 可能表现为 backend active 但没有明显 wait。
成本变量包括：
- 每行 tuple 数量。
- partition tree 层级数。
- 每层 partition 数。
- partition key expression 成本。
- LIST/RANGE bound lookup 是否能命中 last-found cache。
- root/child descriptor 是否需要转换。
- 每个 leaf 是否首次触达。
- leaf index 数量。
- trigger 函数成本。
- RLS/WCO/constraint 表达式成本。
- FDW batch 是否能聚合。
第一次触达 leaf 的成本包括 relation open、RowExclusiveLock、index open、WCO/RETURNING expression translation、routing slot 创建和 FDW begin callback。
如果批量插入随机分区键，可能触达很多 leaf。
这会放大 relation/index open 和 local memory 增长。
如果批量插入按分区键排序，连续命中同一 leaf。
`get_partition_for_tuple()` 的 LIST/RANGE last-found cache 和 `dispatch->indexes[]` 都更容易命中。
这会降低 per-row CPU。
tuple conversion 成本随列数增长。
`execute_attr_map_slot()` 需要按 attr map 重排 Datum/null。
列很多、descriptor 差异大、每行都跨层转换时，成本会很明显。
generated columns、RLS WCO 和 CHECK constraints 运行在 leaf descriptor 上。
它们可能调用用户函数。
所以 routing 之后的成本不只是写 heap。
trigger 成本可能完全淹没 routing。
但 trigger 又会改变是否需要重新 partition check。
因此诊断时要把“路由搜索慢”和“leaf trigger/constraint 慢”拆开。
跨模块传播：
- planner 决定 root、resultRelations、junk attr、RETURNING/WCO 原始表达式。
- executor routing 决定最终 leaf `ResultRelInfo`。
- relcache/partition directory 提供 partition descriptor 和 partition qual。
- trigger manager 负责 BEFORE/AFTER 时序和 transition table。
- table AM 负责实际 insert/update/delete。
- index AM 负责 index tuple 和 uniqueness/exclusion。
- FDW 负责 foreign partition 写入。
- lock manager 保证 relation 级兼容性。
- WAL 和 storage 层保证物理持久性。
这里没有后台进程主动推进 routing 状态。
`PartitionTupleRouting` 是 backend-local。
但写入后的 WAL、checkpoint、autovacuum、logical decoding 和 stats 会被后续系统消费。
跨分区 UPDATE 产生 DELETE+INSERT，会影响 WAL 量、index 维护量、trigger queue 和外键检查。
这也是为什么修改分区键的 UPDATE 通常比同分区 UPDATE 更贵。
## 10. 观测与诊断入口
本节的 runtime truth 是：
```text
同一条 INSERT/UPDATE 语句中的每一行，最终写入哪个 leaf partition，是 executor runtime 根据 tuple 值和当前 partition descriptor 决定的；EXPLAIN 不逐行展示这个选择。
```
能直接观测的内容：
- SQL 错误信息：找不到 partition、违反 partition constraint、BEFORE trigger 移动 row 不支持。
- `RETURNING tableoid`：看到最终写入的 leaf relation OID。
- `EXPLAIN (ANALYZE)`：看到 `ModifyTable` 总体耗时和 rows，但看不到每行 routing。
- `pg_stat_all_tables`：看到各 leaf 的 insert/update/delete 累计计数。
- trigger 函数内部日志：看到 trigger 被哪个 relation 调用。
- `pg_locks`：看到涉及 relation 的锁，但不是每行 routing 决策。
- `perf` / flamegraph：看到 `ExecFindPartition`、`FormPartitionKeyDatum`、`get_partition_for_tuple`、`execute_attr_map_slot` 等 CPU 热点。
- gdb 断点：直接观察 `ResultRelInfo`、slot descriptor、partition key Datums。
只能推断的内容：
- 每行路由到哪个 leaf，除非加 `RETURNING tableoid` 或插桩。
- 首次触达 leaf 的初始化成本占比。
- default partition constraint recheck 是否频繁触发。
- route search 和 trigger/constraint 成本的相对比例。
几乎不可见的内容：
- `PartitionTupleRouting.partitions[]` 的增长过程。
- `dispatch->indexes[]` 的缓存命中。
- `partdesc->last_found_count` 对 LIST/RANGE lookup 的帮助。
- root-to-child map 是否为 NULL 的每行判断成本。
一个简单 SQL 观测入口：
```sql
CREATE TABLE p (id int, payload text) PARTITION BY RANGE (id);
CREATE TABLE p_0 PARTITION OF p FOR VALUES FROM (0) TO (100);
CREATE TABLE p_1 PARTITION OF p FOR VALUES FROM (100) TO (200);
INSERT INTO p VALUES (1, 'a'), (101, 'b')
RETURNING tableoid::regclass, *;
```
这会显示两行进入不同 leaf。
它验证的是 executor 最终写入目标。
它不展示 `ExecFindPartition()` 的中间路径。
观察 UPDATE row movement：
```sql
UPDATE p
SET id = id + 100
WHERE id = 1
RETURNING tableoid::regclass, *;
```
如果从 root `p` 更新，row 可以移动到 `p_1`。
如果直接更新 `p_0` 并把 id 改出范围，应看到 partition constraint violation。
gdb 断点建议：
```text
break ExecPrepareTupleRouting
break ExecFindPartition
break ExecPartitionCheck
break ExecCrossPartitionUpdate
commands
  bt 5
  continue
end
```
实际调试时不要无条件打印所有 tuple。
批量 INSERT 会让断点非常频繁。
可以用条件断点限制 relation OID 或 key 值。
perf 入口：
```text
perf record -g -- psql -c "INSERT INTO p SELECT g, md5(g::text) FROM generate_series(1,100000) g"
perf report
```
如果数据按分区键排序，热点可能集中在 table AM/index/trigger。
如果分区键随机且 leaf 很多，`ExecFindPartition()`、bound compare 和 attr map 可能更明显。
注意 `pg_stat_*` 是累计视图。
它能说明最终哪些 leaf 被写入。
它不能说明每行 routing 成本，也不能区分第一次 leaf 初始化和后续 hot path。
## 11. 常见误区
误区一：以为 planner 已经决定每一行 INSERT 的 leaf partition。
纠正：planner 建立 partitioned INSERT 的 `ModifyTable` 框架；每行 leaf 选择在 executor 的 `ExecFindPartition()` 中按 tuple 值完成。
误区二：以为 `ExecFindPartition()` 返回后就可以直接写入。
纠正：返回的是 leaf `ResultRelInfo`。之后还要 root-to-child conversion、BEFORE trigger、generated column、WCO、constraints、partition check、table AM/FDW。
误区三：以为 BEFORE INSERT trigger 改了分区键会自动重新路由。
纠正：INSERT 已经在 leaf 上触发 BEFORE trigger。改出 leaf 会报不支持移动 row 的错误。
误区四：以为 UPDATE 改分区键就是普通 heap update。
纠正：跨分区 UPDATE 是源 leaf DELETE 加 root INSERT。CTID 链不能跨 relation。
误区五：以为 partition constraint 属于 `ExecConstraints()`。
纠正：`ExecConstraints()` 明确不检查 partition constraint。partition constraint 由 `ExecPartitionCheck()` 单独处理。
误区六：以为 `TupleConversionMap *map == NULL` 表示没初始化。
纠正：NULL map 可能是有效结果，表示无需转换。要结合 valid flag 判断。
误区七：以为 `ResultRelInfo *` 指向 leaf 就意味着 cleanup 责任相同。
纠正：leaf `ResultRelInfo` 可能 borrowed，也可能 routing 动态创建。`is_borrowed_rel[]` 决定 cleanup。
误区八：以为 EXPLAIN 能展示每行路由。
纠正：EXPLAIN 展示 `ModifyTable` 节点总体执行，不逐行展示 `ExecFindPartition()` 的选择。
## 12. 课堂实验
### 实验 1：区分找不到 partition 与 constraint violation
目标：观察两个错误分别来自哪个源码边界。
SQL：
```sql
DROP TABLE IF EXISTS p CASCADE;
CREATE TABLE p (id int, note text) PARTITION BY RANGE (id);
CREATE TABLE p_0 PARTITION OF p FOR VALUES FROM (0) TO (10);
INSERT INTO p VALUES (20, 'missing partition');
```
预期：报 `no partition of relation`。
源码边界：`ExecFindPartition()` 中 `get_partition_for_tuple()` 返回 -1。
再执行：
```sql
UPDATE p_0 SET id = 20 WHERE id = 1;
```
先插入一行：
```sql
INSERT INTO p VALUES (1, 'in p_0');
UPDATE p_0 SET id = 20 WHERE id = 1;
```
预期：直接更新 leaf 时，报 partition constraint violation。
源码边界：`ExecUpdateAct()` 检查源 leaf partition constraint，不能 row movement，于是 `ExecPartitionCheckEmitError()`。
画出两条路径：
```text
INSERT root -> no child bound -> no partition error
UPDATE leaf -> target relation fixed -> partition constraint error
```
### 实验 2：观察 root UPDATE 的 row movement
目标：确认 root UPDATE 能触发 DELETE+INSERT。
SQL：
```sql
DROP TABLE IF EXISTS p CASCADE;
CREATE TABLE p (id int, note text) PARTITION BY RANGE (id);
CREATE TABLE p_0 PARTITION OF p FOR VALUES FROM (0) TO (10);
CREATE TABLE p_1 PARTITION OF p FOR VALUES FROM (10) TO (20);
INSERT INTO p VALUES (1, 'move me');
UPDATE p
SET id = 11
WHERE id = 1
RETURNING tableoid::regclass, id, note;
```
预期：返回 `p_1`。
建议断点：
```text
break ExecUpdateAct
break ExecCrossPartitionUpdate
break ExecPrepareTupleRouting
```
观察点：
- `ExecUpdateAct()` 中 `partition_constraint_failed` 变 true。
- `ExecCrossPartitionUpdate()` 先 `ExecDelete()`。
- `ExecGetChildToRootMap()` 可能把源 leaf tuple 转回 root rowtype。
- `ExecInsert()` 从 root 重新路由到 `p_1`。
### 实验 3：BEFORE INSERT trigger 不能把 row 搬到另一个 leaf
目标：观察 trigger 顺序。
SQL：
```sql
DROP TABLE IF EXISTS p CASCADE;
CREATE TABLE p (id int, note text) PARTITION BY RANGE (id);
CREATE TABLE p_0 PARTITION OF p FOR VALUES FROM (0) TO (10);
CREATE TABLE p_1 PARTITION OF p FOR VALUES FROM (10) TO (20);
CREATE OR REPLACE FUNCTION bump_id()
RETURNS trigger
LANGUAGE plpgsql
AS $$
BEGIN
  NEW.id := 11;
  RETURN NEW;
END;
$$;
CREATE TRIGGER p0_bi
BEFORE INSERT ON p_0
FOR EACH ROW
EXECUTE FUNCTION bump_id();
INSERT INTO p VALUES (1, 'trigger move');
```
预期：报 moving row to another partition during a BEFORE trigger is not supported。
源码边界：`ExecBRInsertTriggers()` 在 leaf trigger 改写后调用 `ExecPartitionCheck(relinfo, slot, estate, false)`。
讨论点：为什么不自动重新路由到 `p_1`？
### 实验 4：观察首次触达 leaf 的初始化成本
目标：区分 routing search 和 leaf initialization。
准备多个分区：
```sql
DROP TABLE IF EXISTS p CASCADE;
CREATE TABLE p (id int, note text) PARTITION BY RANGE (id);
CREATE TABLE p_0 PARTITION OF p FOR VALUES FROM (0) TO (1000);
CREATE TABLE p_1 PARTITION OF p FOR VALUES FROM (1000) TO (2000);
CREATE TABLE p_2 PARTITION OF p FOR VALUES FROM (2000) TO (3000);
CREATE TABLE p_3 PARTITION OF p FOR VALUES FROM (3000) TO (4000);
```
比较两种写法：
```sql
EXPLAIN (ANALYZE, BUFFERS)
INSERT INTO p
SELECT g, md5(g::text)
FROM generate_series(0, 3999) g;
```
再比较随机 key：
```sql
EXPLAIN (ANALYZE, BUFFERS)
INSERT INTO p
SELECT (random() * 3999)::int, md5(g::text)
FROM generate_series(1, 4000) g;
```
观测说明：
EXPLAIN 只给总耗时。
要区分 `ExecInitPartitionInfo()` 和 `get_partition_for_tuple()`，需要 perf 或源码插桩。
源码练习：在 `ExecInitPartitionInfo()` 和 `ExecFindPartition()` 中临时加计数日志。
不要提交该修改。
## 13. 讨论题
1. 为什么 partitioned INSERT 在 init 阶段建立 routing，而 UPDATE 通常等到 row movement 时才建立？
2. 为什么 `ExecFindPartition()` 中间层可能需要 parent-to-child conversion，而 `ExecPrepareTupleRouting()` 还要再做 root-to-child conversion？
3. `ExecConstraints()` 为什么明确不检查 partition constraint？如果合并到一个函数会破坏哪些边界？
4. INSERT 的 BEFORE trigger 为什么不允许把 row 改到另一个 leaf？如果允许重新路由，AFTER trigger 和 transition table 会遇到什么问题？
5. 直接 UPDATE leaf partition 和 UPDATE root partition 修改分区键，为什么一个报 constraint violation，另一个可以 row movement？
6. `ResultRelInfo` borrowed 与动态创建的 cleanup 责任为什么不能只靠 relation OID 判断？
7. `RETURNING tableoid` 能证明什么？不能证明什么？
8. 当批量 INSERT 很慢但没有 wait event 时，你会如何区分 partition routing、tuple conversion、trigger、index 和 table AM 成本？
## 14. 本节小结
本节唯一主问题是：
```text
INSERT/UPDATE 如何在 executor 中选择目标分区，partition constraint、tuple conversion 和 trigger 顺序如何协作？
```
核心链路是：
```text
ModifyTable receives tuple
  -> root or source ResultRelInfo establishes semantic context
  -> ExecFindPartition selects leaf for INSERT/root re-insert
  -> tuple conversion makes slot descriptor match target
  -> BEFORE trigger may change tuple
  -> partition constraint validates target membership
  -> table AM/FDW writes
  -> index/AFTER/transition/RETURNING expose side effects
```
INSERT 从 root tuple 值开始路由。
UPDATE 从源 leaf row identity 开始。
UPDATE 修改分区键时，executor 用 DELETE+root INSERT 实现 row movement。
`PartitionTupleRouting` 是 backend-local、query-lifetime 的懒初始化状态。
它缓存已触达的 dispatch 和 leaf `ResultRelInfo`，但不会预建所有分区。
`ResultRelInfo` 是 leaf 写入语义的聚合点。
它同时承载 trigger、WCO、constraints、RETURNING、index、FDW 和 tuple conversion map。
partition constraint 不属于普通 `ExecConstraints()`。
它由 `ExecPartitionCheck()` 单独处理。
INSERT 路由后，如果 BEFORE trigger 改出当前 leaf，不会重新路由，而是报错。
UPDATE root 修改分区键时可以 row movement，但直接 UPDATE leaf 不行。
ownership 上，MemoryContext 释放内存，slot/relation/index/FDW callback 仍需要显式 cleanup。
`ExecCleanupTupleRouting()` 只关闭 routing 自己动态打开或创建的对象。
观测上，`RETURNING tableoid` 和 leaf stats 能看到最终目标。
EXPLAIN 不能逐行展示 routing。
路由搜索、conversion 和 lazy leaf initialization 的成本通常需要 perf、gdb 或源码插桩推断。
可迁移规律是：
```text
运行时 dispatch 机制必须同时定义选择、转换、验证、用户回调和 cleanup 的顺序；
如果只描述“选中目标”，就无法解释错误路径和 observability。
```
本节仍有 workload-dependent 边界。
分区数、层级、key 分布、列数、trigger 函数、index 数、FDW 能力和并发 DDL 都会改变成本和异常概率。
源码结论基于本地 `/home/nail/postgres` `master` 的 `0e1f1ed157e9`。
不同版本可能调整 helper 函数拆分，但本节的主不变量仍应按 route、convert、trigger、check、write、cleanup 的顺序验证。
