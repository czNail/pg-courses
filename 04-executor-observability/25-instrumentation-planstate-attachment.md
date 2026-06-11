# PostgreSQL Instrumentation 如何挂到 PlanState

## 课程定位

前置知识：已经理解 ExecutorStart 会把 Plan tree 初始化成 PlanState tree，也知道 ExecProcNode 是执行节点之间的 tuple 拉取协议。

本节唯一主问题：

```text
EXPLAIN ANALYZE 为什么能拿到每个计划节点的运行时统计，而普通执行又不会无条件承担这些计时成本？
```

核心矛盾：执行器需要在显式开启时给出节点级 rows、loops、time、buffers、WAL 等细粒度事实，但 ExecProcNode 是高频热路径，不能让每次普通查询都经过完整观测逻辑。

学完后应能判断：能从一条 EXPLAIN ANALYZE 输出反推 Instrumentation 是在哪里分配、挂载、启动、停止和被 ExplainNode 读取的。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

第 24 节把 DML 的执行边界收束在 ModifyTable 上。

从本节开始，主线从“节点怎样执行”转向“节点执行过什么如何被看到”。

Instrumentation 是 EXPLAIN ANALYZE 的第一个核心对象。

它不是计划节点本身，也不是优化器估算。

它是执行期挂在 PlanState 上的一小块观测状态。

后续第 26 节会继续追问 actual rows、loops、startup time 和 total time 的累计边界。

本节刻意只处理一个问题：观测状态如何在执行器或统计体系中找到稳定边界。

凡是不能帮助解释这个边界的函数和字段，都只在后续课程或源码练习中出现。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
ExplainOnePlan 根据选项形成 instrument_options，CreateQueryDesc 把选项交给 ExecutorStart，ExecInitNode 为每个 PlanState 分配 NodeInstrumentation，首轮 ExecProcNode 再把调用入口切到带观测的 wrapper。
```

这句话里有三层含义。

第一层是入口边界：用户看到的 SQL 选项、统计视图或扩展 hook，必须先被转换成执行期状态。

第二层是状态边界：执行路径不能在每个热函数里直接做复杂诊断，只能在少数明确对象上累计事实。

第三层是解释边界：输出或视图只是读取这些事实，不能把估算、累计、平均、延迟刷新混成一个语义。

本节后面的所有源码 walkthrough 都围绕这三层展开。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 作用 |
| --- | --- | --- |
| 1 | `src/backend/commands/explain.c` | ExplainOnePlan 解析 ANALYZE、TIMING、BUFFERS、WAL、IO 选项，创建 QueryDesc，并在执行后调用 ExplainPrintPlan / ExplainNode 读取统计。 |
| 2 | `src/include/executor/execdesc.h` | QueryDesc 保存 instrument_options、query_instr_options 和 query_instr，是 explain 命令与 executor 的交接结构。 |
| 3 | `src/backend/tcop/pquery.c` | CreateQueryDesc 把 planned statement、snapshot、DestReceiver 和 instrumentation 选项放入 QueryDesc。 |
| 4 | `src/backend/executor/execMain.c` | standard_ExecutorStart 创建 EState，把 queryDesc->instrument_options 复制到 estate->es_instrument，并按 query_instr_options 创建查询级 Instrumentation。 |
| 5 | `src/backend/executor/execProcnode.c` | ExecInitNode 构造 PlanState tree，给节点挂 instrument，并通过 ExecProcNodeFirst 决定是否启用 wrapper。 |
| 6 | `src/backend/executor/instrument.c` | InstrAllocNode、ExecProcNodeInstr、InstrStartNode、InstrStopNode 定义节点级采样动作。 |
| 7 | `src/include/executor/instrument.h` | 定义 Instrumentation、NodeInstrumentation、InstrumentOption 和 BufferUsage / WalUsage。 |
| 8 | `src/include/nodes/execnodes.h` | PlanState 中的 instrument、worker_instrument、ExecProcNodeReal 等字段说明执行状态与观测状态如何相邻。 |
| 9 | `src/backend/executor/execParallel.c` | 并行查询中 worker instrumentation 的共享布局和回收路径。 |

阅读时不要按文件名背 API。

先找入口状态，再找运行期持有者，然后看 cleanup、fallback 和输出路径。

如果一个函数只是在表格中出现，而没有参与本节主问题，就不要把它扩展成新的主线。

## 4. 关键数据结构与状态边界

本节需要关注的不是单个字段名字，而是字段组合在生命周期中的语义。

### 4.1 `QueryDesc.instrument_options`

来自 EXPLAIN 选项的节点级观测位图，只影响 PlanState 上是否分配 NodeInstrumentation。

### 4.2 `QueryDesc.query_instr_options`

扩展可请求的查询级观测位图，pg_stat_statements 会使用它获得整条语句的总时间、buffer 和 WAL。

### 4.3 `EState.es_instrument`

ExecutorStart 后在整棵 PlanState 初始化期间可见的观测开关。

### 4.4 `PlanState.instrument`

挂在每个执行节点上的 NodeInstrumentation 指针；为 NULL 表示该节点不会走节点级 wrapper。

### 4.5 `PlanState.ExecProcNodeReal`

节点真实执行函数，保留给 wrapper 调用，避免观测逻辑覆盖节点自己的方法。

### 4.6 `NodeInstrumentation.running`

表示本轮执行周期已经产生首个 tuple，是 InstrEndLoop 能否累计 loops 的关键状态。

### 4.7 `ResultRelInfo.ri_TrigInstrument`

触发器级别的 instrumentation，说明执行器观测不只挂在 PlanState，也能挂在 DML side effect 对象上。

贯穿本节的不变量是：观测状态不应该破坏被观测对象的执行语义。

## 5. 主流程源码 walkthrough

主流程按时间推进，而不是按文件归类。

```text
ExplainOnePlan()
  ->
