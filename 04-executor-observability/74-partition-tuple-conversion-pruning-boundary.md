# PostgreSQL partition tuple conversion 与 pruning boundary
## 课程定位
前置知识：已经读过 04 目录中 `ExecutorStart`、`TupleTableSlot`、`ModifyTable`、`partition routing` 和 EXPLAIN 观测相关课程。
还应理解 PostgreSQL declarative partitioning 的基本 SQL 语义。
本节唯一主问题：
```text
执行期 partition pruning、tuple routing 和 row movement 如何与 planner 分区路径、ModifyTable 和 ResultRelInfo 对接？
```
核心矛盾：
```text
planner 希望把分区结构压缩成稳定的 subplan 映射和可裁剪边界，
executor 却必须在运行时按参数、tuple 值、rowtype 和触发器结果重新决定哪些分区参与读写。
```
一句话运行模型：
```text
读路径用 PartitionPruneState 把 planner 的 PartitionPruneInfo 转成运行时 subplan bitmap；
写路径用 PartitionTupleRouting 把 root tuple 路由到 leaf ResultRelInfo；
row movement 再把 UPDATE 拆成 source leaf DELETE 与 root语义 INSERT。
```
学完后应能判断：一个分区 SQL 的异常或慢路径，究竟发生在 planner 分区路径、Append/MergeAppend runtime pruning、ModifyTable tuple routing、tuple conversion、row movement，还是 ResultRelInfo 的执行期状态边界上。
本课基于本地 `/home/nail/postgres` 源码。
源码基线：
```text
branch: master
commit: 0e1f1ed157e9
```
本节不是分区功能总览。
本节只围绕一个边界问题展开：planner 生成的分区读路径，如何和 executor 中基于 tuple 的写路径共享同一个分区语义，同时又不混淆 subplan、Relation、ResultRelInfo 和 tuple descriptor。
## 1. 本节在总主线中的位置
04 目录前面的课程已经建立了 executor 的几条主线。
`ExecutorStart()` 把 planner 产物变成 `EState` 和 `PlanState`。
`ExecProcNode()` 让执行节点按需拉取 tuple。
`TupleTableSlot` 让 tuple 值、物理 tuple、descriptor 和 pin 生命周期分离。
`ModifyTable` 把上游 tuple 流转成 INSERT、UPDATE、DELETE、MERGE 的副作用。
第 73 节已经讲过 `partition routing` 在 `ModifyTable` 内如何选择 leaf partition。
本节继续往外扩一圈。
我们要把读路径的 runtime partition pruning、写路径的 tuple routing，以及 UPDATE 分区键变更后的 row movement 放在同一个边界模型里。
这个边界很容易被误读。
读查询里，分区表现为 `Append` 或 `MergeAppend` 的 child subplans。
写查询里，分区表现为 `ResultRelInfo` 数组、partition dispatch、tuple conversion map 和 leaf relation。
UPDATE 分区键时，分区又表现为跨 relation 的 row movement。
这些不是三套独立机制。
它们共享 catalog/relcache 中的 partition bound 与 partition key 语义。
但它们在 executor 中持有的状态不同。
读路径的状态回答：
```text
当前参数下，哪些 subplan 还可能产出 tuple？
```
写路径的状态回答：
```text
当前 tuple 值下，应该写入哪个 leaf ResultRelInfo？
```
row movement 的状态回答：
```text
当前 UPDATE 后 tuple 已经不属于源 leaf，应如何保持 SQL UPDATE 语义并完成 delete+insert？
```
这三句话必须区分。
否则诊断时会出现典型混乱：
- 把 `Append` 的子计划裁剪结果当成 DML 写入目标。
- 把 `ResultRelInfo` 当成 planner subplan。
- 把 tuple conversion map 当成 partition pruning 条件。
- 把 row movement 当成 heap 内的一次普通 `table_tuple_update()`。
- 把 `no partition of relation found for row` 和 scan pruning 误归为同一类错误。
本节把这些状态放进同一条时间线：
```text
planner 建立分区路径和 PartitionPruneInfo
  -> ExecutorStart 初始化 Append/MergeAppend pruning state
  -> ExecProcNode 按参数裁剪读 subplans
  -> ModifyTable 消费 tuple 并定位 source ResultRelInfo
  -> PartitionTupleRouting 按 tuple 值定位 destination ResultRelInfo
  -> tuple conversion 让 slot descriptor 与目标 relation 对齐
  -> row movement 在 UPDATE 跨分区时执行 delete+insert
```
本节的目标不是背函数。
目标是建立一个判断框架：当一个 partitioned DML 或 partitioned SELECT 表现异常时，先问它当前处在“subplan boundary”“tuple routing boundary”还是“row movement boundary”。
## 2. 核心矛盾与一句话运行模型
分区看起来像一个统一 SQL abstraction。
源码里它不是单一对象。
planner 看到的是 inheritance/partition tree、partition bounds、RelOptInfo、Path 和 subplan。
executor 读路径看到的是 `AppendState`、`MergeAppendState`、`PartitionPruneState` 和 bitmapset。
executor 写路径看到的是 `ModifyTableState`、`ResultRelInfo`、`PartitionTupleRouting`、`PartitionDispatch`、slot 和 tuple conversion map。
storage 看到的是最终 leaf relation 的 table AM 调用。
核心 tension 是：
```text
分区语义必须对 SQL 用户表现为一个 relation tree；
但 hot path 必须尽量用已经确定的 leaf relation、slot descriptor 和 subplan index 工作。
```
如果把所有分区状态都做成一个万能 runtime 对象，读写路径会互相拖慢。
读路径不需要打开所有写入相关 trigger、index、FDW insert callback。
写路径不需要为每个 leaf 保存 scan subplan instrumentation。
row movement 又必须同时碰到源 leaf、root rowtype 和目标 leaf。
PostgreSQL 的选择是边界分离：
- planner 保存稳定映射。
- executor scan node 保存 subplan pruning 状态。
- ModifyTable 保存 DML target relation 状态。
- partition routing 只在需要写入 leaf 时初始化 leaf `ResultRelInfo`。
- tuple conversion map 只负责 rowtype 对齐，不负责判断分区是否匹配。
一句话运行模型可以更细化成三段。
读路径：
```text
PartitionPruneInfo
  -> ExecDoInitialPruning()
  -> ExecInitPartitionExecPruning()
  -> PartitionPruneState
  -> ExecFindMatchingSubPlans()
  -> Append/MergeAppend valid subplan bitmap
```
写路径：
```text
root tuple slot
  -> ExecPrepareTupleRouting()
  -> ExecFindPartition()
  -> leaf ResultRelInfo
  -> root-to-child tuple conversion
  -> trigger / constraint / table AM 或 FDW
```
row movement：
```text
UPDATE 新 tuple 不再满足 source leaf partition constraint
  -> ExecCrossPartitionUpdate()
  -> source leaf DELETE
  -> child-to-root conversion
  -> root INSERT
  -> 再走 tuple routing 找 destination leaf
```
这里的关键不是调用链长。
关键是每段链路回答的问题不同。
`PartitionPruneState` 不是“运行时找分区写入”的对象。
它只裁剪读路径中的 subplans。
`PartitionTupleRouting` 不是“优化器 pruning 的执行版”。
它用当前 tuple 的 partition key 值在 executor 中找 leaf relation。
`TupleConversionMap` 不是“转换到正确分区”的决策器。
它只在已经选定 source 或 destination relation 后，让 slot 的 descriptor 符合调用者预期。
`ResultRelInfo` 不是 planner path。
它是某个 relation 在当前 DML 语句中的执行期状态包。
本节围绕这些边界展开。
一个简单的 mental model 是：
```text
pruning works on subplan indexes;
routing works on tuple values;
conversion works on tuple descriptors;
row movement works on DML semantics across relations.
```
这四句话会贯穿后续所有章节。
## 3. 核心文件分工与阅读顺序
阅读顺序按 runtime boundary 展开，不按文件名排序。
| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/partitioning/partprune.c` | planner 侧生成 pruning steps 和 `PartitionPruneInfo`，决定哪些信息能带到 executor。 |
| 2 | `src/include/partitioning/partprune.h` | pruning context、step、planner/executor 共享入口的声明边界。 |
| 3 | `src/backend/optimizer/util/plancat.c` | optimizer 从 relcache/catalog 装载 partition key、bound、partition qual 和 child relation 信息。 |
| 4 | `src/backend/executor/nodeAppend.c` | `Append` 如何在 init、rescan 和执行期使用 runtime pruning 的 valid subplan bitmap。 |
| 5 | `src/backend/executor/nodeMergeAppend.c` | `MergeAppend` 在有序 merge 输入下如何接入 pruning，并处理 child subplan 初始化。 |
| 6 | `src/backend/executor/execPartition.c` | `ExecDoInitialPruning()`、`ExecInitPartitionExecPruning()`、`ExecFindMatchingSubPlans()`、tuple routing、partition dispatch 和 cleanup。 |
| 7 | `src/include/executor/execPartition.h` | `PartitionPruneState`、`PartitionTupleRouting`、`PartitionDispatch` 等执行期结构边界。 |
| 8 | `src/backend/executor/nodeModifyTable.c` | `ExecPrepareTupleRouting()`、`ExecInsert()`、`ExecUpdateAct()`、`ExecCrossPartitionUpdate()` 与 `ResultRelInfo` 对接。 |
| 9 | `src/include/nodes/execnodes.h` | `ModifyTableState`、`ResultRelInfo`、`AppendState`、`MergeAppendState` 的 owner 与生命周期。 |
| 10 | `src/backend/executor/execUtils.c` | root/child tuple conversion map、slot 和 projection 相关 helper。 |
第一轮阅读只抓边界：
```text
partprune.c 生成 planner pruning 信息
  -> nodeAppend.c / nodeMergeAppend.c 消费 subplan bitmap
  -> nodeModifyTable.c 消费 ResultRelInfo 与 row identity
  -> execPartition.c 在 executor 里同时服务 pruning 与 routing
