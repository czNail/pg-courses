# PostgreSQL plan cache / executor boundary
## 课程定位
前置知识：已经理解 extended query protocol、prepared statement、generic/custom plan 选择，以及 executor 的 `ExecutorStart()` / `ExecutorRun()` / `ExecutorEnd()` 生命周期。
本节唯一主问题：
```text
plan cache 输出的 PlannedStmt 如何成为 executor 可执行契约，哪些状态仍必须运行期补齐？
```
核心矛盾：
```text
plan cache 希望把 planner 输出固化成可复用的 PlannedStmt
  vs
executor 不能直接执行长期缓存对象，因为 snapshot、参数值、Relation 指针、ResultRelInfo、PlanState、ResourceOwner 和 instrumentation 都属于本次执行现场。
```
学完后应能判断：一个现象到底属于 plan cache 交出的静态契约，还是 `PortalStart()`、`CreateQueryDesc()`、`ExecutorStart()` 在当前事务里补齐运行态时发生。
本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e9`。
本节不重新讲 generic/custom 选择和 invalidation 传播；只讲 `CachedPlan->stmt_list` 进入 executor 前后的契约边界。
---
## 1. 本节在总主线中的位置
04 目录前面已经从 executor 内部建立了这条线：
```text
ExecutorStart()
  -> CreateExecutorState()
  -> InitPlan()
  -> ExecInitNode()
  -> ExecutorRun()
  -> ExecutorFinish()
  -> ExecutorEnd()
```
第 57 到第 61 节从协议和 plan cache 外层建立了另一条线：
```text
Parse / PREPARE
  -> CachedPlanSource
  -> GetCachedPlan()
  -> CachedPlan
  -> List<PlannedStmt>
```
本节接在中间，回答 `CachedPlan` 里的 `PlannedStmt` 到底交给 executor 什么。
直觉上容易把 `PlannedStmt` 当成“已经可执行的计划”。
源码里更准确的说法是：
```text
PlannedStmt 是 executor startup 的静态合同。
QueryDesc 是一次 executor invocation 的输入包。
EState / PlanState 才是本次运行的可变执行状态。
```
`PlannedStmt` 已经包含：
```text
commandType
planTree
rtable
permInfos
resultRelationRelids
rowMarks
subplans
partPruneInfos
paramExecTypes
parallelModeNeeded
jitFlags
relationOids / invalItems
```
`PlannedStmt` 不包含：
```text
本次 MVCC snapshot
本次 ParamListInfo 中的 Datum
本次 DestReceiver
本次 portal context / resowner
本次 EState / PlanState
本次打开的 Relation *
本次 ResultRelInfo、ExecRowMark、ExprState、TupleTableSlot
本次 instrumentation 和 query-level timing state
```
因此本节主流程是：
```text
GetCachedPlan()
  -> PortalDefineQuery()
  -> PortalStart()
  -> CreateQueryDesc()
  -> ExecutorStart()
  -> InitPlan()
  -> ExecInitNode()
```
这条线的关键不是函数名，而是每一步把“静态合同”推进成“运行现场”。
---
## 2. 核心矛盾与一句话运行模型
一句话运行模型：
```text
plan cache 返回 refcounted CachedPlan；Portal 接住其中的 stmt_list 和 cplan ref；QueryDesc 把一个 PlannedStmt 与本次 snapshot、params、DestReceiver、queryEnv 组合；ExecutorStart 再创建 EState、打开 relation、构造 ResultRelInfo、初始化 PlanState 和 instrumentation。
```
这个模型里有五层对象：
```text
CachedPlanSource:
  保存能重新生成计划的 query 语义来源。
CachedPlan:
  保存 planner 输出的 stmt_list，并用 refcount 管内存生命期。
Portal:
  保存一次执行请求的 statement list、cplan ref、参数、snapshot 处理和 cleanup hook。
QueryDesc:
  保存一个 executor invocation 的静态计划和运行期输入。
EState / PlanState:
  保存 executor 可变运行态。
