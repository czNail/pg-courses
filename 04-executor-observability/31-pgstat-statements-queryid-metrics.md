# PostgreSQL pg_stat_statements 的 Query ID 与执行指标

## 课程定位

前置知识：已经理解 pg_stat 核心体系如何把 backend-local 统计刷新到共享状态。

本节唯一主问题：

```text
pg_stat_statements 如何用 query id 聚合同类 SQL 的执行次数、时间、行数、block I/O、WAL 和 JIT 指标，为什么它适合找热点但不能替代单次执行剖析？
```

核心矛盾：线上诊断需要跨执行聚合同一类语句，但 SQL 文本、常量、权限、嵌套语句、utility、prepared statement 和 query text 存储都会让“同一类查询”的边界复杂化。

学完后应能判断：能从 pg_stat_statements 的一行记录追到 JumbleQuery、EnableQueryId、executor hook、query_instr、pgss_store 和共享 hash entry。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

第 30 节讲的是核心 pg_stat。

本节转向 contrib 扩展 pg_stat_statements。

它不记录每个 PlanState 节点。

它记录 query id 维度的聚合事实。

它与 EXPLAIN 互补。

它先帮你找“哪类 SQL 值得单独 explain”。

本节刻意只处理一个问题：观测状态如何在执行器或统计体系中找到稳定边界。

凡是不能帮助解释这个边界的函数和字段，都只在后续课程或源码练习中出现。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
核心 parser/analyzer 为 Query 计算 queryId，pg_stat_statements 通过 EnableQueryId 和 executor hooks 请求 query-level instrumentation，ExecutorEnd 时把总时间、rows、buffers、WAL、JIT 等汇总交给 pgss_store，pgss_store 再按 userid、dbid、queryid、toplevel 更新共享 hash entry 和 query text 存储。
```

这句话里有三层含义。

第一层是入口边界：用户看到的 SQL 选项、统计视图或扩展 hook，必须先被转换成执行期状态。

第二层是状态边界：执行路径不能在每个热函数里直接做复杂诊断，只能在少数明确对象上累计事实。

第三层是解释边界：输出或视图只是读取这些事实，不能把估算、累计、平均、延迟刷新混成一个语义。

本节后面的所有源码 walkthrough 都围绕这三层展开。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 作用 |
| --- | --- | --- |
| 1 | `src/backend/nodes/queryjumblefuncs.c` | JumbleQuery、DoJumble、EnableQueryId 和 compute_query_id 处理 query id 生成。 |
| 2 | `src/include/nodes/queryjumble.h` | compute_query_id 枚举和 query jumble 公开接口。 |
| 3 | `src/backend/parser/analyze.c` | parse analysis 后为 Query 设置 queryId，并调用 pgstat_report_query_id。 |
| 4 | `src/backend/optimizer/plan/planner.c` | 把 parse->queryId 复制到 PlannedStmt。 |
| 5 | `src/backend/executor/execMain.c` | ExecutorStart 报告 query id，standard_ExecutorStart 分配 query_instr。 |
| 6 | `src/include/executor/execdesc.h` | QueryDesc 中 query_instr_options 和 query_instr 支持查询级观测。 |
| 7 | `src/backend/utils/activity/backend_status.c` | pgstat_report_query_id 把当前 query id 暴露给 backend status。 |
| 8 | `contrib/pg_stat_statements/pg_stat_statements.c` | pgss_ExecutorStart、pgss_ExecutorEnd、pgss_store、pg_stat_statements_internal 实现扩展核心。 |
| 9 | `contrib/pg_stat_statements/sql/select.sql` | 回归测试展示 query normalization、calls、rows 等行为。 |

阅读时不要按文件名背 API。

先找入口状态，再找运行期持有者，然后看 cleanup、fallback 和输出路径。

如果一个函数只是在表格中出现，而没有参与本节主问题，就不要把它扩展成新的主线。

## 4. 关键数据结构与状态边界

本节需要关注的不是单个字段名字，而是字段组合在生命周期中的语义。

### 4.1 `Query.queryId`

parse/analyze 阶段得到的规范化查询标识。

### 4.2 `PlannedStmt.queryId`

planner 输出中携带的 query id，executor 和 hook 读取它。

### 4.3 `QueryDesc.query_instr_options`

pg_stat_statements 请求整条语句 instrumentation 的位置。

### 4.4 `QueryDesc.query_instr`

ExecutorStart 分配的查询级 Instrumentation。

### 4.5 `pgssHashKey`

userid、dbid、queryid、toplevel 共同定义共享 hash entry。

### 4.6 `Counters`

calls、total_time、min/max/mean、rows、buffer、WAL、JIT 等聚合指标。

### 4.7 `external query text file`

pg_stat_statements 把规范化 query text 存在共享 hash 外，减少共享内存膨胀。

贯穿本节的不变量是：观测状态不应该破坏被观测对象的执行语义。

## 5. 主流程源码 walkthrough

主流程按时间推进，而不是按文件归类。

```text
EnableQueryId()
  ->
