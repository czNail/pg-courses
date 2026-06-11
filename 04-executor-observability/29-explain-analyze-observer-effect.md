# PostgreSQL EXPLAIN ANALYZE 的观测扰动

## 课程定位

前置知识：已经理解 EXPLAIN ANALYZE 如何挂载 instrumentation、累计 rows/time/resources，并格式化输出。

本节唯一主问题：

```text
EXPLAIN ANALYZE 会怎样改变被观测查询的运行成本，什么时候应该关闭 timing 或改用 pg_stat / auto_explain？
```

核心矛盾：诊断需要真实执行并采集细节，但采集动作本身会改变热路径、时钟读取频率、buffer/WAL 归因、JIT 行为、并行汇总和输出开销。

学完后应能判断：能判断某次 EXPLAIN ANALYZE 的耗时是否可能被观测开销放大，并选择 TIMING OFF、BUFFERS、pg_stat_statements、auto_explain 或 profiler 的合适边界。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

第 28 节讲了输出如何形成。

本节回到一个诊断陷阱。

EXPLAIN ANALYZE 是运行查询，不是静态解释。

它把观测逻辑插入执行路径。

观测越细，扰动越大。

第 30 节开始会转向累计统计体系，作为 EXPLAIN 的互补。

本节刻意只处理一个问题：观测状态如何在执行器或统计体系中找到稳定边界。

凡是不能帮助解释这个边界的函数和字段，都只在后续课程或源码练习中出现。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
EXPLAIN ANALYZE 通过 instrument_options 改写 executor 热路径，TIMING 让每个节点调用读时钟，BUFFERS/WAL 让节点窗口计算计数差值，JIT 和 worker 汇总增加额外收集成本，最终 ExplainNode 还要遍历和格式化整棵 PlanState。
```

这句话里有三层含义。

第一层是入口边界：用户看到的 SQL 选项、统计视图或扩展 hook，必须先被转换成执行期状态。

第二层是状态边界：执行路径不能在每个热函数里直接做复杂诊断，只能在少数明确对象上累计事实。

第三层是解释边界：输出或视图只是读取这些事实，不能把估算、累计、平均、延迟刷新混成一个语义。

本节后面的所有源码 walkthrough 都围绕这三层展开。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 作用 |
| --- | --- | --- |
| 1 | `src/backend/commands/explain.c` | ExplainOnePlan 选择 INSTRUMENT_TIMER 或 INSTRUMENT_ROWS，并在执行后格式化输出。 |
| 2 | `src/backend/executor/instrument.c` | ExecProcNodeInstr、InstrStart、InstrStop 展示节点级观测的热路径成本。 |
| 3 | `src/backend/executor/execProcnode.c` | ExecProcNodeFirst 在是否启用 instrumentation 时改变节点调用入口。 |
| 4 | `src/backend/storage/buffer/bufmgr.c` | BUFFERS 指标来源于 buffer manager 计数更新。 |
| 5 | `src/backend/access/transam/xlog.c` | WAL 指标来源于 WAL 插入计数。 |
| 6 | `src/backend/jit/jit.c` | JIT instrumentation 汇总和 JIT 相关成本。 |
| 7 | `src/backend/executor/execParallel.c` | 并行 worker instrumentation、buffer、WAL 和 JIT 汇总路径。 |
| 8 | `contrib/auto_explain/auto_explain.c` | auto_explain 通过 executor hook 在真实 workload 中采样计划。 |
| 9 | `contrib/pg_stat_statements/pg_stat_statements.c` | pg_stat_statements 使用查询级 instrumentation 聚合热点，不提供单次节点级细节。 |

阅读时不要按文件名背 API。

先找入口状态，再找运行期持有者，然后看 cleanup、fallback 和输出路径。

如果一个函数只是在表格中出现，而没有参与本节主问题，就不要把它扩展成新的主线。

## 4. 关键数据结构与状态边界

本节需要关注的不是单个字段名字，而是字段组合在生命周期中的语义。

### 4.1 `instrument_options`

决定节点级 wrapper 是否启用，以及是否采样 timer、buffers、WAL、IO。

### 4.2 `ExplainState.timing`

决定 ANALYZE 时使用 INSTRUMENT_TIMER 还是 INSTRUMENT_ROWS。

### 4.3 `NodeInstrumentation.instr.need_timer`

每次 InstrStart / InstrStop 是否读取时钟。

### 4.4 `pgBufferUsage / pgWalUsage`

BUFFERS/WAL 通过差值进入节点输出，但计数本身持续维护。

### 4.5 `worker_instrument`

并行 worker 运行后需要汇总到 leader。

### 4.6 `ExplainState.str`

输出字符串缓冲，巨大计划或 JSON 输出会产生额外内存和 CPU。

### 4.7 `auto_explain GUC`

决定线上日志采样阈值、buffers、timing、nested statement 等行为。

贯穿本节的不变量是：观测状态不应该破坏被观测对象的执行语义。

## 5. 主流程源码 walkthrough

主流程按时间推进，而不是按文件归类。

```text
用户请求 EXPLAIN ANALYZE
  ->