CreateQueryDesc()
  ->
ExecutorStart()
  ->
ExecInitNode()
  ->
ExecSetExecProcNode()
  ->
ExecProcNodeFirst()
  ->
ExecProcNodeInstr()
  ->
ExplainPrintPlan()
  ->
ExplainNode()
```

### 5.1 `ExplainOnePlan()`

时间线推进到 `ExplainOnePlan()` 时，关键变化是：根据 es->analyze、es->timing、es->buffers、es->wal、es->io 组合 instrument_option。

这一步把用户输出选项变成 executor 可理解的低层位图。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.2 `CreateQueryDesc()`

时间线推进到 `CreateQueryDesc()` 时，关键变化是：把 instrument_option 写入 QueryDesc.instrument_options。

QueryDesc 成为 explain 命令和 executor 生命周期之间的稳定边界。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.3 `ExecutorStart()`

时间线推进到 `ExecutorStart()` 时，关键变化是：进入 standard_ExecutorStart，创建 EState 并复制 es_instrument。

后续 ExecInitNode 只看 EState，不再依赖 ExplainState。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.4 `ExecInitNode()`

时间线推进到 `ExecInitNode()` 时，关键变化是：递归构造 PlanState tree，并在 estate->es_instrument 非零时分配 NodeInstrumentation。

观测状态和节点状态一一相邻，但仍然是可选对象。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.5 `ExecSetExecProcNode()`

时间线推进到 `ExecSetExecProcNode()` 时，关键变化是：把节点真实函数保存到 ExecProcNodeReal，把外部入口先设为 ExecProcNodeFirst。

首次调用时再决定是否切 wrapper，减少初始化阶段的特殊分支。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.6 `ExecProcNodeFirst()`

时间线推进到 `ExecProcNodeFirst()` 时，关键变化是：若 node->instrument 非 NULL，把入口切为 ExecProcNodeInstr；否则直接切回真实函数。

普通执行之后不再经过观测 wrapper。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.7 `ExecProcNodeInstr()`

时间线推进到 `ExecProcNodeInstr()` 时，关键变化是：在真实节点函数前后调用 InstrStartNode 和 InstrStopNode。

节点级时间和 tuple 数从统一边界进入 NodeInstrumentation。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.8 `ExplainPrintPlan()`

时间线推进到 `ExplainPrintPlan()` 时，关键变化是：从 QueryDesc 拿到顶层 PlanState。

Explain 阶段读取执行后的 PlanState，而不是重新解释 Plan。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.9 `ExplainNode()`

时间线推进到 `ExplainNode()` 时，关键变化是：遍历 PlanState tree，读取 planstate->instrument 并输出 actual 字段。

输出格式是观测状态的解释，不是优化器估算的替代。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

主流程结束后，用户看到的是输出或视图；源码中真正重要的是状态已经在哪个边界完成折叠。

## 6. 状态随时间推进的完整故事

把本节主问题压成一条对象生命周期，可以避免在函数清单里迷路。

阶段 1：创建或启用

观察 `QueryDesc.instrument_options` 在 `ExplainOnePlan()` 前后的变化。

这个阶段不要急着解释用户输出。

先确认状态是否已经产生，是否仍是本地状态，是否已经被折叠成共享或输出状态。

只有完成这个判断，后面的诊断结论才不会跨层。

阶段 2：挂载或关联

观察 `QueryDesc.query_instr_options` 在 `CreateQueryDesc()` 前后的变化。

这个阶段不要急着解释用户输出。

先确认状态是否已经产生，是否仍是本地状态，是否已经被折叠成共享或输出状态。

只有完成这个判断，后面的诊断结论才不会跨层。

阶段 3：热路径累计

观察 `EState.es_instrument` 在 `ExecutorStart()` 前后的变化。

这个阶段不要急着解释用户输出。

先确认状态是否已经产生，是否仍是本地状态，是否已经被折叠成共享或输出状态。

只有完成这个判断，后面的诊断结论才不会跨层。

阶段 4：周期折叠

观察 `PlanState.instrument` 在 `ExecInitNode()` 前后的变化。

这个阶段不要急着解释用户输出。

先确认状态是否已经产生，是否仍是本地状态，是否已经被折叠成共享或输出状态。

只有完成这个判断，后面的诊断结论才不会跨层。

阶段 5：输出或刷新

观察 `PlanState.ExecProcNodeReal` 在 `ExecSetExecProcNode()` 前后的变化。

这个阶段不要急着解释用户输出。

先确认状态是否已经产生，是否仍是本地状态，是否已经被折叠成共享或输出状态。

只有完成这个判断，后面的诊断结论才不会跨层。

阶段 6：清理或失效

观察 `NodeInstrumentation.running` 在 `ExecProcNodeFirst()` 前后的变化。

这个阶段不要急着解释用户输出。

先确认状态是否已经产生，是否仍是本地状态，是否已经被折叠成共享或输出状态。

只有完成这个判断，后面的诊断结论才不会跨层。

完整故事的终点不是“函数返回”，而是用户可以稳定看到或推断这个状态。

## 7. 生命周期 / ownership / cleanup

1. Instrumentation 分配在 executor per-query context 下，生命周期随 ExecutorEnd 清理。

这里要同时回答谁创建、谁持有、谁在正常路径读取、谁在异常路径兜底。

2. PlanState 持有 instrument 指针，但并不拥有跨查询可复用的统计对象。

这里要同时回答谁创建、谁持有、谁在正常路径读取、谁在异常路径兜底。

3. ExplainOnePlan 在 ExecutorEnd 前打印计划，因为 ExecutorEnd 会释放 EState 和 PlanState。

这里要同时回答谁创建、谁持有、谁在正常路径读取、谁在异常路径兜底。

4. ERROR 路径依赖 executor context 与 ResourceOwner 清理，不要求每个节点手动释放 NodeInstrumentation。

这里要同时回答谁创建、谁持有、谁在正常路径读取、谁在异常路径兜底。

5. query_instr 是 QueryDesc 级别对象，给扩展汇总整条语句使用，和每个节点的 PlanState.instrument 不同。

这里要同时回答谁创建、谁持有、谁在正常路径读取、谁在异常路径兜底。

一个常见错误是把“能读到”误认为“拥有”。

执行器、统计系统和扩展 hook 中，这两者经常分离。

## 8. 正确性机制层次

正确性不是由单个计数器保证的。

层次 1：普通执行不设置 instrument_options 时，PlanState.instrument 为 NULL，ExecProcNodeFirst 会把入口改回真实函数。

把这个层次和本节主流程对齐，才能知道它保证什么、不能保证什么。

层次 2：启用 ANALYZE 但关闭 TIMING 时仍使用 INSTRUMENT_ROWS，保证 rows / loops 可见而避免节点级计时。

把这个层次和本节主流程对齐，才能知道它保证什么、不能保证什么。

层次 3：BUFFERS 和 WAL 不单独表示时间，它们通过全局累加计数器的差值进入节点 instrumentation。

把这个层次和本节主流程对齐，才能知道它保证什么、不能保证什么。

层次 4：触发器 instrumentation 使用 TriggerInstrumentation，避免把触发器执行时间误塞进普通节点 rows。

把这个层次和本节主流程对齐，才能知道它保证什么、不能保证什么。

层次 5：并行 worker 的 instrumentation 需要 DSM 布局和 leader 汇总，不能只看 leader 的 PlanState.instrument。

把这个层次和本节主流程对齐，才能知道它保证什么、不能保证什么。

本节涉及的是观测正确性，不是事务隔离本身。

观测正确性的最低要求是：不要因为收集指标而改变执行语义。

## 9. 错误路径 / 异常路径 / fallback

异常路径 1：EXEC_FLAG_EXPLAIN_ONLY 只初始化计划而不运行，PlanState 可能存在但没有运行统计。

遇到这个路径时，先判断状态是缺失、延迟、被聚合，还是被有意隐藏。

异常路径 2：MultiExecProcNode 不能自动推断返回 tuple 数，Hash 和 BitmapIndexScan 等节点需要自己补 instrumentation。

遇到这个路径时，先判断状态是缺失、延迟、被聚合，还是被有意隐藏。

异常路径 3：auto_explain 可能通过 hook 打开 instrumentation，即使用户没有显式运行 EXPLAIN。

遇到这个路径时，先判断状态是缺失、延迟、被聚合，还是被有意隐藏。

异常路径 4：节点从未执行时 ExplainNode 会输出 never executed，而不是把 NULL 统计解释为零成本。

遇到这个路径时，先判断状态是缺失、延迟、被聚合，还是被有意隐藏。

异常路径 5：Gather shutdown 期间可能传播 worker buffer usage，ExecShutdownNode_walker 会把 shutdown 包进节点 instrumentation。

遇到这个路径时，先判断状态是缺失、延迟、被聚合，还是被有意隐藏。

异常路径的课程价值在于保护诊断结论。

看不到某个指标，不等于底层事件没有发生。

## 10. 成本、资源与跨模块传播

成本点 1：最热成本是每次 ExecProcNode 的 start / stop 包装，尤其是计时和 buffer/WAL 差值计算。

它对应的跨模块传播要从执行器边界继续追到存储、WAL、JIT、MemoryContext、统计共享状态或扩展 hook。

成本点 2：PostgreSQL 把观测对象做成可选指针，普通路径只在首轮入口切换时付一次判断成本。

它对应的跨模块传播要从执行器边界继续追到存储、WAL、JIT、MemoryContext、统计共享状态或扩展 hook。

成本点 3：INSTR_TIME_SET_CURRENT_FAST 仍然不是免费操作，高频 tuple 节点会被 TIMING 放大扰动。

它对应的跨模块传播要从执行器边界继续追到存储、WAL、JIT、MemoryContext、统计共享状态或扩展 hook。

成本点 4：BUFFERS 和 WAL 会读取进程级累加计数器并计算差值，成本低于系统调用但不是零。

它对应的跨模块传播要从执行器边界继续追到存储、WAL、JIT、MemoryContext、统计共享状态或扩展 hook。

成本点 5：输出阶段 ExplainNode 遍历整棵 PlanState，成本通常小于执行成本，但对巨大计划也能变成可见延迟。

它对应的跨模块传播要从执行器边界继续追到存储、WAL、JIT、MemoryContext、统计共享状态或扩展 hook。

成本分析必须问两个问题。

第一，事件发生在热路径还是收尾路径。

第二，成本是每 tuple、每 node、每 query、每 flush，还是每输出字段支付。

## 11. 观测与诊断入口

入口 1：EXPLAIN (ANALYZE, TIMING OFF) 可以验证 rows / loops 仍存在而 time 消失。

观察结果出来后，要回到第 3 节的源码入口确认它的来源。

入口 2：对同一查询比较普通执行、EXPLAIN ANALYZE 和 EXPLAIN ANALYZE TIMING OFF，可以感受 wrapper 成本。

观察结果出来后，要回到第 3 节的源码入口确认它的来源。

入口 3：在 gdb 中断 ExecInitNode，观察 estate->es_instrument 是否非零。

观察结果出来后，要回到第 3 节的源码入口确认它的来源。

入口 4：在 gdb 中断 ExecProcNodeFirst，观察 node->instrument 是否决定入口切换。

观察结果出来后，要回到第 3 节的源码入口确认它的来源。

入口 5：在 gdb 中断 ExplainNode，观察输出 actual 字段前是否先调用 InstrEndLoop。

观察结果出来后，要回到第 3 节的源码入口确认它的来源。

诊断闭环的顺序应当是：看到现象，定位状态，回到源码，再修正解释。

## 12. 课堂实验

实验 1：构造一张小表，分别执行 EXPLAIN 与 EXPLAIN ANALYZE，比较是否有 actual 字段。

实验前先写下你预期哪个状态会变化。

实验后再用源码解释为什么变化发生在这个边界，而不是相邻模块。

实验 2：对 generate_series 进行 Seq Scan，打开 TIMING OFF，验证 loops 和 rows 仍被输出。

实验前先写下你预期哪个状态会变化。

实验后再用源码解释为什么变化发生在这个边界，而不是相邻模块。

实验 3：在源码中临时打印 estate->es_instrument，比较 ANALYZE、BUFFERS、WAL 选项下的位图。

实验前先写下你预期哪个状态会变化。

实验后再用源码解释为什么变化发生在这个边界，而不是相邻模块。

实验 4：用 auto_explain 打开慢查询日志，观察不是 EXPLAIN 命令也能产生 instrumentation。

实验前先写下你预期哪个状态会变化。

实验后再用源码解释为什么变化发生在这个边界，而不是相邻模块。

课堂实验不追求压测结果。

它只验证一个 runtime 现象能否回到源码状态。

## 13. 常见误区

误区 1：不要把 Plan 的 cost 字段当成 Instrumentation。cost 来自 planner，actual 来自 PlanState.instrument。

修正这个误区的方法是重新确认粒度：节点、查询、对象、backend、本地 pending、共享 entry 或输出格式。

误区 2：不要以为每个节点函数内部都有计时代码。多数节点依赖统一 wrapper。

修正这个误区的方法是重新确认粒度：节点、查询、对象、backend、本地 pending、共享 entry 或输出格式。

误区 3：不要把 query_instr 和 PlanState.instrument 混为一谈。前者是整条语句，后者是节点级。

修正这个误区的方法是重新确认粒度：节点、查询、对象、backend、本地 pending、共享 entry 或输出格式。

误区 4：不要把 EXPLAIN ONLY 的 PlanState 当成运行状态，它可能没有任何 loops。

修正这个误区的方法是重新确认粒度：节点、查询、对象、backend、本地 pending、共享 entry 或输出格式。

大多数误读都不是因为字段名难，而是因为把两个生命周期不同的事实放在一起比较。

## 14. 源码复盘清单

复盘 1：从 `ExplainOnePlan()` 回到 `src/backend/commands/explain.c`。

先确认 `QueryDesc.instrument_options` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：根据 es->analyze、es->timing、es->buffers、es->wal、es->io 组合 instrument_option。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：最热成本是每次 ExecProcNode 的 start / stop 包装，尤其是计时和 buffer/WAL 差值计算。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 2：从 `CreateQueryDesc()` 回到 `src/include/executor/execdesc.h`。

先确认 `QueryDesc.query_instr_options` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：把 instrument_option 写入 QueryDesc.instrument_options。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：PostgreSQL 把观测对象做成可选指针，普通路径只在首轮入口切换时付一次判断成本。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 3：从 `ExecutorStart()` 回到 `src/backend/tcop/pquery.c`。

先确认 `EState.es_instrument` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：进入 standard_ExecutorStart，创建 EState 并复制 es_instrument。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：INSTR_TIME_SET_CURRENT_FAST 仍然不是免费操作，高频 tuple 节点会被 TIMING 放大扰动。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 4：从 `ExecInitNode()` 回到 `src/backend/executor/execMain.c`。

先确认 `PlanState.instrument` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：递归构造 PlanState tree，并在 estate->es_instrument 非零时分配 NodeInstrumentation。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：BUFFERS 和 WAL 会读取进程级累加计数器并计算差值，成本低于系统调用但不是零。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 5：从 `ExecSetExecProcNode()` 回到 `src/backend/executor/execProcnode.c`。

先确认 `PlanState.ExecProcNodeReal` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：把节点真实函数保存到 ExecProcNodeReal，把外部入口先设为 ExecProcNodeFirst。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：输出阶段 ExplainNode 遍历整棵 PlanState，成本通常小于执行成本，但对巨大计划也能变成可见延迟。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 6：从 `ExecProcNodeFirst()` 回到 `src/backend/executor/instrument.c`。

先确认 `NodeInstrumentation.running` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：若 node->instrument 非 NULL，把入口切为 ExecProcNodeInstr；否则直接切回真实函数。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：最热成本是每次 ExecProcNode 的 start / stop 包装，尤其是计时和 buffer/WAL 差值计算。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 7：从 `ExecProcNodeInstr()` 回到 `src/include/executor/instrument.h`。

先确认 `ResultRelInfo.ri_TrigInstrument` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：在真实节点函数前后调用 InstrStartNode 和 InstrStopNode。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：PostgreSQL 把观测对象做成可选指针，普通路径只在首轮入口切换时付一次判断成本。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 8：从 `ExplainPrintPlan()` 回到 `src/include/nodes/execnodes.h`。

先确认 `QueryDesc.instrument_options` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：从 QueryDesc 拿到顶层 PlanState。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：INSTR_TIME_SET_CURRENT_FAST 仍然不是免费操作，高频 tuple 节点会被 TIMING 放大扰动。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 9：从 `ExplainNode()` 回到 `src/backend/executor/execParallel.c`。

先确认 `QueryDesc.query_instr_options` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：遍历 PlanState tree，读取 planstate->instrument 并输出 actual 字段。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：BUFFERS 和 WAL 会读取进程级累加计数器并计算差值，成本低于系统调用但不是零。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 10：从 `ExplainOnePlan()` 回到 `src/backend/commands/explain.c`。

先确认 `EState.es_instrument` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：根据 es->analyze、es->timing、es->buffers、es->wal、es->io 组合 instrument_option。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：输出阶段 ExplainNode 遍历整棵 PlanState，成本通常小于执行成本，但对巨大计划也能变成可见延迟。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 11：从 `CreateQueryDesc()` 回到 `src/include/executor/execdesc.h`。

先确认 `PlanState.instrument` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：把 instrument_option 写入 QueryDesc.instrument_options。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：最热成本是每次 ExecProcNode 的 start / stop 包装，尤其是计时和 buffer/WAL 差值计算。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 12：从 `ExecutorStart()` 回到 `src/backend/tcop/pquery.c`。

先确认 `PlanState.ExecProcNodeReal` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：进入 standard_ExecutorStart，创建 EState 并复制 es_instrument。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：PostgreSQL 把观测对象做成可选指针，普通路径只在首轮入口切换时付一次判断成本。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 13：从 `ExecInitNode()` 回到 `src/backend/executor/execMain.c`。

先确认 `NodeInstrumentation.running` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：递归构造 PlanState tree，并在 estate->es_instrument 非零时分配 NodeInstrumentation。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：INSTR_TIME_SET_CURRENT_FAST 仍然不是免费操作，高频 tuple 节点会被 TIMING 放大扰动。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 14：从 `ExecSetExecProcNode()` 回到 `src/backend/executor/execProcnode.c`。

先确认 `ResultRelInfo.ri_TrigInstrument` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：把节点真实函数保存到 ExecProcNodeReal，把外部入口先设为 ExecProcNodeFirst。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：BUFFERS 和 WAL 会读取进程级累加计数器并计算差值，成本低于系统调用但不是零。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 15：从 `ExecProcNodeFirst()` 回到 `src/backend/executor/instrument.c`。

先确认 `QueryDesc.instrument_options` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：若 node->instrument 非 NULL，把入口切为 ExecProcNodeInstr；否则直接切回真实函数。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：输出阶段 ExplainNode 遍历整棵 PlanState，成本通常小于执行成本，但对巨大计划也能变成可见延迟。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 16：从 `ExecProcNodeInstr()` 回到 `src/include/executor/instrument.h`。

先确认 `QueryDesc.query_instr_options` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：在真实节点函数前后调用 InstrStartNode 和 InstrStopNode。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：最热成本是每次 ExecProcNode 的 start / stop 包装，尤其是计时和 buffer/WAL 差值计算。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 17：从 `ExplainPrintPlan()` 回到 `src/include/nodes/execnodes.h`。

先确认 `EState.es_instrument` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：从 QueryDesc 拿到顶层 PlanState。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：PostgreSQL 把观测对象做成可选指针，普通路径只在首轮入口切换时付一次判断成本。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 18：从 `ExplainNode()` 回到 `src/backend/executor/execParallel.c`。

先确认 `PlanState.instrument` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：遍历 PlanState tree，读取 planstate->instrument 并输出 actual 字段。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：INSTR_TIME_SET_CURRENT_FAST 仍然不是免费操作，高频 tuple 节点会被 TIMING 放大扰动。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 19：从 `ExplainOnePlan()` 回到 `src/backend/commands/explain.c`。

先确认 `PlanState.ExecProcNodeReal` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：根据 es->analyze、es->timing、es->buffers、es->wal、es->io 组合 instrument_option。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：BUFFERS 和 WAL 会读取进程级累加计数器并计算差值，成本低于系统调用但不是零。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 20：从 `CreateQueryDesc()` 回到 `src/include/executor/execdesc.h`。

先确认 `NodeInstrumentation.running` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：把 instrument_option 写入 QueryDesc.instrument_options。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：输出阶段 ExplainNode 遍历整棵 PlanState，成本通常小于执行成本，但对巨大计划也能变成可见延迟。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 21：从 `ExecutorStart()` 回到 `src/backend/tcop/pquery.c`。

先确认 `ResultRelInfo.ri_TrigInstrument` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：进入 standard_ExecutorStart，创建 EState 并复制 es_instrument。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：最热成本是每次 ExecProcNode 的 start / stop 包装，尤其是计时和 buffer/WAL 差值计算。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 22：从 `ExecInitNode()` 回到 `src/backend/executor/execMain.c`。

先确认 `QueryDesc.instrument_options` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：递归构造 PlanState tree，并在 estate->es_instrument 非零时分配 NodeInstrumentation。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：PostgreSQL 把观测对象做成可选指针，普通路径只在首轮入口切换时付一次判断成本。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 23：从 `ExecSetExecProcNode()` 回到 `src/backend/executor/execProcnode.c`。

先确认 `QueryDesc.query_instr_options` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：把节点真实函数保存到 ExecProcNodeReal，把外部入口先设为 ExecProcNodeFirst。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：INSTR_TIME_SET_CURRENT_FAST 仍然不是免费操作，高频 tuple 节点会被 TIMING 放大扰动。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 24：从 `ExecProcNodeFirst()` 回到 `src/backend/executor/instrument.c`。

先确认 `EState.es_instrument` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：若 node->instrument 非 NULL，把入口切为 ExecProcNodeInstr；否则直接切回真实函数。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：BUFFERS 和 WAL 会读取进程级累加计数器并计算差值，成本低于系统调用但不是零。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 25：从 `ExecProcNodeInstr()` 回到 `src/include/executor/instrument.h`。

先确认 `PlanState.instrument` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：在真实节点函数前后调用 InstrStartNode 和 InstrStopNode。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：输出阶段 ExplainNode 遍历整棵 PlanState，成本通常小于执行成本，但对巨大计划也能变成可见延迟。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 26：从 `ExplainPrintPlan()` 回到 `src/include/nodes/execnodes.h`。

先确认 `PlanState.ExecProcNodeReal` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：从 QueryDesc 拿到顶层 PlanState。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：最热成本是每次 ExecProcNode 的 start / stop 包装，尤其是计时和 buffer/WAL 差值计算。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 27：从 `ExplainNode()` 回到 `src/backend/executor/execParallel.c`。

先确认 `NodeInstrumentation.running` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：遍历 PlanState tree，读取 planstate->instrument 并输出 actual 字段。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：PostgreSQL 把观测对象做成可选指针，普通路径只在首轮入口切换时付一次判断成本。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 28：从 `ExplainOnePlan()` 回到 `src/backend/commands/explain.c`。

先确认 `ResultRelInfo.ri_TrigInstrument` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：根据 es->analyze、es->timing、es->buffers、es->wal、es->io 组合 instrument_option。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：INSTR_TIME_SET_CURRENT_FAST 仍然不是免费操作，高频 tuple 节点会被 TIMING 放大扰动。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

## 15. 讨论题

1. 如果所有查询都默认启用节点级 timing，会在哪些 workload 上首先出问题？

回答时必须引用至少一个第 3 节源码入口。

2. 扩展为什么应该请求 query_instr，而不是偷读每个 PlanState.instrument？

回答时必须引用至少一个第 3 节源码入口。

3. MultiExec 节点为什么不能完全复用 ExecProcNodeInstr？

回答时必须引用至少一个第 3 节源码入口。

4. 并行查询里 leader 汇总 worker instrumentation 时，哪些信息已经不是单 backend 局部事实？

回答时必须引用至少一个第 3 节源码入口。

讨论题的目标不是扩展范围，而是检查本节唯一主问题是否已经能独立解释。

## 16. 本节小结

Instrumentation 的核心设计不是“收集尽可能多的数据”，而是“按需在 PlanState 边界挂一块可解释的运行状态”。

这个边界让 EXPLAIN ANALYZE 能回到每个节点，又让普通执行保留接近无观测的热路径。

理解这一节后，actual rows、loops、timing 的解释才有源码落点。

可以带走的可迁移规律是：

```text
先找到状态归属，再解释输出字段；先确认生命周期，再讨论成本和正确性。
```

下一节继续沿同一条主线，把这个边界推进到相邻观测机制。
