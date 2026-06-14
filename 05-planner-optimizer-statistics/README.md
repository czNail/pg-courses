# Planner / Optimizer / Statistics

本目录当前 68 节，包含 15 个优化器主题组；parser/analyzer/rewrite、ProcessUtility/DDL 和 planner 侧 partitioning 课程已经补齐 planner 之前和 DDL 侧入口。

课程安排：

0. Parser / Analyzer / Rewrite：SQL 如何先变成语义化 `Query`。
1. Planner 主流程与 Query / Plan 边界。
2. Query preprocessing 与 RestrictInfo。
3. ANALYZE 与 pg_statistic。
4. Extended Statistics。
5. Selectivity 选择率估算。
6. Cost Model。
7. RelOptInfo / Path / PathTarget。
8. Base relation path。
9. Join search 与 join path。
10. Upper planning：aggregate、sort、limit。
11. Path 到 Plan：createplan.c。
12. 慢 SQL Planner 诊断方法。
13. ProcessUtility / DDL / event trigger。
14. Partitioning planner side。

第 1 项 `Planner 主流程与 Query / Plan 边界` 建议先拆成 4 个独立主题，后续生成课程时按“一个主问题一课”处理：

1. [Planner 入口与优化器阶段边界](01-planner-entry-phase-boundaries.md)：一条已经 parse / analyze / rewrite 完成的 SQL 进入 `planner()` 后，为什么还要经过 `standard_planner()`、`subquery_planner()`、`grouping_planner()` 和 `query_planner()` 多层分工，哪些阶段负责语义保持，哪些阶段负责搜索计划空间？
2. [Query 树到 PlannedStmt 的结构转换](02-query-to-plannedstmt-boundary.md)：`Query`、`PlannerInfo`、`Plan` 和 `PlannedStmt` 分别保存什么状态，为什么 optimizer 不能直接在 `Query` 上生成可执行节点，而要先构造 planner-local 的搜索上下文？
3. [PlannerInfo 与全局优化上下文](03-plannerinfo-global-context.md)：`PlannerInfo` 为什么集中保存 rangetable、equivalence class、append relation、placeholder、join info、upper rel 等 planner 状态，它如何避免把优化过程散落到各个语法节点上？
4. [Hook、GUC 与 planner 可插拔边界](04-planner-hooks-guc-boundary.md)：`planner_hook`、`set_rel_pathlist_hook`、`set_join_pathlist_hook`、`enable_*` GUC 和 cost 参数能改变哪些决策，为什么它们只能影响搜索空间和代价，不能破坏 Query / Plan 的语义边界？

第 2 项 `Query preprocessing 与 RestrictInfo` 建议先拆成 5 个独立主题，后续生成课程时按“一个主问题一课”处理：

1. [表达式 canonicalize 与隐式 AND 拆分](05-preprocess-expression-canonicalize.md)：为什么 planner 要先把 where / join qual / having 等表达式做常量折叠、函数内联、布尔表达式规范化和 AND clause 拆分，后续选择率、索引匹配和等价类推导依赖哪些形态？
2. [子查询 pullup 与 jointree 改写](06-subquery-pullup-jointree.md)：哪些 simple subquery 可以被拉平到上层 join tree，为什么 security barrier、聚合、limit、volatile 表达式和 lateral 引用会阻止这种改写？
3. [外连接约束与 join tree 语义保护](07-outer-join-reduction-constraints.md)：planner 为什么要记录 outer join 的 nullable side、join order 约束和可消除条件，哪些 predicate 能把 outer join 简化为 inner join，哪些不能跨越 null-extension 边界？
4. [RestrictInfo 的包装意义](08-restrictinfo-clause-metadata.md)：为什么一个普通 qual 进入 optimizer 后要包装成 `RestrictInfo`，它如何携带 required relids、nullable relids、pseudoconstant、leakproof、security level、mergejoinable / hashjoinable 等元数据？
5. [Predicate 下推、延迟执行与 join qual 分类](09-restrictinfo-pushdown-delay.md)：同一个表达式为什么有时是 base restriction，有时是 join qual，有时必须 delay 到 outer join 之后执行，`RestrictInfo` 如何决定它能下推到哪些 relation 或 joinrel？

第 3 项 `ANALYZE 与 pg_statistic` 建议先拆成 5 个独立主题，后续生成课程时按“一个主问题一课”处理：