ExecutorStart 初始化 PlanState
  ->
ExecProcNodeFirst 切换入口
  ->
InstrStart / InstrStop
  ->
ExecutorRun 真正执行查询
  ->
ExecParallelFinish / cleanup
  ->
ExplainPrintPlan / ExplainNode
  ->
用户解读耗时
```

### 5.1 `用户请求 EXPLAIN ANALYZE`

时间线推进到 `用户请求 EXPLAIN ANALYZE` 时，关键变化是：ExplainOnePlan 选择要收集的 instrumentation。

观测需求在执行前变成 executor 选项。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.2 `ExecutorStart 初始化 PlanState`

时间线推进到 `ExecutorStart 初始化 PlanState` 时，关键变化是：每个节点可能获得 NodeInstrumentation。

此时执行路径已经不同于普通查询。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.3 `ExecProcNodeFirst 切换入口`

时间线推进到 `ExecProcNodeFirst 切换入口` 时，关键变化是：有 instrumentation 时进入 ExecProcNodeInstr。

后续每个 tuple 拉取都经过 wrapper。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.4 `InstrStart / InstrStop`

时间线推进到 `InstrStart / InstrStop` 时，关键变化是：按选项读取时钟和资源计数差值。

细粒度越高，热路径附加动作越多。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.5 `ExecutorRun 真正执行查询`

时间线推进到 `ExecutorRun 真正执行查询` 时，关键变化是：DML、函数、JIT、并行 worker 都会发生真实副作用或真实资源消耗。

EXPLAIN ANALYZE 不是 dry run。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.6 `ExecParallelFinish / cleanup`

时间线推进到 `ExecParallelFinish / cleanup` 时，关键变化是：leader 汇总 worker 观测数据。

并行计划的观测成本还包含跨进程合并。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.7 `ExplainPrintPlan / ExplainNode`

时间线推进到 `ExplainPrintPlan / ExplainNode` 时，关键变化是：遍历 PlanState 并构造输出。

输出本身也会消耗 CPU 和内存。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.8 `用户解读耗时`

时间线推进到 `用户解读耗时` 时，关键变化是：需要区分被观测 workload 成本和观测机制成本。

诊断结论必须回扣选项和 workload。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

主流程结束后，用户看到的是输出或视图；源码中真正重要的是状态已经在哪个边界完成折叠。

## 6. 状态随时间推进的完整故事

把本节主问题压成一条对象生命周期，可以避免在函数清单里迷路。

阶段 1：创建或启用

观察 `instrument_options` 在 `用户请求 EXPLAIN ANALYZE` 前后的变化。

这个阶段不要急着解释用户输出。

先确认状态是否已经产生，是否仍是本地状态，是否已经被折叠成共享或输出状态。

只有完成这个判断，后面的诊断结论才不会跨层。

阶段 2：挂载或关联

观察 `ExplainState.timing` 在 `ExecutorStart 初始化 PlanState` 前后的变化。

这个阶段不要急着解释用户输出。

先确认状态是否已经产生，是否仍是本地状态，是否已经被折叠成共享或输出状态。

只有完成这个判断，后面的诊断结论才不会跨层。

阶段 3：热路径累计

观察 `NodeInstrumentation.instr.need_timer` 在 `ExecProcNodeFirst 切换入口` 前后的变化。

这个阶段不要急着解释用户输出。

先确认状态是否已经产生，是否仍是本地状态，是否已经被折叠成共享或输出状态。

只有完成这个判断，后面的诊断结论才不会跨层。

阶段 4：周期折叠

观察 `pgBufferUsage / pgWalUsage` 在 `InstrStart / InstrStop` 前后的变化。

这个阶段不要急着解释用户输出。

先确认状态是否已经产生，是否仍是本地状态，是否已经被折叠成共享或输出状态。

只有完成这个判断，后面的诊断结论才不会跨层。

阶段 5：输出或刷新

观察 `worker_instrument` 在 `ExecutorRun 真正执行查询` 前后的变化。

这个阶段不要急着解释用户输出。

先确认状态是否已经产生，是否仍是本地状态，是否已经被折叠成共享或输出状态。

只有完成这个判断，后面的诊断结论才不会跨层。

阶段 6：清理或失效

观察 `ExplainState.str` 在 `ExecParallelFinish / cleanup` 前后的变化。

这个阶段不要急着解释用户输出。

先确认状态是否已经产生，是否仍是本地状态，是否已经被折叠成共享或输出状态。

只有完成这个判断，后面的诊断结论才不会跨层。

完整故事的终点不是“函数返回”，而是用户可以稳定看到或推断这个状态。

## 7. 生命周期 / ownership / cleanup

1. 观测状态只活在本次 executor 生命周期内。

这里要同时回答谁创建、谁持有、谁在正常路径读取、谁在异常路径兜底。

2. EXPLAIN ANALYZE 执行结束前不能释放 PlanState，否则 ExplainNode 读不到统计。

这里要同时回答谁创建、谁持有、谁在正常路径读取、谁在异常路径兜底。

3. 输出字符串在 ExplainState 中累计，命令结束后释放。

这里要同时回答谁创建、谁持有、谁在正常路径读取、谁在异常路径兜底。

4. auto_explain 的状态由 hook 和 GUC 控制，随语句执行而临时创建。

这里要同时回答谁创建、谁持有、谁在正常路径读取、谁在异常路径兜底。

5. pg_stat_statements 的聚合状态跨语句保留，属于共享扩展状态，不是单次 EXPLAIN 状态。

这里要同时回答谁创建、谁持有、谁在正常路径读取、谁在异常路径兜底。

一个常见错误是把“能读到”误认为“拥有”。

执行器、统计系统和扩展 hook 中，这两者经常分离。

## 8. 正确性机制层次

正确性不是由单个计数器保证的。

层次 1：EXPLAIN ANALYZE 会执行查询；对 DML 需要事务回滚或测试环境保护。

把这个层次和本节主流程对齐，才能知道它保证什么、不能保证什么。

层次 2：TIMING OFF 只关闭节点级 timing，不关闭执行，也不关闭 rows / loops。

把这个层次和本节主流程对齐，才能知道它保证什么、不能保证什么。

层次 3：BUFFERS 可以帮助解释缓存行为，但也会让 instrumentation 保存更多差值状态。

把这个层次和本节主流程对齐，才能知道它保证什么、不能保证什么。

层次 4：JIT 统计可能让短查询看起来更慢，因为编译成本本来就集中且可被观测。

把这个层次和本节主流程对齐，才能知道它保证什么、不能保证什么。

层次 5：并行计划的 worker 汇总会让 leader 输出更完整，但也增加收集和合并成本。

把这个层次和本节主流程对齐，才能知道它保证什么、不能保证什么。

本节涉及的是观测正确性，不是事务隔离本身。

观测正确性的最低要求是：不要因为收集指标而改变执行语义。

## 9. 错误路径 / 异常路径 / fallback

异常路径 1：EXPLAIN 不带 ANALYZE 只展示计划，不运行执行节点，因此没有实际副作用。

遇到这个路径时，先判断状态是缺失、延迟、被聚合，还是被有意隐藏。

异常路径 2：EXPLAIN ANALYZE CREATE TABLE AS WITH NO DATA 有特殊执行方向，不能当普通 SELECT。

遇到这个路径时，先判断状态是缺失、延迟、被聚合，还是被有意隐藏。

异常路径 3：auto_explain 可能只在超过阈值时记录，样本天然有选择偏差。

遇到这个路径时，先判断状态是缺失、延迟、被聚合，还是被有意隐藏。

异常路径 4：pg_stat_statements 聚合多次执行，隐藏单次参数、缓存状态和 worker 倾斜。

遇到这个路径时，先判断状态是缺失、延迟、被聚合，还是被有意隐藏。

异常路径 5：SERIALIZE 选项会改变输出接收路径，用于观察结果序列化成本。

遇到这个路径时，先判断状态是缺失、延迟、被聚合，还是被有意隐藏。

异常路径的课程价值在于保护诊断结论。

看不到某个指标，不等于底层事件没有发生。

## 10. 成本、资源与跨模块传播

成本点 1：节点级时钟读取在 tuple 很小、节点很多、loops 很高时最明显。

它对应的跨模块传播要从执行器边界继续追到存储、WAL、JIT、MemoryContext、统计共享状态或扩展 hook。

成本点 2：BUFFERS/WAL 差值计算通常比时钟便宜，但会放大 PlanState instrumentation 内存。

它对应的跨模块传播要从执行器边界继续追到存储、WAL、JIT、MemoryContext、统计共享状态或扩展 hook。

成本点 3：格式化 JSON/YAML/XML 会增加字符串构造和 escaping 成本。

它对应的跨模块传播要从执行器边界继续追到存储、WAL、JIT、MemoryContext、统计共享状态或扩展 hook。

成本点 4：VERBOSE 输出表达式反解析，可能明显增加 explain 阶段 CPU。

它对应的跨模块传播要从执行器边界继续追到存储、WAL、JIT、MemoryContext、统计共享状态或扩展 hook。

成本点 5：线上频繁 auto_explain with timing/buffers 会变成新的性能变量。

它对应的跨模块传播要从执行器边界继续追到存储、WAL、JIT、MemoryContext、统计共享状态或扩展 hook。

成本分析必须问两个问题。

第一，事件发生在热路径还是收尾路径。

第二，成本是每 tuple、每 node、每 query、每 flush，还是每输出字段支付。

## 11. 观测与诊断入口

入口 1：先用 EXPLAIN ANALYZE TIMING OFF 看 rows/loops，再决定是否需要打开 timing。

观察结果出来后，要回到第 3 节的源码入口确认它的来源。

入口 2：用 pg_stat_statements 找热点 queryid，再对代表性参数做单次 EXPLAIN ANALYZE。

观察结果出来后，要回到第 3 节的源码入口确认它的来源。

入口 3：对 DML 使用 BEGIN; EXPLAIN ANALYZE ...; ROLLBACK; 避免保留副作用。

观察结果出来后，要回到第 3 节的源码入口确认它的来源。

入口 4：比较 FORMAT TEXT 与 JSON 的耗时，感受输出成本。

观察结果出来后，要回到第 3 节的源码入口确认它的来源。

入口 5：对短小高频函数查询比较 timing on/off，观察时钟扰动比例。

观察结果出来后，要回到第 3 节的源码入口确认它的来源。

诊断闭环的顺序应当是：看到现象，定位状态，回到源码，再修正解释。

## 12. 课堂实验

实验 1：构造 generate_series 交叉查询，比较 TIMING ON/OFF 的总耗时。

实验前先写下你预期哪个状态会变化。

实验后再用源码解释为什么变化发生在这个边界，而不是相邻模块。

实验 2：执行一个 INSERT 的 EXPLAIN ANALYZE 后 ROLLBACK，确认副作用不会提交但执行确实发生。

实验前先写下你预期哪个状态会变化。

实验后再用源码解释为什么变化发生在这个边界，而不是相邻模块。

实验 3：打开 auto_explain.log_min_duration，观察真实 SQL 结束后才记录计划。

实验前先写下你预期哪个状态会变化。

实验后再用源码解释为什么变化发生在这个边界，而不是相邻模块。

实验 4：用 pg_stat_statements 找 calls 高的语句，再只对一个样本做节点级 explain。

实验前先写下你预期哪个状态会变化。

实验后再用源码解释为什么变化发生在这个边界，而不是相邻模块。

课堂实验不追求压测结果。

它只验证一个 runtime 现象能否回到源码状态。

## 13. 常见误区

误区 1：不要把 EXPLAIN ANALYZE 的耗时当作完全无扰动的生产耗时。

修正这个误区的方法是重新确认粒度：节点、查询、对象、backend、本地 pending、共享 entry 或输出格式。

误区 2：不要因为 TIMING OFF 没有 time 字段就以为没有执行查询。

修正这个误区的方法是重新确认粒度：节点、查询、对象、backend、本地 pending、共享 entry 或输出格式。

误区 3：不要把 pg_stat_statements 的平均时间当成某一次参数的执行计划。

修正这个误区的方法是重新确认粒度：节点、查询、对象、backend、本地 pending、共享 entry 或输出格式。

误区 4：不要在生产 DML 上随意运行 EXPLAIN ANALYZE 而不控制事务。

修正这个误区的方法是重新确认粒度：节点、查询、对象、backend、本地 pending、共享 entry 或输出格式。

大多数误读都不是因为字段名难，而是因为把两个生命周期不同的事实放在一起比较。

## 14. 源码复盘清单

复盘 1：从 `用户请求 EXPLAIN ANALYZE` 回到 `src/backend/commands/explain.c`。

先确认 `instrument_options` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：ExplainOnePlan 选择要收集的 instrumentation。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：节点级时钟读取在 tuple 很小、节点很多、loops 很高时最明显。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 2：从 `ExecutorStart 初始化 PlanState` 回到 `src/backend/executor/instrument.c`。

先确认 `ExplainState.timing` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：每个节点可能获得 NodeInstrumentation。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：BUFFERS/WAL 差值计算通常比时钟便宜，但会放大 PlanState instrumentation 内存。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 3：从 `ExecProcNodeFirst 切换入口` 回到 `src/backend/executor/execProcnode.c`。

先确认 `NodeInstrumentation.instr.need_timer` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：有 instrumentation 时进入 ExecProcNodeInstr。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：格式化 JSON/YAML/XML 会增加字符串构造和 escaping 成本。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 4：从 `InstrStart / InstrStop` 回到 `src/backend/storage/buffer/bufmgr.c`。

先确认 `pgBufferUsage / pgWalUsage` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：按选项读取时钟和资源计数差值。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：VERBOSE 输出表达式反解析，可能明显增加 explain 阶段 CPU。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 5：从 `ExecutorRun 真正执行查询` 回到 `src/backend/access/transam/xlog.c`。

先确认 `worker_instrument` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：DML、函数、JIT、并行 worker 都会发生真实副作用或真实资源消耗。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：线上频繁 auto_explain with timing/buffers 会变成新的性能变量。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 6：从 `ExecParallelFinish / cleanup` 回到 `src/backend/jit/jit.c`。

先确认 `ExplainState.str` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：leader 汇总 worker 观测数据。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：节点级时钟读取在 tuple 很小、节点很多、loops 很高时最明显。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 7：从 `ExplainPrintPlan / ExplainNode` 回到 `src/backend/executor/execParallel.c`。

先确认 `auto_explain GUC` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：遍历 PlanState 并构造输出。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：BUFFERS/WAL 差值计算通常比时钟便宜，但会放大 PlanState instrumentation 内存。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 8：从 `用户解读耗时` 回到 `contrib/auto_explain/auto_explain.c`。

先确认 `instrument_options` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：需要区分被观测 workload 成本和观测机制成本。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：格式化 JSON/YAML/XML 会增加字符串构造和 escaping 成本。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 9：从 `用户请求 EXPLAIN ANALYZE` 回到 `contrib/pg_stat_statements/pg_stat_statements.c`。

先确认 `ExplainState.timing` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：ExplainOnePlan 选择要收集的 instrumentation。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：VERBOSE 输出表达式反解析，可能明显增加 explain 阶段 CPU。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 10：从 `ExecutorStart 初始化 PlanState` 回到 `src/backend/commands/explain.c`。

先确认 `NodeInstrumentation.instr.need_timer` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：每个节点可能获得 NodeInstrumentation。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：线上频繁 auto_explain with timing/buffers 会变成新的性能变量。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 11：从 `ExecProcNodeFirst 切换入口` 回到 `src/backend/executor/instrument.c`。

先确认 `pgBufferUsage / pgWalUsage` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：有 instrumentation 时进入 ExecProcNodeInstr。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：节点级时钟读取在 tuple 很小、节点很多、loops 很高时最明显。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 12：从 `InstrStart / InstrStop` 回到 `src/backend/executor/execProcnode.c`。

先确认 `worker_instrument` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：按选项读取时钟和资源计数差值。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：BUFFERS/WAL 差值计算通常比时钟便宜，但会放大 PlanState instrumentation 内存。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 13：从 `ExecutorRun 真正执行查询` 回到 `src/backend/storage/buffer/bufmgr.c`。

先确认 `ExplainState.str` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：DML、函数、JIT、并行 worker 都会发生真实副作用或真实资源消耗。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：格式化 JSON/YAML/XML 会增加字符串构造和 escaping 成本。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 14：从 `ExecParallelFinish / cleanup` 回到 `src/backend/access/transam/xlog.c`。

先确认 `auto_explain GUC` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：leader 汇总 worker 观测数据。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：VERBOSE 输出表达式反解析，可能明显增加 explain 阶段 CPU。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 15：从 `ExplainPrintPlan / ExplainNode` 回到 `src/backend/jit/jit.c`。

先确认 `instrument_options` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：遍历 PlanState 并构造输出。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：线上频繁 auto_explain with timing/buffers 会变成新的性能变量。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 16：从 `用户解读耗时` 回到 `src/backend/executor/execParallel.c`。

先确认 `ExplainState.timing` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：需要区分被观测 workload 成本和观测机制成本。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：节点级时钟读取在 tuple 很小、节点很多、loops 很高时最明显。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 17：从 `用户请求 EXPLAIN ANALYZE` 回到 `contrib/auto_explain/auto_explain.c`。

先确认 `NodeInstrumentation.instr.need_timer` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：ExplainOnePlan 选择要收集的 instrumentation。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：BUFFERS/WAL 差值计算通常比时钟便宜，但会放大 PlanState instrumentation 内存。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 18：从 `ExecutorStart 初始化 PlanState` 回到 `contrib/pg_stat_statements/pg_stat_statements.c`。

先确认 `pgBufferUsage / pgWalUsage` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：每个节点可能获得 NodeInstrumentation。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：格式化 JSON/YAML/XML 会增加字符串构造和 escaping 成本。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 19：从 `ExecProcNodeFirst 切换入口` 回到 `src/backend/commands/explain.c`。

先确认 `worker_instrument` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：有 instrumentation 时进入 ExecProcNodeInstr。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：VERBOSE 输出表达式反解析，可能明显增加 explain 阶段 CPU。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 20：从 `InstrStart / InstrStop` 回到 `src/backend/executor/instrument.c`。

先确认 `ExplainState.str` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：按选项读取时钟和资源计数差值。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：线上频繁 auto_explain with timing/buffers 会变成新的性能变量。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 21：从 `ExecutorRun 真正执行查询` 回到 `src/backend/executor/execProcnode.c`。

先确认 `auto_explain GUC` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：DML、函数、JIT、并行 worker 都会发生真实副作用或真实资源消耗。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：节点级时钟读取在 tuple 很小、节点很多、loops 很高时最明显。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 22：从 `ExecParallelFinish / cleanup` 回到 `src/backend/storage/buffer/bufmgr.c`。

先确认 `instrument_options` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：leader 汇总 worker 观测数据。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：BUFFERS/WAL 差值计算通常比时钟便宜，但会放大 PlanState instrumentation 内存。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 23：从 `ExplainPrintPlan / ExplainNode` 回到 `src/backend/access/transam/xlog.c`。

先确认 `ExplainState.timing` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：遍历 PlanState 并构造输出。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：格式化 JSON/YAML/XML 会增加字符串构造和 escaping 成本。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 24：从 `用户解读耗时` 回到 `src/backend/jit/jit.c`。

先确认 `NodeInstrumentation.instr.need_timer` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：需要区分被观测 workload 成本和观测机制成本。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：VERBOSE 输出表达式反解析，可能明显增加 explain 阶段 CPU。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 25：从 `用户请求 EXPLAIN ANALYZE` 回到 `src/backend/executor/execParallel.c`。

先确认 `pgBufferUsage / pgWalUsage` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：ExplainOnePlan 选择要收集的 instrumentation。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：线上频繁 auto_explain with timing/buffers 会变成新的性能变量。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 26：从 `ExecutorStart 初始化 PlanState` 回到 `contrib/auto_explain/auto_explain.c`。

先确认 `worker_instrument` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：每个节点可能获得 NodeInstrumentation。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：节点级时钟读取在 tuple 很小、节点很多、loops 很高时最明显。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 27：从 `ExecProcNodeFirst 切换入口` 回到 `contrib/pg_stat_statements/pg_stat_statements.c`。

先确认 `ExplainState.str` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：有 instrumentation 时进入 ExecProcNodeInstr。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：BUFFERS/WAL 差值计算通常比时钟便宜，但会放大 PlanState instrumentation 内存。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 28：从 `InstrStart / InstrStop` 回到 `src/backend/commands/explain.c`。

先确认 `auto_explain GUC` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：按选项读取时钟和资源计数差值。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：格式化 JSON/YAML/XML 会增加字符串构造和 escaping 成本。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 29：从 `ExecutorRun 真正执行查询` 回到 `src/backend/executor/instrument.c`。

先确认 `instrument_options` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：DML、函数、JIT、并行 worker 都会发生真实副作用或真实资源消耗。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：VERBOSE 输出表达式反解析，可能明显增加 explain 阶段 CPU。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

## 15. 讨论题

1. 什么样的 workload 最容易被节点级 timing 扰动？

回答时必须引用至少一个第 3 节源码入口。

2. 为什么 TIMING OFF 是诊断大多数 row-flow 问题的合理第一步？

回答时必须引用至少一个第 3 节源码入口。

3. auto_explain 与手工 EXPLAIN ANALYZE 的样本偏差分别是什么？

回答时必须引用至少一个第 3 节源码入口。

4. 如果 EXPLAIN 输出巨大，formatter 成本是否应该算进查询诊断？

回答时必须引用至少一个第 3 节源码入口。

讨论题的目标不是扩展范围，而是检查本节唯一主问题是否已经能独立解释。

## 16. 本节小结

EXPLAIN ANALYZE 是带插桩的真实执行。

它的价值在于把 runtime 状态落到节点边界，代价是改变热路径并增加输出成本。

严肃诊断需要在 EXPLAIN、pg_stat、auto_explain 和 profiler 之间选择合适粒度。

可以带走的可迁移规律是：

```text
先找到状态归属，再解释输出字段；先确认生命周期，再讨论成本和正确性。
```

下一节继续沿同一条主线，把这个边界推进到相邻观测机制。