```
第二轮阅读再抓状态：
```text
PartitionPruneInfo
  -> PartitionPruneState
  -> AppendState.as_valid_subplans / MergeAppendState.ms_valid_subplans
```
以及：
```text
PartitionTupleRouting
  -> PartitionDispatch
  -> ResultRelInfo
  -> TupleConversionMap
```
第三轮阅读才看异常路径：
```text
no matching partition
default partition constraint invalidation
runtime pruning parameter change
uninitialized subplan
BEFORE trigger moves tuple out of selected leaf
cross-partition UPDATE concurrency conflict
ON CONFLICT DO UPDATE attempts row movement
```
这种顺序能避免把源码读成“分区 API 清单”。
本节真正关心的是状态谁创建、谁持有、谁能访问、何时失效、出现 ERROR 时谁清理。
## 4. 关键数据结构与状态
本节的数据结构大多是 backend-local。
它们通常挂在 `EState`、`PlanState` 或 `ModifyTableState` 下。
普通指针不能跨 backend 共享。
分区 catalog 信息通过 relcache/syscache 进入 backend。
shared invalidation 让 relcache 在 DDL 后失效，但当前 executor 已持有的 relation locks 和 plan snapshot 决定本语句能看到什么。
### 4.1 `PartitionPruneInfo`
`PartitionPruneInfo` 是 planner 写给 executor 的 pruning 契约。
它不保存 tuple。
它也不保存 `ResultRelInfo`。
它保存的是“如何把 partition bound 判断结果映射到 plan 的 subplan index”。
关键语义包括：
- 哪些 partitioned rel 需要 runtime pruning。
- 每个 partitioned rel 对应哪些 pruning steps。
- partition index 如何映射到 child subplan index。
- nested partition tree 中，非 leaf partition 与下层 rel 的关系。
- 哪些 subplan 无法被 partition pruning 管理。
- 哪些 executor params 会影响 pruning 结果。
这个结构的边界非常重要。
planner 可以在 plan tree 中重排 child plans。
partition bound 的 ordinal 不是 subplan 的 ordinal。
因此 executor 不能拿 partition bound index 直接当 subplan index。
必须经过 `subplan_map` 这类映射。
换句话说：
```text
partition ordinal is a catalog/partition descriptor concept;
subplan index is a plan/executor concept;
PartitionPruneInfo bridges them.
```
这也是 runtime pruning 复杂性的来源之一。
执行期不是重新规划。
执行期只是用 planner 预先留下的映射和 pruning steps，在参数已知时选出 still-valid subplans。
### 4.2 `PartitionPruneState`
`PartitionPruneState` 是 executor 侧对象，由 `ExecDoInitialPruning()` 根据 `EState.es_part_prune_infos` 创建，并通过 `ExecInitPartitionExecPruning()` 挂到 `AppendState` 或 `MergeAppendState`。
它回答的问题是：
```text
当前 executor 参数下，哪些 child subplans 可以跳过？
```
当前结构里最关键的是 `econtext`、`execparamids`、`other_subplans`、`prune_context`、`do_initial_prune`、`do_exec_prune` 和 `partprunedata[]`。
`other_subplans` 保存 pruning 管不到但仍必须执行的 subplans。
`do_initial_prune` 与 `do_exec_prune` 区分 executor startup pruning 和后续 PARAM_EXEC 变化后的 per-scan pruning。
`PartitionPruneState` 是 scan-path 状态。
它不负责 DML tuple routing。
在 gdb 中看到它时，应该先问：
```text
它挂在哪个 Append/MergeAppend node 上？
valid subplan bitmap 是 init-time 结果，还是 exec-time rescan 后结果？
触发它重算的是外部 param、内部 param，还是普通 rescan？
```
不要问：
```text
它最后写入哪个 partition？
```
这不是它的职责。
### 4.3 `AppendState` / `MergeAppendState`
`AppendState` 和 `MergeAppendState` 都可能持有 pruning state，并把 surviving child plans 记录到 `as_valid_subplans` 或 `ms_valid_subplans`。
`Append` 从 valid child subplan 拉 tuple；`MergeAppend` 还要把 valid child 放进有序 merge 结构。
它们不会创建写入用的 `ResultRelInfo`。
如果上层是 `ModifyTable`，写入目标仍由 DML path 决定。
### 4.4 `ModifyTableState`
`ModifyTableState` 是 DML 的顶层 owner，分区相关状态集中在 `resultRelInfo`、`rootResultRelInfo`、`mt_partition_tuple_routing`、`mt_root_tuple_slot`、junk attr、transition capture 和 EPQ state。
它可能从 child scan subplan 得到 source tuple，也可能从 VALUES/SELECT 得到 root 语义 tuple。
UPDATE/DELETE 先用 row identity 找 source `ResultRelInfo`；INSERT 或 row movement 再用 `PartitionTupleRouting` 找 destination leaf。
### 4.5 `ResultRelInfo`
`ResultRelInfo` 是某个 relation 在当前 DML 语句中的执行期状态包，不是 relcache entry，也不是 planner path。
它聚合 `Relation` descriptor、trigger、index、constraint、WCO、RETURNING、FDW/table AM callback、partition check expression 和 root/child conversion map。
同一个 partitioned DML 可能同时有 root、source leaf、destination leaf、routing 懒初始化和 borrowed `ResultRelInfo`。
所以 `ResultRelInfo *` 指针本身不是语义。
语义来自：
```text
ResultRelInfo + root relation + operation + owner + conversion map + current slot
```
诊断 DML 时，必须先确认当前指针是 source 还是 destination。
尤其在 cross-partition UPDATE 中，源 leaf DELETE 和目标 leaf INSERT 都会经过 `ResultRelInfo`。
把两者混淆，会导致错误地解释 trigger、RETURNING 和 row count。
### 4.6 `PartitionTupleRouting`
`PartitionTupleRouting` 是 executor 写路径的路由状态。
它由 `ExecSetupPartitionTupleRouting()` 创建。
它通常挂在 `ModifyTableState.mt_partition_tuple_routing`。
它的核心职责是：
```text
给定 root table 语义下的 tuple，找到应写入的 leaf ResultRelInfo。
```
它的当前结构以 `partition_root`、`memcxt`、`partition_dispatch_info`、`nonleaf_partitions`、`partitions` 和 `is_borrowed_rel` 为核心；partition descriptor 查询目录在 `EState.es_partition_directory`。
`PartitionTupleRouting` 与 `PartitionPruneState` 的最大差异是输入。
`PartitionPruneState` 的输入是 executor params。
`PartitionTupleRouting` 的输入是 tuple values。
一个参数化查询可能先在读路径裁剪 subplans，再在写路径把更新后的 tuple 路由到另一个 leaf。
这两步使用同一套 partition key 语义。
但运行时对象完全不同。
### 4.7 `PartitionDispatch`
`PartitionDispatch` 表示一个 partitioned relation 层级上的 dispatch 节点。
它不是 leaf relation 写入状态。
它用于在某一级 partitioned table 上计算 partition key，并查找下一层 partition。
当前 `PartitionDispatchData` 持有 `reldesc`、`key`、`keystate`、`partdesc`、可选 `tupslot` / `tupmap`，以及 child partition 到 dispatch/result-rel 数组的 `indexes[]`。
多级分区路由就是重复：
```text
当前 dispatch relation rowtype
  -> 计算 partition key
  -> 找到 child index
  -> 如果 child 是 leaf，返回 ResultRelInfo
  -> 如果 child 还是 partitioned table，转换 slot 后进入下一个 dispatch
