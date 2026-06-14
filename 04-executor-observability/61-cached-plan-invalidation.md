# PostgreSQL cached plan invalidation
## 课程定位
前置知识：已经理解 frontend/backend protocol message loop，知道 extended query 的 `Parse` 会创建 prepared statement，`Bind` / `Execute` 会通过 portal 进入执行器。
前置知识：已经读过 executor 生命周期课程，知道 executor 消费的是 `PlannedStmt`，而不是 SQL 字符串或 raw parse tree。
前置知识：知道 relcache、syscache、GUC、search_path 都是 backend-local 语义环境的一部分，但它们会被共享 invalidation 或本 backend 配置变化推动。
本节唯一主问题：
```text
catalog invalidation、search_path、GUC 和 relcache 变化如何让 cached plan 失效或重建？
```
核心矛盾：prepared statement 需要跨多次执行复用分析结果和计划，降低 parse/analyze/rewrite/plan 成本；但 SQL 名称解析、权限/RLS、函数、类型、operator、relation schema、planner GUC 和 relcache 状态都可能在两次执行之间变化，旧计划不能被盲目执行。
学完后应能判断：一次 prepared statement 变慢、重新规划、结果 tuple descriptor 改变或执行时报 `cached plan must not change result type` 时，问题发生在 query tree revalidation、generic plan validity、custom plan 重建、search_path matcher，还是 relcache/syscache callback。
学完后还应能解释：invalidation 不是锁；它不会阻止 DDL 发生，只是在下次安全边界让本 backend 放弃过期语义。
本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e`。
本节只讲 cached plan 的失效和重建。
generic/custom plan 的选择策略只讲到 invalidation 需要的程度，不展开成本估算公式。
Portal 暂停、DestReceiver 输出和 executor consuming `PlannedStmt` 会在后续课程继续拆开。
## 1. 本节在总主线中的位置
第 57 节从 post-auth backend 的消息循环讲起。
后续待补课程会把 simple query、extended query、prepared statement、portal、DestReceiver 串起来。
本节处在 prepared statement 和 executor 之间。
它回答一个很容易被线上现象逼出来的问题：
```text
为什么我已经 PREPARE 过的语句，有时仍然会重新 parse/rewrite/plan？
```
或者换成更接近源码的问题：
```text
为什么 CachedPlanSource 和 CachedPlan 都有 is_valid，
并且 search_path、RLS role、relcache/syscache callback 还要共同参与判断？
```
从外层看，prepared statement 像是“保存一条 SQL”。
从内核看，prepared statement 保存的是一组分层缓存：
```text
SQL text / raw parse tree
  -> analyzed and rewritten query tree
  -> generic PlannedStmt list
  -> execution-time custom PlannedStmt list
```
每一层依赖的环境不同。
SQL text 本身通常不会失效。
analyzed query tree 依赖名称解析、rewrite、RLS、对象依赖和 search_path。
generic plan 依赖 planner 可见的 relation/schema/function/type/operator 语义、role-sensitive RLS 和 plan-time environment。
custom plan 依赖当前参数值，通常执行完就释放。
本节主线是：
```text
prepared statement 被保存
  -> DDL/GUC/search_path/role/relcache 变化
  -> callback 或环境检查把某一层标成 invalid
  -> GetCachedPlan() 在下次执行时 revalidate
  -> 重用 generic plan 或重建 query tree / generic plan / custom plan
```
这条线补上 executor 之前的一段 observability blind spot。
如果只看 `ExecutorStart()`，你只能看到最终计划。
如果要解释为什么同一个 prepared statement 这次突然花了 planner 时间，必须回到 plan cache 的 invalidation 边界。
## 2. 核心矛盾与一句话运行模型
一句话运行模型：
```text
CachedPlanSource 保存可重新生成计划的 query state；
invalidation callback 只把相关 CachedPlanSource 或 CachedPlan 标成 invalid；
GetCachedPlan() 在下一次执行前用 RevalidateCachedQuery() 和 CheckCachedPlan() 决定复用、重写或重规划。
```
这里的系统 tension 是：
```text
跨执行复用计划以降低延迟和 CPU
  vs
catalog、search_path、GUC、role/RLS、relcache 变化后旧语义不能继续执行
```
PostgreSQL 没有把 prepared statement 设计成“拿锁冻结所有依赖对象”。
它选择另一条路径：
```text
DDL 和 catalog 变化正常提交
  -> sinval 传播到其他 backend
  -> backend 在安全边界接收 invalidation
  -> plan cache entry 标记过期
  -> 下一次执行时按需重建
