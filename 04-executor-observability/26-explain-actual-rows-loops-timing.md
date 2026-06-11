# PostgreSQL Actual Rows、Loops 与 Timing 的采样边界

## 课程定位

前置知识：已经知道 EXPLAIN ANALYZE 会给 PlanState 挂 NodeInstrumentation，也知道执行器节点通过 ExecProcNode 拉取 tuple。

本节唯一主问题：

```text
actual rows、loops、startup time 和 total time 到底在什么边界上累计，为什么 loops 会改变用户对时间和行数的解读？
```

核心矛盾：用户需要把 EXPLAIN 输出理解成每个节点的真实执行事实，但执行器是 demand-driven、可 rescan、可短路、可异步的，单次函数调用和用户看到的一行 actual 字段并不一一对应。

学完后应能判断：能判断一行 actual 输出是在说每轮平均值、累计值还是未执行状态，并能回到 InstrStartNode、InstrStopNode、InstrEndLoop 解释。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

第 25 节回答了 instrumentation 如何挂到 PlanState。

本节只追问挂上之后如何累计。

actual rows 不是优化器估计。

loops 不是循环语法次数。

startup time 也不是 ExecutorStart 的耗时。

这些字段都来自 NodeInstrumentation 的运行周期。

本节刻意只处理一个问题：观测状态如何在执行器或统计体系中找到稳定边界。

凡是不能帮助解释这个边界的函数和字段，都只在后续课程或源码练习中出现。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
ExecProcNodeInstr 在每次节点入口前后采样，InstrStopNode 把本轮 tuple 数和计时放入临时计数，InstrEndLoop 在一个执行周期结束时把 firsttuple、counter、tuplecount 折叠进 startup、total、ntuples、nloops，ExplainNode 再按 nloops 做平均展示。
```

这句话里有三层含义。

第一层是入口边界：用户看到的 SQL 选项、统计视图或扩展 hook，必须先被转换成执行期状态。

第二层是状态边界：执行路径不能在每个热函数里直接做复杂诊断，只能在少数明确对象上累计事实。

第三层是解释边界：输出或视图只是读取这些事实，不能把估算、累计、平均、延迟刷新混成一个语义。

本节后面的所有源码 walkthrough 都围绕这三层展开。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 作用 |
| --- | --- | --- |
| 1 | `src/backend/executor/instrument.c` | InstrStartNode、InstrStopNode、InstrEndLoop 和 ExecProcNodeInstr 是 rows、loops、timing 的累计核心。 |
| 2 | `src/include/executor/instrument.h` | NodeInstrumentation 定义 running、counter、firsttuple、tuplecount、startup、ntuples、nloops、nfiltered 字段。 |
| 3 | `src/backend/executor/execProcnode.c` | ExecProcNodeFirst 决定是否进入 ExecProcNodeInstr，ExecShutdownNode_walker 处理 shutdown 时的观测归属。 |
| 4 | `src/backend/commands/explain.c` | ExplainNode 调用 InstrEndLoop，并把 startup、total、ntuples 除以 nloops 输出。 |
| 5 | `src/backend/executor/nodeBitmapIndexscan.c` | BitmapIndexScan 这种 MultiExec 节点手工调用 InstrStartNode / InstrStopNode。 |
| 6 | `src/backend/executor/nodeHash.c` | Hash 节点构建 hash table 时自己报告 tuple 数，说明 MultiExec 的边界不同。 |
| 7 | `src/backend/executor/nodeLimit.c` | Limit 体现上层短路如何让子节点 loops 和 rows 不同于全表事实。 |
| 8 | `src/backend/executor/nodeNestloop.c` | Nested Loop 的内侧节点 rescan 是 loops 放大的常见来源。 |

阅读时不要按文件名背 API。

先找入口状态，再找运行期持有者，然后看 cleanup、fallback 和输出路径。

如果一个函数只是在表格中出现，而没有参与本节主问题，就不要把它扩展成新的主线。

## 4. 关键数据结构与状态边界

本节需要关注的不是单个字段名字，而是字段组合在生命周期中的语义。

### 4.1 `NodeInstrumentation.counter`

当前执行周期内累计的运行时间，尚未进入最终 total。

### 4.2 `NodeInstrumentation.firsttuple`

本周期第一次产出 tuple 时的 counter，用于 startup time。

### 4.3 `NodeInstrumentation.tuplecount`

本周期产出的 tuple 数，InstrEndLoop 后进入 ntuples。

### 4.4 `NodeInstrumentation.nloops`

完成了多少个执行周期，是 ExplainNode 计算平均 rows/time 的除数。

### 4.5 `NodeInstrumentation.nfiltered1`

scan qual 或 join qual 过滤掉的 tuple 数，输出为 Rows Removed by Filter 或 Join Filter。

### 4.6 `NodeInstrumentation.nfiltered2`

节点第二类过滤计数，具体标签由 ExplainNode 按节点类型解释。

### 4.7 `PlanState.chgParam`

参数变化会触发 rescan，是 loops 放大的重要来源。

贯穿本节的不变量是：观测状态不应该破坏被观测对象的执行语义。

## 5. 主流程源码 walkthrough

主流程按时间推进，而不是按文件归类。

```text
ExecProcNodeInstr()
  ->