1. [ANALYZE 采样与统计目标](10-analyze-sampling-statistics-target.md)：`ANALYZE` 为什么只采样而不全表扫描，采样行数、`default_statistics_target`、列级统计目标和表大小如何共同决定统计信息的精度与成本？
2. [typanalyze 与类型相关统计](11-analyze-typanalyze-type-specific.md)：不同数据类型为什么需要 `typanalyze` 定制统计逻辑，标量、数组、范围、tsvector 等类型如何决定 MCV、histogram、correlation 和其它 slot 的含义？
3. [pg_statistic 存储模型](12-pg-statistic-storage-model.md)：`pg_statistic` 为什么用 kind / operator / collation / numbers / values 多 slot 结构保存列统计，planner 如何按列、类型和操作符族取回可用于估算的信息？
4. [MCV、histogram、nullfrac 与 ndistinct](13-column-stats-mcv-histogram-ndistinct.md)：最常见值列表、直方图、空值比例和 distinct 估算分别回答什么问题，它们在等值、范围、不等值和 IS NULL 估算中如何组合？
5. [Correlation 与物理顺序成本影响](14-column-stats-correlation-cost.md)：列值与 heap 物理顺序的相关性为什么会影响 index scan 成本，`pg_stats.correlation` 如何把同样的选择率转化成不同的随机 I/O 预期？

第 4 项 `Extended Statistics` 建议先拆成 4 个独立主题，后续生成课程时按“一个主问题一课”处理：

1. [扩展统计对象与表达式统计](15-extended-statistics-object-expression.md)：为什么单列统计无法描述多列相关性和表达式分布，`CREATE STATISTICS` 如何把列组、表达式、schema 对象和 ANALYZE 采集流程连接起来？
2. [Dependencies 与条件独立性修正](16-extended-stats-dependencies.md)：当 `a = ? AND b = ?` 中两列并不独立时，functional dependencies 如何修正多个 filter 选择率相乘造成的低估或高估？
3. [Multivariate MCV 与组合值分布](17-extended-stats-multivariate-mcv.md)：多列 MCV 为什么能直接描述常见组合值，planner 如何在多个 clause 同时命中时优先使用组合频率，而不是把单列频率机械相乘？
4. [ndistinct 扩展统计与 group 估算](18-extended-stats-ndistinct-grouping.md)：多列 distinct 数为什么对 GROUP BY、聚合和 join cardinality 重要，扩展 ndistinct 如何避免把组合基数估得过大或过小？

第 5 项 `Selectivity 选择率估算` 建议先拆成 5 个独立主题，后续生成课程时按“一个主问题一课”处理：

1. [选择率 API 与默认兜底](19-selectivity-api-defaults.md)：`restriction_selectivity()`、`join_selectivity()`、操作符 `oprrest` / `oprjoin` 和默认选择率如何协作，为什么缺失统计信息时 planner 必须使用可解释但粗糙的 fallback？
2. [等值、范围与 pattern 选择率](20-selectivity-equality-range-pattern.md)：`eqsel`、`scalarltsel`、`scalarineqsel`、LIKE / regex 估算如何使用 MCV、histogram、collation 和 prefix 信息，哪些模式会退化为默认估算？
3. [Clause 组合与独立性假设](21-selectivity-clause-combination.md)：多个 restriction clause 为什么默认按独立性相乘，OR / NOT / ScalarArrayOpExpr / NullTest 等表达式如何组合选择率，extended statistics 在哪里介入修正？
4. [Join 选择率与 join cardinality](22-selectivity-join-cardinality.md)：inner / outer / semi / anti join 的行数估算分别需要哪些输入，等值 join 如何使用 ndistinct、MCV 和 nullfrac，为什么 join 估错会迅速放大到后续 plan？
5. [Rows 估算传播与 clamp 规则](23-rows-estimation-propagation-clamp.md)：base rel、joinrel、appendrel 和 upper rel 的 rows 如何逐层传递，为什么 planner 要对极小、极大和不确定行数做 clamp，避免代价模型出现不可用结果？

第 6 项 `Cost Model` 建议先拆成 5 个独立主题，后续生成课程时按“一个主问题一课”处理：

