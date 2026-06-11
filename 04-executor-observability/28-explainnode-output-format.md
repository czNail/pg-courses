# PostgreSQL ExplainNode 如何格式化执行状态输出

## 课程定位

前置知识：已经理解 EXPLAIN ANALYZE 的执行期状态来自 PlanState.instrument，也知道不同资源指标的来源。

本节唯一主问题：

```text
ExplainNode 如何把 PlanState tree、plan-time 信息、runtime instrumentation、worker 信息和 text/json/yaml/xml 格式要求合成一份稳定输出？
```

核心矛盾：EXPLAIN 必须兼顾人类可读的 text 输出和机器可解析的结构化输出，同时又要保留 PostgreSQL 历史输出格式、节点差异和扩展节点信息。

学完后应能判断：能从 EXPLAIN 输出的一行字段定位到 ExplainNode、ExplainProperty*、ExplainOpenGroup/CloseGroup 的格式化边界。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

第 25 到第 27 节解释了数据从哪里来。

本节解释数据怎样被排成输出。

ExplainNode 不是 executor 的执行函数。

它是一个读取 PlanState 和 Plan 的 formatter。

它需要在同一棵树上同时呈现 plan-time 与 runtime。

后续第 29 节会讨论这种输出本身带来的观测扰动。

本节刻意只处理一个问题：观测状态如何在执行器或统计体系中找到稳定边界。

凡是不能帮助解释这个边界的函数和字段，都只在后续课程或源码练习中出现。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
ExplainPrintPlan 从 QueryDesc 取顶层 PlanState，ExplainNode 递归遍历 PlanState tree，先识别节点类型和 plan-time 属性，再读取 instrumentation、worker 和节点私有统计，最后通过 ExplainProperty 与 group API 映射到 text/json/yaml/xml。
```

这句话里有三层含义。

第一层是入口边界：用户看到的 SQL 选项、统计视图或扩展 hook，必须先被转换成执行期状态。

第二层是状态边界：执行路径不能在每个热函数里直接做复杂诊断，只能在少数明确对象上累计事实。

第三层是解释边界：输出或视图只是读取这些事实，不能把估算、累计、平均、延迟刷新混成一个语义。

本节后面的所有源码 walkthrough 都围绕这三层展开。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 作用 |
| --- | --- | --- |
| 1 | `src/backend/commands/explain.c` | ExplainOnePlan、ExplainPrintPlan、ExplainNode、show_plan_tlist、show_instrumentation_count 和节点私有输出函数。 |
| 2 | `src/backend/commands/explain_format.c` | ExplainOpenGroup、ExplainCloseGroup、ExplainProperty*、ExplainBeginOutput、ExplainEndOutput 处理格式差异。 |
| 3 | `src/backend/commands/explain_state.c` | NewExplainState 初始化 ExplainState。 |
| 4 | `src/include/commands/explain_state.h` | ExplainState 保存 analyze、verbose、buffers、format、indent、workers_state 等输出状态。 |
| 5 | `src/include/commands/explain_format.h` | 格式化层对外暴露的 property 和 group API。 |
| 6 | `src/include/commands/explain.h` | ExplainOnePlan、ExplainPrintPlan、ExplainPrintJITSummary 等入口声明。 |
| 7 | `src/backend/utils/adt/ruleutils.c` | 表达式、targetlist、qual 反解析会被 explain 输出调用。 |
| 8 | `src/backend/commands/explain_dr.c` | EXPLAIN SERIALIZE 相关 DestReceiver 和输出度量。 |

阅读时不要按文件名背 API。

先找入口状态，再找运行期持有者，然后看 cleanup、fallback 和输出路径。

如果一个函数只是在表格中出现，而没有参与本节主问题，就不要把它扩展成新的主线。

## 4. 关键数据结构与状态边界

本节需要关注的不是单个字段名字，而是字段组合在生命周期中的语义。

### 4.1 `ExplainState.format`

决定 text、json、yaml、xml 的输出协议。

### 4.2 `ExplainState.indent`

text 输出的缩进状态，和结构化格式的 group nesting 不完全等价。

### 4.3 `ExplainState.analyze`

决定是否读取 runtime instrumentation。

### 4.4 `ExplainState.verbose`

决定是否输出更完整的 targetlist、query id 和 worker 细节。

### 4.5 `ExplainState.workers_state`

在打印某个 PlanState 时暂存 worker 输出缓冲。

### 4.6 `PlanState.plan`

formatter 读取 plan-time cost、rows、width、node type 的入口。

### 4.7 `PlanState.instrument`

formatter 读取 actual rows、loops、time、buffers、WAL 的入口。

贯穿本节的不变量是：观测状态不应该破坏被观测对象的执行语义。

## 5. 主流程源码 walkthrough

主流程按时间推进，而不是按文件归类。

```text
NewExplainState()
  ->
