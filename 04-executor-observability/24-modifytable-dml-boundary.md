# PostgreSQL ModifyTable 与 DML 执行边界

## 课程定位

前置知识：理解 TupleTableSlot、junk
attribute、ResultRelInfo、table AM、触发器、约束、RETURNING 和
EvalPlanQual。

本节唯一主问题：

```text
ModifyTable 如何把上游 tuple 流转换成 INSERT / UPDATE / DELETE / MERGE 的真实表修改，同时串起 junk attr、partition routing、约束、trigger、RETURNING 和 table/FDW 回调？
```

核心矛盾：执行器上层希望 DML 仍像普通节点一样一次返回一个 slot；但一次 DML tuple
可能触发表路由、行锁、约束、索引维护、触发器、FDW/table AM 调用、并发冲突重试和
RETURNING 副作用。

学完后应能判断：DML 慢或行为异常时，问题落在上游 plan、row identity junk
column、partition routing、约束/trigger、table AM
修改、并发冲突，还是 RETURNING 输出。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

04 目录的主线已经从
ExecutorStart、PlanState、TupleTableSlot、ExprContext 和
executor memory context 走到具体执行节点。

第 23 节看到轻节点如何用少量状态改变 tuple 流控制。

本组课程的读法不是把节点函数逐个背下来，而是抓住一个运行时对象如何随 ExecProcNode
的一次次调用推进。

每节都会把源码状态落回 EXPLAIN、pg_stat、wait event、临时文件或 gdb
断点中能看到的现象。

本节只回答一个问题：ModifyTable 如何把上游 tuple 流转换成 INSERT /
UPDATE / DELETE / MERGE 的真实表修改，同时串起 junk
attr、partition routing、约束、trigger、RETURNING 和
table/FDW 回调？

后续阅读不要把本节写成节点百科。所有函数、字段、指标都必须回到这一条主线。

第 25 节会转入可观测性，解释 Instrumentation 如何挂到 PlanState。

从课程组位置看，17-24 节是常见执行节点源码 walkthrough。它夹在 scan/access
method 边界和 EXPLAIN instrumentation
之间，因此每节都要同时回答两个问题：节点如何产生或消费 tuple，以及这个行为最终如何被观测。

本节的重点不是覆盖所有分支，而是建立一个能迁移到慢 SQL 诊断的 mental model。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
ModifyTable 从子计划拉取 candidate slot。
INSERT 直接使用 projected slot，UPDATE/DELETE/MERGE 还要从 junk attr 中取 row identity。
具体操作进入 ExecInsert、ExecUpdate、ExecDelete 或 ExecMerge，再调用 table AM 或 FDW，并穿过约束、触发器、索引维护和 RETURNING。
它把“读出来的 tuple”变成“对目标关系的副作用”，因此是 executor 与 storage/trigger/catalog 交界处。
```

这句话背后有三层含义。

第一层是执行协议：上层仍然通过 `ExecProcNode()` 一次要一个 slot。

第二层是状态边界：节点内部可以缓存、重扫、阻塞、展开或副作用化 tuple，但必须给上层稳定语义。

第三层是诊断边界：EXPLAIN 看到的是状态推进后的事实，不能直接等同于优化器估算或单个函数耗时。

本节的 tension 可以压缩成：

```text
保持统一 tuple 拉取协议和低 hot path 成本
  vs