1. [成本单位与 cpu / io 参数](24-cost-model-units-parameters.md)：`seq_page_cost`、`random_page_cost`、`cpu_tuple_cost`、`cpu_operator_cost`、parallel cost 等参数为什么只是相对单位，planner 如何用它们比较不同 Path 而不是预测真实耗时？
2. [SeqScan、IndexScan 与 BitmapScan 成本](25-cost-scan-paths.md)：顺序扫描、索引扫描、index-only scan、bitmap heap scan 的成本模型分别如何估算 heap page、index page、recheck、visibility map 和 tuple CPU 成本？
3. [Sort、Hash、Material 与内存溢出成本](26-cost-memory-spill-nodes.md)：`work_mem` 如何影响 sort、hash aggregate、hash join、materialize 的内存批次、磁盘 I/O 和 startup / total cost，为什么同一个计划在内存边界附近会突然变差？
4. [Nested Loop、Merge Join、Hash Join 成本](27-cost-join-algorithms.md)：三种 join 算法的 startup cost、rescan cost、排序需求、hash build/probe 和 outer rows 放大效应有什么不同，planner 为什么会在行数估错时选错 join 类型？
5. [Parallel Path 成本与 worker 数选择](28-cost-parallel-paths.md)：并行扫描和并行 join 为什么要估算 setup、tuple communication、leader participation 和 partial path 行数，`max_parallel_workers_per_gather` 等参数如何限制搜索空间？

第 7 项 `RelOptInfo / Path / PathTarget` 建议先拆成 4 个独立主题，后续生成课程时按“一个主问题一课”处理：

1. [RelOptInfo 作为搜索空间节点](29-reloptinfo-search-node.md)：为什么 optimizer 用 `RelOptInfo` 表示 base rel、joinrel、upper rel 和其它 relation-like 对象，`relids` 如何成为 join 搜索和 clause 归属的核心标识？
2. [Path 作为候选实现方式](30-path-candidate-implementation.md)：同一个 `RelOptInfo` 为什么会挂多个 `Path`，`startup_cost`、`total_cost`、`rows`、`pathkeys`、`param_info`、parallel awareness 等字段如何描述一种可比较的实现方式？
3. [PathTarget 与投影成本](31-pathtarget-projection-cost.md)：为什么 targetlist 在 optimizer 中要变成 `PathTarget`，表达式求值成本、宽度、sortgroupref 和 projection placement 如何影响 scan、join、aggregate 和 final plan？
4. [Parameterized Path 与 lateral / nestloop 依赖](32-parameterized-paths.md)：一个 Path 为什么可能依赖外层 relation 提供参数，parameterized path 如何支持 LATERAL、相关子查询和索引内层 nested loop，同时又限制 join order 的可交换性？

第 8 项 `Base relation path` 建议先拆成 4 个独立主题，后续生成课程时按“一个主问题一课”处理：

1. [set_base_rel_size 与 base rows 估算](33-base-rel-size-estimation.md)：planner 为什么先为每个 base relation 估算 rows、width、restriction cost 和可用统计信息，错误的 base cardinality 如何污染后续全部 path 选择？
2. [set_base_rel_pathlists 与 scan path 生成](34-base-rel-pathlist-generation.md)：一个普通表会生成哪些 seqscan、indexscan、bitmap、tidscan、sample scan、parallel scan 候选 Path，`add_path()` 如何按成本和 pathkeys 剪枝？
3. [Index path 匹配与 clause / order 利用](35-index-path-clause-order-matching.md)：planner 如何判断一个 RestrictInfo 能匹配 btree、hash、gin、gist、brin 等索引条件，为什么同一个索引既可能用于过滤，也可能用于输出顺序和 index-only scan？
4. [Partition、Append 与 Foreign / Custom Path](36-base-rel-special-paths.md)：分区表、继承表、FDW 和 custom scan 为什么会扩展 base rel path 生成过程，partition pruning、append path 和 foreign path 如何进入同一套成本比较框架？

第 9 项 `Join search 与 join path` 建议先拆成 5 个独立主题，后续生成课程时按“一个主问题一课”处理：