```
这里的 conversion 很关键。
不同层级 partitioned table 和 leaf 可能有不同列顺序、dropped column、generated column 或 attisdropped 位置。
路由不能假设同一个 `TupleTableSlot` descriptor 可以一路传到底。
### 4.8 tuple conversion map
tuple conversion map 负责在两个 tuple descriptor 之间搬运属性。
它回答的问题是：
```text
这个 slot 的 attribute values 如何按另一个 relation 的 TupleDesc 重新排列？
```
它不回答：
```text
这个 tuple 应该属于哪个 partition？
```
常见方向包括：
- root-to-child。
- child-to-root。
- source leaf-to-root。
- root-to-destination leaf。
- `RETURNING` 或 trigger 需要的 old/new tuple descriptor。
conversion 的输出通常是另一个 slot。
这意味着它引入了 per-tuple CPU 成本和 memory context 生命周期。
但它也让 executor 能保持一个强不变量：
```text
table AM / trigger / constraint 总是看到与当前 ResultRelInfo 匹配的 tuple descriptor。
```
如果没有这个不变量，dropped column、attribute reorder 和 inherited table 历史兼容性会污染所有下游代码。
### 4.9 partition constraint expression
partition constraint 是 leaf relation 层面的正确性边界。
它通常在路由后检查。
它与 pruning 条件相关，但不是同一个对象。
runtime pruning 用 pruning steps 判断 subplans 是否可能匹配。
partition constraint 用 executor expression 判断当前 tuple 是否仍属于目标 leaf。
这一区分解释了两个常见错误：
```text
no partition of relation found for row
```
通常发生在 routing 时找不到 matching leaf。
```text
new row for relation ... violates partition constraint
```
通常发生在已经有 target relation 后，当前 tuple 不满足该 leaf constraint。
特别是 `BEFORE ROW` trigger 改写 tuple 后，第二类错误更常见。
### 4.10 planner path 与 executor state 的映射关系
最终需要记住四个 index 空间。
| 空间 | 属于谁 | 典型对象 | 不能混淆为 |
| --- | --- | --- | --- |
| partition bound index | relcache/partition descriptor | `PartitionDesc` | subplan index |
| subplan index | plan/executor scan node | `Append` child plan | relation oid |
| result rel index | ModifyTable | `ResultRelInfo` 数组 | partition bound index |
| dispatch child index | tuple routing | `PartitionDispatch` | child plan position |
很多分区 bug 和诊断误判，本质是把这几个 index 空间混用了。
本节后面的主流程会反复回到这张表。
## 5. 主流程源码 walkthrough
本节走两条主链路，再把它们在 `ModifyTable` 上汇合。
第一条是读路径 runtime pruning。
第二条是写路径 tuple routing 和 row movement。
### 5.1 planner 侧留下 pruning 契约
planner 首先从 catalog/relcache 获取 partition metadata。
`plancat.c` 负责把 relation 相关信息装入 optimizer 可用的结构。
这里会涉及 partition key、partition bound、partition qual、child relation 等信息。
这些信息进入 optimizer 后，planner 可以做 plan-time pruning。
如果查询条件在计划期已经足够确定，某些 partitions 不会变成 child paths。
如果条件依赖 runtime params，planner 不能直接删掉对应 subplans。
它需要留下 runtime pruning 信息。
抽象流程是：
```text
get relation partition metadata
  -> build partitioned RelOptInfo
  -> generate partition pruning steps
  -> build PartitionPruneInfo for Append/MergeAppend
  -> store part_prune_index in plan node
```
这里要注意“留下信息”和“重新规划”的区别。
runtime pruning 不会在执行期重新生成 path。
它只执行 planner 已经生成的 pruning steps，并把结果映射成 subplan bitmap。
因此运行期能裁剪的范围受 planner 产物限制。
如果 planner 没有留下可执行的 pruning step，executor 不会凭空从 SQL qual 里重新推导分区排除。
### 5.2 `Append` 初始化 runtime pruning
`nodeAppend.c` 是 runtime pruning 最直观的入口。
`ExecutorStart` 初始化 `EState` 时会把 `plannedstmt->partPruneInfos` 放到 `estate->es_part_prune_infos`，随后 `ExecDoInitialPruning()` 建立 `es_part_prune_states` 和 `es_part_prune_results`。
`ExecInitAppend()` 看到 plan node 上的 `part_prune_index` 后，再调用 `ExecInitPartitionExecPruning()` 取回对应 pruning state 和 initial valid subplans。
抽象调用链：
```text
ExecutorStart()
  -> ExecDoInitialPruning()
     -> CreatePartitionPruneState()
     -> maybe ExecFindMatchingSubPlans(initial_prune=true)
ExecInitAppend()
  -> ExecInitPartitionExecPruning()
     -> read estate lists by part_prune_index
     -> initialize exec pruning context if needed
  -> initialize required child planstates