ExplainOnePlan()
  ->
ExplainPrintPlan()
  ->
ExplainNode()
  ->
ExplainOpenGroup()
  ->
plan-time 属性输出
  ->
runtime 属性输出
  ->
节点私有详情输出
  ->
递归子节点
  ->
ExplainCloseGroup()
```

### 5.1 `NewExplainState()`

时间线推进到 `NewExplainState()` 时，关键变化是：创建 ExplainState 并设置默认选项。

输出格式状态从命令解析进入 formatter。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.2 `ExplainOnePlan()`

时间线推进到 `ExplainOnePlan()` 时，关键变化是：执行或只初始化计划，并保留 QueryDesc 到打印结束。

ExplainNode 需要 PlanState，所以不能先 ExecutorEnd。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.3 `ExplainPrintPlan()`

时间线推进到 `ExplainPrintPlan()` 时，关键变化是：取得 queryDesc->planstate 并输出顶层 Query group。

格式化从整棵 PlanState tree 的根开始。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.4 `ExplainNode()`

时间线推进到 `ExplainNode()` 时，关键变化是：根据 nodeTag(plan) 识别节点名、策略、join type、operation。

节点类型输出来自 Plan，而不是 Instrumentation。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.5 `ExplainOpenGroup()`

时间线推进到 `ExplainOpenGroup()` 时，关键变化是：为当前节点建立格式化 group。

text 和 json/yaml/xml 在这里分叉。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.6 `plan-time 属性输出`

时间线推进到 `plan-time 属性输出` 时，关键变化是：输出 cost、plan rows、plan width、parallel aware 等。

这些属性不需要 ANALYZE。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.7 `runtime 属性输出`

时间线推进到 `runtime 属性输出` 时，关键变化是：调用 InstrEndLoop 后输出 actual startup/total/rows/loops。

这些属性需要 analyze 和 instrument。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.8 `节点私有详情输出`

时间线推进到 `节点私有详情输出` 时，关键变化是：按节点类型输出 Sort Method、Hash Batches、Heap Blocks、Index Searches 等。

ExplainNode 保留大量历史和节点差异。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.9 `递归子节点`

时间线推进到 `递归子节点` 时，关键变化是：按 outer、inner、subplan、initplan 等关系继续 ExplainNode。

输出结构是 PlanState tree 和附属 subplan 的组合。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.10 `ExplainCloseGroup()`

时间线推进到 `ExplainCloseGroup()` 时，关键变化是：关闭当前节点 group。

结构化格式依赖严格成对的 group 边界。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

主流程结束后，用户看到的是输出或视图；源码中真正重要的是状态已经在哪个边界完成折叠。

## 6. 状态随时间推进的完整故事

把本节主问题压成一条对象生命周期，可以避免在函数清单里迷路。

阶段 1：创建或启用

观察 `ExplainState.format` 在 `NewExplainState()` 前后的变化。

这个阶段不要急着解释用户输出。

先确认状态是否已经产生，是否仍是本地状态，是否已经被折叠成共享或输出状态。

只有完成这个判断，后面的诊断结论才不会跨层。

阶段 2：挂载或关联

观察 `ExplainState.indent` 在 `ExplainOnePlan()` 前后的变化。

这个阶段不要急着解释用户输出。

先确认状态是否已经产生，是否仍是本地状态，是否已经被折叠成共享或输出状态。

只有完成这个判断，后面的诊断结论才不会跨层。

阶段 3：热路径累计

观察 `ExplainState.analyze` 在 `ExplainPrintPlan()` 前后的变化。

这个阶段不要急着解释用户输出。

先确认状态是否已经产生，是否仍是本地状态，是否已经被折叠成共享或输出状态。

只有完成这个判断，后面的诊断结论才不会跨层。

阶段 4：周期折叠

观察 `ExplainState.verbose` 在 `ExplainNode()` 前后的变化。

这个阶段不要急着解释用户输出。

先确认状态是否已经产生，是否仍是本地状态，是否已经被折叠成共享或输出状态。

只有完成这个判断，后面的诊断结论才不会跨层。

阶段 5：输出或刷新

观察 `ExplainState.workers_state` 在 `ExplainOpenGroup()` 前后的变化。

这个阶段不要急着解释用户输出。

先确认状态是否已经产生，是否仍是本地状态，是否已经被折叠成共享或输出状态。

只有完成这个判断，后面的诊断结论才不会跨层。

阶段 6：清理或失效

观察 `PlanState.plan` 在 `plan-time 属性输出` 前后的变化。

这个阶段不要急着解释用户输出。

先确认状态是否已经产生，是否仍是本地状态，是否已经被折叠成共享或输出状态。

只有完成这个判断，后面的诊断结论才不会跨层。

完整故事的终点不是“函数返回”，而是用户可以稳定看到或推断这个状态。

## 7. 生命周期 / ownership / cleanup

1. ExplainState 由 EXPLAIN 命令创建，生命周期覆盖格式化输出。

这里要同时回答谁创建、谁持有、谁在正常路径读取、谁在异常路径兜底。

2. QueryDesc 和 EState 必须活到 ExplainPrintPlan 结束，因为 PlanState.instrument 还要被读取。

这里要同时回答谁创建、谁持有、谁在正常路径读取、谁在异常路径兜底。

3. text 输出主要写入 ExplainState.str，结构化输出也通过同一 StringInfo 构造。

这里要同时回答谁创建、谁持有、谁在正常路径读取、谁在异常路径兜底。

4. workers_state 只在打印当前节点 worker 信息时暂时切换，离开节点后恢复。

这里要同时回答谁创建、谁持有、谁在正常路径读取、谁在异常路径兜底。

5. ExecutorEnd 在 ExplainPrintPlan 后执行，释放 PlanState 和 instrumentation。

这里要同时回答谁创建、谁持有、谁在正常路径读取、谁在异常路径兜底。

一个常见错误是把“能读到”误认为“拥有”。

执行器、统计系统和扩展 hook 中，这两者经常分离。

## 8. 正确性机制层次

正确性不是由单个计数器保证的。

层次 1：ExplainNode 先打印 plan-time 信息，再按 analyze 判断 runtime 信息。

把这个层次和本节主流程对齐，才能知道它保证什么、不能保证什么。

层次 2：结构化输出必须使用 ExplainProperty*，不能随意 append text，否则 json/yaml/xml 会破坏。

把这个层次和本节主流程对齐，才能知道它保证什么、不能保证什么。

层次 3：text 格式有历史兼容性，某些字段被拼接在同一行，不能按结构化输出反推内部模型。

把这个层次和本节主流程对齐，才能知道它保证什么、不能保证什么。

层次 4：节点私有输出必须由节点类型决定，通用 PlanState.instrument 不包含 Sort Method 或 Hash Batches。

把这个层次和本节主流程对齐，才能知道它保证什么、不能保证什么。

层次 5：ExplainNode 在打印前调用 InstrEndLoop，确保最后一个未结束周期进入 totals。

把这个层次和本节主流程对齐，才能知道它保证什么、不能保证什么。

本节涉及的是观测正确性，不是事务隔离本身。

观测正确性的最低要求是：不要因为收集指标而改变执行语义。

## 9. 错误路径 / 异常路径 / fallback

异常路径 1：Utility statement 没有 plan structure，ExplainOneUtility 走不同输出。

遇到这个路径时，先判断状态是缺失、延迟、被聚合，还是被有意隐藏。

异常路径 2：ForeignScan 和 CustomScan 可以提供扩展细节，formatter 必须保留扩展边界。

遇到这个路径时，先判断状态是缺失、延迟、被聚合，还是被有意隐藏。

异常路径 3：never executed 是 analyze 模式下 nloops 为 0 的输出语义。

遇到这个路径时，先判断状态是缺失、延迟、被聚合，还是被有意隐藏。

异常路径 4：worker 输出可能被 hide_workers 抑制，不代表 worker 没有参与。

遇到这个路径时，先判断状态是缺失、延迟、被聚合，还是被有意隐藏。

异常路径 5：EXPLAIN SERIALIZE 会引入 DestReceiver 和 serialization metrics，不等同于普通执行输出。

遇到这个路径时，先判断状态是缺失、延迟、被聚合，还是被有意隐藏。

异常路径的课程价值在于保护诊断结论。

看不到某个指标，不等于底层事件没有发生。

## 10. 成本、资源与跨模块传播

成本点 1：格式化成本通常不在执行热路径，但巨大计划或 verbose 输出会产生明显字符串构造成本。

它对应的跨模块传播要从执行器边界继续追到存储、WAL、JIT、MemoryContext、统计共享状态或扩展 hook。

成本点 2：ruleutils 反解析表达式可能遍历复杂 expression tree。

它对应的跨模块传播要从执行器边界继续追到存储、WAL、JIT、MemoryContext、统计共享状态或扩展 hook。

成本点 3：结构化格式比 text 更严格，group 和 property 边界会增加代码复杂度。

它对应的跨模块传播要从执行器边界继续追到存储、WAL、JIT、MemoryContext、统计共享状态或扩展 hook。

成本点 4：worker 信息需要额外缓冲和合并，避免直接打乱父节点输出顺序。

它对应的跨模块传播要从执行器边界继续追到存储、WAL、JIT、MemoryContext、统计共享状态或扩展 hook。

成本点 5：为了历史兼容，text 输出保留许多特殊拼接逻辑，维护成本高于理想化 formatter。

它对应的跨模块传播要从执行器边界继续追到存储、WAL、JIT、MemoryContext、统计共享状态或扩展 hook。

成本分析必须问两个问题。

第一，事件发生在热路径还是收尾路径。

第二，成本是每 tuple、每 node、每 query、每 flush，还是每输出字段支付。

## 11. 观测与诊断入口

入口 1：同一查询分别用 FORMAT TEXT 和 FORMAT JSON，比较字段层次和同一属性名称。

观察结果出来后，要回到第 3 节的源码入口确认它的来源。

入口 2：打开 VERBOSE，观察 Output、Query Identifier、worker 细节是否增加。

观察结果出来后，要回到第 3 节的源码入口确认它的来源。

入口 3：对 Sort、Hash、BitmapHeapScan 使用 EXPLAIN ANALYZE，观察节点私有字段如何出现。

观察结果出来后，要回到第 3 节的源码入口确认它的来源。

入口 4：在 gdb 中断 ExplainPropertyFloat，查看 Actual Rows 从 ExplainNode 进入格式层。

观察结果出来后，要回到第 3 节的源码入口确认它的来源。

入口 5：在 gdb 中断 ExplainOpenGroup，查看 text 与 json 的分支。

观察结果出来后，要回到第 3 节的源码入口确认它的来源。

诊断闭环的顺序应当是：看到现象，定位状态，回到源码，再修正解释。

## 12. 课堂实验

实验 1：执行 EXPLAIN (FORMAT JSON) SELECT ...，用 jq 查看 Plan 数组层次。

实验前先写下你预期哪个状态会变化。

实验后再用源码解释为什么变化发生在这个边界，而不是相邻模块。

实验 2：执行 EXPLAIN (ANALYZE, VERBOSE) 包含 join 和 sort 的查询，标出哪些字段来自 Plan，哪些来自 Instrumentation。

实验前先写下你预期哪个状态会变化。

实验后再用源码解释为什么变化发生在这个边界，而不是相邻模块。

实验 3：执行 EXPLAIN (ANALYZE, SUMMARY OFF)，观察 summary 字段变化而节点字段仍在。

实验前先写下你预期哪个状态会变化。

实验后再用源码解释为什么变化发生在这个边界，而不是相邻模块。

实验 4：对一个 never executed 的子计划构造 CASE 或 InitPlan 场景，观察输出语义。

实验前先写下你预期哪个状态会变化。

实验后再用源码解释为什么变化发生在这个边界，而不是相邻模块。

课堂实验不追求压测结果。

它只验证一个 runtime 现象能否回到源码状态。

## 13. 常见误区

误区 1：不要以为 ExplainNode 执行计划。它只是读取已经存在的 PlanState。

修正这个误区的方法是重新确认粒度：节点、查询、对象、backend、本地 pending、共享 entry 或输出格式。

误区 2：不要把 text 输出的缩进当成源码里的唯一结构。结构化格式才暴露 group 层次。

修正这个误区的方法是重新确认粒度：节点、查询、对象、backend、本地 pending、共享 entry 或输出格式。

误区 3：不要把节点私有字段误认为所有节点都有的通用 instrumentation。

修正这个误区的方法是重新确认粒度：节点、查询、对象、backend、本地 pending、共享 entry 或输出格式。

误区 4：不要因为 JSON 有字段就认为字段总会出现；很多字段受选项、节点类型和执行状态控制。

修正这个误区的方法是重新确认粒度：节点、查询、对象、backend、本地 pending、共享 entry 或输出格式。

大多数误读都不是因为字段名难，而是因为把两个生命周期不同的事实放在一起比较。

## 14. 源码复盘清单

复盘 1：从 `NewExplainState()` 回到 `src/backend/commands/explain.c`。

先确认 `ExplainState.format` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：创建 ExplainState 并设置默认选项。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：格式化成本通常不在执行热路径，但巨大计划或 verbose 输出会产生明显字符串构造成本。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 2：从 `ExplainOnePlan()` 回到 `src/backend/commands/explain_format.c`。

先确认 `ExplainState.indent` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：执行或只初始化计划，并保留 QueryDesc 到打印结束。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：ruleutils 反解析表达式可能遍历复杂 expression tree。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 3：从 `ExplainPrintPlan()` 回到 `src/backend/commands/explain_state.c`。

先确认 `ExplainState.analyze` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：取得 queryDesc->planstate 并输出顶层 Query group。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：结构化格式比 text 更严格，group 和 property 边界会增加代码复杂度。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 4：从 `ExplainNode()` 回到 `src/include/commands/explain_state.h`。

先确认 `ExplainState.verbose` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：根据 nodeTag(plan) 识别节点名、策略、join type、operation。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：worker 信息需要额外缓冲和合并，避免直接打乱父节点输出顺序。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 5：从 `ExplainOpenGroup()` 回到 `src/include/commands/explain_format.h`。

先确认 `ExplainState.workers_state` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：为当前节点建立格式化 group。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：为了历史兼容，text 输出保留许多特殊拼接逻辑，维护成本高于理想化 formatter。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 6：从 `plan-time 属性输出` 回到 `src/include/commands/explain.h`。

先确认 `PlanState.plan` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：输出 cost、plan rows、plan width、parallel aware 等。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：格式化成本通常不在执行热路径，但巨大计划或 verbose 输出会产生明显字符串构造成本。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 7：从 `runtime 属性输出` 回到 `src/backend/utils/adt/ruleutils.c`。

先确认 `PlanState.instrument` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：调用 InstrEndLoop 后输出 actual startup/total/rows/loops。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：ruleutils 反解析表达式可能遍历复杂 expression tree。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 8：从 `节点私有详情输出` 回到 `src/backend/commands/explain_dr.c`。

先确认 `ExplainState.format` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：按节点类型输出 Sort Method、Hash Batches、Heap Blocks、Index Searches 等。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：结构化格式比 text 更严格，group 和 property 边界会增加代码复杂度。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 9：从 `递归子节点` 回到 `src/backend/commands/explain.c`。

先确认 `ExplainState.indent` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：按 outer、inner、subplan、initplan 等关系继续 ExplainNode。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：worker 信息需要额外缓冲和合并，避免直接打乱父节点输出顺序。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 10：从 `ExplainCloseGroup()` 回到 `src/backend/commands/explain_format.c`。

先确认 `ExplainState.analyze` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：关闭当前节点 group。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：为了历史兼容，text 输出保留许多特殊拼接逻辑，维护成本高于理想化 formatter。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 11：从 `NewExplainState()` 回到 `src/backend/commands/explain_state.c`。

先确认 `ExplainState.verbose` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：创建 ExplainState 并设置默认选项。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：格式化成本通常不在执行热路径，但巨大计划或 verbose 输出会产生明显字符串构造成本。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 12：从 `ExplainOnePlan()` 回到 `src/include/commands/explain_state.h`。

先确认 `ExplainState.workers_state` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：执行或只初始化计划，并保留 QueryDesc 到打印结束。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：ruleutils 反解析表达式可能遍历复杂 expression tree。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 13：从 `ExplainPrintPlan()` 回到 `src/include/commands/explain_format.h`。

先确认 `PlanState.plan` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：取得 queryDesc->planstate 并输出顶层 Query group。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：结构化格式比 text 更严格，group 和 property 边界会增加代码复杂度。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 14：从 `ExplainNode()` 回到 `src/include/commands/explain.h`。

先确认 `PlanState.instrument` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：根据 nodeTag(plan) 识别节点名、策略、join type、operation。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：worker 信息需要额外缓冲和合并，避免直接打乱父节点输出顺序。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 15：从 `ExplainOpenGroup()` 回到 `src/backend/utils/adt/ruleutils.c`。

先确认 `ExplainState.format` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：为当前节点建立格式化 group。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：为了历史兼容，text 输出保留许多特殊拼接逻辑，维护成本高于理想化 formatter。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 16：从 `plan-time 属性输出` 回到 `src/backend/commands/explain_dr.c`。

先确认 `ExplainState.indent` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：输出 cost、plan rows、plan width、parallel aware 等。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：格式化成本通常不在执行热路径，但巨大计划或 verbose 输出会产生明显字符串构造成本。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 17：从 `runtime 属性输出` 回到 `src/backend/commands/explain.c`。

先确认 `ExplainState.analyze` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：调用 InstrEndLoop 后输出 actual startup/total/rows/loops。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：ruleutils 反解析表达式可能遍历复杂 expression tree。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 18：从 `节点私有详情输出` 回到 `src/backend/commands/explain_format.c`。

先确认 `ExplainState.verbose` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：按节点类型输出 Sort Method、Hash Batches、Heap Blocks、Index Searches 等。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：结构化格式比 text 更严格，group 和 property 边界会增加代码复杂度。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 19：从 `递归子节点` 回到 `src/backend/commands/explain_state.c`。

先确认 `ExplainState.workers_state` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：按 outer、inner、subplan、initplan 等关系继续 ExplainNode。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：worker 信息需要额外缓冲和合并，避免直接打乱父节点输出顺序。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 20：从 `ExplainCloseGroup()` 回到 `src/include/commands/explain_state.h`。

先确认 `PlanState.plan` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：关闭当前节点 group。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：为了历史兼容，text 输出保留许多特殊拼接逻辑，维护成本高于理想化 formatter。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 21：从 `NewExplainState()` 回到 `src/include/commands/explain_format.h`。

先确认 `PlanState.instrument` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：创建 ExplainState 并设置默认选项。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：格式化成本通常不在执行热路径，但巨大计划或 verbose 输出会产生明显字符串构造成本。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 22：从 `ExplainOnePlan()` 回到 `src/include/commands/explain.h`。

先确认 `ExplainState.format` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：执行或只初始化计划，并保留 QueryDesc 到打印结束。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：ruleutils 反解析表达式可能遍历复杂 expression tree。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 23：从 `ExplainPrintPlan()` 回到 `src/backend/utils/adt/ruleutils.c`。

先确认 `ExplainState.indent` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：取得 queryDesc->planstate 并输出顶层 Query group。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：结构化格式比 text 更严格，group 和 property 边界会增加代码复杂度。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 24：从 `ExplainNode()` 回到 `src/backend/commands/explain_dr.c`。

先确认 `ExplainState.analyze` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：根据 nodeTag(plan) 识别节点名、策略、join type、operation。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：worker 信息需要额外缓冲和合并，避免直接打乱父节点输出顺序。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 25：从 `ExplainOpenGroup()` 回到 `src/backend/commands/explain.c`。

先确认 `ExplainState.verbose` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：为当前节点建立格式化 group。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：为了历史兼容，text 输出保留许多特殊拼接逻辑，维护成本高于理想化 formatter。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 26：从 `plan-time 属性输出` 回到 `src/backend/commands/explain_format.c`。

先确认 `ExplainState.workers_state` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：输出 cost、plan rows、plan width、parallel aware 等。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：格式化成本通常不在执行热路径，但巨大计划或 verbose 输出会产生明显字符串构造成本。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

## 15. 讨论题

1. 为什么 ExplainNode 同时读取 Plan 和 PlanState，而不是只读取其中一个？

回答时必须引用至少一个第 3 节源码入口。

2. text 格式历史兼容性对新增观测字段有什么约束？

回答时必须引用至少一个第 3 节源码入口。

3. 扩展节点应该怎样输出信息，才能同时兼容 text 和 json？

回答时必须引用至少一个第 3 节源码入口。

4. 如果 ExplainNode 在 ExecutorEnd 后运行，会失去哪些状态？

回答时必须引用至少一个第 3 节源码入口。

讨论题的目标不是扩展范围，而是检查本节唯一主问题是否已经能独立解释。

## 16. 本节小结

ExplainNode 是执行状态到用户输出的格式化边界。

它把 Plan 的估计、PlanState 的运行状态、worker 汇总和节点私有统计放进统一输出协议。

理解 formatter 后，才能从 EXPLAIN 字段反查正确源码层次。

可以带走的可迁移规律是：

```text
先找到状态归属，再解释输出字段；先确认生命周期，再讨论成本和正确性。
```

下一节继续沿同一条主线，把这个边界推进到相邻观测机制。