节点必须保存足够状态，才能实现本节 SQL 语义、异常 cleanup 和可观测性
```

读源码时要不断把这组 tension 放回当前节点。否则很容易把某个 helper 函数误读成独立主题。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | src/backend/executor/nodeModifyTable.c | `ExecModifyTable()`、`ExecInsert()`、`ExecUpdate()`、`ExecDelete()`、`ExecMerge()` 和 RETURNING。 |
| 2 | src/backend/executor/execMain.c | `ExecConstraints()`、`ExecWithCheckOptions()`、`ExecUpdateLockMode()` 和 ResultRelInfo 初始化协作。 |
| 3 | src/backend/executor/execJunk.c | `ExecFindJunkAttributeInTlist()`、`ExecGetJunkAttribute()` 读取 ctid/tableoid/wholerow。 |
| 4 | src/backend/executor/execPartition.c | `ExecSetupPartitionTupleRouting()`、`ExecCleanupTupleRouting()` 和分区路由。 |
| 5 | src/backend/executor/execIndexing.c | `ExecInsertIndexTuples()` 维护索引和唯一约束冲突检查。 |
| 6 | src/include/access/tableam.h | `table_tuple_insert()`、`table_tuple_update()`、`table_tuple_delete()`、`table_tuple_lock()`。 |
| 7 | src/include/nodes/execnodes.h | `ModifyTableState`、`ResultRelInfo`、`EPQState` 的状态边界。 |
| 8 | src/include/foreign/fdwapi.h | FDW 的 ExecForeignInsert/Update/Delete 回调边界。 |

阅读顺序按 mental model
展开：入口状态、状态结构、主流程、rescan/cleanup、异常或 fallback、观测入口。

不要从文件顶部线性读到文件底部。先找谁创建状态、谁推进状态、谁释放状态，再补充具体分支。

## 4. 关键数据结构与状态边界

本节不复制结构体源码。关键是把 raw field 放回 owner、生命周期、并发或资源边界中解释。

### `ModifyTableState`

DML 节点的总状态，保存 operation、result
rels、returning、routing 和 EPQ。

owner / 生命周期：ExecInitModifyTable 创建，ExecModifyTable
推进。

诊断边界：它不是普通 scan state，因为它的核心输出是副作用。

单独看到 `ModifyTableState` 的值并不等于理解了语义。必须同时看当前
PlanState、ExprContext、slot 或资源 owner。

如果这个状态出现在 EXPLAIN 或 gdb 中，应先判断它属于本次调用、本轮 rescan、本条
SQL，还是整个 backend 生命周期。

### `ResultRelInfo`

目标 relation 的触发器、索引、约束、FDW、RETURNING 等执行期状态。

owner / 生命周期：ExecInitResultRelation 和
ExecInitModifyTable 初始化。

诊断边界：同一 ModifyTable 可能有多个 target relation。

单独看到 `ResultRelInfo` 的值并不等于理解了语义。必须同时看当前
PlanState、ExprContext、slot 或资源 owner。

如果这个状态出现在 EXPLAIN 或 gdb 中，应先判断它属于本次调用、本轮 rescan、本条
SQL，还是整个 backend 生命周期。

### junk attribute

UPDATE/DELETE/MERGE 用 ctid、tableoid、wholerow
等隐藏列定位目标行。

owner / 生命周期：子计划 targetlist 中携带，ExecGetJunkAttribute
读取。

诊断边界：缺少 row identity 会让 DML 无法定位原 tuple。

单独看到 junk attribute 的值并不等于理解了语义。必须同时看当前
PlanState、ExprContext、slot 或资源 owner。

如果这个状态出现在 EXPLAIN 或 gdb 中，应先判断它属于本次调用、本轮 rescan、本条
SQL，还是整个 backend 生命周期。

### `PartitionTupleRouting`

INSERT 或跨分区 UPDATE 的目标分区选择状态。

owner / 生命周期：ExecSetupPartitionTupleRouting
创建，ExecCleanupTupleRouting 释放。

诊断边界：路由会改变最终 ResultRelInfo。

单独看到 `PartitionTupleRouting` 的值并不等于理解了语义。必须同时看当前
PlanState、ExprContext、slot 或资源 owner。

如果这个状态出现在 EXPLAIN 或 gdb 中，应先判断它属于本次调用、本轮 rescan、本条
SQL，还是整个 backend 生命周期。

### `EPQState`

并发更新冲突后重新检查目标行的执行状态。

owner / 生命周期：ModifyTable 和 LockRows 使用。

诊断边界：它说明 DML 不是单纯拿到 ctid 就能修改。

单独看到 `EPQState` 的值并不等于理解了语义。必须同时看当前
PlanState、ExprContext、slot 或资源 owner。

如果这个状态出现在 EXPLAIN 或 gdb 中，应先判断它属于本次调用、本轮 rescan、本条
SQL，还是整个 backend 生命周期。

### RETURNING projection

把修改前后 tuple 转成用户可见结果。

owner / 生命周期：ExecProcessReturning 使用 old/new slot 和
ResultRelInfo。

诊断边界：RETURNING 会让 DML 节点也像普通节点一样返回 slot。

单独看到 RETURNING projection 的值并不等于理解了语义。必须同时看当前
PlanState、ExprContext、slot 或资源 owner。

如果这个状态出现在 EXPLAIN 或 gdb 中，应先判断它属于本次调用、本轮 rescan、本条
SQL，还是整个 backend 生命周期。

## 5. 主流程源码 walkthrough

主流程按时间推进。下面的链路不是 API 清单，而是本节状态从初始化、被消费、发生边界变化，到返回上层
slot 或完成副作用的故事。

```text
`ExecInitModifyTable()` -> 初始化 ResultRelInfo、junk attr 编号、RETURNING projection、partition routing 和子计划。
`ExecModifyTable()` 拉取子计划 -> 从 outerPlan 获取 candidate slot。
INSERT 路径 -> `ExecInsert()` 检查 BEFORE trigger、路由、WCO、constraints，再调用 table AM/FDW。
UPDATE 路径 -> `ExecUpdate()` 用 tupleid 定位旧行，准备新 slot，检查约束并调用 `table_tuple_update()`。
DELETE 路径 -> `ExecDelete()` 用 tupleid 定位旧行，触发器通过后调用 `table_tuple_delete()`。
MERGE 路径 -> `ExecMerge()` 根据 MATCHED/NOT MATCHED 分派到 update/delete/insert 子动作。
索引与约束 -> `ExecInsertIndexTuples()`、`ExecConstraints()`、`ExecWithCheckOptions()` 在 table 修改周围执行。
RETURNING 与收尾 -> `ExecProcessReturning()` 生成输出 slot；执行结束时清理 routing、trigger 和子节点。
```

### 5.1 `ExecInitModifyTable()`

初始化 ResultRelInfo、junk attr 编号、RETURNING
projection、partition routing 和子计划。

状态变化：ModifyTableState 持有 DML 所需的多模块状态。

正确性或资源边界：初始化阶段决定后续每行如何定位目标。

调试时可以在 `ExecInitModifyTable()`
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

### 5.2 `ExecModifyTable()` 拉取子计划

从 outerPlan 获取 candidate slot。

状态变化：INSERT 使用输入 slot，UPDATE/DELETE 读取 ctid/tableoid
等 junk attr。

正确性或资源边界：上游 plan 仍按普通 tuple 协议工作。

调试时可以在 `ExecModifyTable()` 拉取子计划
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

### 5.3 INSERT 路径

`ExecInsert()` 检查 BEFORE
trigger、路由、WCO、constraints，再调用 table AM/FDW。

状态变化：新 tuple 进入目标 relation，索引随后维护。

正确性或资源边界：INSERT 的 row identity 是新 slot 而不是旧 ctid。

调试时可以在 INSERT 路径
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

### 5.4 UPDATE 路径

`ExecUpdate()` 用 tupleid 定位旧行，准备新 slot，检查约束并调用
`table_tuple_update()`。

状态变化：并发冲突可能进入 EPQ 或报错。

正确性或资源边界：UPDATE 可能跨分区转换成 delete+insert。

调试时可以在 UPDATE 路径
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

### 5.5 DELETE 路径

`ExecDelete()` 用 tupleid 定位旧行，触发器通过后调用
`table_tuple_delete()`。

状态变化：RETURNING 可能需要 old tuple。

正确性或资源边界：DELETE 输出和副作用顺序必须清晰。

调试时可以在 DELETE 路径
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

### 5.6 MERGE 路径

`ExecMerge()` 根据 MATCHED/NOT MATCHED 分派到
update/delete/insert 子动作。

状态变化：状态机保证跟随 update chain 时不会 livelock。

正确性或资源边界：MERGE 把 join 结果和 DML action 绑定。

调试时可以在 MERGE 路径
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

### 5.7 索引与约束

`ExecInsertIndexTuples()`、`ExecConstraints()`、`ExecWithCheckOptions()`
在 table 修改周围执行。

状态变化：错误会中止当前语句/事务。

正确性或资源边界：这些不是普通 filter，而是 DML correctness 边界。

调试时可以在 索引与约束
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

### 5.8 RETURNING 与收尾

`ExecProcessReturning()` 生成输出 slot；执行结束时清理
routing、trigger 和子节点。

状态变化：DML 副作用已经发生。

正确性或资源边界：RETURNING 的 rows 不等于内部检查次数。

调试时可以在 RETURNING 与收尾
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

## 6. 生命周期 / ownership / cleanup

### 创建

ModifyTableState 和 ResultRelInfo 在
ExecutorStart/ExecInitModifyTable 中建立。

这一路径的核心不是谁调用了 `pfree()`，而是对象挂在哪个 executor context、哪个
ResourceOwner 或哪个节点私有状态下。

正常路径和 ERROR 路径都要能回到同一个 owner
边界，否则就会出现泄漏、重复释放或使用失效状态。

### 持有

每个目标关系持有触发器、索引、FDW、RETURNING、partition routing
等运行时状态。

这一路径的核心不是谁调用了 `pfree()`，而是对象挂在哪个 executor context、哪个
ResourceOwner 或哪个节点私有状态下。

正常路径和 ERROR 路径都要能回到同一个 owner
边界，否则就会出现泄漏、重复释放或使用失效状态。

### 每行推进

ExecModifyTable 每拉一行就完成一次 DML action 或跳过被并发语义淘汰的行。

这一路径的核心不是谁调用了 `pfree()`，而是对象挂在哪个 executor context、哪个
ResourceOwner 或哪个节点私有状态下。

正常路径和 ERROR 路径都要能回到同一个 owner
边界，否则就会出现泄漏、重复释放或使用失效状态。

### 释放

ExecEndModifyTable 关闭子节点、清理 partition routing 和
ResultRelInfo 附属状态。

这一路径的核心不是谁调用了 `pfree()`，而是对象挂在哪个 executor context、哪个
ResourceOwner 或哪个节点私有状态下。

正常路径和 ERROR 路径都要能回到同一个 owner
边界，否则就会出现泄漏、重复释放或使用失效状态。

### 异常

约束、触发器、table AM 或 FDW 抛 ERROR 时，已做修改依赖事务
abort/WAL/ResourceOwner 回滚和释放。

这一路径的核心不是谁调用了 `pfree()`，而是对象挂在哪个 executor context、哪个
ResourceOwner 或哪个节点私有状态下。

正常路径和 ERROR 路径都要能回到同一个 owner
边界，否则就会出现泄漏、重复释放或使用失效状态。

在 gdb 中检查生命周期时，建议同时看
`estate->es_query_cxt`、节点私有指针、slot 类型和子节点是否已经 end。

## 7. 正确性机制层次

| 层次 | 本节机制 |
| --- | --- |
| row identity | UPDATE/DELETE 必须通过 junk attr 找到目标 tuple。 |
| MVCC 与锁 | table_tuple_update/delete/lock 处理并发可见性和 TM_Result。 |
| 约束与 WCO | ExecConstraints 和 ExecWithCheckOptions 在写入边界维护关系语义。 |
| 触发器顺序 | BEFORE/AFTER trigger 影响是否执行、如何改 slot、何时 drain side effects。 |
| 索引维护 | 表修改后必须维护索引和唯一冲突检查。 |
| 事务语义 | DML 副作用靠事务 abort、WAL 和 undo-like MVCC 后果保持一致。 |

row identity 这一层保证的是：UPDATE/DELETE 必须通过 junk attr
找到目标 tuple。

它不能替代其它层次。例如 MVCC 不等于互斥，内存上下文不等于语义仍然有效，资源预算也不等于正确性。

MVCC 与锁 这一层保证的是：table_tuple_update/delete/lock
处理并发可见性和 TM_Result。

它不能替代其它层次。例如 MVCC 不等于互斥，内存上下文不等于语义仍然有效，资源预算也不等于正确性。

约束与 WCO 这一层保证的是：ExecConstraints 和
ExecWithCheckOptions 在写入边界维护关系语义。

它不能替代其它层次。例如 MVCC 不等于互斥，内存上下文不等于语义仍然有效，资源预算也不等于正确性。

触发器顺序 这一层保证的是：BEFORE/AFTER trigger 影响是否执行、如何改
slot、何时 drain side effects。

它不能替代其它层次。例如 MVCC 不等于互斥，内存上下文不等于语义仍然有效，资源预算也不等于正确性。

索引维护 这一层保证的是：表修改后必须维护索引和唯一冲突检查。

它不能替代其它层次。例如 MVCC 不等于互斥，内存上下文不等于语义仍然有效，资源预算也不等于正确性。

事务语义 这一层保证的是：DML 副作用靠事务 abort、WAL 和 undo-like MVCC
后果保持一致。

它不能替代其它层次。例如 MVCC 不等于互斥，内存上下文不等于语义仍然有效，资源预算也不等于正确性。

PostgreSQL 在 executor 中通常不是靠单一机制保证正确性，而是靠状态机、slot
生命周期、表达式语义、存储层回调和 cleanup 顺序共同收束。

## 8. 错误路径 / 异常路径 / fallback

### 并发更新

table AM 返回 TM_Updated/TM_Deleted 等结果，ModifyTable 可能
EPQ、跳过或报错。

这个分支不应该被看成“少见边角”。它通常正是线上问题、性能突变或解释 EXPLAIN 异常字段的入口。

阅读源码时要确认 fallback 是否改变输出语义，还是只改变资源使用、执行顺序或等待方式。

### 分区路由

目标分区根据 tuple 值决定，跨分区 UPDATE 可能改写成 delete+insert。

这个分支不应该被看成“少见边角”。它通常正是线上问题、性能突变或解释 EXPLAIN 异常字段的入口。

阅读源码时要确认 fallback 是否改变输出语义，还是只改变资源使用、执行顺序或等待方式。

### FDW 目标

有 FDW callback 时走 ExecForeignInsert/Update/Delete。

这个分支不应该被看成“少见边角”。它通常正是线上问题、性能突变或解释 EXPLAIN 异常字段的入口。

阅读源码时要确认 fallback 是否改变输出语义，还是只改变资源使用、执行顺序或等待方式。

### 触发器返回 NULL

BEFORE trigger 可取消当前行操作。

这个分支不应该被看成“少见边角”。它通常正是线上问题、性能突变或解释 EXPLAIN 异常字段的入口。

阅读源码时要确认 fallback 是否改变输出语义，还是只改变资源使用、执行顺序或等待方式。

### RETURNING 缺失

没有 RETURNING 时 DML 节点主要返回空 slot 并更新 es_processed。

这个分支不应该被看成“少见边角”。它通常正是线上问题、性能突变或解释 EXPLAIN 异常字段的入口。

阅读源码时要确认 fallback 是否改变输出语义，还是只改变资源使用、执行顺序或等待方式。

如果某个 fallback 发生在热路径中，建议先用 EXPLAIN、日志或计数确认频率，再决定是否进入
gdb。

## 9. 成本、资源与跨模块传播

### 上游 plan

DML 成本首先受产生 candidate tuple 的计划影响。

这个成本通常不会停在本节点内部。它会传播到 buffer manager、临时文件、表达式执行、上层
loops、并行 worker 或统计输出。

诊断时要避免只看节点总耗时。更好的问题是：这个成本随
rows、groups、loops、work_mem、tuple width
或并发度中的哪一个变量扩张。

### row identity

UPDATE/DELETE 需要 ctid/tableoid 等 junk attr 和可能的 row
lock。

这个成本通常不会停在本节点内部。它会传播到 buffer manager、临时文件、表达式执行、上层
loops、并行 worker 或统计输出。

诊断时要避免只看节点总耗时。更好的问题是：这个成本随
rows、groups、loops、work_mem、tuple width
或并发度中的哪一个变量扩张。

### 索引数量

每个 INSERT/UPDATE 可能维护多个索引。

这个成本通常不会停在本节点内部。它会传播到 buffer manager、临时文件、表达式执行、上层
loops、并行 worker 或统计输出。

诊断时要避免只看节点总耗时。更好的问题是：这个成本随
rows、groups、loops、work_mem、tuple width
或并发度中的哪一个变量扩张。

### 触发器/约束

函数调用和检查可成为主要 CPU 成本。

这个成本通常不会停在本节点内部。它会传播到 buffer manager、临时文件、表达式执行、上层
loops、并行 worker 或统计输出。

诊断时要避免只看节点总耗时。更好的问题是：这个成本随
rows、groups、loops、work_mem、tuple width
或并发度中的哪一个变量扩张。

### 分区数

partition routing 和 ResultRelInfo 数量会扩大初始化和每行路由成本。

这个成本通常不会停在本节点内部。它会传播到 buffer manager、临时文件、表达式执行、上层
loops、并行 worker 或统计输出。

诊断时要避免只看节点总耗时。更好的问题是：这个成本随
rows、groups、loops、work_mem、tuple width
或并发度中的哪一个变量扩张。

成本分析必须回到
workload。相同节点在小结果集、热缓存、冷缓存、重复参数或高并发下可能呈现完全不同的瓶颈。

## 10. 观测与诊断入口

### EXPLAIN ModifyTable

看 Insert/Update/Delete/Merge 节点与子计划 rows。

看到这个现象后，下一步要回到第 3 节的源码入口，找哪个状态被创建、推进或清理。

不要把累计视图、单次 EXPLAIN、采样 profiler 和 gdb
断点混成同一种证据。它们的时间窗口不同。

### RETURNING rows

返回行数反映最终输出，不等于内部冲突重试次数。

看到这个现象后，下一步要回到第 3 节的源码入口，找哪个状态被创建、推进或清理。

不要把累计视图、单次 EXPLAIN、采样 profiler 和 gdb
断点混成同一种证据。它们的时间窗口不同。

### Buffers/WAL

DML 会产生 heap/index buffer dirties 和 WAL。

看到这个现象后，下一步要回到第 3 节的源码入口，找哪个状态被创建、推进或清理。

不要把累计视图、单次 EXPLAIN、采样 profiler 和 gdb
断点混成同一种证据。它们的时间窗口不同。

### pg_stat_all_tables

n_tup_ins/upd/del 是累计视图，不是单条语句精确 trace。

看到这个现象后，下一步要回到第 3 节的源码入口，找哪个状态被创建、推进或清理。

不要把累计视图、单次 EXPLAIN、采样 profiler 和 gdb
断点混成同一种证据。它们的时间窗口不同。

### 触发器时间

EXPLAIN ANALYZE 可显示 trigger time。

看到这个现象后，下一步要回到第 3 节的源码入口，找哪个状态被创建、推进或清理。

不要把累计视图、单次 EXPLAIN、采样 profiler 和 gdb
断点混成同一种证据。它们的时间窗口不同。

### gdb

断点
`ExecModifyTable()`、`ExecUpdate()`、`table_tuple_update()`、`ExecInsertIndexTuples()`。

看到这个现象后，下一步要回到第 3 节的源码入口，找哪个状态被创建、推进或清理。

不要把累计视图、单次 EXPLAIN、采样 profiler 和 gdb
断点混成同一种证据。它们的时间窗口不同。

一个可靠诊断闭环应该长这样：先看到 runtime
现象，再定位状态边界，然后用源码解释为什么该状态会产生这个输出。

### 诊断闭环示例

这一小节给出一条从 runtime 现象回到源码的完整路径。它不引入新的主题，只把第 10
节的观测入口串成一个可操作的排查顺序。

#### 现场现象

DML 语句的 ModifyTable 节点耗时高，RETURNING 行数不多，但
WAL、Buffers、trigger time 或索引维护成本明显。

这一步要写下一个具体证据，而不是只写“可能是执行器慢”。证据可以来自
EXPLAIN、日志、pg_stat、wait event、临时文件统计、perf 栈或 gdb 断点。

如果证据不能指向本节主状态，就说明还没有回到正确层次，需要退回计划树重新定位。

#### 第一步

第一步把上游 candidate rows、实际修改行数、RETURNING rows、WAL 和
trigger time 分开看。

这一步要写下一个具体证据，而不是只写“可能是执行器慢”。证据可以来自
EXPLAIN、日志、pg_stat、wait event、临时文件统计、perf 栈或 gdb 断点。

如果证据不能指向本节主状态，就说明还没有回到正确层次，需要退回计划树重新定位。

#### 回到状态

回到 `ModifyTableState`、`ResultRelInfo`、junk attribute
和 `PartitionTupleRouting`，确认每行如何定位目标关系和目标 tuple。

这一步要写下一个具体证据，而不是只写“可能是执行器慢”。证据可以来自
EXPLAIN、日志、pg_stat、wait event、临时文件统计、perf 栈或 gdb 断点。

如果证据不能指向本节主状态，就说明还没有回到正确层次，需要退回计划树重新定位。

#### 确认边界

再沿 `ExecInsert()`、`ExecUpdate()`、`ExecDelete()` 或
`ExecMerge()` 进入 table AM/FDW，区分副作用、约束和输出投影。

这一步要写下一个具体证据，而不是只写“可能是执行器慢”。证据可以来自
EXPLAIN、日志、pg_stat、wait event、临时文件统计、perf 栈或 gdb 断点。

如果证据不能指向本节主状态，就说明还没有回到正确层次，需要退回计划树重新定位。

#### 排除相邻原因

如果上游 scan 已经很慢，ModifyTable 只是消费慢输入；如果 WAL 或 trigger
time 高，瓶颈在写入边界而不是 tuple 拉取。

这一步要写下一个具体证据，而不是只写“可能是执行器慢”。证据可以来自
EXPLAIN、日志、pg_stat、wait event、临时文件统计、perf 栈或 gdb 断点。

如果证据不能指向本节主状态，就说明还没有回到正确层次，需要退回计划树重新定位。

#### 最小断点路径

```text
break ExecModifyTable
break ExecGetJunkAttribute
break ExecUpdate
break table_tuple_update
```

断点只用于确认状态推进顺序，不建议在高频循环里长时间单步。更有效的做法是记录入口参数、关键状态字段和返回
slot 是否为空。

完成一次闭环后，再决定是否需要扩大到 buffer、lock、temporary
file、statistics 或 planner estimate 层面。

#### 课堂复盘口径

复盘时把结论压成三句话。

第一句说明看到的 runtime 现象，例如
rows、loops、temp、WAL、Buffers、trigger time、wait event
或函数栈。

第二句说明源码中的主状态如何推进，必须点名一个字段、一个 owner 或一个状态机分支。

第三句说明这个状态为什么会产生观察到的现象，并指出它属于正常路径、fallback、rescan、cleanup
还是异常路径。

本节的具体落点是：把 junk row identity、ResultRelInfo 和 table
AM/FDW 副作用连成一条线。

如果三句话无法首尾相接，说明诊断仍停留在症状描述，还没有完成从 runtime 到 source
的闭环。

这个复盘口径也能帮助区分执行器问题和 planner estimate、schema
设计、数据分布、GUC 或硬件 I/O 问题。

## 11. 常见误区

误区 1：ModifyTable 不是普通投影节点，它的核心语义是副作用。

纠正这个误区的办法，是把结论拆成状态、生命周期、成本变量和观测证据四个部分。

如果四个部分不能互相解释，说明还没有真正回到源码主线。

误区 2：UPDATE/DELETE 不是靠用户可见列定位行，而是依赖 junk row
identity。

纠正这个误区的办法，是把结论拆成状态、生命周期、成本变量和观测证据四个部分。

如果四个部分不能互相解释，说明还没有真正回到源码主线。

误区 3：RETURNING 不是修改前的简单 SELECT，它在 DML action 后按
old/new slot 投影。

纠正这个误区的办法，是把结论拆成状态、生命周期、成本变量和观测证据四个部分。

如果四个部分不能互相解释，说明还没有真正回到源码主线。

误区 4：触发器和约束不是额外 filter，它们能改变或阻止写入。

纠正这个误区的办法，是把结论拆成状态、生命周期、成本变量和观测证据四个部分。

如果四个部分不能互相解释，说明还没有真正回到源码主线。

误区 5：FDW 目标表会把写入边界交给 FDW callback。

纠正这个误区的办法，是把结论拆成状态、生命周期、成本变量和观测证据四个部分。

如果四个部分不能互相解释，说明还没有真正回到源码主线。

## 12. 课堂实验

实验目标不是制造某个固定计划，而是练习从可见现象回到源码边界。不同数据分布和版本可能得到不同计划，重点是解释差异。

### 实验 1：RETURNING 路径

```sql
EXPLAIN (ANALYZE, BUFFERS, WAL) UPDATE t SET v = v + 1 WHERE k < 100 RETURNING k, v;
```

观察重点：观察 ModifyTable rows、WAL 和 RETURNING 输出。

如果结果和预期不同，先检查统计信息、GUC、数据分布和并发环境，再回到源码判断是否走了相邻路径。

把 EXPLAIN 中的 rows、loops、buffers、temp、WAL 或 trigger
time 与本节主状态对应起来。

### 实验 2：索引维护成本

```sql
CREATE INDEX ON t(v);
EXPLAIN (ANALYZE, BUFFERS, WAL) UPDATE t SET v = v + 1 WHERE k < 10000;
```

观察重点：比较有无索引时 WAL/buffers。

如果结果和预期不同，先检查统计信息、GUC、数据分布和并发环境，再回到源码判断是否走了相邻路径。

把 EXPLAIN 中的 rows、loops、buffers、temp、WAL 或 trigger
time 与本节主状态对应起来。

### 实验 3：分区路由

```sql
EXPLAIN (ANALYZE, BUFFERS) INSERT INTO p SELECT * FROM src;
```

观察重点：观察 ModifyTable 与 partition routing 成本。

如果结果和预期不同，先检查统计信息、GUC、数据分布和并发环境，再回到源码判断是否走了相邻路径。

把 EXPLAIN 中的 rows、loops、buffers、temp、WAL 或 trigger
time 与本节主状态对应起来。

### 实验 4：源码断点

```text
break ExecModifyTable
break ExecUpdate
break ExecGetJunkAttribute
break table_tuple_update
```

观察重点：跟踪 candidate slot 如何变成真实表修改。

如果结果和预期不同，先检查统计信息、GUC、数据分布和并发环境，再回到源码判断是否走了相邻路径。

把 EXPLAIN 中的 rows、loops、buffers、temp、WAL 或 trigger
time 与本节主状态对应起来。

实验结束后建议恢复会话级 GUC，避免影响后续课程。

## 13. 讨论题

1. 为什么 DML 要把 row identity 做成 junk attribute？

回答时要求同时给出一个源码入口、一个运行时状态和一个可观测现象。只给概念性答案不够。

2. ModifyTable 如何保持普通 ExecProcNode 协议，同时执行副作用？

回答时要求同时给出一个源码入口、一个运行时状态和一个可观测现象。只给概念性答案不够。

3. 触发器返回 NULL 与普通 WHERE 过滤有什么区别？

回答时要求同时给出一个源码入口、一个运行时状态和一个可观测现象。只给概念性答案不够。

4. 跨分区 UPDATE 为什么会让 UPDATE 接近 delete+insert？

回答时要求同时给出一个源码入口、一个运行时状态和一个可观测现象。只给概念性答案不够。

5. 从 WAL/Buffers 判断 DML 成本时哪些指标最关键？

回答时要求同时给出一个源码入口、一个运行时状态和一个可观测现象。只给概念性答案不够。

讨论题的目标是把本节 runtime 现象压缩成可迁移规律，而不是展开新的主题。

## 14. 本节小结

ModifyTable 是 executor 从 tuple 流进入表副作用的边界。

INSERT、UPDATE、DELETE、MERGE 共享节点框架，但 row identity
和副作用顺序不同。

ResultRelInfo 是 DML 目标关系的运行时状态中心。

诊断 DML 要把上游计划、junk attr、table AM、索引、触发器、约束和
RETURNING 分开看。

可迁移规律：当一个节点把纯数据流转换成持久副作用时，正确性来自多个模块按固定顺序交接。

把本节带回 04 目录主线，可以得到一个稳定读法：PlanState 不是静态结构，而是每次
ExecProcNode 调用都会推进的运行时状态。

下一节继续沿同一读法，只是把主状态换成相邻执行节点或观测对象。