```
init-time pruning 的输入可能包括 executor startup 阶段已经知道的 params。
它可以减少需要初始化的 child planstates。
这对大量分区表很重要。
因为初始化每个 child scan 都可能打开 relation、初始化 expression state、分配 slot 和 instrumentation。
如果 init-time pruning 能证明某些 child 不会用到，就可以避免这部分启动成本。
但这不是无条件成立。
如果参数只有执行过程中才变化，或者 rescan 时才确定，就需要 exec-time pruning。
### 5.3 `Append` 执行期更新 valid subplans
执行期，`Append` 的主要任务是从 valid child subplans 中拉 tuple。
如果相关 `PARAM_EXEC` 发生变化，runtime pruning 结果需要重算。
抽象流程：
```text
ExecReScanAppend()
  -> if pruning params changed
       ExecFindMatchingSubPlans()
       update as_valid_subplans
  -> rescan selected child planstates
```
然后执行：
```text
ExecAppend()
  -> choose next valid subplan
  -> ExecProcNode(child)
  -> if child exhausted, move to next valid subplan
```
这里的 bitmap 是 `subplan index` 空间。
它不直接告诉你 partition OID。
要从 EXPLAIN 或 gdb 诊断时，需要通过 plan tree 和 child relation 映射回具体 leaf。
### 5.4 `MergeAppend` 的特殊性
`MergeAppend` 也可以接入 partition pruning。
差异在于它要维护多个 child 的有序 merge 状态。
被 pruning 掉的 child 不应进入 heap 或 sort support 的比较循环。
抽象流程：
```text
ExecInitMergeAppend()
  -> ExecInitPartitionExecPruning()
  -> initialize valid child planstates
  -> setup sort support / binary heap for active children
```
执行时：
```text
ExecMergeAppend()
  -> pull smallest tuple among active children
  -> advance that child
  -> maintain heap only for valid children
```
`MergeAppend` 的 pruning 成本不只是少扫几个 relation。
它还减少 merge heap 的参与者数量。
但如果 ORDER BY / pathkey 要求导致必须保留多个 child path，planner/executor 仍要在正确性边界内维护排序。
### 5.5 pruning 结果如何回到可观测行为
runtime pruning 最直接的可见入口是 EXPLAIN。
典型现象包括：
- `Subplans Removed` 显示 init-time pruning 移除的 subplans。
- `EXPLAIN ANALYZE` 中某些 child plan 没有执行 loops。
- 参数化 nested loop 下，inner append 的实际 loops 与 child loops 分布不均。
- planning time 已裁剪的 partitions 根本不出现在 plan tree。
要小心这几个口径：
`Subplans Removed` 不是 executor 每次 rescan 的完整历史。
它通常更接近 init-time pruning 的静态可见结果。
exec-time pruning 可能表现为 child plan loops 为 0 或远低于父节点 loops。
但 EXPLAIN 不会把每次 param 值对应的 pruned bitmap 全部列出来。
这就是“能观测”和“只能推断”的边界。
### 5.6 `ModifyTable` 消费上游 tuple
现在切到写路径。
`ModifyTable` 的上游可以是普通 plan，也可以包含 `Append` 或 `MergeAppend`。
例如：
```text
UPDATE partitioned_table
SET key = key + 100
WHERE key = $1;
```
读阶段可能先通过 runtime pruning 只扫描一个 source leaf。
但 update 后的新 key 可能属于另一个 leaf。
这时写路径不能复用读路径的 pruning 结果。
原因很简单：
```text
pruning answered where old rows may be found;
routing answers where new rows must be written.
```
`ExecModifyTable()` 从 subplan 拉取 candidate slot 后，根据 operation 进入 `ExecInsert()`、`ExecUpdate()`、`ExecDelete()` 或 `ExecMerge()`。
对于 UPDATE/DELETE，junk attr 帮助定位源 tuple。
对于 inherited/partitioned UPDATE，`tableoid` 或 planner 生成的 row identity 信息让 executor 找到 source `ResultRelInfo`。
这一刻仍然不是 destination leaf routing。
它只是确定“要修改哪张源表上的哪一行”。
### 5.7 INSERT 的 tuple routing
INSERT into partitioned root 的主链路是：
```text
ExecInsert()
  -> ExecPrepareTupleRouting()
     -> ExecFindPartition()
        -> walk PartitionDispatch levels
        -> find leaf ResultRelInfo
     -> convert tuple to leaf descriptor if needed
  -> BEFORE ROW INSERT trigger on leaf
  -> generated columns / WCO / constraints
  -> table_tuple_insert() or FDW insert
  -> indexes / AFTER triggers / RETURNING
```
`ExecPrepareTupleRouting()` 是 `ModifyTable` 与 partition routing 的关键交界。
它接收 root 语义下的 slot。
它调用 `ExecFindPartition()` 找到 leaf。
如果 leaf relation descriptor 与当前 slot 不同，它会准备或使用 tuple conversion map。
返回后，`ExecInsert()` 面对的是 leaf `ResultRelInfo` 和 leaf descriptor 的 slot。
这保证了后面的 trigger、constraint、index 和 table AM 不需要理解 root partition tree。
但 trigger 会制造一个重要异常。
`BEFORE ROW INSERT` trigger 在 leaf 上执行。
如果 trigger 改写 tuple，使它不再满足该 leaf 的 partition constraint，PostgreSQL 通常报错。
它不会在 trigger 后重新 routing 到另一个 leaf。
这个选择维持了 trigger 顺序和 transition table 语义。
### 5.8 UPDATE 不移动分区时的路径
普通 UPDATE 的主链路是：
```text
ExecUpdate()
  -> locate source ResultRelInfo and old tuple
  -> ExecUpdatePrologue()
     -> BEFORE ROW UPDATE trigger
  -> ExecUpdateAct()
     -> check new tuple against source leaf partition constraint
     -> table_tuple_update()
     -> indexes / AFTER triggers / RETURNING
```
如果新 tuple 仍属于 source leaf，就可以在同一 relation 内完成 update。
这时不需要 destination tuple routing。
tuple conversion 仍可能出现在 RETURNING、root/child projection 或 transition table 路径。
但 partition routing 不参与写入目标选择。
### 5.9 UPDATE 跨分区时的 row movement
如果 UPDATE 后的 tuple 不再满足 source leaf partition constraint，语义变成跨分区 UPDATE。
抽象流程：
```text
source leaf UPDATE candidate
  -> new tuple violates source partition constraint
  -> ExecCrossPartitionUpdate()
     -> DELETE old tuple from source leaf
     -> convert new tuple to root rowtype
     -> INSERT through root partition routing
     -> destination leaf table insert
```
这里不是 storage 内部移动 tuple。
不同 partitions 是不同 relations。
它们有不同 relfilenode、indexes、triggers、constraints、FDW/table AM 状态。
CTID 不能跨 relation 保持同一行 identity。
所以 row movement 是 executor 层的 delete+insert 语义组合。
这带来几个后果：
- 源 leaf 的 DELETE trigger 和目标 leaf 的 INSERT trigger 都可能参与。
- 外键、transition table 和 RETURNING 要按 SQL 语义处理 old/new tuple。
- 并发 UPDATE/DELETE 可能进入 EPQ 或冲突处理路径。
- `ON CONFLICT DO UPDATE` 不允许把冲突行移动到另一个 partition。
- row movement 可能比同 leaf UPDATE 产生更多 WAL、index 操作和 trigger 成本。
### 5.10 tuple conversion 在 row movement 中的位置
row movement 需要至少两个方向的 conversion。
第一步是 source leaf to root。
原因是 destination leaf 还没确定。
routing API 以 root relation 语义理解 tuple。
所以从 source leaf 得到的新 tuple，要先转回 root descriptor。
第二步是 root to destination leaf。
`ExecFindPartition()` 找到目标 leaf 后，再把 root slot 转成 leaf descriptor。
抽象链路：
```text
source leaf slot
  -> child-to-root map
  -> root slot
  -> routing by partition key values
  -> root-to-destination-child map
  -> destination leaf slot