```
这带来几个直接后果。
第一，失效是 lazy 的。
DDL 提交时不会同步扫描所有 backend 的 plan cache 并重建计划。
第二，失效是分层的。
有时只需要丢 generic plan。
有时必须重新 analyze/rewrite query tree。
有时 current `search_path` 和保存的 matcher 不一致，即使没有接到新的 relcache 消息，也要重走 query rewrite。
第三，prepared statement 的“缓存命中”不是单一布尔值。
你需要区分：
```text
CachedPlanSource.is_valid
CachedPlanSource.gplan
CachedPlan.is_valid
plansource->search_path
plansource->dependsOnRLS
plan->dependsOnRole
generic/custom choice
```
第四，GUC 的影响不是都通过 invalidation callback 进入。
`plan_cache_mode` 直接影响 `choose_custom_plan()`。
planner enable/cost 类 GUC 可能改变新计划的形状。
已有 generic plan 是否被强制重建，要看它是否被 invalidation、role/RLS、search_path 或 explicit reset 触发；不能把“SET 了一个 planner GUC”简单等同于“所有 cached plan 立刻无效”。
本节把这些现象压缩成一个可迁移模型：
```text
cache 的正确性不来自缓存对象本身；
它来自 dependency extraction、environment matcher、invalidation delivery 和 use-before-execute revalidation 的组合。
```
## 3. 核心文件分工与阅读顺序
| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/utils/plancache.h` | `CachedPlanSource`、`CachedPlan`、`CachedExpression` 的字段语义、lifetime 和外部入口。 |
| 2 | `src/backend/utils/cache/plancache.c` | plan cache 创建、保存、失效 callback、`RevalidateCachedQuery()`、`CheckCachedPlan()`、`GetCachedPlan()`、generic/custom plan 重建。 |
| 3 | `src/backend/utils/cache/inval.c` | shared invalidation 的接收、事务结束发送、local execute、syscache/relcache callback 注册和触发。 |
| 4 | `src/backend/utils/cache/relcache.c` | relation cache entry 如何被刷新、清理或整体 invalidated，以及 relcache invalidation 如何向 plan cache callback 传播。 |
| 5 | `src/backend/catalog/namespace.c` | `SearchPathMatcher`、`GetSearchPathMatcher()`、`SearchPathMatchesCurrentEnvironment()`、`activePathGeneration`。 |
| 6 | `src/backend/tcop/postgres.c` | extended query path 中 `Parse` / `Bind` / `Execute` 如何触发 prepared statement 和 `GetCachedPlan()`。 |
| 7 | `src/include/utils/inval.h` | invalidation 对外接口、callback 类型和事务边界入口。 |
建议阅读顺序不是按 DDL 调用链从头追。
先读 `plancache.h`。
因为本节真正要解释的是 state transition，而 state transition 写在结构体字段组合里。
然后读 `plancache.c` 头部注释和 callback 注册。
这能先建立一个事实：
```text
plan cache 不主动监听所有 catalog table；
它注册 relcache callback 和少量 syscache callback，
再用 query tree 中保存的 dependency OID 列表做过滤。
```
接着读 `RevalidateCachedQuery()` 和 `CheckCachedPlan()`。
这两个函数是执行前的语义关口。
最后再回到 `inval.c` 和 `relcache.c`，理解 invalidation message 如何到达 callback。
不要从 `inval.c` 的所有消息类型开始背。
invalidation 子系统服务很多 cache。
本节只关心它怎样把“某个对象可能变了”转成 plan cache 的 `is_valid = false`。
## 4. 关键数据结构与状态
### 4.1. `CachedPlanSource`
`CachedPlanSource` 更像 cached query，不只是 cached plan。
它保存“重新得到计划所需的上游状态”。
本节关注字段：
| 字段 | 语义 |
| --- | --- |
| `raw_parse_tree` | 原始 parse tree；用于重新 analyze/rewrite。 |
| `analyzed_parse_tree` | 已分析 query 的输入形式；某些调用者绕过 raw parse tree，但仍可能需要 rewrite/plan。 |
| `query_string` | 原始 SQL 文本；用于错误信息、重新规划上下文和 observability。 |
| `commandTag` | command completion 和 portal 行为所需的命令类型信息。 |
| `param_types` / `num_params` | prepared statement 参数类型契约。 |
| `fixed_result` / `resultDesc` | 是否禁止结果 tuple descriptor 改变，以及当前保存的结果形状。 |
| `query_list` | analyzed and rewritten query tree list。 |
| `relationOids` | query tree 依赖的 relation OID 列表。 |
| `invalItems` | query tree 依赖的非 relation cache item。 |
| `search_path` | query tree 生成时的 search path matcher。 |
| `rewriteRoleId` | query rewrite/RLS 相关 role。 |
| `rewriteRowSecurity` | query rewrite 时的 `row_security` 环境。 |
| `dependsOnRLS` | query tree 是否对 RLS role/environment 敏感。 |
| `gplan` | 可复用的 generic `CachedPlan`。 |
| `is_saved` | 是否已经放入长期 plan cache list。 |
| `is_valid` | `query_list` 当前是否仍可作为语义基础。 |
| `generation` | plan source 的 generation，用于调用者判断变化。 |
| `context` / `query_context` | 分别管理 plan source 元数据和 query tree 生命周期。 |
`CachedPlanSource.is_valid` 不是“最终 plan 可以执行”。
它只表示 analyzed/rewrite 后的 `query_list` 还没有被证明过期。
如果 `query_list` 有效，generic plan 仍可能无效。
如果 generic plan 无效，custom plan 仍可以在当前参数下新建。
如果 `query_list` 无效，必须先回到 raw parse tree 重新 analyze/rewrite。
### 4.2. `CachedPlan`
`CachedPlan` 表示从 `CachedPlanSource` 派生出来的一组 `PlannedStmt`。
本节关注字段：
| 字段 | 语义 |
| --- | --- |
| `stmt_list` | planner 输出的 `PlannedStmt` / utility statement list。 |
| `is_saved` | 是否放在长期 context，通常对应 saved `CachedPlanSource` 的 generic plan。 |
| `is_valid` | 这个 planned statement list 是否仍可执行。 |
| `planRoleId` | plan 生成时的 role。 |
| `dependsOnRole` | plan 是否依赖当前 role，典型来自 RLS。 |
| `saved_xmin` | transient plan 记录生成时的 `TransactionXmin`；如果后续 `TransactionXmin` 推进，需要重新规划。 |
| `generation` | 父 `CachedPlanSource` 的 generation，用于识别 plan 是否来自当前一代。 |
| `refcount` | 当前使用者数量，防止正在执行的 plan 被释放。 |
| `context` | plan 本体的 memory context。 |
generic plan 通常挂在 `plansource->gplan` 下。
custom plan 通常由 `GetCachedPlan()` 为当前参数创建，调用者拿到引用，执行结束后释放引用。
因此 `CachedPlan.is_valid` 的意义也要和 lifetime 一起读。
一个 custom plan 执行完就释放，它不需要成为长期 invalidation 目标。
一个 generic plan 可能被多次执行，所以必须通过 relcache/syscache callback 标 invalid。
### 4.3. `CachedExpression`
`CachedExpression` 是表达式级缓存。
它使用同一套 invalidation 思路，但本节不展开表达式执行。
需要知道的是，`plancache.c` 里还有 `cached_expression_list`。
callback 同样会扫描它，把依赖过期的表达式标成 invalid。
这解释了为什么 plan cache callback 名字看起来比 prepared statement 更宽。
它不只服务 SQL-level prepared statement。
### 4.4. `SearchPathMatcher`
`SearchPathMatcher` 定义在 namespace 相关代码里。
它不是简单保存 `search_path` GUC 字符串。
它保存的是当前有效 search path 的语义摘要：
```text
activeSearchPath
activeCreationNamespace
activeTempCreationPending
activePathGeneration
```
`namespace_search_path` 只是用户设置的文本。
真正决定名称解析的是经过权限过滤、`pg_catalog`/temp schema 特殊规则和当前用户处理后的 active path。
这就是为什么 `CachedPlanSource` 保存 matcher，而不是保存一段 GUC 字符串。
同一个字符串在不同 role 下可能得到不同 effective path。
同一个 schema 名称在 DROP/CREATE 后也可能对应不同 OID。
`SearchPathMatchesCurrentEnvironment()` 的核心语义是：
```text
如果当前 active path 和 plan source 保存的 matcher 不一致，
旧 query tree 的 unqualified name binding 不能被假定仍然正确。
```
### 4.5. dependency list
`relationOids` 和 `invalItems` 来自 parse/analyze/rewrite/planner dependency extraction。
它们是 plan cache 能够做精确失效的基础。
可以把它们理解成：
```text
query tree 或 plan tree 读过哪些 catalog 语义，
如果这些语义变了，下次执行前必须重新确认。
```
它们不是 lock。
保存 OID 不代表对象不能被 DROP/ALTER。
它们只是让 callback 在收到 invalidation 时知道哪些 `CachedPlanSource` 或 `CachedPlan` 可能受影响。
### 4.6. saved list
`plancache.c` 维护 backend-local 的 saved plan source list 和 saved expression list。
这些 list 不是 shared memory。
每个 backend 有自己的 prepared statement 和 plan cache 状态。
其他 backend 的 DDL 不会直接修改你的内存。
它只会通过 shared invalidation 让你的 backend 在接收消息后执行本地 callback。
这解释了两个线上现象：
```text
同一个 DDL 提交后，不同 backend 可能在不同时间点发现自己的 cached plan invalid。
同一个 prepared statement 名称只在当前 session 内有效，除非外层 pooler 做了额外协议映射。
```
## 5. 主流程源码 walkthrough
### 5.1. 创建 plan source
extended query 的 `Parse` message 会创建 `CachedPlanSource`。
从本节角度看，核心链路可以压缩成：
```text
exec_parse_message()
  -> CreateCachedPlan()
  -> pg_analyze_and_rewrite_varparams()
  -> CompleteCachedPlan()
  -> StorePreparedStatement() 或保存 unnamed statement
```
`CreateCachedPlan()` 建立 plan source 的 memory context 和基本字段。
它还没有完成 analyzed/rewrite query tree。
`CompleteCachedPlan()` 把 `querytree_list`、参数类型、result descriptor、dependency、search path matcher 等信息填入 `CachedPlanSource`。
如果这个 statement 需要 revalidation，`CompleteCachedPlan()` 会收集：
```text
relationOids
invalItems
dependsOnRLS
rewriteRoleId / rewriteRowSecurity
search_path
```
随后把 `plansource->is_valid` 置为 true。
这一步的语义是：
```text
当前 query tree 在当前环境下成立，可以作为以后 GetCachedPlan() 的输入。
```
它不是说 generic plan 已经存在。
generic plan 可能还没有生成。
### 5.2. 保存 plan source
`SaveCachedPlan()` 把一个 transient `CachedPlanSource` 移到更长生命周期。
对 named prepared statement 来说，这让它跨协议消息存活。
关键动作是：
```text
move plansource context to CacheMemoryContext
insert plansource into saved_plan_list
mark is_saved
```
`saved_plan_list` 是 invalidation callback 扫描的对象集合。
没保存的 one-shot plan 不需要长期 callback 管理。
如果一个 `CachedPlanSource` 没有保存，它可以生成计划，但不能指望在事务或消息边界之外继续存在。
### 5.3. 第一次执行
`Bind` 或执行 prepared statement 时会调用 `GetCachedPlan()`。
主流程是：
```text
GetCachedPlan(plansource, boundParams, owner, queryEnv)
  -> RevalidateCachedQuery(plansource, queryEnv)
  -> choose_custom_plan(plansource, boundParams)
  -> custom?
       -> BuildCachedPlan(..., boundParams, queryEnv)
     generic?
       -> CheckCachedPlan(plansource)
       -> maybe BuildCachedPlan(..., NULL, queryEnv)
       -> attach refcount / owner
```
第一步永远是 query tree 层 revalidation。
这保证 planner 不会建立在已经错误的 name binding、rewrite/RLS 或 result descriptor 上。
如果 query tree 看起来仍有效，`RevalidateCachedQuery()` 还会先 `AcquirePlannerLocks()`，再检查 callback 是否已经把它标 invalid。
这是为了覆盖一个 race：invalidation 可能在拿锁之前到达。
如果 race 发生，源码会释放刚拿到的锁，再走完整 re-analysis/rewrite。
之后才讨论 custom/generic。
如果选择 custom plan，本次会用当前参数进入 planner。
如果选择 generic plan，则优先检查已有 `plansource->gplan`。
如果没有 generic plan，或 generic plan 已无效，就重新构造一个 generic `CachedPlan`。
### 5.4. DDL 提交后的 invalidation delivery
另一个 backend 执行 DDL，例如：
```sql
ALTER TABLE t ADD COLUMN c int;
```
DDL 修改 catalog，并在事务结束时通过 invalidation 机制发送相关消息。
简化链路是：
```text
catalog update
  -> RegisterRelcacheInvalidation() / RegisterCatcacheInvalidation()
  -> CommandEndInvalidationMessages()
  -> AtEOXact_Inval(commit)
  -> SendSharedInvalidMessages()
```
收到消息的 backend 不一定立刻正在执行 SQL。
它会在事务开始、命令边界、接收 interrupt、访问 cache 等安全点调用：
```text
AcceptInvalidationMessages()
  -> ReceiveSharedInvalidMessages()
  -> LocalExecuteInvalidationMessage()
  -> syscache / relcache local invalidation
  -> registered callbacks
```
plan cache 在初始化时注册 callback：
```text
CacheRegisterRelcacheCallback(PlanCacheRelCallback, 0)
CacheRegisterSyscacheCallback(PROCOID, PlanCacheObjectCallback, 0)
CacheRegisterSyscacheCallback(TYPEOID, PlanCacheObjectCallback, 0)
CacheRegisterSyscacheCallback(NAMESPACEOID, PlanCacheSysCallback, 0)
...
```
回调执行时不会重建计划。
它只扫描本 backend 的 saved plan/expression list，把受影响对象标为 invalid。
### 5.5. relcache callback 如何标记
`PlanCacheRelCallback()` 收到 relation OID。
如果 OID 是 `InvalidOid`，含义接近“全局 relcache flush”，会让更多 entry 失效。
如果是具体 relation OID，它会扫描 `saved_plan_list`。
对每个 plan source：
```text
if relationOids contains relid:
    plansource->is_valid = false
    if gplan exists:
        gplan->is_valid = false
```
即使 query tree 本身没依赖这个 relation，generic plan 也可能依赖 planner 阶段读到的 relation。
所以 callback 还要检查 generic plan 的 dependency。
这就是两个 invalid flag 同时存在的原因。
query tree 层和 plan 层的 dependency 不完全一样。
### 5.6. syscache callback 如何标记
`PlanCacheObjectCallback()` 处理更具体的 syscache 依赖，例如 function、domain 或 type 相关对象。
它同样扫描 dependency list。
如果 `invalItems` 里有匹配 cache item，就标记 `plansource->is_valid = false`。
如果 generic plan 的 dependency 匹配，就标记 `gplan->is_valid = false`。
`PlanCacheSysCallback()` 更粗。
它用于 namespace、operator、access method operator、foreign server、FDW 等变化。
这类变化难以用现有 saved dependency 精确过滤，或影响名称解析/规划环境，因此直接 `ResetPlanCache()`。
`ResetPlanCache()` 会遍历 saved plan list，把需要 revalidation 的 plan source 和 generic plan 标 invalid。
这不是物理释放所有内存。
它是语义失效。
真正释放或重建发生在后续调用路径。
### 5.7. search_path 检查
search_path 变化不一定来自 shared invalidation。
本 backend 执行：
```sql
SET search_path = s2, public;
```
不会靠另一个 backend 发 sinval。
但下一次 `GetCachedPlan()` 会走：
```text
RevalidateCachedQuery()
  -> SearchPathMatchesCurrentEnvironment(plansource->search_path)
  -> mismatch?
       plansource->is_valid = false
       gplan->is_valid = false
```
如果 unqualified name binding 可能改变，旧 query tree 不能继续使用。
这里有一个细节：matcher 比较的是 effective path。
如果 SET 前后文本不同，但 effective path 相同，不一定需要重写。
反过来，如果文本相同但 role 权限或 schema OID 变化导致 effective path 不同，仍可能不匹配。
### 5.8. RLS 和 role 检查
RLS/rewrite 对当前 role 和 `row_security` 环境敏感。
`CachedPlanSource` 保存 `rewriteRoleId`、`rewriteRowSecurity` 和 `dependsOnRLS`。
`RevalidateCachedQuery()` 会检查当前 role 是否仍匹配。
如果 statement 依赖 RLS，而 role 或 `row_security` 变化，就不能复用旧 query tree。
generic `CachedPlan` 也有 `dependsOnRole`。
`CheckCachedPlan()` 会在 plan 层检查 role-sensitive plan 是否还成立。
这说明 role 变化可能打掉 query tree，也可能只打掉 generic plan。
具体取决于依赖在哪一层被记录。
### 5.9. query tree 重建
当 `plansource->is_valid` 为 false 时，`RevalidateCachedQuery()` 会重新走：
```text
raw parse tree
  -> parse analysis
  -> rewrite
  -> dependency extraction
  -> resultDesc recompute
  -> search_path matcher refresh
```
这里有一个关键正确性边界。
如果 prepared statement 已经暴露过 result type，而重写后的 result descriptor 发生不兼容变化，PostgreSQL 不能假装没事。
调用者可能已经按旧 tuple descriptor 准备了客户端解码或 portal 输出。
因此 `fixed_result = true` 的 cached plan 会检测 result type 改变并报错。
extended query `Parse` 和 SQL-level `PREPARE` 当前都以 fixed result 完成 plan source。
这就是线上常见错误：
```text
cached plan must not change result type
```
这个错误不是 planner 任性。
它保护的是 prepared statement/portal 对外结果契约。
### 5.10. generic plan 重建
如果 query tree 仍有效，但 `plansource->gplan` 不存在或 `gplan->is_valid` 为 false，`CheckCachedPlan()` 返回 false。
随后 `GetCachedPlan()` 可以调用 `BuildCachedPlan()` 重新生成 generic plan。
generic plan 的参数值传 NULL。
planner 会基于参数类型和统计信息，而不是当前具体参数值，生成可复用计划。
`BuildCachedPlan()` 还会为 plan 收集新的 dependency。
这使下一次 relcache/syscache invalidation 能更精确地打掉 plan。
如果已有 generic plan 看起来有效，`CheckCachedPlan()` 会先 `AcquireExecutorLocks()`，再检查 invalidation race。
如果 plan 是 transient plan，它还会比较 `saved_xmin` 和当前 `TransactionXmin`，避免继续使用依赖旧 xmin horizon 的计划。
### 5.11. custom plan 重建
custom plan 不依赖长期 `gplan`。
当 `choose_custom_plan()` 选择 custom 时，`BuildCachedPlan()` 会用当前 `boundParams`。
custom plan 的生命周期通常绑定到本次调用。
它仍然需要 `RevalidateCachedQuery()` 先保证 query tree 正确。
但它不需要在 saved list 中等待未来 invalidation。
如果同一个 prepared statement 多次执行且参数分布差异很大，custom plan 可能不断重建。
这不是 invalidation。
这是 generic/custom 策略选择。
诊断时必须把这两者分开。
### 5.12. simple validity check
`CachedPlanAllowsSimpleValidityCheck()` 和 `CachedPlanIsSimplyValid()` 是热路径优化。
某些调用者已经持有一个 `CachedPlan` 指针，希望下一次快速判断是否还能复用。
simple check 能省掉完整 `GetCachedPlan()` 的一部分成本。
但它有严格前提：
```text
plan source 和 plan 必须都适合 simple validity check
search_path 必须匹配
plan source / plan 的 is_valid 必须仍为 true
```
如果前提不满足，必须退回完整 revalidation。
这里的设计信号是：PostgreSQL 愿意为 hot path 提供快速检查，但不会让快速检查绕过语义环境匹配。
## 6. 生命周期 / ownership / cleanup
### 6.1. 谁创建
`CreateCachedPlan()` 创建 `CachedPlanSource`。
创建者通常是 tcop 层处理 `Parse` message、SQL-level `PREPARE` 或 SPI 相关入口。
one-shot path 也可以创建不保存的 plan source。
创建时的 memory context 决定了它是否能跨消息、跨事务或只活到当前调用结束。
### 6.2. 谁补全
`CompleteCachedPlan()` 补全 analyzed/rewrite query tree、dependency、result descriptor、search_path matcher 和 RLS 标记。
补全后，`CachedPlanSource` 才能被 `GetCachedPlan()` 用来生成 `CachedPlan`。
一个没有 complete 的 plan source 只是半成品。
### 6.3. 谁保存
`SaveCachedPlan()` 把 plan source 放到长期 context 和 saved list。
SQL-level prepared statement、extended query named statement、PL/pgSQL 或 SPI 的某些缓存路径都可能间接依赖这个机制。
saved list 是 backend-local。
这决定了 invalidation callback 的扫描范围：
```text
只扫描当前 backend 保存过的 plan source；
不会也不能直接扫描其他 backend 内存。
```
### 6.4. 谁持有正在使用的 plan
`GetCachedPlan()` 返回 `CachedPlan *`。
调用者通常通过 resource owner 或显式 release 机制持有引用。
`CachedPlan.refcount` 防止 plan 正在执行时被释放。
注意：
```text
refcount 保护内存生命周期；
is_valid 保护语义新鲜度；
二者不是同一个维度。
```
一个正在执行的 plan 即使之后收到 invalidation，也通常不会被中途拆掉。
它会完成当前执行，下一次执行前再 revalidate。
### 6.5. 谁释放
`DropCachedPlan()` 释放 `CachedPlanSource`。
对 named prepared statement，`DEALLOCATE` 或 session 结束会触发释放。
对 unnamed statement，新一次 unnamed Parse 或 simple query 可能先 drop 旧 unnamed state。
generic plan 通过 `ReleaseGenericPlan()` 释放或解除挂接。
custom plan 在调用者释放引用后随其 context 回收。
### 6.6. ERROR / abort 时谁兜底
ERROR 不应该让 plan cache 保留半完成状态。
`RevalidateCachedQuery()` 里重新 analyze/rewrite 前，会先把相关状态标 invalid，确保失败后不会继续使用旧的错误中间状态。
内存由 MemoryContext 层兜底。
正在使用的 plan 引用由 ResourceOwner 或调用者释放路径兜底。
事务 abort 会处理本事务内登记的 invalidation state。
`AtEOXact_Inval(false)` 会按 abort 语义处理本地 invalidation，避免未提交 catalog 变化污染其他 backend。
### 6.7. 长期对象如何失效
长期对象不靠生命周期自然结束来维持正确性。
它靠四类机制被标记：
```text
relcache callback
syscache callback
search_path / role / RLS environment check
explicit ResetPlanCache()
```
标记 invalid 后，对象可以继续占内存。
这是 lazy rebuild 的代价。
它减少 DDL 提交时的同步工作，但把重建成本推到下一次使用这个 plan 的 backend。
## 7. 正确性机制层次
### 7.1. invalidation 不是 lock
DDL 需要自己的 lock 规则。
plan cache invalidation 不提供互斥。
它只告诉其他 backend：
```text
你之前缓存的某些 catalog-derived 语义可能过期了。
下次使用前请重新确认。
```
如果把 invalidation 当成 lock，就会误判两个问题。
第一，为什么 DDL 提交后 prepared statement 还能在某些 backend 短时间看似没变化。
第二，为什么正在执行的 plan 不会被强制中断。
### 7.2. dependency extraction 保证精确性上限
plan cache callback 的精确程度取决于 dependency list。
如果 relation OID 或 cache item 被记录，callback 可以只打掉相关 plan。
如果变化类型太宽或难以精确匹配，就只能 `ResetPlanCache()`。
因此正确性优先级是：
```text
宁可过度失效并多重建一次；
不能漏失效并执行旧语义。
```
### 7.3. search_path matcher 保护名称解析
unqualified name binding 是 parse analysis 的产物。
一旦 effective search path 改变，旧 query tree 里保存的 OID 可能不再对应用户此刻期望的对象。
matcher 不是为了性能。
它是为了让 prepared statement 的跨执行复用不破坏 SQL 名称解析语义。
### 7.4. result descriptor 保护客户端契约
prepared statement 的结果列形状是对外契约。
当 revalidation 发现 result descriptor 变化时，不能默默替换。
否则客户端可能按旧列数、旧类型或旧 format code 解析结果。
因此 `cached plan must not change result type` 是 correctness error，不是缓存实现泄漏。
### 7.5. relcache/syscache 分层保护对象语义
relcache 管 relation descriptor。
syscache 管 catalog tuple cache。
plan cache 不直接读取所有 catalog table 的更新细节。
它依赖这两层把“对象语义变化”转成 callback。
这保持了模块边界：
```text
catalog update path 负责登记 invalidation；
inval.c 负责传播；
relcache/syscache 负责本地 cache 刷新；
plancache.c 负责 cached query/plan 失效。
```
### 7.6. MemoryContext 和 ResourceOwner 只管生命周期
MemoryContext 删除能释放内存。
ResourceOwner 能在 abort 时释放引用。
它们不能判断 cached plan 语义是否仍正确。
所以不能用“对象还没释放”推导“对象还能执行”。
必须同时看 `is_valid` 和环境 matcher。
### 7.7. snapshot 不是主要保护机制
构建计划可能需要 snapshot 读取 catalog。
但 snapshot 只决定读到哪个 catalog 版本。
它不保证跨执行缓存永远正确。
跨执行正确性仍要靠 invalidation 和 revalidation。
这解释了为什么 plan cache code 同时关心 snapshot 和 invalidation。
它们服务不同层次。
## 8. 错误路径 / 异常路径 / fallback
### 8.1. DDL 改变 result type
实验中最容易看到的是：
```sql
PREPARE p AS SELECT * FROM t;
ALTER TABLE t ADD COLUMN c int;
EXECUTE p;
```
如果 prepared statement 的 result descriptor 因 `SELECT *` 展开而改变，revalidation 不能静默继续。
错误路径是：
```text
relcache invalidation marks plansource invalid
  -> next EXECUTE calls RevalidateCachedQuery()
  -> query tree/resultDesc recomputed
  -> resultDesc differs from old contract
  -> ERROR: cached plan must not change result type
```
这个错误比“自动返回新列”更安全。
prepared statement 的客户端可能已经基于旧 RowDescription 做了解码准备。
### 8.2. DROP/CREATE 同名对象
另一类常见边界是：
```sql
PREPARE p AS SELECT * FROM t;
DROP TABLE t;
CREATE TABLE t(...);
EXECUTE p;
```
旧 query tree 里保存的是旧 relation OID。
DROP 会发送 relcache invalidation。
CREATE 同名对象不等于旧对象复活。
下一次执行必须重新 parse/analyze，让名称解析绑定到当前 search path 下的新 OID，或报告对象不存在/契约变化。
这说明 plan cache 不能只比较 SQL 文本。
对象身份是 OID 和 catalog state，不是名字字符串。
### 8.3. search_path 改变但没有 DDL
只执行：
```sql
SET search_path = s2, public;
EXECUTE p;
```
也可能让 plan source invalid。
这不是 sinval 驱动。
这是 `RevalidateCachedQuery()` 中的 environment matcher 驱动。
fallback 是重新 analyze/rewrite。
如果新 path 下 unqualified name 解析到另一个对象，可能得到新计划，或因为 result type 改变而报错。
### 8.4. namespace/operator 等粗粒度 syscache 变化
某些 syscache callback 直接调用 `ResetPlanCache()`。
这会造成看似“无关 SQL 也重新规划”的现象。
这是正确性换性能的保守路径。
当依赖关系很难低成本精确判断时，PostgreSQL 选择扩大失效范围。
研发诊断时要避免把这种过度失效误判成内存泄漏或 prepared statement 丢失。
它只是下次执行要重建。
### 8.5. generic plan invalid，custom plan 可继续新建
如果 generic plan 被打掉，`CachedPlanSource` 仍可能有效。
下一次执行可能选择 custom plan，直接用当前参数 build。
这时你会看到 planner 成本，但不一定看到 query rewrite 成本。
这类现象在 profiling 上会表现为 planner 栈出现，但 parse analysis/rewrite 栈不明显。
### 8.6. invalidation queue pressure
shared invalidation 依赖消息队列。
如果 backend 很久不接收 invalidation，队列压力可能导致更粗的 cache reset。
从 plan cache 视角看，结果是更多 cached state 被标 invalid。
这仍然保持 correctness。
代价是下一次访问 cache 时重建范围变大。
### 8.7. ERROR 中断重建
如果 `RevalidateCachedQuery()` 重建 query tree 途中 ERROR，例如对象不存在、权限变化、RLS policy 导致 rewrite 失败，旧状态不能被当作 fallback 继续执行。
正确 fallback 是把 statement 留在 invalid 状态，并把 ERROR 返回给调用者。
下次执行可以再次尝试 revalidation。
这避免了“重建失败就偷偷执行旧计划”的严重错误。
## 9. 成本、资源与跨模块传播
### 9.1. 热路径成本
prepared statement 正常命中时，热路径希望接近：
```text
GetCachedPlan()
  -> cheap validity checks
  -> refcount existing generic plan
  -> executor consumes PlannedStmt
```
这条路径的成本主要是少量分支、matcher 检查、list 状态检查和 refcount/ResourceOwner 操作。
如果 generic plan 可用，避免了 parse/analyze/rewrite/planner 的大部分 CPU。
### 9.2. invalidation callback 成本
callback 收到消息时会扫描当前 backend 的 saved plan list。
成本大致随：
```text
当前 backend 保存的 prepared statement 数量
每个 statement dependency list 长度
invalidation 消息数量
```
增长。
这不是集群全局扫描。
每个 backend 只扫描自己的 list。
但在连接池大量 session 每个都缓存很多 prepared statement 时，同一个 DDL 会把重建成本分散到很多 backend 的后续执行上。
### 9.3. revalidation 成本
`plansource->is_valid = false` 后，下次执行要重新 analyze/rewrite。
成本随 SQL 复杂度、RTE 数、依赖对象数、rewrite rule/RLS policy、权限检查和 catalog cache miss 增长。
如果 search_path 很长，名称解析相关成本也会增加。
这类成本通常出现在执行前，用户可能只看到 EXECUTE 延迟变高。
### 9.4. replanning 成本
generic plan invalid 后需要 planner。
planner 成本随 relation 数、join 数、partition 数、统计信息访问、path 构造数量增长。
对大 join 或大分区表，generic plan 重建可能比 executor 执行小结果集还贵。
这解释了为什么 DDL 后第一次执行常常是慢的。
### 9.5. relcache/syscache 传播
跨模块传播链路是：
```text
catalog tuple update
  -> syscache/relcache invalidation registration
  -> shared invalidation message
  -> receiving backend local cache invalidation
  -> plan cache callback
  -> next GetCachedPlan() revalidation
```
任何一层都不是“计划重建”本身。
计划重建只发生在最后一层被使用时。
### 9.6. GUC 影响边界
需要区分四类 GUC 或 GUC-like planner environment。
第一类是 plan cache 自己的策略 GUC，例如 `plan_cache_mode`。
它影响 `choose_custom_plan()` 是倾向 generic、custom 还是自动。
第二类是 planner cost/enable GUC。
它们影响新计划形状，但不一定主动打掉已有 generic plan。
第三类是 search_path 这类名称解析环境 GUC。
它通过 matcher 直接影响 `CachedPlanSource` 是否还能复用 query tree。
第四类是 `row_security` 这类 rewrite/RLS 环境。
只有当 statement `dependsOnRLS` 时，role 或 `row_security` 变化才会让 query tree revalidation 走慢路径。
把所有 GUC 都归为“会 invalid cached plan”是不准确的。
### 9.7. relcache init file 与 plan cache
relcache 有自己的 init file 和 nailed relation 处理。
这些是 relcache 启动和刷新成本优化。
plan cache 不直接管理 relcache init file。
当 relcache invalidation 表明 relation descriptor 语义变化，plan cache 只关心自己的 dependency 是否命中。
这是模块边界。
### 9.8. 后台进程参与程度
本节主要是 backend-local plan cache。
没有一个专门后台进程负责重建所有 cached plan。
相关后台或辅助路径只间接参与：
```text
postmaster/backend 负责 session 生命周期
autovacuum/DDL backend 可能产生 catalog/relcache invalidation
checkpointer/walwriter 负责 catalog 变化的持久化相关 WAL/I/O
receiving backend 在安全点接收 sinval
```
plan cache 的 invalidation 和 rebuild 仍发生在使用它的 backend 内。
### 9.9. 资源压力的形态
资源压力通常不是 shared memory，而是 backend-local memory 和 CPU。
大量 saved prepared statements 会占用 `CacheMemoryContext` 子 context。
大量 invalidation 后，旧 generic plan 可能等待引用释放。
重建 query tree 和 plan 会制造短生命周期内存峰值。
诊断时应看 `pg_backend_memory_contexts`、perf 栈和执行前延迟，而不是只看 shared_buffers 或 WAL。
## 10. 观测与诊断入口
### 10.1. 能直接看到什么
SQL 层能直接看到 prepared statement：
```sql
SELECT name, statement, prepare_time, parameter_types, generic_plans, custom_plans
FROM pg_prepared_statements;
```
这里的 `generic_plans` 和 `custom_plans` 能帮助判断策略选择。
它不能直接告诉你某次 generic plan 为什么 invalid。
它也不能列出 `relationOids` 或 `invalItems`。
### 10.2. 能通过错误看到什么
最典型错误是：
```text
ERROR: cached plan must not change result type
```
这个错误说明 revalidation 已经发生，并且新旧 result descriptor 不兼容。
它不是“没有收到 invalidation”。
恰恰相反，它通常说明 invalidation 或 environment mismatch 把 statement 推到了重建路径。
### 10.3. 能通过计数推断什么
`pg_prepared_statements.generic_plans` / `custom_plans` 增长可以推断：
```text
是否一直在使用 custom plan
是否开始生成 generic plan
DDL/search_path 后是否又出现新的 plan build
```
但这些计数不是 invalidation 计数。
如果 generic plan invalid 后重新生成，计数变化能提示发生了 rebuild。
如果 custom plan 本来每次都生成，计数增长不能说明发生了 invalidation。
### 10.4. server log
可以临时使用：
```sql
SET log_min_duration_statement = 0;
SET log_statement = 'ddl';
```
结合 DDL 时间点和 EXECUTE 延迟，观察 DDL 后第一次执行是否明显变慢。
也可以打开 planner 相关日志或统计 GUC 做实验，但线上要谨慎。
这些日志粒度通常是 statement，不是 plan cache entry。
### 10.5. `pg_stat_statements`
`pg_stat_statements` 可以看到 mean/min/max 时间变化。
它适合发现“DDL 后第一次 EXECUTE 慢”或“某类 prepared statement 执行时间有尖峰”。
它不能区分 parse/rewrite/planner/executor 时间。
如果要拆分，需要 perf、gdb、tracepoint 或临时源码日志。
### 10.6. `pg_backend_memory_contexts`
prepared statement 和 generic plan 占 backend-local memory。
可以在同一 session 中观察：
```sql
SELECT name, total_bytes, used_bytes
FROM pg_backend_memory_contexts
WHERE name LIKE '%CachedPlan%' OR name LIKE '%Cached%'
ORDER BY total_bytes DESC;
```
不同版本 context 名称会变化。
这个视图能提示缓存对象存在和大致占用。
它不能告诉你某个 plan 是否 valid。
### 10.7. gdb 断点
源码跟读时可以在这些函数下断点：
```text
RevalidateCachedQuery
CheckCachedPlan
BuildCachedPlan
PlanCacheRelCallback
PlanCacheObjectCallback
PlanCacheSysCallback
ResetPlanCache
SearchPathMatchesCurrentEnvironment
```
观察变量：
```text
plansource->is_valid
plansource->gplan
plansource->gplan->is_valid
plansource->relationOids
plansource->invalItems
plansource->search_path
plansource->dependsOnRLS
plansource->rewriteRowSecurity
plansource->gplan->saved_xmin
```
这个实验比看 SQL 结果更直接。
因为 SQL 层没有暴露这些字段。
### 10.8. perf / flamegraph
如果怀疑重建成本，可以抓 EXECUTE 的 CPU 栈。
关注是否出现：
```text
RevalidateCachedQuery
pg_analyze_and_rewrite_withcb
QueryRewrite
planner
BuildCachedPlan
SearchPathMatchesCurrentEnvironment
```
如果只看到 executor 节点栈，慢点不在 plan cache。
如果看到 planner 栈但没看到 rewrite 栈，可能只是 generic/custom plan 重建。
如果看到 rewrite/analyze 栈，可能是 `CachedPlanSource` 失效。
### 10.9. 哪些状态看不到
普通 SQL 看不到：
```text
plansource->is_valid
gplan->is_valid
relationOids
invalItems
saved_plan_list 长度
sinval 消息具体命中哪个 plan
search_path matcher generation
```
这些只能通过源码日志、debugger、扩展或间接现象推断。
因此线上诊断要避免过度确定。
你可以说“现象符合 plan source revalidation”，但不能只靠 `pg_prepared_statements` 证明某个 relcache callback 命中了哪个 entry。
## 11. 常见误区
### 11.1. 误区：prepared statement 一定不再规划
prepared statement 只是给复用提供可能。
它仍然可能因为 invalidation、search_path、RLS/role 或 generic/custom 策略重新规划。
对参数敏感语句，custom plan 甚至可能长期每次规划。
### 11.2. 误区：invalidation 会立刻重建所有计划
invalidation callback 只标 invalid。
重建发生在下一次使用。
这就是 DDL 提交很快，但某个业务请求第一次访问旧 prepared statement 时变慢的原因。
### 11.3. 误区：SET search_path 只是字符串变化
plan cache 关心 effective path。
权限、temp schema、`pg_catalog` 规则和 schema OID 都影响 matcher。
因此 search_path 的语义不能用 GUC 文本直接判断。
### 11.4. 误区：只要 refcount 大于 0，plan 就有效
refcount 只说明内存不能释放。
它不说明语义没有过期。
正在执行的 plan 和下一次执行前的 plan validity 是两个问题。
### 11.5. 误区：所有 GUC 都会 invalid cached plan
`plan_cache_mode` 影响选择策略。
planner GUC 影响新计划。
search_path 影响名称解析 revalidation。
这些路径不同。
不要把 GUC 变化统一解释成 relcache invalidation。
### 11.6. 误区：`cached plan must not change result type` 是缓存 bug
多数情况下这是正确性保护。
旧 prepared statement 对客户端暴露过结果形状。
DDL 或 search_path 让新结果形状改变时，静默继续会更危险。
### 11.7. 误区：plan cache 是 shared cache
plan cache 是 backend-local。
shared invalidation message 是跨 backend 的通知机制。
通知到达后，每个 backend 修改自己的本地 cache 状态。
## 12. 课堂实验
### 12.1. 实验一：观察 result type 改变
目标：看到 relcache invalidation 让 prepared statement 进入 revalidation，并因结果契约变化报错。
步骤：
```sql
CREATE TABLE pc_inv_t(a int);
INSERT INTO pc_inv_t VALUES (1);
PREPARE pc_inv_p AS SELECT * FROM pc_inv_t;
EXECUTE pc_inv_p;
ALTER TABLE pc_inv_t ADD COLUMN b text;
EXECUTE pc_inv_p;
```
预期：
```text
第二次 EXECUTE 报 cached plan must not change result type。
```
回到源码解释：
```text
ALTER TABLE
  -> relcache invalidation
  -> PlanCacheRelCallback marks plansource invalid
  -> RevalidateCachedQuery recomputes resultDesc
  -> resultDesc differs
  -> ERROR
```
扩展练习：把 `SELECT *` 改成 `SELECT a`。
如果结果 descriptor 不变，下一次 EXECUTE 更可能成功，但计划仍可能被重建。
### 12.2. 实验二：search_path 让同名对象重绑定
目标：区分 search_path mismatch 和 relcache invalidation。
步骤：
```sql
CREATE SCHEMA pc_s1;
CREATE SCHEMA pc_s2;
CREATE TABLE pc_s1.t(a int);
CREATE TABLE pc_s2.t(a int, b int);
SET search_path = pc_s1;
PREPARE pc_sp AS SELECT * FROM t;
EXECUTE pc_sp;
SET search_path = pc_s2;
EXECUTE pc_sp;
```
预期：
```text
第二次 EXECUTE 可能因结果类型变化报错。
如果两个表结果形状相同，则可能成功但绑定到新 search_path 语义。
```
回到源码解释：
```text
SET search_path
  -> namespace active path generation changes
  -> RevalidateCachedQuery calls SearchPathMatchesCurrentEnvironment
  -> mismatch marks query tree invalid
  -> re-analyze resolves t again
```
扩展练习：让两个 schema 下的 `t` 列形状完全一样，再用 `EXPLAIN EXECUTE` 比较计划 relation OID。
### 12.3. 实验三：generic/custom 计数
目标：不要把 custom plan 每次重建误判成 invalidation。
步骤：
```sql
CREATE TABLE pc_param_t(a int, b text);
INSERT INTO pc_param_t
SELECT g, md5(g::text)
FROM generate_series(1, 10000) g;
CREATE INDEX ON pc_param_t(a);
PREPARE pc_param_p(int) AS SELECT * FROM pc_param_t WHERE a = $1;
EXECUTE pc_param_p(1);
EXECUTE pc_param_p(2);
EXECUTE pc_param_p(3);
SELECT name, generic_plans, custom_plans
FROM pg_prepared_statements
WHERE name = 'pc_param_p';
```
然后尝试：
```sql
SET plan_cache_mode = force_generic_plan;
EXECUTE pc_param_p(4);
SELECT name, generic_plans, custom_plans
FROM pg_prepared_statements
WHERE name = 'pc_param_p';
```
观察点：
```text
custom_plans 增长不等于 invalidation。
plan_cache_mode 改变策略，但不等同于 relcache callback。
```
源码对应：
```text
GetCachedPlan
  -> choose_custom_plan
  -> BuildCachedPlan custom or generic
```
### 12.4. 实验四：gdb 跟踪 relcache callback
目标：从 DDL 到 callback 再到 revalidation 画出完整时间线。
建议断点：
```text
PlanCacheRelCallback
RevalidateCachedQuery
CheckCachedPlan
BuildCachedPlan
```
操作：
```text
session A: PREPARE p AS SELECT a FROM pc_inv_t;
session B: ALTER TABLE pc_inv_t ALTER COLUMN a TYPE bigint;
session A: EXECUTE p;
```
观察：
```text
PlanCacheRelCallback 是否在 session A 接收 invalidation 时触发。
plansource->is_valid 何时从 true 变 false。
RevalidateCachedQuery 是否重新生成 query_list。
BuildCachedPlan 是否重新规划。
```
注意：不同安全点接收 invalidation 的时机会有差异。
不要假设 DDL commit 后 session A 立刻停在 callback。
### 12.5. 实验五：源码加日志
目标：建立轻量可观测性。
只在本地实验分支修改 `plancache.c`，不要用于生产。
建议在这些位置打 DEBUG 日志：
```text
PlanCacheRelCallback: relid, hit plansource count, hit gplan count
PlanCacheSysCallback: cacheid, ResetPlanCache count
RevalidateCachedQuery: search_path mismatch, role mismatch, resultDesc change
CheckCachedPlan: gplan invalid reason
```
日志要避免打印完整 SQL 或用户参数，防止泄漏敏感信息。
这个实验能让你看到 SQL 层看不到的 invalidation 粒度。
## 13. 讨论题
1. 为什么 `CachedPlanSource` 和 `CachedPlan` 都需要 `is_valid`，只保留一个 flag 会丢失什么语义？
2. 为什么 invalidation callback 只标 invalid，而不是在 DDL commit 时替所有 backend 重建计划？
3. `search_path` 改变为什么可能让 query tree 失效，即使没有任何 relation DDL？
4. `plan_cache_mode = force_custom_plan` 带来的 planner CPU，和 relcache invalidation 导致的 replanning，如何从现象上区分？
5. 为什么 `cached plan must not change result type` 是正确性保护，而不是用户体验上的多余错误？
6. 如果一个 backend 缓存了十万个 prepared statements，DDL 后 plan cache callback 成本会怎样扩张？这个成本为什么仍然不是集群全局同步扫描？
7. 为什么 refcount 不能说明 cached plan 语义有效？它到底保护了什么？
8. 对 extension 作者来说，哪些 plan cache 内部结构不能当 public ABI 依赖？如果需要缓存 catalog-derived 语义，应模仿哪类 invalidation 边界？
## 14. 本节小结
本节唯一主问题是：
```text
catalog invalidation、search_path、GUC 和 relcache 变化如何让 cached plan 失效或重建？
```
核心链路是：
```text
CachedPlanSource 保存 raw/analyzed/rewrite state
  -> CachedPlan 保存 generic PlannedStmt
  -> relcache/syscache/search_path/role/GUC environment 改变
  -> callback 或 execution-time matcher 标 invalid
  -> GetCachedPlan 在下一次执行前 revalidate
  -> 复用、重建 generic plan、重建 custom plan，或报 result type 错误
```
核心状态边界是：
```text
CachedPlanSource.is_valid 保护 query tree 语义
CachedPlan.is_valid 保护 planned statement 语义
SearchPathMatcher 保护 unqualified name binding
dependency list 连接 catalog/relcache/syscache invalidation
refcount 只保护内存生命周期
```
ownership 上，plan cache 是 backend-local。
saved prepared statement 进入当前 backend 的 saved list。
shared invalidation 只是跨 backend 传递“语义可能过期”的消息。
真正的标记、重建和释放都发生在接收 backend 的本地内存中。
异常路径上，PostgreSQL 宁可过度失效和下一次执行变慢，也不能漏掉过期语义。
如果 revalidation 后 result descriptor 改变，就报 `cached plan must not change result type`，保护客户端结果契约。
如果 search_path 或 role/RLS 环境改变，就重新 analyze/rewrite。
如果 generic plan 无效，就重新规划；如果策略选择 custom plan，则本次按参数重新规划。
可观测性上，SQL 层能看到 `pg_prepared_statements` 的 generic/custom 计数和典型错误。
它看不到 `plansource->is_valid`、`gplan->is_valid`、dependency list 或 callback 命中。
要做源码级诊断，需要 gdb、perf、临时 DEBUG 日志或内存上下文视图配合。
本节沉淀的可迁移规律是：
```text
长期缓存的正确性不靠缓存对象自己证明；
它靠依赖记录、环境摘要、异步 invalidation、执行前 revalidation 和保守 fallback 共同维持。
```
这个规律会继续出现在 relcache、typcache、catcache、PL/pgSQL plan cache、扩展自建 cache 和 executor runtime state 中。
下一节如果继续追 Portal 生命周期，需要带着这个边界看：
```text
Portal 拿到的是某一刻通过 GetCachedPlan() 验证后的 CachedPlan；
Portal 的暂停、恢复和 cleanup 不能替代 plan cache 的语义 revalidation。
```