JumbleQuery()
  ->
planner()
  ->
pgss_ExecutorStart()
  ->
standard_ExecutorStart()
  ->
pgss_ExecutorRun / Finish()
  ->
pgss_ExecutorEnd()
  ->
pgss_store()
  ->
pg_stat_statements_internal()
```

### 5.1 `EnableQueryId()`

时间线推进到 `EnableQueryId()` 时，关键变化是：扩展加载时要求核心在 auto 模式下生成 query id。

扩展不自己重写 parser。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.2 `JumbleQuery()`

时间线推进到 `JumbleQuery()` 时，关键变化是：对 Query tree 的稳定字段做 hash，忽略常量差异。

同类查询聚合依赖这个规范化边界。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.3 `planner()`

时间线推进到 `planner()` 时，关键变化是：把 Query.queryId 传入 PlannedStmt.queryId。

执行期不再拿原始 Query tree。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.4 `pgss_ExecutorStart()`

时间线推进到 `pgss_ExecutorStart()` 时，关键变化是：如果 queryId 非零且扩展启用，设置 queryDesc->query_instr_options |= INSTRUMENT_ALL。

扩展请求查询级统计，而不是节点级 PlanState.instrument。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.5 `standard_ExecutorStart()`

时间线推进到 `standard_ExecutorStart()` 时，关键变化是：看到 query_instr_options 后分配 queryDesc->query_instr。

查询级计时窗口覆盖 ExecutorRun/Finish。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.6 `pgss_ExecutorRun / Finish()`

时间线推进到 `pgss_ExecutorRun / Finish()` 时，关键变化是：维护 nesting_level，区分 top-level 和 nested statements。

聚合 key 需要知道语句层次。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.7 `pgss_ExecutorEnd()`

时间线推进到 `pgss_ExecutorEnd()` 时，关键变化是：从 query_instr、estate 和 plannedstmt 取 total time、rows、buffers、WAL、JIT、parallel worker 数。

执行完成后才有完整聚合事实。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.8 `pgss_store()`

时间线推进到 `pgss_store()` 时，关键变化是：清理 query text，查找或创建共享 hash entry，并更新 Counters。

共享状态更新受 LWLock 和 spinlock 保护。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.9 `pg_stat_statements_internal()`

时间线推进到 `pg_stat_statements_internal()` 时，关键变化是：读取共享 hash 和 query text file，按权限隐藏 queryid 或 query text。

视图输出是聚合快照。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

主流程结束后，用户看到的是输出或视图；源码中真正重要的是状态已经在哪个边界完成折叠。

## 6. 状态随时间推进的完整故事

把本节主问题压成一条对象生命周期，可以避免在函数清单里迷路。

阶段 1：创建或启用

观察 `Query.queryId` 在 `EnableQueryId()` 前后的变化。

这个阶段不要急着解释用户输出。

先确认状态是否已经产生，是否仍是本地状态，是否已经被折叠成共享或输出状态。

只有完成这个判断，后面的诊断结论才不会跨层。

阶段 2：挂载或关联

观察 `PlannedStmt.queryId` 在 `JumbleQuery()` 前后的变化。

这个阶段不要急着解释用户输出。

先确认状态是否已经产生，是否仍是本地状态，是否已经被折叠成共享或输出状态。

只有完成这个判断，后面的诊断结论才不会跨层。

阶段 3：热路径累计

观察 `QueryDesc.query_instr_options` 在 `planner()` 前后的变化。

这个阶段不要急着解释用户输出。

先确认状态是否已经产生，是否仍是本地状态，是否已经被折叠成共享或输出状态。

只有完成这个判断，后面的诊断结论才不会跨层。

阶段 4：周期折叠

观察 `QueryDesc.query_instr` 在 `pgss_ExecutorStart()` 前后的变化。

这个阶段不要急着解释用户输出。

先确认状态是否已经产生，是否仍是本地状态，是否已经被折叠成共享或输出状态。

只有完成这个判断，后面的诊断结论才不会跨层。

阶段 5：输出或刷新

观察 `pgssHashKey` 在 `standard_ExecutorStart()` 前后的变化。

这个阶段不要急着解释用户输出。

先确认状态是否已经产生，是否仍是本地状态，是否已经被折叠成共享或输出状态。

只有完成这个判断，后面的诊断结论才不会跨层。

阶段 6：清理或失效

观察 `Counters` 在 `pgss_ExecutorRun / Finish()` 前后的变化。

这个阶段不要急着解释用户输出。

先确认状态是否已经产生，是否仍是本地状态，是否已经被折叠成共享或输出状态。

只有完成这个判断，后面的诊断结论才不会跨层。

完整故事的终点不是“函数返回”，而是用户可以稳定看到或推断这个状态。

## 7. 生命周期 / ownership / cleanup

1. queryId 从 Query 传播到 PlannedStmt，再进入 QueryDesc。

这里要同时回答谁创建、谁持有、谁在正常路径读取、谁在异常路径兜底。

2. query_instr 在本次执行中分配，ExecutorEnd 后释放。

这里要同时回答谁创建、谁持有、谁在正常路径读取、谁在异常路径兜底。

3. pgss shared hash 在 shared_preload_libraries 加载扩展后存在，跨语句累积。

这里要同时回答谁创建、谁持有、谁在正常路径读取、谁在异常路径兜底。

4. query text 文件独立于 shared hash entry 管理，需要 garbage collection。

这里要同时回答谁创建、谁持有、谁在正常路径读取、谁在异常路径兜底。

5. reset 函数可以按 userid、dbid、queryid 或 minmax_only 调整统计生命周期。

这里要同时回答谁创建、谁持有、谁在正常路径读取、谁在异常路径兜底。

一个常见错误是把“能读到”误认为“拥有”。

执行器、统计系统和扩展 hook 中，这两者经常分离。

## 8. 正确性机制层次

正确性不是由单个计数器保证的。

层次 1：queryId 为零时 pg_stat_statements 不跟踪，避免 utility 嵌套和不可归一语句双计数。

把这个层次和本节主流程对齐，才能知道它保证什么、不能保证什么。

层次 2：聚合 key 包含 userid、dbid、queryid、toplevel，不只是 query 文本。

把这个层次和本节主流程对齐，才能知道它保证什么、不能保证什么。

层次 3：rows 来自 estate->es_total_processed，不等于某个节点 actual rows。

把这个层次和本节主流程对齐，才能知道它保证什么、不能保证什么。

层次 4：buffers 和 WAL 来自 query_instr 的总量，不提供节点归属。

把这个层次和本节主流程对齐，才能知道它保证什么、不能保证什么。

层次 5：权限不足的用户可能看不到 query text 或 queryid，但统计行仍有权限边界。

把这个层次和本节主流程对齐，才能知道它保证什么、不能保证什么。

本节涉及的是观测正确性，不是事务隔离本身。

观测正确性的最低要求是：不要因为收集指标而改变执行语义。

## 9. 错误路径 / 异常路径 / fallback

异常路径 1：compute_query_id=off 且没有其他模块提供 queryId 时，pgss_store 会直接返回。

遇到这个路径时，先判断状态是缺失、延迟、被聚合，还是被有意隐藏。

异常路径 2：ProcessUtility 路径需要单独 hook，utility 语句和 optimizable 语句的 queryId 处理不同。

遇到这个路径时，先判断状态是缺失、延迟、被聚合，还是被有意隐藏。

异常路径 3：prepared statement、EXECUTE、规则重写和 nested statement 会影响 sourceText 与 queryId 的关系。

遇到这个路径时，先判断状态是缺失、延迟、被聚合，还是被有意隐藏。

异常路径 4：hash entry 不存在时需要写 query text 文件，再升级锁创建 entry。

遇到这个路径时，先判断状态是缺失、延迟、被聚合，还是被有意隐藏。

异常路径 5：共享 hash 满时可能需要 entry eviction 或 query text garbage collection。

遇到这个路径时，先判断状态是缺失、延迟、被聚合，还是被有意隐藏。

异常路径的课程价值在于保护诊断结论。

看不到某个指标，不等于底层事件没有发生。

## 10. 成本、资源与跨模块传播

成本点 1：query jumble 增加 parse/analyze 阶段成本，但换来跨执行聚合能力。

它对应的跨模块传播要从执行器边界继续追到存储、WAL、JIT、MemoryContext、统计共享状态或扩展 hook。

成本点 2：query-level instrumentation 比节点级便宜，但仍会采样总时间、buffers、WAL。

它对应的跨模块传播要从执行器边界继续追到存储、WAL、JIT、MemoryContext、统计共享状态或扩展 hook。

成本点 3：pgss_store 更新共享 hash 需要锁，热点 workload 上可能形成扩展层竞争。

它对应的跨模块传播要从执行器边界继续追到存储、WAL、JIT、MemoryContext、统计共享状态或扩展 hook。

成本点 4：query text 外部文件减少共享内存消耗，但引入文件写入和 GC 复杂性。

它对应的跨模块传播要从执行器边界继续追到存储、WAL、JIT、MemoryContext、统计共享状态或扩展 hook。

成本点 5：统计聚合隐藏参数差异，降低诊断细节以换取长期趋势。

它对应的跨模块传播要从执行器边界继续追到存储、WAL、JIT、MemoryContext、统计共享状态或扩展 hook。

成本分析必须问两个问题。

第一，事件发生在热路径还是收尾路径。

第二，成本是每 tuple、每 node、每 query、每 flush，还是每输出字段支付。

## 11. 观测与诊断入口

入口 1：创建 pg_stat_statements 扩展后执行相同 SQL 不同常量，观察 calls 增加在同一 queryid。

观察结果出来后，要回到第 3 节的源码入口确认它的来源。

入口 2：打开 track_planning，观察 plans 与 planning time 指标。

观察结果出来后，要回到第 3 节的源码入口确认它的来源。

入口 3：执行 DML，观察 rows、wal_records、wal_bytes 的聚合。

观察结果出来后，要回到第 3 节的源码入口确认它的来源。

入口 4：用不同用户执行同类 SQL，观察 userid 维度分裂。

观察结果出来后，要回到第 3 节的源码入口确认它的来源。

入口 5：用 pg_stat_statements_reset 指定 queryid，观察 reset scope。

观察结果出来后，要回到第 3 节的源码入口确认它的来源。

诊断闭环的顺序应当是：看到现象，定位状态，回到源码，再修正解释。

## 12. 课堂实验

实验 1：设置 shared_preload_libraries 加载 pg_stat_statements，重启后 CREATE EXTENSION。

实验前先写下你预期哪个状态会变化。

实验后再用源码解释为什么变化发生在这个边界，而不是相邻模块。

实验 2：执行 SELECT * FROM t WHERE id = 1 和 id = 2，确认 query 文本规范化。

实验前先写下你预期哪个状态会变化。

实验后再用源码解释为什么变化发生在这个边界，而不是相邻模块。

实验 3：执行同一查询多次后查看 calls、mean_exec_time、rows。

实验前先写下你预期哪个状态会变化。

实验后再用源码解释为什么变化发生在这个边界，而不是相邻模块。

实验 4：对热点 queryid 再执行 EXPLAIN ANALYZE，比较聚合平均和单次计划。

实验前先写下你预期哪个状态会变化。

实验后再用源码解释为什么变化发生在这个边界，而不是相邻模块。

课堂实验不追求压测结果。

它只验证一个 runtime 现象能否回到源码状态。

## 13. 常见误区

误区 1：不要把 pg_stat_statements 当成单次 plan profiler。它没有 PlanState tree。

修正这个误区的方法是重新确认粒度：节点、查询、对象、backend、本地 pending、共享 entry 或输出格式。

误区 2：不要把同一个 queryid 当成相同参数、相同缓存、相同计划。

修正这个误区的方法是重新确认粒度：节点、查询、对象、backend、本地 pending、共享 entry 或输出格式。

误区 3：不要忽略 userid、dbid、toplevel 维度。queryid 相同也可能分成多行。

修正这个误区的方法是重新确认粒度：节点、查询、对象、backend、本地 pending、共享 entry 或输出格式。

误区 4：不要把 rows 解释成扫描行数。它是语句处理行数。

修正这个误区的方法是重新确认粒度：节点、查询、对象、backend、本地 pending、共享 entry 或输出格式。

大多数误读都不是因为字段名难，而是因为把两个生命周期不同的事实放在一起比较。

## 14. 源码复盘清单

复盘 1：从 `EnableQueryId()` 回到 `src/backend/nodes/queryjumblefuncs.c`。

先确认 `Query.queryId` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：扩展加载时要求核心在 auto 模式下生成 query id。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：query jumble 增加 parse/analyze 阶段成本，但换来跨执行聚合能力。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 2：从 `JumbleQuery()` 回到 `src/include/nodes/queryjumble.h`。

先确认 `PlannedStmt.queryId` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：对 Query tree 的稳定字段做 hash，忽略常量差异。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：query-level instrumentation 比节点级便宜，但仍会采样总时间、buffers、WAL。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 3：从 `planner()` 回到 `src/backend/parser/analyze.c`。

先确认 `QueryDesc.query_instr_options` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：把 Query.queryId 传入 PlannedStmt.queryId。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：pgss_store 更新共享 hash 需要锁，热点 workload 上可能形成扩展层竞争。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 4：从 `pgss_ExecutorStart()` 回到 `src/backend/optimizer/plan/planner.c`。

先确认 `QueryDesc.query_instr` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：如果 queryId 非零且扩展启用，设置 queryDesc->query_instr_options |= INSTRUMENT_ALL。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：query text 外部文件减少共享内存消耗，但引入文件写入和 GC 复杂性。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 5：从 `standard_ExecutorStart()` 回到 `src/backend/executor/execMain.c`。

先确认 `pgssHashKey` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：看到 query_instr_options 后分配 queryDesc->query_instr。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：统计聚合隐藏参数差异，降低诊断细节以换取长期趋势。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 6：从 `pgss_ExecutorRun / Finish()` 回到 `src/include/executor/execdesc.h`。

先确认 `Counters` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：维护 nesting_level，区分 top-level 和 nested statements。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：query jumble 增加 parse/analyze 阶段成本，但换来跨执行聚合能力。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 7：从 `pgss_ExecutorEnd()` 回到 `src/backend/utils/activity/backend_status.c`。

先确认 `external query text file` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：从 query_instr、estate 和 plannedstmt 取 total time、rows、buffers、WAL、JIT、parallel worker 数。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：query-level instrumentation 比节点级便宜，但仍会采样总时间、buffers、WAL。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 8：从 `pgss_store()` 回到 `contrib/pg_stat_statements/pg_stat_statements.c`。

先确认 `Query.queryId` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：清理 query text，查找或创建共享 hash entry，并更新 Counters。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：pgss_store 更新共享 hash 需要锁，热点 workload 上可能形成扩展层竞争。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 9：从 `pg_stat_statements_internal()` 回到 `contrib/pg_stat_statements/sql/select.sql`。

先确认 `PlannedStmt.queryId` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：读取共享 hash 和 query text file，按权限隐藏 queryid 或 query text。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：query text 外部文件减少共享内存消耗，但引入文件写入和 GC 复杂性。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 10：从 `EnableQueryId()` 回到 `src/backend/nodes/queryjumblefuncs.c`。

先确认 `QueryDesc.query_instr_options` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：扩展加载时要求核心在 auto 模式下生成 query id。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：统计聚合隐藏参数差异，降低诊断细节以换取长期趋势。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 11：从 `JumbleQuery()` 回到 `src/include/nodes/queryjumble.h`。

先确认 `QueryDesc.query_instr` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：对 Query tree 的稳定字段做 hash，忽略常量差异。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：query jumble 增加 parse/analyze 阶段成本，但换来跨执行聚合能力。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 12：从 `planner()` 回到 `src/backend/parser/analyze.c`。

先确认 `pgssHashKey` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：把 Query.queryId 传入 PlannedStmt.queryId。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：query-level instrumentation 比节点级便宜，但仍会采样总时间、buffers、WAL。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 13：从 `pgss_ExecutorStart()` 回到 `src/backend/optimizer/plan/planner.c`。

先确认 `Counters` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：如果 queryId 非零且扩展启用，设置 queryDesc->query_instr_options |= INSTRUMENT_ALL。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：pgss_store 更新共享 hash 需要锁，热点 workload 上可能形成扩展层竞争。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 14：从 `standard_ExecutorStart()` 回到 `src/backend/executor/execMain.c`。

先确认 `external query text file` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：看到 query_instr_options 后分配 queryDesc->query_instr。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：query text 外部文件减少共享内存消耗，但引入文件写入和 GC 复杂性。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 15：从 `pgss_ExecutorRun / Finish()` 回到 `src/include/executor/execdesc.h`。

先确认 `Query.queryId` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：维护 nesting_level，区分 top-level 和 nested statements。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：统计聚合隐藏参数差异，降低诊断细节以换取长期趋势。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 16：从 `pgss_ExecutorEnd()` 回到 `src/backend/utils/activity/backend_status.c`。

先确认 `PlannedStmt.queryId` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：从 query_instr、estate 和 plannedstmt 取 total time、rows、buffers、WAL、JIT、parallel worker 数。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：query jumble 增加 parse/analyze 阶段成本，但换来跨执行聚合能力。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 17：从 `pgss_store()` 回到 `contrib/pg_stat_statements/pg_stat_statements.c`。

先确认 `QueryDesc.query_instr_options` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：清理 query text，查找或创建共享 hash entry，并更新 Counters。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：query-level instrumentation 比节点级便宜，但仍会采样总时间、buffers、WAL。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 18：从 `pg_stat_statements_internal()` 回到 `contrib/pg_stat_statements/sql/select.sql`。

先确认 `QueryDesc.query_instr` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：读取共享 hash 和 query text file，按权限隐藏 queryid 或 query text。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：pgss_store 更新共享 hash 需要锁，热点 workload 上可能形成扩展层竞争。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 19：从 `EnableQueryId()` 回到 `src/backend/nodes/queryjumblefuncs.c`。

先确认 `pgssHashKey` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：扩展加载时要求核心在 auto 模式下生成 query id。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：query text 外部文件减少共享内存消耗，但引入文件写入和 GC 复杂性。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 20：从 `JumbleQuery()` 回到 `src/include/nodes/queryjumble.h`。

先确认 `Counters` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：对 Query tree 的稳定字段做 hash，忽略常量差异。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：统计聚合隐藏参数差异，降低诊断细节以换取长期趋势。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 21：从 `planner()` 回到 `src/backend/parser/analyze.c`。

先确认 `external query text file` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：把 Query.queryId 传入 PlannedStmt.queryId。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：query jumble 增加 parse/analyze 阶段成本，但换来跨执行聚合能力。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 22：从 `pgss_ExecutorStart()` 回到 `src/backend/optimizer/plan/planner.c`。

先确认 `Query.queryId` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：如果 queryId 非零且扩展启用，设置 queryDesc->query_instr_options |= INSTRUMENT_ALL。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：query-level instrumentation 比节点级便宜，但仍会采样总时间、buffers、WAL。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 23：从 `standard_ExecutorStart()` 回到 `src/backend/executor/execMain.c`。

先确认 `PlannedStmt.queryId` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：看到 query_instr_options 后分配 queryDesc->query_instr。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：pgss_store 更新共享 hash 需要锁，热点 workload 上可能形成扩展层竞争。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 24：从 `pgss_ExecutorRun / Finish()` 回到 `src/include/executor/execdesc.h`。

先确认 `QueryDesc.query_instr_options` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：维护 nesting_level，区分 top-level 和 nested statements。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：query text 外部文件减少共享内存消耗，但引入文件写入和 GC 复杂性。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 25：从 `pgss_ExecutorEnd()` 回到 `src/backend/utils/activity/backend_status.c`。

先确认 `QueryDesc.query_instr` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：从 query_instr、estate 和 plannedstmt 取 total time、rows、buffers、WAL、JIT、parallel worker 数。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：统计聚合隐藏参数差异，降低诊断细节以换取长期趋势。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 26：从 `pgss_store()` 回到 `contrib/pg_stat_statements/pg_stat_statements.c`。

先确认 `pgssHashKey` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：清理 query text，查找或创建共享 hash entry，并更新 Counters。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：query jumble 增加 parse/analyze 阶段成本，但换来跨执行聚合能力。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 27：从 `pg_stat_statements_internal()` 回到 `contrib/pg_stat_statements/sql/select.sql`。

先确认 `Counters` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：读取共享 hash 和 query text file，按权限隐藏 queryid 或 query text。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：query-level instrumentation 比节点级便宜，但仍会采样总时间、buffers、WAL。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 28：从 `EnableQueryId()` 回到 `src/backend/nodes/queryjumblefuncs.c`。

先确认 `external query text file` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：扩展加载时要求核心在 auto 模式下生成 query id。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：pgss_store 更新共享 hash 需要锁，热点 workload 上可能形成扩展层竞争。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 29：从 `JumbleQuery()` 回到 `src/include/nodes/queryjumble.h`。

先确认 `Query.queryId` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：对 Query tree 的稳定字段做 hash，忽略常量差异。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：query text 外部文件减少共享内存消耗，但引入文件写入和 GC 复杂性。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

## 15. 讨论题

1. queryId 规范化应该忽略哪些差异，保留哪些差异？

回答时必须引用至少一个第 3 节源码入口。

2. 为什么 pg_stat_statements 请求 query_instr，而不是打开每个 PlanState instrumentation？

回答时必须引用至少一个第 3 节源码入口。

3. query text 文件独立存储带来了哪些一致性和 GC 问题？

回答时必须引用至少一个第 3 节源码入口。

4. 如何从 pg_stat_statements 选择一个值得手工 EXPLAIN 的样本？

回答时必须引用至少一个第 3 节源码入口。

讨论题的目标不是扩展范围，而是检查本节唯一主问题是否已经能独立解释。

## 16. 本节小结

pg_stat_statements 把 query id 作为聚合边界，把查询级 instrumentation 作为执行事实来源。

它擅长找热点、趋势和资源大户，不擅长解释单次节点行为。

正确工作流通常是先用 pg_stat_statements 定位，再用 EXPLAIN 或 profiler 下钻。

可以带走的可迁移规律是：

```text
先找到状态归属，再解释输出字段；先确认生命周期，再讨论成本和正确性。
```

下一节继续沿同一条主线，把这个边界推进到相邻观测机制。