```
这解释了为什么 tuple conversion 不只是 INSERT into partitioned table 的细节。
它也是 row movement 正确性的核心。
只要 source 和 destination leaf 的 physical descriptor 不完全一致，跳过中间 root 语义就会出错。
### 5.11 `ResultRelInfo` borrowed 与 lazy init
partition routing 找到 leaf 后，需要一个 leaf `ResultRelInfo`。
有两种来源。
第一种是 `ModifyTable` 初始化时已有的 result rel。
例如 UPDATE/DELETE/MERGE 的 target leaf 可能已经在 result rel array 中。
routing 可以借用它。
第二种是 routing 时第一次触达该 leaf。
executor 需要打开 relation，初始化 `ResultRelInfo`、indexes、triggers、WCO、RETURNING 或 FDW state。
这就是 lazy init。
它降低了大量分区表上单行写入的启动成本。
但也让 runtime 里出现更多条件分支。
诊断内存泄漏或 Relation 未关闭时，需要知道这个 `ResultRelInfo` 是 borrowed 还是 routing 自己创建。
borrowed 对象不由 `ExecCleanupTupleRouting()` 关闭。
动态创建对象需要在 cleanup 中释放 relation/index/FDW 相关状态。
### 5.12 `Append` pruning 与 `ModifyTable` routing 的汇合点
把读写路径放在同一条 SQL 中看：
```text
UPDATE root
SET partkey = partkey + 10
WHERE partkey = $1;
```
运行时可能发生：
```text
Append runtime pruning uses $1
  -> scans only source partitions that can contain old rows
  -> ModifyTable receives old row
  -> UPDATE expression computes new partkey
  -> source leaf partition constraint fails
  -> row movement converts child-to-root
  -> routing uses new partkey
  -> destination leaf receives insert