真实节点函数返回 slot
  ->
InstrStopNode()
  ->
ExecProcNode() 被上层重复调用
  ->
InstrEndLoop()
  ->
ExecReScan()
  ->
ExplainNode()
  ->
show_instrumentation_count()
```

### 5.1 `ExecProcNodeInstr()`

时间线推进到 `ExecProcNodeInstr()` 时，关键变化是：调用 InstrStartNode 后进入真实节点函数。

这定义一次节点调用的时间窗口。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.2 `真实节点函数返回 slot`

时间线推进到 `真实节点函数返回 slot` 时，关键变化是：返回非空 slot 时本次调用计 1 行，返回空 slot 时计 0 行。

actual rows 来自 tuple 产出边界，而不是扫描过的物理行数。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.3 `InstrStopNode()`

时间线推进到 `InstrStopNode()` 时，关键变化是：增加 tuplecount，累加 counter，并在第一次产出时记录 firsttuple。

startup time 是本周期第一次成功产出的时间。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.4 `ExecProcNode() 被上层重复调用`

时间线推进到 `ExecProcNode() 被上层重复调用` 时，关键变化是：上层节点继续拉取，直到子节点返回空 slot。

一轮执行周期可以包含很多次函数调用。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.5 `InstrEndLoop()`

时间线推进到 `InstrEndLoop()` 时，关键变化是：把 firsttuple、counter、tuplecount 折叠到 startup、total、ntuples、nloops。

loops 在这里增加，而不是每次 ExecProcNode 增加。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.6 `ExecReScan()`

时间线推进到 `ExecReScan()` 时，关键变化是：在参数化计划、Nested Loop 内侧或 Material 重扫时开始新周期。

用户看到的 loops 往往来自 rescan，而不是 SQL 中的 loop。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.7 `ExplainNode()`

时间线推进到 `ExplainNode()` 时，关键变化是：在打印节点前强制 InstrEndLoop，并用 nloops 计算平均 rows 和 time。

输出中的 actual rows/time 是每 loop 平均值。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.8 `show_instrumentation_count()`

时间线推进到 `show_instrumentation_count()` 时，关键变化是：把 nfiltered 按 nloops 平均后输出过滤行数。

Rows Removed by Filter 也受 loops 解释影响。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

主流程结束后，用户看到的是输出或视图；源码中真正重要的是状态已经在哪个边界完成折叠。

## 6. 状态随时间推进的完整故事

把本节主问题压成一条对象生命周期，可以避免在函数清单里迷路。

阶段 1：创建或启用

观察 `NodeInstrumentation.counter` 在 `ExecProcNodeInstr()` 前后的变化。

这个阶段不要急着解释用户输出。

先确认状态是否已经产生，是否仍是本地状态，是否已经被折叠成共享或输出状态。

只有完成这个判断，后面的诊断结论才不会跨层。

阶段 2：挂载或关联

观察 `NodeInstrumentation.firsttuple` 在 `真实节点函数返回 slot` 前后的变化。

这个阶段不要急着解释用户输出。

先确认状态是否已经产生，是否仍是本地状态，是否已经被折叠成共享或输出状态。

只有完成这个判断，后面的诊断结论才不会跨层。

阶段 3：热路径累计

观察 `NodeInstrumentation.tuplecount` 在 `InstrStopNode()` 前后的变化。

这个阶段不要急着解释用户输出。

先确认状态是否已经产生，是否仍是本地状态，是否已经被折叠成共享或输出状态。

只有完成这个判断，后面的诊断结论才不会跨层。

阶段 4：周期折叠

观察 `NodeInstrumentation.nloops` 在 `ExecProcNode() 被上层重复调用` 前后的变化。

这个阶段不要急着解释用户输出。

先确认状态是否已经产生，是否仍是本地状态，是否已经被折叠成共享或输出状态。

只有完成这个判断，后面的诊断结论才不会跨层。

阶段 5：输出或刷新

观察 `NodeInstrumentation.nfiltered1` 在 `InstrEndLoop()` 前后的变化。

这个阶段不要急着解释用户输出。

先确认状态是否已经产生，是否仍是本地状态，是否已经被折叠成共享或输出状态。

只有完成这个判断，后面的诊断结论才不会跨层。

阶段 6：清理或失效

观察 `NodeInstrumentation.nfiltered2` 在 `ExecReScan()` 前后的变化。

这个阶段不要急着解释用户输出。

先确认状态是否已经产生，是否仍是本地状态，是否已经被折叠成共享或输出状态。

只有完成这个判断，后面的诊断结论才不会跨层。

完整故事的终点不是“函数返回”，而是用户可以稳定看到或推断这个状态。

## 7. 生命周期 / ownership / cleanup

1. counter、firsttuple、tuplecount 属于当前运行周期，InstrEndLoop 后会清零。

这里要同时回答谁创建、谁持有、谁在正常路径读取、谁在异常路径兜底。

2. startup、total、ntuples、nloops 属于已完成周期的累计结果，打印时按 loops 展示平均值。

这里要同时回答谁创建、谁持有、谁在正常路径读取、谁在异常路径兜底。

3. 节点从未产出 tuple 但运行过，也可能有 total time 和 rows=0。

这里要同时回答谁创建、谁持有、谁在正常路径读取、谁在异常路径兜底。

4. 节点从未执行则 nloops 为 0，ExplainNode 输出 never executed。

这里要同时回答谁创建、谁持有、谁在正常路径读取、谁在异常路径兜底。

5. ERROR 中断执行时，未打印的 instrumentation 随 executor context 释放，不形成可查询历史。

这里要同时回答谁创建、谁持有、谁在正常路径读取、谁在异常路径兜底。

一个常见错误是把“能读到”误认为“拥有”。

执行器、统计系统和扩展 hook 中，这两者经常分离。

## 8. 正确性机制层次

正确性不是由单个计数器保证的。

层次 1：rows 是节点向父节点返回的 tuple 数，不是表访问层看到的 tuple 数。

把这个层次和本节主流程对齐，才能知道它保证什么、不能保证什么。

层次 2：loops 是执行周期数，常由 rescan、参数化 inner scan、子计划重复调用造成。

把这个层次和本节主流程对齐，才能知道它保证什么、不能保证什么。

层次 3：startup time 是第一次返回 tuple 前消耗，不等于节点初始化时间。

把这个层次和本节主流程对齐，才能知道它保证什么、不能保证什么。

层次 4：total time 是本节点 wrapper 覆盖的时间，通常包含子节点时间，因此不能直接当作独占时间。

把这个层次和本节主流程对齐，才能知道它保证什么、不能保证什么。

层次 5：TIMING OFF 会避免节点级时间采样，但 rows 和 loops 仍可通过 INSTRUMENT_ROWS 得到。

把这个层次和本节主流程对齐，才能知道它保证什么、不能保证什么。

本节涉及的是观测正确性，不是事务隔离本身。

观测正确性的最低要求是：不要因为收集指标而改变执行语义。

## 9. 错误路径 / 异常路径 / fallback

异常路径 1：MultiExecProcNode 无法统一知道返回了多少 tuple，需要节点自己调用 InstrStopNode 或 InstrUpdateTupleCount。

遇到这个路径时，先判断状态是缺失、延迟、被聚合，还是被有意隐藏。

异常路径 2：异步节点使用 async_mode 处理 firsttuple 边界，避免第一次真正产出被错误记录。

遇到这个路径时，先判断状态是缺失、延迟、被聚合，还是被有意隐藏。

异常路径 3：Limit 可能让子节点早停，子节点 rows 不是基表全量行数。

遇到这个路径时，先判断状态是缺失、延迟、被聚合，还是被有意隐藏。

异常路径 4：Nested Loop 内侧节点可能 loops 很高，但每 loop rows 很少，这正是点查计划的典型形态。

遇到这个路径时，先判断状态是缺失、延迟、被聚合，还是被有意隐藏。

异常路径 5：并行 worker 的 rows 和 loops 需要在 ExplainNode 中分 worker 展示或汇总。

遇到这个路径时，先判断状态是缺失、延迟、被聚合，还是被有意隐藏。

异常路径的课程价值在于保护诊断结论。

看不到某个指标，不等于底层事件没有发生。

## 10. 成本、资源与跨模块传播

成本点 1：打开 timing 时，每次节点调用都要读取时钟，高频小 tuple 计划会被明显扰动。

它对应的跨模块传播要从执行器边界继续追到存储、WAL、JIT、MemoryContext、统计共享状态或扩展 hook。

成本点 2：loops 越大，InstrEndLoop 的解释越重要，否则总时间和平均时间会被混读。

它对应的跨模块传播要从执行器边界继续追到存储、WAL、JIT、MemoryContext、统计共享状态或扩展 hook。

成本点 3：过滤计数是附加计数，开销低于时钟采样，但仍在节点热路径附近。

它对应的跨模块传播要从执行器边界继续追到存储、WAL、JIT、MemoryContext、统计共享状态或扩展 hook。

成本点 4：返回空 slot 也会进入 InstrStopNode，只是 tuple 数为 0。

它对应的跨模块传播要从执行器边界继续追到存储、WAL、JIT、MemoryContext、统计共享状态或扩展 hook。

成本点 5：节点级 total 通常包含子节点等待和 CPU，因此多个节点时间相加会重复计算。

它对应的跨模块传播要从执行器边界继续追到存储、WAL、JIT、MemoryContext、统计共享状态或扩展 hook。

成本分析必须问两个问题。

第一，事件发生在热路径还是收尾路径。

第二，成本是每 tuple、每 node、每 query、每 flush，还是每输出字段支付。

## 11. 观测与诊断入口

入口 1：比较 Nested Loop 内侧 Index Scan 的 loops 与外侧 rows，能看到参数化 inner scan 的重扫。

观察结果出来后，要回到第 3 节的源码入口确认它的来源。

入口 2：使用 LIMIT 可以看到下层节点 rows 被上层需求截断。

观察结果出来后，要回到第 3 节的源码入口确认它的来源。

入口 3：使用 TIMING OFF 可以保留 rows / loops 来降低时钟扰动。

观察结果出来后，要回到第 3 节的源码入口确认它的来源。

入口 4：在 gdb 观察 InstrEndLoop 前后的 tuplecount、ntuples、nloops，能看到周期折叠。

观察结果出来后，要回到第 3 节的源码入口确认它的来源。

入口 5：观察 Rows Removed by Filter 时必须同时看 loops，否则过滤行数会被误读。

观察结果出来后，要回到第 3 节的源码入口确认它的来源。

诊断闭环的顺序应当是：看到现象，定位状态，回到源码，再修正解释。

## 12. 课堂实验

实验 1：创建外表 100 行、内表带索引，执行 Nested Loop，观察内侧 Index Scan loops。

实验前先写下你预期哪个状态会变化。

实验后再用源码解释为什么变化发生在这个边界，而不是相邻模块。

实验 2：给查询加 LIMIT 1，比较 Seq Scan 的 actual rows 与全表行数。

实验前先写下你预期哪个状态会变化。

实验后再用源码解释为什么变化发生在这个边界，而不是相邻模块。

实验 3：执行 EXPLAIN (ANALYZE, TIMING OFF) 与默认 ANALYZE，比较输出字段和耗时。

实验前先写下你预期哪个状态会变化。

实验后再用源码解释为什么变化发生在这个边界，而不是相邻模块。

实验 4：在 nodeBitmapIndexscan.c 中查找手工 instrumentation，理解 MultiExec 例外。

实验前先写下你预期哪个状态会变化。

实验后再用源码解释为什么变化发生在这个边界，而不是相邻模块。

课堂实验不追求压测结果。

它只验证一个 runtime 现象能否回到源码状态。

## 13. 常见误区

误区 1：不要把 actual rows 理解成扫描过的 heap tuple 数。它是节点向父节点交付的行数。

修正这个误区的方法是重新确认粒度：节点、查询、对象、backend、本地 pending、共享 entry 或输出格式。

误区 2：不要把 loops 当成 SQL 循环。它是 executor 对节点执行周期的计数。

修正这个误区的方法是重新确认粒度：节点、查询、对象、backend、本地 pending、共享 entry 或输出格式。

误区 3：不要把 startup time 当成 ExecInitNode 时间。初始化不在节点 wrapper 中。

修正这个误区的方法是重新确认粒度：节点、查询、对象、backend、本地 pending、共享 entry 或输出格式。

误区 4：不要把节点 total time 相加当成查询总时间。父节点时间通常包含子节点。

修正这个误区的方法是重新确认粒度：节点、查询、对象、backend、本地 pending、共享 entry 或输出格式。

大多数误读都不是因为字段名难，而是因为把两个生命周期不同的事实放在一起比较。

## 14. 源码复盘清单

复盘 1：从 `ExecProcNodeInstr()` 回到 `src/backend/executor/instrument.c`。

先确认 `NodeInstrumentation.counter` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：调用 InstrStartNode 后进入真实节点函数。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：打开 timing 时，每次节点调用都要读取时钟，高频小 tuple 计划会被明显扰动。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 2：从 `真实节点函数返回 slot` 回到 `src/include/executor/instrument.h`。

先确认 `NodeInstrumentation.firsttuple` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：返回非空 slot 时本次调用计 1 行，返回空 slot 时计 0 行。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：loops 越大，InstrEndLoop 的解释越重要，否则总时间和平均时间会被混读。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 3：从 `InstrStopNode()` 回到 `src/backend/executor/execProcnode.c`。

先确认 `NodeInstrumentation.tuplecount` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：增加 tuplecount，累加 counter，并在第一次产出时记录 firsttuple。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：过滤计数是附加计数，开销低于时钟采样，但仍在节点热路径附近。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 4：从 `ExecProcNode() 被上层重复调用` 回到 `src/backend/commands/explain.c`。

先确认 `NodeInstrumentation.nloops` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：上层节点继续拉取，直到子节点返回空 slot。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：返回空 slot 也会进入 InstrStopNode，只是 tuple 数为 0。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 5：从 `InstrEndLoop()` 回到 `src/backend/executor/nodeBitmapIndexscan.c`。

先确认 `NodeInstrumentation.nfiltered1` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：把 firsttuple、counter、tuplecount 折叠到 startup、total、ntuples、nloops。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：节点级 total 通常包含子节点等待和 CPU，因此多个节点时间相加会重复计算。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 6：从 `ExecReScan()` 回到 `src/backend/executor/nodeHash.c`。

先确认 `NodeInstrumentation.nfiltered2` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：在参数化计划、Nested Loop 内侧或 Material 重扫时开始新周期。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：打开 timing 时，每次节点调用都要读取时钟，高频小 tuple 计划会被明显扰动。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 7：从 `ExplainNode()` 回到 `src/backend/executor/nodeLimit.c`。

先确认 `PlanState.chgParam` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：在打印节点前强制 InstrEndLoop，并用 nloops 计算平均 rows 和 time。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：loops 越大，InstrEndLoop 的解释越重要，否则总时间和平均时间会被混读。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 8：从 `show_instrumentation_count()` 回到 `src/backend/executor/nodeNestloop.c`。

先确认 `NodeInstrumentation.counter` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：把 nfiltered 按 nloops 平均后输出过滤行数。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：过滤计数是附加计数，开销低于时钟采样，但仍在节点热路径附近。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 9：从 `ExecProcNodeInstr()` 回到 `src/backend/executor/instrument.c`。

先确认 `NodeInstrumentation.firsttuple` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：调用 InstrStartNode 后进入真实节点函数。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：返回空 slot 也会进入 InstrStopNode，只是 tuple 数为 0。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 10：从 `真实节点函数返回 slot` 回到 `src/include/executor/instrument.h`。

先确认 `NodeInstrumentation.tuplecount` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：返回非空 slot 时本次调用计 1 行，返回空 slot 时计 0 行。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：节点级 total 通常包含子节点等待和 CPU，因此多个节点时间相加会重复计算。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 11：从 `InstrStopNode()` 回到 `src/backend/executor/execProcnode.c`。

先确认 `NodeInstrumentation.nloops` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：增加 tuplecount，累加 counter，并在第一次产出时记录 firsttuple。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：打开 timing 时，每次节点调用都要读取时钟，高频小 tuple 计划会被明显扰动。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 12：从 `ExecProcNode() 被上层重复调用` 回到 `src/backend/commands/explain.c`。

先确认 `NodeInstrumentation.nfiltered1` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：上层节点继续拉取，直到子节点返回空 slot。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：loops 越大，InstrEndLoop 的解释越重要，否则总时间和平均时间会被混读。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 13：从 `InstrEndLoop()` 回到 `src/backend/executor/nodeBitmapIndexscan.c`。

先确认 `NodeInstrumentation.nfiltered2` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：把 firsttuple、counter、tuplecount 折叠到 startup、total、ntuples、nloops。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：过滤计数是附加计数，开销低于时钟采样，但仍在节点热路径附近。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 14：从 `ExecReScan()` 回到 `src/backend/executor/nodeHash.c`。

先确认 `PlanState.chgParam` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：在参数化计划、Nested Loop 内侧或 Material 重扫时开始新周期。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：返回空 slot 也会进入 InstrStopNode，只是 tuple 数为 0。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 15：从 `ExplainNode()` 回到 `src/backend/executor/nodeLimit.c`。

先确认 `NodeInstrumentation.counter` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：在打印节点前强制 InstrEndLoop，并用 nloops 计算平均 rows 和 time。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：节点级 total 通常包含子节点等待和 CPU，因此多个节点时间相加会重复计算。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 16：从 `show_instrumentation_count()` 回到 `src/backend/executor/nodeNestloop.c`。

先确认 `NodeInstrumentation.firsttuple` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：把 nfiltered 按 nloops 平均后输出过滤行数。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：打开 timing 时，每次节点调用都要读取时钟，高频小 tuple 计划会被明显扰动。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 17：从 `ExecProcNodeInstr()` 回到 `src/backend/executor/instrument.c`。

先确认 `NodeInstrumentation.tuplecount` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：调用 InstrStartNode 后进入真实节点函数。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：loops 越大，InstrEndLoop 的解释越重要，否则总时间和平均时间会被混读。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 18：从 `真实节点函数返回 slot` 回到 `src/include/executor/instrument.h`。

先确认 `NodeInstrumentation.nloops` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：返回非空 slot 时本次调用计 1 行，返回空 slot 时计 0 行。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：过滤计数是附加计数，开销低于时钟采样，但仍在节点热路径附近。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 19：从 `InstrStopNode()` 回到 `src/backend/executor/execProcnode.c`。

先确认 `NodeInstrumentation.nfiltered1` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：增加 tuplecount，累加 counter，并在第一次产出时记录 firsttuple。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：返回空 slot 也会进入 InstrStopNode，只是 tuple 数为 0。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 20：从 `ExecProcNode() 被上层重复调用` 回到 `src/backend/commands/explain.c`。

先确认 `NodeInstrumentation.nfiltered2` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：上层节点继续拉取，直到子节点返回空 slot。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：节点级 total 通常包含子节点等待和 CPU，因此多个节点时间相加会重复计算。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 21：从 `InstrEndLoop()` 回到 `src/backend/executor/nodeBitmapIndexscan.c`。

先确认 `PlanState.chgParam` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：把 firsttuple、counter、tuplecount 折叠到 startup、total、ntuples、nloops。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：打开 timing 时，每次节点调用都要读取时钟，高频小 tuple 计划会被明显扰动。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 22：从 `ExecReScan()` 回到 `src/backend/executor/nodeHash.c`。

先确认 `NodeInstrumentation.counter` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：在参数化计划、Nested Loop 内侧或 Material 重扫时开始新周期。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：loops 越大，InstrEndLoop 的解释越重要，否则总时间和平均时间会被混读。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 23：从 `ExplainNode()` 回到 `src/backend/executor/nodeLimit.c`。

先确认 `NodeInstrumentation.firsttuple` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：在打印节点前强制 InstrEndLoop，并用 nloops 计算平均 rows 和 time。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：过滤计数是附加计数，开销低于时钟采样，但仍在节点热路径附近。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 24：从 `show_instrumentation_count()` 回到 `src/backend/executor/nodeNestloop.c`。

先确认 `NodeInstrumentation.tuplecount` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：把 nfiltered 按 nloops 平均后输出过滤行数。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：返回空 slot 也会进入 InstrStopNode，只是 tuple 数为 0。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 25：从 `ExecProcNodeInstr()` 回到 `src/backend/executor/instrument.c`。

先确认 `NodeInstrumentation.nloops` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：调用 InstrStartNode 后进入真实节点函数。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：节点级 total 通常包含子节点等待和 CPU，因此多个节点时间相加会重复计算。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 26：从 `真实节点函数返回 slot` 回到 `src/include/executor/instrument.h`。

先确认 `NodeInstrumentation.nfiltered1` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：返回非空 slot 时本次调用计 1 行，返回空 slot 时计 0 行。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：打开 timing 时，每次节点调用都要读取时钟，高频小 tuple 计划会被明显扰动。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 27：从 `InstrStopNode()` 回到 `src/backend/executor/execProcnode.c`。

先确认 `NodeInstrumentation.nfiltered2` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：增加 tuplecount，累加 counter，并在第一次产出时记录 firsttuple。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：loops 越大，InstrEndLoop 的解释越重要，否则总时间和平均时间会被混读。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 28：从 `ExecProcNode() 被上层重复调用` 回到 `src/backend/commands/explain.c`。

先确认 `PlanState.chgParam` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：上层节点继续拉取，直到子节点返回空 slot。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：过滤计数是附加计数，开销低于时钟采样，但仍在节点热路径附近。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 29：从 `InstrEndLoop()` 回到 `src/backend/executor/nodeBitmapIndexscan.c`。

先确认 `NodeInstrumentation.counter` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：把 firsttuple、counter、tuplecount 折叠到 startup、total、ntuples、nloops。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：返回空 slot 也会进入 InstrStopNode，只是 tuple 数为 0。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

## 15. 讨论题

1. 为什么 ExplainNode 要把 rows 和 time 除以 nloops，而不是直接展示总数？

回答时必须引用至少一个第 3 节源码入口。

2. 如果某节点频繁返回空 slot，它的 rows 和 total time 应该如何解释？

回答时必须引用至少一个第 3 节源码入口。

3. Nested Loop 内侧 Index Scan 的 loops 很大一定是坏计划吗？

回答时必须引用至少一个第 3 节源码入口。

4. TIMING OFF 牺牲了什么信息，换回了什么诊断可靠性？

回答时必须引用至少一个第 3 节源码入口。

讨论题的目标不是扩展范围，而是检查本节唯一主问题是否已经能独立解释。

## 16. 本节小结

actual rows、loops 和 timing 的关键在采样边界。

ExecProcNodeInstr 记录一次调用，InstrEndLoop 定义一次周期，ExplainNode 按周期解释输出。

看懂 loops，才能避免把平均值、累计值和未执行状态混在一起。

可以带走的可迁移规律是：

```text
先找到状态归属，再解释输出字段；先确认生命周期，再讨论成本和正确性。
```

下一节继续沿同一条主线，把这个边界推进到相邻观测机制。