1. [Joinrel 构造与动态规划搜索](37-joinrel-dynamic-programming.md)：optimizer 为什么按 relids 集合逐层构造 joinrel，standard join search 如何在 join order 数量爆炸前尽量枚举有意义的连接组合？
2. [Join order 约束与合法性检查](38-join-order-legality.md)：outer join、semi / anti join、lateral、PlaceHolderVar 和 join removal 会给 join 顺序施加哪些限制，`join_is_legal()` 如何避免生成语义错误的 join tree？
3. [Nested Loop / Merge / Hash Join Path 生成](39-join-path-generation.md)：`add_paths_to_joinrel()` 如何根据 join type、join quals、pathkeys、hashability、mergejoinability 和 parameterization 生成三类 join path 候选？
4. [GEQO 与大连接查询搜索](40-geqo-large-join-search.md)：当 join relation 太多导致动态规划不可承受时，GEQO 为什么把 join order 搜索变成遗传算法问题，它牺牲了哪些确定性和可解释性？
5. [Join Path 剪枝、pathkeys 与 cheapest path](41-join-path-pruning-cheapest.md)：`add_path()` 为什么不能只保留最低 total cost 的 path，排序顺序、startup cost、parameterization、parallel safety 和 row ordering 如何让多个 path 同时有保留价值？

第 10 项 `Upper planning：aggregate、sort、limit` 建议先拆成 5 个独立主题，后续生成课程时按“一个主问题一课”处理：

1. [UpperRel 阶段划分](42-upperrel-stage-pipeline.md)：base / join search 结束后，为什么 planner 还要按 grouping、window、distinct、order、limit、modify table 等阶段构造 upper relation，而不是直接把节点接到 cheapest path 上？
2. [GroupAggregate、HashAggregate 与 Mixed Aggregate](43-upper-aggregate-paths.md)：GROUP BY 和 aggregate 为什么可能选择排序聚合、哈希聚合或混合策略，分组基数、pathkeys、work_mem、partial aggregation 和 rollup 如何影响路径选择？
3. [WindowAgg、Distinct 与 SetOp Path](44-upper-window-distinct-setop.md)：window function、DISTINCT、UNION / INTERSECT / EXCEPT 需要哪些排序、哈希或分组语义，planner 如何在复用已有 pathkeys 和新增排序之间取舍？
4. [Sort、Incremental Sort 与 Limit](45-upper-sort-incremental-limit.md)：ORDER BY / LIMIT 为什么会改变 startup cost 的重要性，已有 pathkeys、incremental sort、top-N heapsort 和 limit_tuples 如何让 planner 选择看似更贵但更快返回前几行的路径？
5. [Parallel upper planning 与 Gather](46-upper-parallel-gather.md)：partial path 如何经过 partial aggregate、parallel append、gather、gather merge 和 finalize aggregate 变成完整结果，哪些 upper 操作会阻断并行计划？

第 11 项 `Path 到 Plan：createplan.c` 建议先拆成 4 个独立主题，后续生成课程时按“一个主问题一课”处理：

1. [Best Path 选择与 create_plan 入口](47-createplan-bestpath-entry.md)：当每个 relation 已经有 cheapest path 后，`create_plan()` 如何从 final_rel 的 best path 递归生成可执行 Plan tree，哪些 Path 信息会被保留，哪些只是 planner 阶段的临时决策？
2. [Scan、Join、Agg Plan 节点生成](48-createplan-node-construction.md)：`create_scan_plan()`、`create_join_plan()`、`create_agg_plan()` 等函数如何把 Path 字段转成 executor 需要的 targetlist、qual、joinqual、hash clauses、sort keys 和子计划？
3. [Projection、resjunk 与表达式替换](49-createplan-projection-resjunk.md)：为什么最终 Plan 的 targetlist 不等于 SQL 输出列列表，resjunk 列、sortgroupref、Var 替换、PlaceHolderVar 和 projection 节点如何保证 executor 能拿到排序、分组和 join 所需的中间值？
4. [Plan finalization、initPlan 与 executor contract](50-createplan-finalize-executor-contract.md)：`set_plan_references()`、SubPlan / InitPlan、param id、range table index 和 plan node metadata 如何完成 planner 到 executor 的契约，使执行器不再依赖 optimizer 的搜索结构？

第 12 项 `慢 SQL Planner 诊断方法` 建议先拆成 5 个独立主题，后续生成课程时按“一个主问题一课”处理：