```
这条链路是本节最重要的现场模型。
它说明：
读路径 pruning 与写路径 routing 可以在同一条 SQL 中同时出现。
它们都依赖 partition key。
但它们处理的是不同时间点的值。
pruning 处理 WHERE 条件和 executor params 下的 old row search space。
routing 处理 UPDATE 后 new tuple 的 destination。
row movement 把这两个世界连接起来。
### 5.13 planner/executor boundary 的不变量
主流程可以压缩成五个不变量。
第一，executor 不重新规划 partition paths。
它只消费 planner 给出的 pruning steps、subplan mapping 和 plan tree。
第二，runtime pruning 只影响 scan child subplans。
它不决定 DML destination leaf。
第三，tuple routing 只根据当前 tuple 值选择 leaf `ResultRelInfo`。
它不修改 planner subplan tree。
第四，tuple conversion 只解决 descriptor 对齐。
它不证明 partition constraint 一定满足。
第五，row movement 是 DML 语义层的 delete+insert。
它不是 heap/table AM 内部移动。
这些不变量是后面正确性、异常和成本分析的基础。
## 6. 生命周期 / ownership / cleanup
本节对象的生命周期分三层。
### 6.1 plan tree 生命周期
`PartitionPruneInfo` 属于 planned statement。
它由 planner 创建。
executor 读取它，但不拥有它的长期语义。
prepared statement 或 cached plan 场景下，plan cache invalidation 决定它是否还能复用。
DDL 改变分区结构后，relcache invalidation 与 plan invalidation 会让旧 plan 失效。
如果旧 plan 正在执行，当前语句基于已有 snapshot 和 locks 完成。
不要把 runtime pruning 当成处理 DDL 变更的机制。
### 6.2 scan node 生命周期
`PartitionPruneState` 由 `ExecDoInitialPruning()` 在 `EState` 中创建，`AppendState` / `MergeAppendState` 通过 `ExecInitPartitionExecPruning()` 取得并持有指针。
表达式计算使用 `ExprContext` 和专门的 `prune_context`，`ExecFindMatchingSubPlans()` 结束时会 copy bitmap 并 reset 该临时 context。
`ExecutorEnd()` 释放 planstate tree 时，这些 backend-local 内存被 context reset。
如果执行中 ERROR，PostgreSQL 的错误栈会回滚到上层 context。
普通 backend-local 指针不需要跨事务保留。
但已经打开的 relation、buffer pin、tuple table slot、FDW state 等资源仍要靠 executor cleanup、ResourceOwner 和 memory context 共同收尾。
### 6.3 DML routing 生命周期
`PartitionTupleRouting` 由 `ExecSetupPartitionTupleRouting()` 创建。
owner 通常是 `ModifyTableState`。
它有自己的 memory context 或挂在 executor state 的长生命周期 context 中。
它持有 partition directory、dispatch 数组、leaf result rel 数组和 conversion slot。
cleanup 入口是 `ExecCleanupTupleRouting()`。
cleanup 需要区分 borrowed 和 owned `ResultRelInfo`。
owned relation/index/FDW state 需要释放。
borrowed `ResultRelInfo` 由 `ModifyTableState` 的正常 cleanup 负责。
这一区分是 ownership 的核心。
### 6.4 tuple slot、Relation 与 ERROR cleanup
slot 是 per executor 状态；root slot、child slot、routing slot、RETURNING slot 可能同时存在。
conversion map 输出的 slot 只在当前 tuple 处理链路中有效，per-tuple context reset 后不能继续引用内部临时 Datum。
executor 访问 leaf relation 还需要 relation descriptor、锁和 relcache 有效性；DDL 并发由 lock、relcache invalidation 和 plan invalidation 控制。
分区 routing 和 pruning 大多是 backend-local 内存。
ERROR 时，MemoryContext reset 会释放普通分配。
但是外部资源不能只靠 memory context。
例如 relation refcount、buffer pin、index open state、FDW callbacks、trigger event queue，都有各自 cleanup 路径。
`ExecutorEnd()`、`ExecCloseResultRelations()`、`ExecCleanupTupleRouting()`、ResourceOwner release 和 transaction abort cleanup 共同收尾。
诊断 ERROR 后的泄漏时，先区分：
```text
backend-local memory
relation/index refcount
buffer pin
FDW remote state
trigger queued event
```
不要只看一个 memory context。
## 7. 正确性机制层次
分区执行期正确性不是一个机制保证的。
它是一组边界叠加。
| 层次 | 机制 | 保证 | 不保证 |
| --- | --- | --- | --- |
| planner contract | `PartitionPruneInfo` / subplan map | executor 能把 pruning 结果映射回 subplans | 执行期重新规划 |
| executor params | `PARAM_EXEC` 依赖跟踪 | 参数变化时 pruning 可重算 | 任意表达式都能 runtime pruning |
| relation identity | locks / relcache / plan invalidation | 当前语句看到稳定 relation tree | 无锁 DDL 同时安全修改 |
| tuple descriptor | conversion map / slot | 下游看到匹配 relation 的 tuple | 判断 tuple 属于哪个 partition |
| partition membership | bound lookup / partition constraint | tuple 属于目标 leaf | trigger 改写后自动 reroute |
| DML semantic | `ModifyTable` / `ResultRelInfo` | trigger、constraint、RETURNING、FDW/table AM 顺序正确 | row movement 是原地 update |
| MVCC/concurrency | snapshot / EPQ / table AM result | 并发 UPDATE/DELETE 可重检或报错 | 分区 routing 本身解决并发冲突 |
### 7.1 pruning 的正确性
runtime pruning 必须是保守的。
如果无法证明某个 partition 不可能匹配，就必须保留对应 subplan。
裁剪错误会造成漏读。
保留过多只会造成性能下降。
因此 pruning steps 的表达能力和 parameter tracking 是 correctness boundary。
executor 不能因为“看起来条件很窄”就随意跳过 subplan。
必须通过 planner 留下的 pruning state。
### 7.2 routing 的正确性
tuple routing 必须找到唯一 leaf。
如果找不到，报错。
如果找到 default partition，还要考虑 default partition constraint 是否排除新增 partitions 的范围。
多级分区中，每一级的 partition key 都按该级 relation descriptor 计算。
所以 conversion 必须发生在正确位置。
否则用错 descriptor 计算 key，会把 tuple 路由到错误 leaf。
### 7.3 row movement 的正确性
row movement 的正确性来自 DML 语义重组。
源 relation 上的旧 tuple 要被删除。
目标 relation 上的新 tuple 要被插入。
触发器、外键、transition table、RETURNING 和 command tag 仍要表现为 UPDATE 语义。
这比单纯 delete+insert 复杂。
例如并发冲突时，executor 可能需要 EPQ recheck。
如果目标插入又触发冲突或约束错误，整个语句要按事务错误语义回滚。
### 7.4 descriptor 正确性
PostgreSQL 长期支持 inheritance 和 dropped columns。
不同 partitions 的 physical layout 可能不完全一样。
`TupleDesc` 对齐是 correctness，不是优化。
table AM、trigger、constraint expression、RETURNING projection 都假设 slot descriptor 与当前 `ResultRelInfo` 一致。
跳过 conversion 可能在简单测试中工作。
但遇到 dropped column、列顺序变化、generated column 或不同 child descriptor 时会出错。
### 7.5 不涉及 WAL 的原因
本节不是 WAL 课程。
partition pruning 和 tuple routing 本身不写 WAL。
真正 WAL 由 leaf table AM、index AM、toast、visibility map、FSM 等下游模块产生。
但是 row movement 会放大 WAL。
因为它可能从一次同 leaf UPDATE 变成 source DELETE 加 destination INSERT，再加两边 index 维护。
所以成本章节会讨论 WAL 传播，但正确性主边界仍在 executor DML 语义。
## 8. 错误路径 / 异常路径 / fallback
### 8.1 runtime pruning fallback
如果 planner 没生成 runtime pruning steps，executor 不会临时发明；如果 `PARAM_EXEC` 改变，`Append` 会清空 valid bitmap 并在后续重新调用 `ExecFindMatchingSubPlans()`。
不能证明可裁剪时必须保留 subplan；误裁剪会漏读，漏裁剪只会变慢。
不归 pruning 管的 child subplans 通过 `other_subplans` 保留。
典型现象是 `Subplans Removed` 很少、child loops 分布不均，或 CPU 花在 append rescan/pruning expression 上。
### 8.2 INSERT 找不到 partition
INSERT routing 时，如果 partition key 值没有匹配 leaf，也没有可用 default partition，会报错。
典型错误是：
```text
no partition of relation "..." found for row
```
这发生在写路径。
它与 scan pruning 无关。
即使 SELECT 的 WHERE 可以裁剪到某个 partition，INSERT 的 tuple 值仍可能没有目标 partition。
### 8.3 default partition 约束变化
新增 partition 会改变 default partition 的隐含约束。
executor 通过 relcache、partition descriptor 和 constraint expression 看到当前语句边界内的状态，DDL 并发由 locks 和 invalidation 管控。
default partition 不是永久兜底；它接收的是当前 partition bound 集合下的剩余空间。
### 8.4 BEFORE trigger 改写 partition key
INSERT 路由先选择 leaf，再执行 leaf 的 `BEFORE ROW INSERT` trigger。
如果 trigger 改写 partition key，使 tuple 不再属于该 leaf，通常报 partition constraint 错误。
系统不会自动 reroute。
这是刻意选择。
否则 trigger 能把 tuple 推向任意 leaf，导致 trigger 执行顺序、transition table、FDW state 和 recursion control 变得难以定义。
### 8.5 UPDATE 分区键改变
UPDATE 后 tuple 如果仍属于源 leaf，就走普通 update。
如果不属于源 leaf，且语句语义允许跨分区 UPDATE，就进入 row movement。
异常包括：
- source tuple 被并发修改，需要 EPQ。
- source DELETE 失败或发现 tuple 已被删除。
- destination INSERT 触发 constraint、trigger 或 unique violation。
- `ON CONFLICT DO UPDATE` 试图移动 row。
- FDW partition 不支持所需写入回调。
这些异常不能只看分区 routing。
它们落在 DML 并发、table AM、trigger 或 FDW 边界。
### 8.6 tuple conversion 失败
conversion map 通常在 tuple descriptors 不兼容时失败。
例如目标缺少必要属性、类型不兼容，或 inheritance/partition tree 不满足 executor 预期。
这类错误往往在 executor init 或第一次触达 leaf 时暴露。
它不是 pruning 失败。
它说明已经选定某个 relation，但 slot 无法安全转换成该 relation 的 rowtype。
### 8.7 lazy init 中的 ERROR
第一次路由到某个 leaf 时，executor 可能打开 relation、初始化 indexes、trigger、FDW state 和 slots。
ERROR 后 cleanup 依赖 executor memory context、ResourceOwner 和已经注册的 cleanup 路径；诊断时应问：
```text
对象是否已经放入 routing array？
is_borrowed_rel 是否已设置？
Relation/index/FDW state 是否需要关闭？
```
### 8.8 cached plan 失效
prepared statement 可能缓存包含 partition pruning 信息的 plan。
如果 DDL 改变 partition tree，plan cache invalidation 会使旧 plan 失效。
下一次执行重新规划。
runtime pruning 不是 plan invalidation 的替代品。
它不能修复过期的 subplan map。
这一区分对线上问题很重要。
如果观察到 prepared statement 在 DDL 后重新 planning，先看 invalidation。
不要把它归因于 executor pruning 变慢。
## 9. 成本、资源与跨模块传播
分区执行期成本来自四个维度。
### 9.1 分区数量
partition 数量增加会影响 planner 和 executor。
planner 要构造更多 child paths、bounds 和 mapping。
executor init 可能要初始化更多 child planstates。
runtime pruning 能减少实际扫描。
但 pruning state 自身也有初始化和表达式成本。
如果每次查询只命中一个 partition，大量 partitions 的主要成本可能转移到 planning、plan cache invalidation、ExecutorStart 和 relcache。
如果参数化 rescan 高频发生，成本可能转移到 `ExecFindMatchingSubPlans()`。
### 9.2 tuple 数量
写路径成本随 tuple 数量线性放大。
每行可能经历 partition key expression、bound lookup、tuple conversion、partition constraint、trigger、table AM/FDW 写入、index maintenance 和 RETURNING projection。
单行成本看起来不大。
批量 INSERT/UPDATE 跨大量 partitions 时，conversion 和 lazy init 会变得明显。
### 9.3 descriptor 差异
如果 root 和 leaf descriptors 一致，conversion map 可以很轻或为空。
如果存在 dropped columns、列顺序差异或多级 inheritance，conversion 会进入 hot path。
这会增加 CPU、cache miss 和 per-tuple memory churn。
更重要的是，它让 gdb 观察 slot 时更容易误判。
同一行在 root slot 和 child slot 中的 attnum 语义不同。
### 9.4 row movement 放大
跨分区 UPDATE 通常比同 leaf UPDATE 更贵。
它可能带来：
- source DELETE 的 WAL。
- destination INSERT 的 WAL。
- 两边 index 维护。
- 两边 trigger。
- foreign key check。
- transition table capture。
- tuple conversion 两次。
- EPQ 或并发冲突重试。
因此生产中如果 UPDATE 大量修改 partition key，不能按普通 UPDATE 成本估算。
### 9.5 relcache 和 plan cache 压力
大量 partitions 会让 relcache/syscache 压力上升。
DDL 变更会触发 invalidation。
prepared statements 可能频繁失效重建。
runtime pruning 只能减少执行期扫描。
它不能消除 planning 或 cache invalidation 成本。
这也是分区数量设计需要保守的原因。
### 9.6 跨模块传播
本节至少连接这些模块：
| 模块 | 边界 |
| --- | --- |
| optimizer | 生成 paths、pruning steps、subplan mapping。 |
| relcache/syscache | 提供 partition key、bound、descriptor、constraint。 |
| executor scan nodes | Append/MergeAppend 消费 valid subplans。 |
| ModifyTable | 消费 source tuple，执行 DML semantics。 |
| trigger manager | BEFORE/AFTER trigger 可能改写或观察 tuple。 |
| table AM / index AM | 最终 leaf relation 的物理修改和 WAL。 |
| FDW | foreign partition 写入需要 FDW callbacks。 |
| plan cache | DDL invalidation 后重新规划。 |
没有单个指标能覆盖这些成本。
必须按边界拆开看。
## 10. 观测与诊断入口
本节锚定的 runtime truth 是：
```text
同一条 partitioned SQL 中，old row search space 由 pruning 控制，
new row destination 由 routing 控制，
descriptor correctness 由 conversion 控制。
```
观测目标是把这三者分开。
### 10.1 EXPLAIN 看 pruning
使用：
```sql
EXPLAIN (ANALYZE, VERBOSE, BUFFERS)
SELECT * FROM p WHERE k = 42;
```
关注：
- `Append` 或 `MergeAppend` 下出现哪些 child plans。
- 是否有 `Subplans Removed`。
- child plan 的 `Actual Loops`。
- planning time 是否异常。
- buffers 是否集中在少数 partitions。
解释边界：
`Subplans Removed` 不能代表每次 exec-time pruning 的完整历史。
如果是参数化 nested loop，child loops 才能间接反映 pruning 效果。
如果某个 partition 在 plan-time 已被裁剪，它不会出现在 plan tree。
### 10.2 EXPLAIN 看 ModifyTable
使用：
```sql
EXPLAIN (ANALYZE, VERBOSE, BUFFERS, WAL)
UPDATE p
SET k = k + 100
WHERE k = 42;
```
关注：
- 上游 scan 是否通过 Append/MergeAppend 访问 partitions。
- ModifyTable 的 target relation。
- child scan loops 与实际 affected rows。
- WAL 量是否接近 delete+insert。
- trigger 时间。
解释边界：
EXPLAIN 不会逐行显示 destination leaf routing。
你可能看到 source scan 只扫一个 partition。
但 destination insert 可能进入另一个 partition。
这需要结合 trigger、WAL、row count、错误信息或断点推断。
### 10.3 错误信息分类
常见错误分类：
```text
no partition of relation found for row
```
优先看写路径 routing。
```text
new row for relation violates partition constraint
```
优先看已选 target leaf 后的 constraint check，尤其是 trigger 改写。
```text
tuple to be locked was already moved to another partition
```
优先看 concurrent row movement 和 EPQ。
```text
invalid per-tuple memory access 或 slot descriptor assert
```
优先看 tuple conversion、slot owner 和 descriptor mismatch。
### 10.4 gdb 断点
源码跟读可设置断点：
```text
break ExecDoInitialPruning
break ExecInitPartitionExecPruning
break ExecFindMatchingSubPlans
break ExecPrepareTupleRouting
break ExecFindPartition
break ExecCrossPartitionUpdate
break ExecCleanupTupleRouting
```
观察顺序：
```text
AppendState or MergeAppendState
  -> PartitionPruneState
  -> valid subplans bitmap
  -> ModifyTableState
  -> source ResultRelInfo
  -> PartitionTupleRouting
  -> destination ResultRelInfo
  -> conversion slot descriptor