```
核心 tension 是：
```text
复用越多，plan cache 越有价值；
复用跨过运行期边界，越容易复用错误的 snapshot、参数值、Relation 指针、slot、trigger 状态或 instrumentation。
```
PostgreSQL 的选择是只缓存可以跨执行稳定复用的合同。
它不缓存 `EState`。
它不缓存 `PlanState`。
它不缓存 `Relation *`。
它不缓存 `ResultRelInfo`。
它不缓存 `ExprContext`、`TupleTableSlot` 或 node private state。
所以本节最重要的判断句是：
```text
PlannedStmt 是 executor 可执行契约，不是 executor 已执行态。
```
如果线上问题发生在 `GetCachedPlan()` 之前，优先看 plan cache revalidation、generic/custom、fixed result。
如果问题发生在 `ExecutorStart()` 之后，优先看 snapshot、permission、relation open、runtime pruning、ResultRelInfo、PlanState 和 ResourceOwner。
---
## 3. 核心文件分工与阅读顺序
| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/nodes/plannodes.h` | `PlannedStmt` 字段：plan cache 交给 executor 的静态合同。 |
| 2 | `src/include/utils/plancache.h` | `CachedPlanSource`、`CachedPlan`、`stmt_list`、refcount、validity。 |
| 3 | `src/backend/utils/cache/plancache.c` | `GetCachedPlan()` 如何 revalidate、build、refcount，并设置 `planOrigin`。 |
| 4 | `src/backend/utils/mmgr/portalmem.c` | `PortalDefineQuery()` 如何接住 `CachedPlan` ref，`PortalReleaseCachedPlan()` 如何释放。 |
| 5 | `src/backend/tcop/postgres.c` | extended protocol `Bind` 如何取 plan、定义 portal、启动 portal。 |
| 6 | `src/backend/commands/prepare.c` | SQL `EXECUTE` 如何从 prepared statement 取 `CachedPlan` 并交给 portal。 |
| 7 | `src/backend/tcop/pquery.c` | `CreateQueryDesc()`、`PortalStart()` 如何进入 executor。 |
| 8 | `src/include/executor/execdesc.h` | `QueryDesc` 哪些字段由 caller 提供，哪些由 `ExecutorStart()` 填写。 |
| 9 | `src/backend/executor/execMain.c` | `ExecutorStart()`、`InitPlan()` 如何补齐 `EState`、权限、rowmark、subplan、tuple desc。 |
| 10 | `src/backend/executor/execUtils.c` | `ExecInitRangeTable()`、`ExecGetRangeTableRelation()`、`ExecOpenScanRelation()` 的 runtime relation 边界。 |
| 11 | `src/backend/optimizer/plan/planner.c` | `standard_planner()` 如何构造 `PlannedStmt`。 |
建议先读 `PlannedStmt` 和 `QueryDesc`。
这两个结构正好形成静态合同和运行输入的对照。
再读 `PortalDefineQuery()`，因为这里是 `CachedPlan` refcount ownership 的交接点。
最后读 `ExecutorStart()` 和 `InitPlan()`，确认哪些字段只是被读取，哪些状态必须新建。
源码基线核对：
```text
cd /home/nail/postgres
git rev-parse --abbrev-ref HEAD
git rev-parse --short=12 HEAD
```
本地输出：
```text
master
0e1f1ed157e9
```
---
## 4. 关键数据结构与状态
### 4.1. `CachedPlan`
`CachedPlan` 定义在 `src/include/utils/plancache.h`。
它表示从 `CachedPlanSource` 派生出的 planned statement list。
本节只看这些字段：
| 字段 | 语义 |
| --- | --- |
| `stmt_list` | `PlannedStmt` 列表，是交给 Portal 的计划结果。 |
| `is_valid` | 该 planned statement list 语义上是否仍可复用。 |
| `is_saved` | 是否已放入长期 context。 |
| `planRoleId` / `dependsOnRole` | plan 是否对 role/RLS 敏感。 |
| `saved_xmin` | transient plan 是否需要在 `TransactionXmin` 改变后重建。 |
| `generation` | 对应父 `CachedPlanSource` 的代际。 |
| `refcount` | `CachedPlanSource`、Portal 或调用者持有的活跃引用数。 |
| `context` | 保存 `stmt_list` 和 plan tree 的 memory context。 |
`CachedPlan` 的 refcount 管内存安全。
`CachedPlan.is_valid` 管未来语义复用。
这两件事不能合并。
一个 plan 可以被当前 portal 持有并继续执行，同时已经被 invalidation 标记为不适合下一次执行。
`GetCachedPlan()` 返回前还会遍历 `stmt_list`，把每个 `PlannedStmt->planOrigin` 设置成 generic 或 custom 来源。
这个标签有助于诊断，但不改变 executor startup 流程。
### 4.2. `PlannedStmt`
`PlannedStmt` 定义在 `src/include/nodes/plannodes.h`。
它是本节最重要的合同对象。
字段按职责分成几组。
命令与观测：
| 字段 | 语义 |
| --- | --- |
| `commandType` | SELECT、INSERT、UPDATE、DELETE、MERGE 或 utility。 |
| `queryId` | 从 `Query` 复制，用于统计和观测。 |
| `planId` | 可由插件设置的 plan id。 |
| `planOrigin` | standard、generic cache、custom cache、internal 等来源。 |
| `hasReturning` | DML 是否有 `RETURNING`。 |
| `hasModifyingCTE` | SELECT 中是否有 modifying CTE。 |
| `canSetTag` | 是否设置 completion tag。 |
执行结构：
| 字段 | 语义 |
| --- | --- |
| `planTree` | 顶层 `Plan` tree，`ExecInitNode()` 的入口。 |
| `subplans` | SubPlan / InitPlan 的 plan 列表。 |
| `rewindPlanIDs` | 需要支持 rewind 的 subplan id。 |
| `paramExecTypes` | `PARAM_EXEC` 参数槽类型列表。 |
| `parallelModeNeeded` | 执行时是否需要 parallel mode 限制。 |
| `jitFlags` | JIT 行为标志。 |
range table 与权限：
| 字段 | 语义 |
| --- | --- |
| `rtable` | executor 使用的最终 range table。 |
| `permInfos` | 权限检查所需的 RTE 权限信息。 |
| `unprunableRelids` | 初始不被 runtime pruning 移除或 pruning 需要保留的 RT index。 |
| `resultRelationRelids` | DML target relation 的 RT index 集合。 |
| `rowMarks` | `SELECT FOR UPDATE/SHARE` 等 row mark 计划信息。 |
| `rowMarkRelids` | row mark relation 的 RT index 集合。 |
依赖与扩展：
| 字段 | 语义 |
| --- | --- |
| `partPruneInfos` | runtime initial pruning 需要的 plan-time 信息。 |
| `appendRelations` | inheritance / partition expansion 后的信息。 |
| `relationOids` | relation dependency，用于 plan invalidation。 |
| `invalItems` | 非 relation dependency，用于 plan invalidation。 |
| `utilityStmt` | utility statement 的节点。 |
| `extension_state` | extension 可存放的可复制节点状态。 |
边界判断：
```text
rtable 里有 RTE，不是 Relation *。
rowMarks 里有 PlanRowMark，不是 ExecRowMark。
paramExecTypes 里有类型，不是 ParamExecData 值。
partPruneInfos 里有 pruning 描述，不是本次 pruning 结果。
planTree 里有 Plan，不是 PlanState。
```
### 4.3. `Portal`
Portal 不是本节主角，但它是 plan cache ref 到 executor 的中间 owner。
`PortalDefineQuery()` 只做窄交接：
```text
portal->prepStmtName = prepStmtName
portal->sourceText = sourceText
portal->commandTag = commandTag
portal->stmts = stmts
portal->cplan = cplan
portal->status = PORTAL_DEFINED
```
源码注释强调：如果 `cplan` 非空，调用者必须已经通过 `GetCachedPlan()` 增加 refcount，portal 销毁时会释放。
更重要的是：
```text
GetCachedPlan()
  -> 不允许插入可能 ERROR 的逻辑
  -> PortalDefineQuery()
```
否则 `CachedPlan.refcount` 已经增加，但 Portal 还没接住，ERROR 后会泄漏 plan ref。
`PortalReleaseCachedPlan()` 释放 `portal->cplan` 后会把 `portal->stmts` 置为 `NIL`。
这说明 `portal->stmts` 通常不是独立 ownership，而是受 `portal->cplan` 保护的 borrowed plan list。
### 4.4. `QueryDesc`
`QueryDesc` 定义在 `src/include/executor/execdesc.h`。
`CreateQueryDesc()` 填写第一组字段：
| 字段 | 语义 |
| --- | --- |
| `operation` | 从 `plannedstmt->commandType` 复制。 |
| `plannedstmt` | 本次 executor 要执行的 `PlannedStmt`。 |
| `sourceText` | 原 SQL 文本。 |
| `snapshot` | 本次 query snapshot，会被注册。 |
| `crosscheck_snapshot` | RI update/delete crosscheck snapshot。 |
| `dest` | 本次输出 receiver。 |
| `params` | 本次外部参数值。 |
| `queryEnv` | 本次 query environment。 |
| `instrument_options` | 节点级 instrumentation 请求。 |
| `query_instr_options` | query-level instrumentation 请求。 |
`ExecutorStart()` 填写第二组字段：
| 字段 | 语义 |
| --- | --- |
| `tupDesc` | 顶层输出 tuple descriptor。 |
| `estate` | query-wide executor state。 |
| `planstate` | runtime `PlanState` tree。 |
| `query_instr` | query-level instrumentation 对象。 |
`ExecutorRun()` 更新：
| 字段 | 语义 |
| --- | --- |
| `already_executed` | 是否已经执行过。 |
`CreateQueryDesc()` 返回时，`estate` 和 `planstate` 仍然是 `NULL`。
这正好说明 `PlannedStmt` 还没有变成执行状态。
### 4.5. `EState`
`EState` 属于 executor runtime。
`standard_ExecutorStart()` 会把 `QueryDesc` 和 `PlannedStmt` 转入 `EState`：
```text
estate->es_param_list_info = queryDesc->params
estate->es_param_exec_vals = palloc0(paramExecTypes count)
estate->es_sourceText = queryDesc->sourceText
estate->es_queryEnv = queryDesc->queryEnv
estate->es_snapshot = RegisterSnapshot(queryDesc->snapshot)
estate->es_crosscheck_snapshot = RegisterSnapshot(queryDesc->crosscheck_snapshot)
estate->es_top_eflags = eflags
estate->es_instrument = queryDesc->instrument_options
estate->es_jit_flags = queryDesc->plannedstmt->jitFlags
```
`EState` 还会持有本次执行的 range table runtime arrays、opened relations、rowmarks、result relations、subplan states、tuple table、partition pruning state 和 instrumentation。
这些状态都不能放进 `PlannedStmt`。
### 4.6. `ResultRelInfo`
`PlannedStmt` 只标出哪些 RT index 是 result relation。
运行期需要 `ResultRelInfo`。
`InitResultRelInfo()` 会补齐：
```text
Relation descriptor
trigger descriptor copy
trigger FmgrInfo array
trigger instrumentation
FDW routine
constraint / generated column / returning expression state
partition routing maps
on conflict state
MERGE action state
```
这些状态依赖当前 relcache、trigger cache、FDW handler、partition map 和 executor context。
它们必须运行期创建，也必须运行期清理。
### 4.7. `PlanState`
`PlanState` 是 `Plan` tree 的运行期形态。
`ExecInitNode()` 按 `nodeTag(node)` 分派到具体节点初始化函数。
每个 `ExecInit*` 会创建：
```text
PlanState common header
EState pointer
ExprContext
ExprState for qual/projection
TupleTableSlot
child PlanState
node private state
ExecProcNode function pointer
Instrumentation pointer
```
所以 `PlannedStmt->planTree` 是说明书。
`QueryDesc->planstate` 才是本次执行的状态机。
---
## 5. 主流程源码 walkthrough
### 5.1. `GetCachedPlan()`：拿到 refcounted `CachedPlan`
位置：`src/backend/utils/cache/plancache.c`。
主链路：
```text
GetCachedPlan()
  -> RevalidateCachedQuery()
  -> choose_custom_plan()
  -> CheckCachedPlan() or BuildCachedPlan()
  -> maybe install plansource->gplan
  -> plan->refcount++
  -> maybe ResourceOwnerRememberPlanCacheRef()
  -> set PlannedStmt.planOrigin
  -> return plan
```
这一层的输出是 `CachedPlan`。
它已经过 plan cache validity 检查。
它还没有 executor snapshot。
它还没有 `EState`。
它还没有 `PlanState`。
如果调用者传入 `owner`，plan ref 挂到 `ResourceOwner`。
如果传入 `NULL`，调用者必须用 Portal 或其他结构接住 ref。
`owner && !plansource->is_saved` 会报错，因为未保存的 cached plan 不允许交给 ResourceOwner 管理。
### 5.2. extended `Bind`：把 plan ref 交给 Portal
位置：`src/backend/tcop/postgres.c`。
`Bind` 阶段会：
```text
cplan = GetCachedPlan(psrc, params, NULL, NULL)
PortalDefineQuery(portal,
                  saved_stmt_name,
                  query_string,
                  psrc->commandTag,
                  cplan->stmt_list,
                  cplan)
PortalStart(portal, params, 0, InvalidSnapshot)
```
这里的 `params` 是本次 Bind 的 `ParamListInfo`。
`CachedPlanSource` 保存的是参数类型契约。
`ParamListInfo` 保存的是本次参数值。
两者边界不能混。
### 5.3. SQL `EXECUTE`：同样走 Portal
位置：`src/backend/commands/prepare.c`。
SQL `EXECUTE` 会先计算 SQL 形式的参数表达式，得到 `ParamListInfo`。
然后：
```text
cplan = GetCachedPlan(entry->plansource, paramLI, NULL, NULL)
plan_list = cplan->stmt_list
PortalDefineQuery(portal, NULL, query_string, commandTag, plan_list, cplan)
PortalStart(portal, paramLI, eflags, GetActiveSnapshot())
```
这个路径和 extended protocol 差异在外层协议和参数求值。
交给 executor 的边界仍然是 `PlannedStmt + runtime params + snapshot`。
### 5.4. `PortalDefineQuery()`：只做 ownership 交接
位置：`src/backend/utils/mmgr/portalmem.c`。
这个函数故意很窄。
它不应该检查权限。
它不应该打开 relation。
它不应该启动 executor。
它只把 `stmts` 和 `cplan` 存进 Portal，并把状态推进到 `PORTAL_DEFINED`。
原因是：这里正处于 plan cache refcount 转交窗口。
在 `GetCachedPlan()` 与 `PortalDefineQuery()` 之间插入可能 `ERROR` 的逻辑，是典型 refcount leak 风险。
### 5.5. `PortalStart()`：进入一次执行请求
位置：`src/backend/tcop/pquery.c`。
`PortalStart()` 先切换执行现场：
```text
ActivePortal = portal
CurrentResourceOwner = portal->resowner
PortalContext = portal->portalContext
MemoryContextSwitchTo(PortalContext)
portal->portalParams = params
portal->strategy = ChoosePortalStrategy(portal->stmts)
```
从这一刻开始，同一个 `PlannedStmt` list 已经属于一次具体执行请求。
Portal 提供：
```text
portal memory context
portal resource owner
bound parameters
cursor options
source text
command tag
execution strategy
queryDesc holder
cleanup hook
```
### 5.6. snapshot 在 Portal 层建立
`PORTAL_ONE_SELECT` 路径会在创建 `QueryDesc` 前：
```text
if (snapshot)
  PushActiveSnapshot(snapshot)
else
  PushActiveSnapshot(GetTransactionSnapshot())
```
`ExecutorStart()` 后面断言：
```text
GetActiveSnapshot() == queryDesc->snapshot
```
这说明 snapshot 是本次执行环境，不是 `PlannedStmt` 的缓存内容。
同一 generic plan 在不同事务中会使用不同 snapshot。
把 snapshot 放入 plan cache 会直接破坏 MVCC。
### 5.7. `CreateQueryDesc()`：把一个 `PlannedStmt` 包成 invocation
位置：`src/backend/tcop/pquery.c`。
关键赋值：
```text
qd->operation = plannedstmt->commandType
qd->plannedstmt = plannedstmt
qd->sourceText = sourceText
qd->snapshot = RegisterSnapshot(snapshot)
qd->crosscheck_snapshot = RegisterSnapshot(crosscheck_snapshot)
qd->dest = dest
qd->params = params
qd->queryEnv = queryEnv
qd->instrument_options = instrument_options
qd->estate = NULL
qd->planstate = NULL
qd->already_executed = false
```
断点停在这里时，能看到 `PlannedStmt` 已存在，但 executor runtime state 尚未创建。
`QueryDesc` 是 `PlannedStmt` 与运行期输入的组合点。
### 5.8. `ExecutorStart()`：hook 与 query id
位置：`src/backend/executor/execMain.c`。
入口先做：
```text
pgstat_report_query_id(queryDesc->plannedstmt->queryId, false)
```
然后进入 `ExecutorStart_hook` 或 `standard_ExecutorStart()`。
这说明 executor hook 的入口已经能看到 `QueryDesc`。
但在标准初始化前，`queryDesc->estate` 可能仍为 `NULL`。
扩展在 hook 中必须尊重这个阶段差异。
### 5.9. `standard_ExecutorStart()`：创建 `EState`
核心检查：
```text
Assert(queryDesc != NULL)
Assert(queryDesc->estate == NULL)
Assert(GetActiveSnapshot() == queryDesc->snapshot)
```
然后处理 read-only transaction 和 parallel mode 写入限制：
```text
if ((XactReadOnly || IsInParallelMode()) &&
    !(eflags & EXEC_FLAG_EXPLAIN_ONLY))
  ExecCheckXactReadOnly(queryDesc->plannedstmt)
```
再创建 executor state：
```text
estate = CreateExecutorState()
queryDesc->estate = estate
MemoryContextSwitchTo(estate->es_query_cxt)
```
这一步把执行生命周期从 Portal / QueryDesc 推入 executor。
`estate->es_query_cxt` 是本次 executor runtime 的主要 memory context。
### 5.10. 运行期输入进入 `EState`
`standard_ExecutorStart()` 接着填充：
```text
estate->es_param_list_info = queryDesc->params
estate->es_sourceText = queryDesc->sourceText
estate->es_queryEnv = queryDesc->queryEnv
estate->es_snapshot = RegisterSnapshot(queryDesc->snapshot)
estate->es_crosscheck_snapshot = RegisterSnapshot(queryDesc->crosscheck_snapshot)
estate->es_top_eflags = eflags
estate->es_instrument = queryDesc->instrument_options
estate->es_jit_flags = queryDesc->plannedstmt->jitFlags
```
如果 `plannedstmt->paramExecTypes` 非空，还会分配 `ParamExecData` 数组。
这说明 `PlannedStmt` 保存参数槽类型。
实际参数值来自 `QueryDesc->params`，内部参数值来自运行期 `ParamExecData`。
### 5.11. command id 和 trigger query context
`ExecutorStart()` 根据 `operation` 设置 output command id。
SELECT 只有在 rowmarks 或 modifying CTE 场景下需要：
```text
if (rowMarks != NIL || hasModifyingCTE)
  estate->es_output_cid = GetCurrentCommandId(true)
```
DML 直接获取 command id。
如果不是 skip triggers 或 EXPLAIN-only，还会调用 `AfterTriggerBeginQuery()`。
这些状态依赖当前事务命令边界。
它们不能被 plan cache 保存。
### 5.12. `InitPlan()`：兑现 `PlannedStmt` 合同
`ExecutorStart()` 最后调用：
```text
InitPlan(queryDesc, eflags)
```
`InitPlan()` 提取：
```text
PlannedStmt *plannedstmt = queryDesc->plannedstmt
Plan *plan = plannedstmt->planTree
List *rangeTable = plannedstmt->rtable
EState *estate = queryDesc->estate
```
第一步是权限检查：
```text
ExecCheckPermissions(rangeTable, plannedstmt->permInfos, true)
```
这说明 prepared statement 不是绕过权限的 capability。
plan cache 能保存权限检查所需的信息。
当前执行仍要检查当前 role 是否满足这些权限。
### 5.13. `ExecInitRangeTable()`：建立 RT index 数组
`InitPlan()` 调用：
```text
ExecInitRangeTable(estate,
                   rangeTable,
                   plannedstmt->permInfos,
                   bms_copy(plannedstmt->unprunableRelids))
```
它设置：
```text
estate->es_range_table = rangeTable
estate->es_rteperminfos = permInfos
estate->es_range_table_size = list_length(rangeTable)
estate->es_unpruned_relids = unpruned_relids
estate->es_relations = palloc0(size * sizeof(Relation))
estate->es_result_relations = NULL
estate->es_rowmarks = NULL
```
注意 `es_relations` 初始都是 `NULL`。
`rtable` 不是 opened relation table。
它只是 executor 根据 RT index 打开 relation 的合同。
### 5.14. runtime initial partition pruning
`InitPlan()` 设置：
```text
estate->es_plannedstmt = plannedstmt
estate->es_part_prune_infos = plannedstmt->partPruneInfos
ExecDoInitialPruning(estate)
```
`ExecDoInitialPruning()` 会创建 `PartitionPruneState`，并把结果保存到 `estate->es_part_prune_results`。
如果有 initial pruning step，它会计算本次有效 child subplans。
如果没有 initial pruning，也会追加 `NULL` 以保持和 `es_part_prune_infos` 对齐。
边界是：
```text
partPruneInfos 是 plan-time 描述。
es_part_prune_results 是 runtime 结果。
```
generic plan 仍然可能在 executor runtime 利用参数做 pruning。
### 5.15. `PlanRowMark` 变成 `ExecRowMark`
如果 `plannedstmt->rowMarks` 非空，`InitPlan()` 会创建 `estate->es_rowmarks` 数组。
然后遍历每个 `PlanRowMark`。
运行期会：
```text
跳过 parent rowmark
跳过已被 pruning 的 child relation
必要时调用 ExecGetRangeTableRelation()
检查 CheckValidRowMarkRel()
创建 ExecRowMark
填入 markType / strength / waitPolicy / curCtid 等 runtime 字段
```
`PlanRowMark` 是合同。
`ExecRowMark` 是当前执行状态。
后者包含 relation pointer 和 tuple identity 进度，不能跨执行缓存。
### 5.16. subplans 先于主 plan 初始化
`InitPlan()` 在主 plan 前初始化 `plannedstmt->subplans`。
原因是 `ExecInitSubPlan` 需要在主 tree 初始化时查到这些 state。
伪链路：
```text
foreach subplan in plannedstmt->subplans:
  sp_eflags = eflags without BACKWARD/MARK
  if id in plannedstmt->rewindPlanIDs:
    sp_eflags |= EXEC_FLAG_REWIND
  subplanstate = ExecInitNode(subplan, estate, sp_eflags)
  estate->es_subplanstates = lappend(...)
```
`PlannedStmt` 保存 subplan `Plan`。
`EState` 保存 subplan `PlanState`。
### 5.17. main `Plan` 变成 `PlanState`
主计划初始化：
```text
planstate = ExecInitNode(plan, estate, eflags)
```
`ExecInitNode()` 位于 `src/backend/executor/execProcnode.c`。
它按 node tag 分派：
```text
T_Result      -> ExecInitResult()
T_ModifyTable -> ExecInitModifyTable()
T_SeqScan     -> ExecInitSeqScan()
T_IndexScan   -> ExecInitIndexScan()
T_HashJoin    -> ExecInitHashJoin()
T_Agg         -> ExecInitAgg()
```
每个节点初始化自己的 runtime state。
scan node 会打开 relation 或 index。
join node 会初始化左右子树。
ModifyTable 会构造 result relation runtime 状态。
Sort、Hash、Agg 会准备自己的内存结构和可能的外部资源。
### 5.18. relation 打开是运行期动作
`ExecOpenScanRelation()` 调用 `ExecGetRangeTableRelation()`。
普通 backend 路径中：
```text
rel = table_open(rte->relid, NoLock)
Assert(CheckRelationLockedByMe(rel, rte->rellockmode, false))
```
parallel worker 路径中会用 `rte->rellockmode` 获取自己的 local lock。
这说明：
```text
RTE 记录 relid 和 lockmode。
Relation * 在当前 executor runtime 打开。
```
如果 relation 是未 populated 的 materialized view，`ExecOpenScanRelation()` 会报错。
这类错误属于 runtime relation 状态，不是 plan cache miss。
### 5.19. output tuple descriptor 回写 `QueryDesc`
主 plan 初始化后：
```text
tupType = ExecGetResultType(planstate)
```
SELECT 顶层如果有 junk attrs，会创建 junk filter，并把输出类型替换成 clean tuple descriptor。
最后：
```text
queryDesc->tupDesc = tupType
queryDesc->planstate = planstate
```
此时 `ExecutorStart()` 完成。
`QueryDesc` 同时持有：
```text
plannedstmt: 静态合同
estate: query-wide runtime state
planstate: node runtime state tree
tupDesc: receiver-visible output shape
```
### 5.20. `ExecutorRun()` 与 `ExecutorEnd()`
`ExecutorRun()` 要求 active snapshot 和 `estate->es_snapshot` 一致。
它根据：
```text
operation == CMD_SELECT || plannedstmt->hasReturning
```
决定是否启动 `DestReceiver`。
`DestReceiver` 是本次执行选择，不是 plan cache 状态。
`ExecutorEnd()` 会：
```text
ExecEndPlan(queryDesc->planstate, estate)
UnregisterSnapshot(estate->es_snapshot)
UnregisterSnapshot(estate->es_crosscheck_snapshot)
FreeExecutorState(estate)
queryDesc->tupDesc = NULL
queryDesc->estate = NULL
queryDesc->planstate = NULL
queryDesc->query_instr = NULL
```
它释放 executor runtime state。
它不释放 `Portal->cplan`。
`CachedPlan` ref 由 Portal 或 ResourceOwner 释放。
---
## 6. 生命周期 / ownership / cleanup
生命周期要按 owner 分层。
```text
CachedPlanSource:
  持有 query 语义来源和可选 generic CachedPlan。
CachedPlan:
  持有 stmt_list，用 refcount 防止使用中释放。
Portal:
  持有 cplan ref、stmts borrowed pointer、params、queryDesc、portal context 和 cleanup hook。
QueryDesc:
  持有本次 PlannedStmt、snapshot refs、dest、params，以及 ExecutorStart 后的 estate/planstate 指针。
EState / PlanState:
  持有本次 executor runtime state，在 ExecutorEnd 中释放。
```
### 6.1. `PlannedStmt` 谁创建
`standard_planner()` 在 `src/backend/optimizer/plan/planner.c` 末尾构造 `PlannedStmt`。
它把 `glob->finalrtable`、`glob->finalrteperminfos`、`glob->subplans`、`glob->finalrowmarks`、`glob->relationOids`、`glob->invalItems`、`glob->paramExecTypes` 等写入 result。
它还根据 cost 和 JIT GUC 设置 `jitFlags`。
`BuildCachedPlan()` 把 planner 输出放入 `CachedPlan->stmt_list`。
### 6.2. `CachedPlan` 谁释放
generic plan 通常被 `CachedPlanSource->gplan` 持有。
一次执行通过 `Portal->cplan` 或 `ResourceOwner` 增加 active ref。
`ReleaseCachedPlan()` decrement refcount。
refcount 到 0 时，如果不是 one-shot plan，就删除 `plan->context`。
这释放的是 cached plan 内存，不是 executor runtime。
### 6.3. `QueryDesc` 谁释放
`CreateQueryDesc()` 注册 snapshot。
`FreeQueryDesc()` 要求 `qdesc->estate == NULL`，然后注销 `qdesc->snapshot` 和 `qdesc->crosscheck_snapshot`，再释放 `QueryDesc` 本体。
所以正常顺序是：
```text
ExecutorFinish()
ExecutorEnd()
FreeQueryDesc()
PortalReleaseCachedPlan()
```
具体顺序会由 portal strategy 和 cleanup 路径控制，但 `FreeQueryDesc()` 不能在 `ExecutorEnd()` 前释放。
### 6.4. ERROR / abort 兜底
ERROR 不会沿普通 C 返回路径逐层退回。
兜底来自三层：
```text
Portal cleanup:
  运行 executor shutdown、释放 cplan ref、清理 portal 子 context。
ResourceOwner:
  释放 transient plan refs、snapshot、buffer pin、lock、文件等外部资源。
MemoryContext:
  批量释放 Portal / executor context 下的 palloc 内存。
```
`AtAbort_Portals()` 会处理 failed portal，调用 cleanup hook，释放 cached plan ref，并把 `portal->resowner` 置空，等待事务级 ResourceOwner cleanup 释放外部资源。
---
## 7. 正确性机制层次
| 机制 | 保证 | 不要误解为 |
| --- | --- | --- |
| plan cache revalidation | 返回给 caller 的 `CachedPlan` 在当前语义下可用 | 当前执行永远不会被 DDL 影响 |
| `CachedPlan.refcount` | `stmt_list` 内存使用中不被释放 | plan 语义仍然有效 |
| `planOrigin` | 标记 generic/custom/internal 来源 | 改变 executor 初始化路径 |
| snapshot registration | 本次 query 使用稳定 MVCC snapshot | snapshot 可跨执行缓存 |
| `ExecCheckPermissions()` | 当前 role 满足 `permInfos` | prepared statement 绕过权限 |
| relation open / locks | 当前 backend 有可用 relation descriptor | `rtable` 已经是 opened relation |
| `ResultRelInfo` | DML runtime target state | DML state 可以放入 `PlannedStmt` |
| MemoryContext | 释放普通 executor/portal 内存 | 释放外部 resource ref |
| ResourceOwner | ERROR/abort 时释放外部资源 | 管理所有 palloc chunk |
### 7.1. `PlannedStmt` 的合同边界
`PlannedStmt` 的正确性来自 planner 和 setrefs 阶段。
它已经把 plan tree、range table、subplan、rowmark、permission info、param exec type 和 invalidation dependency 整理成 executor 可读形式。
但它不证明当前事务能执行。
当前事务是否 read-only、当前 snapshot 是什么、当前 role 是否仍有权限、当前 relation 是否可扫描，都必须运行期确认。
### 7.2. active snapshot 不变量
`ExecutorStart()` 断言 active snapshot 等于 `queryDesc->snapshot`。
`PortalStart()` / `PortalRun()` 负责 push。
绕过 portal 直接调用 executor 的内部路径，也必须满足这个不变量。
`QueryDesc` 保存 snapshot ref。
active snapshot stack 提供执行期间的全局语义上下文。
两者都需要。
### 7.3. lock 与 relation 指针
`RTE` 中的 `relid` 和 `rellockmode` 是合同。
普通 executor 打开 relation 时使用 `NoLock`，并 assert 已经持有合适 lock。
parallel worker 需要自己的 local lock。
这说明 lock ownership 和 relation descriptor 都是 runtime 边界。
不能把 `Relation *` 塞进 cached plan。
### 7.4. invalidation 与 active execution
invalidation 只阻止未来错误复用。
它不会直接 free 一个正在被 portal refcount 持有的 `CachedPlan`。
当前 executor 已经有 `QueryDesc`、snapshot、`EState`、opened relations 和 runtime state。
下一次 `GetCachedPlan()` 才会重新验证或重建。
---
## 8. 错误路径 / 异常路径 / fallback
### 8.1. refcount 转交窗口
危险窗口：
```text
cplan = GetCachedPlan(...)
/* 这里如果 ERROR，cplan ref 泄漏 */
PortalDefineQuery(..., cplan)
```
源码注释明确要求中间不能放可能抛错的逻辑。
这是一条 ownership 不变量，不是代码风格。
### 8.2. cached plan result type 改变
`plancache.c` 中 `RevalidateCachedQuery()` 会重新计算 result descriptor。
如果 `plansource->fixed_result` 为 true，且新旧 tuple descriptor 不一致，会报：
```text
cached plan must not change result type
```
这类错误发生在 plan cache revalidation。
不要把它归到 `ExecutorStart()` 的 tuple descriptor 初始化。
### 8.3. read-only transaction 执行 DML
`GetCachedPlan()` 可以正常返回 DML plan。
`ExecutorStart()` 会在当前事务状态下调用 `ExecCheckXactReadOnly()`。
如果事务是 read-only，错误属于 runtime environment 无法满足 `PlannedStmt` 合同。
### 8.4. snapshot 调用错误
debug build 下，如果调用者没有正确设置 active snapshot，`ExecutorStart()` 断言会失败。
这说明内部调用 executor 不能只构造 `QueryDesc`。
还必须保证 active snapshot stack 与 `QueryDesc` 一致。
### 8.5. relation runtime 状态错误
`ExecOpenScanRelation()` 会检查 relation 是否可扫描。
未 populated 的 materialized view 会在这里报错。
这不是 plan cache 没命中。
这是当前 relation runtime state 不满足扫描合同。
### 8.6. runtime pruning 一致性错误
`ExecGetRangeTableRelation()` 对 scan relation 会检查 RT index 是否在 `estate->es_unpruned_relids` 中。
试图打开已被 pruning 的 scan relation 会 `elog(ERROR)`。
这类错误说明 executor runtime 状态与节点初始化路径不一致。
### 8.7. executor finish / end 顺序
`ExecutorEnd()` 断言 `ExecutorFinish()` 已经调用，除非 EXPLAIN-only。
因为 AFTER triggers、FDW 和 queued side effects 必须先 drain。
plan cache 不知道这些运行期副作用是否已经完成。
---
## 9. 成本、资源与跨模块传播
### 9.1. generic plan 不等于 zero startup
命中 generic plan 主要省掉 parse/analyze/rewrite/plan 成本。
每次执行仍要付出：
```text
Portal setup
CreateQueryDesc()
snapshot registration
CreateExecutorState()
ExecCheckPermissions()
ExecInitRangeTable()
ExecDoInitialPruning()
ExecInitNode() for subplans and main plan
relation open
ResultRelInfo / trigger / FDW state initialization
Instrumentation allocation
```
因此短查询中，executor startup 可能成为显著成本。
### 9.2. 成本随什么扩张
plan cache 边界成本随这些变量扩张：
```text
query tree 大小
dependency list 长度
generic/custom 重建频率
```
executor startup 成本随这些变量扩张：
```text
Plan node 数
subplan 数
rtable 长度
rowmark 数
partition 数
result relation 数
trigger / FDW / constraint 状态
instrumentation 选项
```
relation runtime 成本随这些变量扩张：
```text
relcache miss
partition leaf 数
parallel worker 数
lock 等待
index / table AM 初始化需求
```
### 9.3. 参数的两层作用
参数类型属于 `CachedPlanSource` 合同。
参数值属于本次 `ParamListInfo`。
custom plan 可以让 planner 看见参数值。
generic plan 不以参数值定制 plan shape。
但 executor 仍然通过：
```text
estate->es_param_list_info
runtime partition pruning
scan key setup
expression evaluation
SubPlan parameter passing
```
使用本次参数值。
所以 generic plan 不等于执行期不看参数。
### 9.4. 跨模块连接
plan cache：
```text
保证返回的 stmt_list 在执行前经过 revalidation。
```
Portal：
```text
保证 cplan ref、params、snapshot、queryDesc 和 cleanup 边界。
```
executor：
```text
保证把静态 Plan 转成 EState / PlanState，并推进 tuple 流。
```
relcache / syscache：
```text
提供当前 relation、trigger、FDW、type、function 等元数据。
```
lock manager：
```text
保证 relation 访问前已满足 lock contract。
```
observability：
```text
queryId、planId、planOrigin、instrumentation、wait event 和 EXPLAIN 输出分布在不同边界。
```
### 9.5. 后台进程边界
`CachedPlan`、Portal、`QueryDesc`、`EState` 都是 backend-local，没有后台进程维护 shared global plan cache。
但 DDL backend、autovacuum、checkpointer、bgwriter、walwriter 会通过 catalog invalidation、可见性、统计、I/O、WAL 和 lock pressure 改变 executor runtime 成本。
---
## 10. 观测与诊断入口
本节锚定的 runtime truth：
```text
同一个 prepared statement 即使命中 generic plan，每次执行仍会创建新的 QueryDesc、EState 和 PlanState，并使用本次 snapshot、params、DestReceiver、relation runtime state。
```
### 10.1. SQL 级观测
`pg_prepared_statements` 能看到：
```text
name
statement
parameter_types
generic_plans
custom_plans
```
它看不到：
```text
CachedPlan.refcount
portal->stmts 是否借用 cplan->stmt_list
QueryDesc->estate 何时创建
PlanState tree 是否已初始化
estate->es_relations 哪些 relation 已打开
```
`EXPLAIN EXECUTE` 能看到当前计划形状。
`EXPLAIN (ANALYZE, BUFFERS, WAL, MEMORY)` 能看到执行结果和部分资源统计。
但它不能完整分解 plan cache 命中、executor startup 和 tuple loop 成本。
### 10.2. 断点入口
推荐断点：
```text
b GetCachedPlan
b PortalDefineQuery
b PortalStart
b CreateQueryDesc
b ExecutorStart
b InitPlan
b ExecInitNode
b ExecGetRangeTableRelation
b ExecutorEnd
b PortalReleaseCachedPlan
```
观察点：
```text
GetCachedPlan 返回后 plan->refcount
portal->stmts 和 cplan->stmt_list 地址
CreateQueryDesc 后 qd->estate == NULL
ExecutorStart 后 qd->estate != NULL
InitPlan 后 estate->es_range_table_size
ExecGetRangeTableRelation 后 estate->es_relations[rti - 1]
ExecutorEnd 后 queryDesc->estate == NULL
PortalReleaseCachedPlan 后 portal->stmts == NIL
```
### 10.3. 指标误读边界
`generic_plans` 增加只说明 generic plan 被使用。
它不说明 executor startup 为零。
`planning time` 低不说明 `ExecutorStart()` 成本低。
`execution time` 高且 rows 很少时，可能是 startup、lock wait、relation open、trigger/FDW initialization、instrumentation 或 receiver 输出成本。
wait event 只能说明当前等待点。
它不自动告诉你等待发生在 plan cache、executor startup、node execution 还是 output receiver。
### 10.4. 从错误定位层次
`cached plan must not change result type`：优先看 `CachedPlanSource.fixed_result` 和 `RevalidateCachedQuery()`。
`cannot execute INSERT in a read-only transaction`：优先看 `ExecutorStart()` 的 transaction mode check。
`materialized view has not been populated`：优先看 `ExecOpenScanRelation()` 的 runtime scannability。
`permission denied for table ...`：优先看 `ExecCheckPermissions()`、当前 role 和 `permInfos`。
---
## 11. 常见误区
误区 1：`PlannedStmt` 就是可执行对象。
实际：它缺少 snapshot、params、DestReceiver、EState、PlanState、Relation 指针、ResultRelInfo、ExprState、slot 和 instrumentation。
误区 2：generic plan 可以复用 `PlanState`。
实际：generic plan 复用 `PlannedStmt`；`PlanState` 每次 `ExecutorStart()` 都新建。
误区 3：`CachedPlan.refcount` 说明 plan 语义仍有效。
实际：refcount 只管内存安全；validity 和 invalidation 管未来语义复用。
误区 4：`rtable` 已经打开了 relation。
实际：`rtable` 保存 RTE；`Relation *` 在 executor runtime 通过 RT index 打开并放入 `estate->es_relations`。
误区 5：prepared statement 缓存后不用再做权限检查。
实际：`InitPlan()` 仍调用 `ExecCheckPermissions()`。
误区 6：参数只影响 custom plan。
实际：generic plan 不用参数值定制 plan shape，但 executor 仍使用参数值做表达式求值、runtime pruning 和 scan key setup。
误区 7：Portal cleanup 等于 `ExecutorEnd()`。
实际：`ExecutorEnd()` 清理 executor runtime；Portal cleanup 还处理 `QueryDesc`、`CachedPlan` ref、portal context、hold store 和 ResourceOwner 边界。
---
## 12. 课堂实验
### 12.1. 实验一：画出状态交接链
准备：
```sql
CREATE TABLE pc_exec_demo(id int primary key, v text);
INSERT INTO pc_exec_demo
SELECT g, md5(g::text) FROM generate_series(1, 1000) AS g;
PREPARE q(int) AS SELECT * FROM pc_exec_demo WHERE id = $1;
EXECUTE q(1);
```
断点：
```text
b GetCachedPlan
b PortalDefineQuery
b CreateQueryDesc
b ExecutorStart
b InitPlan
b ExecInitNode
```
观察：
```text
cplan->stmt_list
portal->stmts
qd->plannedstmt
qd->estate before ExecutorStart
qd->estate after ExecutorStart
qd->planstate after InitPlan
```
要求画图：
```text
CachedPlan
  -> Portal
  -> QueryDesc
  -> EState
  -> PlanState
```
每条边标注 owner 和 cleanup 函数。
### 12.2. 实验二：generic plan 仍然创建 executor state
SQL：
```sql
CREATE TABLE pc_exec_demo2(id int, v text);
INSERT INTO pc_exec_demo2
SELECT g, md5(g::text) FROM generate_series(1, 10000) AS g;
CREATE INDEX ON pc_exec_demo2(id);
ANALYZE pc_exec_demo2;
PREPARE q2(int) AS SELECT * FROM pc_exec_demo2 WHERE id = $1;
EXECUTE q2(1);
EXECUTE q2(6);
SELECT name, generic_plans, custom_plans
FROM pg_prepared_statements
WHERE name = 'q2';
```
如果 generic plan 计数增加，再用断点确认每次仍进入 `CreateQueryDesc()`、`ExecutorStart()`、`InitPlan()`、`ExecInitNode()`。
结论：
```text
generic plan 命中省掉 planning，不省掉 executor runtime state 初始化。
```
### 12.3. 实验三：观察 relation runtime 打开
断点：
```text
b ExecInitRangeTable
b ExecGetRangeTableRelation
b ExecOpenScanRelation
```
观察：
```text
ExecInitRangeTable 后 estate->es_relations 初始为空。
ExecOpenScanRelation 后相应 RT index 才出现 Relation 指针。
下一次 EXECUTE 会有新的 EState 和新的 es_relations。
```
这个实验验证：
```text
rtable 是合同。
Relation * 是运行期对象。
```
### 12.4. 实验四：read-only DML 定位
SQL：
```sql
BEGIN READ ONLY;
PREPARE qi AS INSERT INTO pc_exec_demo2 VALUES (99999, 'x');
EXECUTE qi;
ROLLBACK;
```
断点：
```text
b GetCachedPlan
b ExecutorStart
b ExecCheckXactReadOnly
```
解释：
```text
GetCachedPlan 可以返回 DML PlannedStmt。
错误发生在 ExecutorStart 的当前事务检查。
```
---
## 13. 讨论题
1. 为什么 `PlannedStmt` 保存 `rtable`，但不能保存 `Relation *`？
2. `CachedPlan.refcount` 与 `CachedPlan.is_valid` 分别保护什么？
3. 为什么 `GetCachedPlan()` 和 `PortalDefineQuery()` 之间不能加入可能 `ERROR` 的逻辑？
4. `CreateQueryDesc()` 注册 snapshot 后，为什么 `ExecutorStart()` 仍要求 active snapshot 匹配？
5. generic plan 命中后，哪些成本消失，哪些 executor startup 成本仍然存在？
6. `PlanRowMark` 和 `ExecRowMark` 的边界是什么？
7. `pg_prepared_statements.generic_plans` 能说明什么，不能说明什么？
8. 如果错误发生在 `ExecOpenScanRelation()`，你如何判断它属于 runtime relation 状态还是 plan cache contract？
---
## 14. 本节小结
本节核心链路：
```text
CachedPlanSource
  -> GetCachedPlan()
  -> CachedPlan->stmt_list
  -> PortalDefineQuery()
  -> PortalStart()
  -> CreateQueryDesc()
  -> ExecutorStart()
  -> InitPlan()
  -> ExecInitNode()
  -> EState / PlanState
```
`PlannedStmt` 是 executor 可执行契约。
它固定命令类型、plan tree、range table、权限信息、rowmarks、subplans、partition pruning 描述、依赖项、JIT flags 和参数槽类型。
它不固定本次 snapshot、参数值、DestReceiver、Relation 指针、ResultRelInfo、EState、PlanState、ExprState、TupleTableSlot 或 instrumentation。
ownership 边界：
```text
CachedPlan refcount 保护 stmt_list。
Portal 接住 cplan ref 和 execution request。
QueryDesc 组合 PlannedStmt 与 runtime inputs。
EState / PlanState 由 ExecutorStart 创建，由 ExecutorEnd 释放。
ResourceOwner 兜底外部资源和 transient refs。
MemoryContext 管普通内存。
```
正确性边界：
```text
plan cache revalidation 管未来复用语义。
refcount 管内存安全。
snapshot registration 管本次 MVCC 视图。
ExecCheckPermissions 管当前权限。
relation open / lock contract 管 runtime relation 访问。
executor cleanup 管运行态收尾。
```
观测边界：`pg_prepared_statements` 能看 generic/custom 计数；`EXPLAIN` 能看计划形状和执行结果；gdb 断点最适合看 `QueryDesc`、`EState`、`PlanState` 的创建与释放。
可迁移规律：
```text
缓存层应该保存可复用的语义合同；
执行层必须在当前事务、当前 snapshot、当前 ResourceOwner、当前 backend 中补齐可变运行态。
把运行态塞进长期缓存，通常会制造 invalidation、ownership 和 cleanup 错误。
```
哪些判断仍依赖上下文：generic/custom 选择依赖 workload 和参数分布；executor startup 成本依赖 plan shape、partition 数、relation 数、trigger/FDW 状态和 instrumentation；planning time、execution time、wait event 的归因需要结合源码断点或 profiling；不同 PostgreSQL 版本可能调整 `PlannedStmt` 字段、`planId`、JIT flags、instrumentation 和 portal 路径。