1. [EXPLAIN 中读出 planner 判断](51-diagnose-explain-planner-judgement.md)：如何从 `EXPLAIN (ANALYZE, BUFFERS)` 的 estimated rows、actual rows、loops、startup / total cost、width 和 chosen node 看出 planner 当时相信了什么？
2. [行数估错定位：统计信息还是谓词模型](52-diagnose-row-misestimation.md)：慢 SQL 中 rows 估错时，如何沿着 plan tree 找第一个失真节点，并判断问题来自 stale stats、低统计目标、列相关性、表达式统计缺失还是 selectivity fallback？
3. [成本参数与硬件现实偏差](53-diagnose-cost-parameter-mismatch.md)：当估算行数基本正确但计划仍然慢，如何检查 `random_page_cost`、`effective_cache_size`、`work_mem`、parallel cost、JIT cost 和缓存状态是否让 cost model 偏离实际硬件？
4. [索引、Join Order 与 Path 搜索诊断](54-diagnose-index-join-path-search.md)：如何判断 planner 没有使用索引、选错 join 算法或 join order 的原因，是 path 不合法、谓词不可下推、表达式不匹配、统计误导、enable GUC 剪枝还是搜索空间限制？
5. [可重复的 Planner 调试流程](55-diagnose-repeatable-workflow.md)：面对一条慢 SQL，如何按“收集 EXPLAIN、确认 schema / stats、定位首个估错点、验证替代 path、最小化 SQL、选择修复手段”的顺序形成可复现的优化器诊断报告？

## 插入阅读路径

以下课程保留在 README 中作为建议阅读路径；编号沿用追加编号，未重命名已有 01-55。

第 0 项 `Parser / Analyzer / Rewrite` 建议在阅读第 1 项 planner 入口之前先读；如果已经从 planner 开始，也应回补这组以理解 `Query` 从何而来：

1. [Raw Parser、Grammar 与 Node Tags](56-raw-parser-grammar-node-tags.md)：raw parser 如何从 SQL 文本生成 raw parse tree，gram.y、Node tag 和 location 如何服务后续错误定位？
2. [parse analysis 中 range table、namespace 与 name resolution](57-parse-analysis-name-resolution.md)：parse analysis 如何解析 range table、列名、namespace、scope 和 `ParseState`？
3. [类型转换、操作符与函数查找](58-type-coercion-operator-function-lookup.md)：类型推断、隐式 coercion、operator/function lookup 如何把语法表达式变成可执行语义？
4. [Query Rewrite / View / Rule](59-query-rewrite-view-rule.md)：view/rule rewrite 如何把用户 Query 改写成等价 Query，哪些边界会影响权限和可更新视图？
5. [RLS / security barrier rewrite 与 qual 下推边界](60-rls-security-barrier-rewrite.md)：RLS、security barrier view 和 leakproof function 如何约束 qual 下推和 planner 搜索空间？

第 13 项 `ProcessUtility / DDL / event trigger` 建议作为 planner hook / utility hook 边界的补课，放在 planner 主流程之后阅读：

1. [ProcessUtility / DDL dispatch](61-processutility-ddl-dispatch.md)：`ProcessUtility()` 如何区分 DDL、transaction command、VACUUM、COPY、EXPLAIN 和其它 utility statement？
2. [DDL command / catalog update](62-ddl-command-catalog-update.md)：CREATE/ALTER/DROP 如何修改 catalog、记录 dependency、发出 invalidation，并在事务 abort 时回滚？
3. [Event Trigger / ProcessUtility 边界](63-event-trigger-processutility-boundary.md)：event trigger、ProcessUtility hook 和 executor/planner hook 的边界在哪里？
4. [DDL locking / invalidation ordering](64-ddl-locking-invalidation-order.md)：DDL 为什么必须先拿对象锁，再改 catalog，再发 invalidation，错误顺序会破坏什么？

第 14 项 `Partitioning planner side` 建议插在第 8 项 Base relation path 和第 9 项 Join search 之间阅读：

1. [partition bound / catalog model](65-partition-bound-catalog-model.md)：分区表的 bound、partition key、partition descriptor 和 relcache 如何表达分区空间？
2. [planner-time partition pruning 与 Append 子路径裁剪](66-partition-pruning-planner.md)：planner-time partition pruning 如何根据约束、参数和表达式裁剪 Append 子路径？
3. [partition-wise join / aggregate](67-partitionwise-join-aggregate.md)：partition-wise join / aggregate 何时合法，为什么需要对齐 partition scheme 和 equivalence class？
4. [inheritance / partition expansion 与 appendrel Var translation](68-inheritance-expansion-appendrel.md)：inheritance/partition expansion 如何构造 appendrel、child rel 和 Var translation？