```
不要只打印 OID。
还要确认当前 slot 的 descriptor。
对 tuple conversion 问题，打印 `slot->tts_tupleDescriptor` 和 `resultRelInfo->ri_RelationDesc->rd_att` 比打印 tuple 值更关键。
### 10.5 perf、pg_stat 与 wait event
CPU overhead 需要 perf/flamegraph 区分 `ExecFindMatchingSubPlans()`、`ExecFindPartition()`、slot deforming、tuple conversion、trigger 和 table/index AM。
`pg_stat_statements` 可看到总耗时、rows、shared blocks、wal bytes。
`pg_stat_io` 可看 relation/index IO 分布，但不直接告诉 pruning bitmap。
`pg_locks` 可看 DDL/DML lock contention。
`pg_stat_activity` wait event 可看 lock、IO 或 WAL flush wait。
这些指标都是间接证据。
它们不能直接回答“这一行路由到了哪个 leaf”。
临时 instrumentation 可以放在 `ExecFindMatchingSubPlans()`、`ExecFindPartition()`、conversion map 创建处和 `ExecCrossPartitionUpdate()`，但 per-tuple 日志只适合实验分支。
### 10.6 能看到、只能推断、看不到
| 类型 | 示例 |
| --- | --- |
| 能直接看到 | plan tree、child loops、Subplans Removed、WAL bytes、trigger time、错误信息。 |
| 只能推断 | exec-time pruning 每次参数下的 bitmap、每行 destination leaf、conversion map 命中频率。 |
| 基本看不到 | backend 内部临时 slot 生命周期、per-tuple context 中短生命周期 Datum、半初始化 routing state。 |
诊断时要诚实标注推断边界。
不要把 EXPLAIN 的 child loops 解释成完整路由历史。
## 11. 常见误区
### 误区一：runtime pruning 会决定 INSERT 写入哪个分区
不会。
runtime pruning 裁剪 scan subplans。
INSERT/UPDATE destination 由 tuple routing 决定。
### 误区二：source partition 就是 destination partition
只对不改变 partition key 的 UPDATE 常常成立。
一旦 UPDATE 后 tuple 不再满足 source leaf constraint，就可能 row movement。
### 误区三：tuple conversion 是性能优化
conversion 首先是 correctness。
它保证下游 relation-specific 代码看到正确 descriptor。
### 误区四：`ResultRelInfo` 等于 relation
`ResultRelInfo` 是当前语句的执行期状态包。
同一个 relation 在不同语句、不同 operation、不同 owner 下状态不同。
### 误区五：default partition 永远兜底
default partition 只代表当前 bound 集合下的剩余空间。
新增 partition 或 constraint 变化会改变它的语义。
### 误区六：EXPLAIN 能显示每行路由
不能。
EXPLAIN 能帮助看 scan pruning 和 DML 总体成本。
每行 routing 需要断点、临时日志或从错误/WAL/trigger 行为推断。
## 12. 课堂实验
### 实验一：区分 plan-time pruning、init-time pruning 和 exec-time pruning
准备表：
```sql
CREATE TABLE p (k int, v text) PARTITION BY RANGE (k);
CREATE TABLE p0 PARTITION OF p FOR VALUES FROM (0) TO (100);
CREATE TABLE p1 PARTITION OF p FOR VALUES FROM (100) TO (200);
CREATE TABLE p2 PARTITION OF p FOR VALUES FROM (200) TO (300);
INSERT INTO p SELECT g, md5(g::text) FROM generate_series(0, 299) g;
ANALYZE p;
```
常量查询：
```sql
EXPLAIN (ANALYZE, VERBOSE, BUFFERS)
SELECT * FROM p WHERE k = 42;
```
参数化查询：
```sql
PREPARE q(int) AS SELECT * FROM p WHERE k = $1;
EXPLAIN (ANALYZE, VERBOSE, BUFFERS) EXECUTE q(42);
EXPLAIN (ANALYZE, VERBOSE, BUFFERS) EXECUTE q(142);
```
观察：
- plan tree 中出现几个 child。
- 是否显示 `Subplans Removed`。
- child loops 是否随参数变化。
源码断点：
```text
ExecDoInitialPruning
ExecInitPartitionExecPruning
ExecFindMatchingSubPlans
```
要求画出：
```text
PartitionPruneInfo -> PartitionPruneState -> valid subplans bitmap
```
### 实验二：证明 routing 与 pruning 处理不同时间点的值
准备：
```sql
CREATE TABLE u (k int, v text) PARTITION BY RANGE (k);
CREATE TABLE u0 PARTITION OF u FOR VALUES FROM (0) TO (100);
CREATE TABLE u1 PARTITION OF u FOR VALUES FROM (100) TO (200);
INSERT INTO u VALUES (42, 'old');
```
执行：
```sql
EXPLAIN (ANALYZE, VERBOSE, BUFFERS, WAL)
UPDATE u SET k = 142 WHERE k = 42 RETURNING tableoid::regclass, *;
```
观察：
- scan 侧应只需要找到 old row。
- RETURNING 的 `tableoid` 反映 new row 所在 relation。
- WAL 可能比普通同 leaf update 更高。
源码断点：
```text
ExecFindMatchingSubPlans
ExecCrossPartitionUpdate
ExecFindPartition
```
要求解释：
```text
old row search space != new row destination
```
### 实验三：触发器改写 partition key 后为什么不 reroute
准备：
```sql
CREATE TABLE ti (k int, v text) PARTITION BY RANGE (k);
CREATE TABLE ti0 PARTITION OF ti FOR VALUES FROM (0) TO (100);
CREATE TABLE ti1 PARTITION OF ti FOR VALUES FROM (100) TO (200);
CREATE OR REPLACE FUNCTION bump_key()
RETURNS trigger LANGUAGE plpgsql AS $$
BEGIN
  NEW.k := NEW.k + 100;
  RETURN NEW;
END;
$$;
CREATE TRIGGER ti_bi
BEFORE INSERT ON ti0
FOR EACH ROW EXECUTE FUNCTION bump_key();
```
执行：
```sql
INSERT INTO ti VALUES (42, 'triggered');
```
预期：
```text
先按 42 路由到 ti0；
BEFORE trigger 把 key 改成 142；
随后 leaf partition constraint 检查失败；
系统不自动 reroute 到 ti1。
```
源码断点：
```text
ExecPrepareTupleRouting
ExecFindPartition
ExecBRInsertTriggers
ExecPartitionCheck
```
要求解释：
```text
routing before BEFORE INSERT trigger 是触发器语义和执行器状态的边界选择。
```
### 实验四：观察 tuple conversion
构造存在 dropped column、列顺序差异或 inheritance 历史布局的测试表；如果本地版本限制不允许直接构造，就用断点观察已有 conversion map。
推荐断点：
```text
ExecGetRootToChildMap
ExecGetChildToRootMap
execute_attr_map_slot
```
观察 root/child slot descriptor、attr map 是否为空，以及 conversion 是否进入 per-tuple hot path。
要求说明：conversion map 不是 routing 决策，它只在 relation 已经确定后修正 descriptor。
## 13. 讨论题
1. 为什么 `PartitionPruneState` 不能直接复用为 `PartitionTupleRouting`？
2. 如果 runtime pruning 漏裁剪一个 partition，会发生什么？如果误裁剪一个 partition，会发生什么？
3. 为什么跨分区 UPDATE 更适合理解为 delete+insert，而不是 table AM 内部 update？
4. `ResultRelInfo` 中哪些状态与 relation catalog 有关，哪些状态只属于当前 DML 语句？
5. `BEFORE INSERT` trigger 改写 partition key 后，为什么 PostgreSQL 不自动 reroute？
6. 大量 partitions 下，为什么执行期 pruning 变快后，瓶颈可能迁移到 planning、relcache 或 plan invalidation？
7. 如何用 EXPLAIN、gdb 和临时日志区分 scan pruning、tuple routing 和 tuple conversion？
8. 如果一个分区 DML 在 DDL 后 prepared statement 第一次执行变慢，应该优先怀疑 runtime pruning 还是 plan cache invalidation？
## 14. 本节小结
本节唯一主问题是：
```text
执行期 partition pruning、tuple routing 和 row movement 如何与 planner 分区路径、ModifyTable 和 ResultRelInfo 对接？
```
核心链路可以压缩为：
```text
planner PartitionPruneInfo
  -> executor PartitionPruneState
  -> Append/MergeAppend valid subplans
  -> ModifyTable source tuple
  -> PartitionTupleRouting destination ResultRelInfo
  -> tuple conversion descriptor alignment
  -> row movement delete+insert when UPDATE crosses partitions
```
核心状态边界是：
- `PartitionPruneState` 处理 subplan indexes。
- `PartitionTupleRouting` 处理 tuple values。
- `TupleConversionMap` 处理 tuple descriptors。
- `ResultRelInfo` 处理当前 DML 语句中的 relation execution state。
- `ModifyTableState` 把 source row identity、DML semantics 和 destination routing 串起来。
ownership 上，planner 产物属于 plan tree。
pruning state 属于 scan node。
routing state 属于 `ModifyTableState`。
borrowed `ResultRelInfo` 不能由 routing cleanup 关闭。
动态创建的 leaf result rel 必须由 routing cleanup 收尾。
ERROR 路径中，backend-local memory 由 memory context 清理，但 relation、index、FDW、trigger 和 pin/refcount 等外部资源仍要走各自 cleanup。
正确性上，runtime pruning 必须保守，不能漏读。
tuple routing 必须找到唯一 leaf 或报错。
tuple conversion 保证下游 descriptor 正确。
row movement 通过 delete+insert 保持跨 relation UPDATE 语义。
观测上，EXPLAIN 擅长看 scan pruning 和总体 DML 成本。
它不能显示每行 routing。
gdb、临时日志和 perf 能补上 executor 内部状态，但要区分 subplan bitmap、destination leaf 和 conversion map。
从本节抽象出的可迁移规律是：
```text
同一个 SQL abstraction 在不同执行阶段会被投影成不同状态空间；
诊断时先确认当前状态空间，再解释字段和指标。
```
对 partitioned table 来说，这四个状态空间分别是 partition bound、subplan index、result rel index 和 tuple descriptor。
只有把它们分开，才能正确解释 runtime pruning、tuple routing、row movement 和 PostgreSQL executor 的分区边界。
