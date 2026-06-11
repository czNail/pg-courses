# PostgreSQL EXPLAIN Buffers、WAL、JIT 与 Memory 指标来源

## 课程定位

前置知识：已经理解节点级 Instrumentation 的挂载和 rows / loops / timing 的累计边界。

本节唯一主问题：

```text
EXPLAIN 中 BUFFERS、WAL、JIT 和 MEMORY 指标分别从哪里来，为什么有些能落到节点级，有些只能作为查询级或规划级事实解释？
```

核心矛盾：用户希望一个 EXPLAIN 输出能解释所有资源消耗，但 PostgreSQL 的资源计数分散在 buffer manager、WAL insertion、JIT、MemoryContext、并行 worker 和 explain 输出层，指标粒度天然不一致。

学完后应能判断：能把 EXPLAIN 的资源字段映射回 pgBufferUsage、pgWalUsage、JitInstrumentation 和 MemoryContextCounters，并知道哪些字段不能过度归因到单个节点。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

第 26 节解释了 rows、loops 和 timing。

本节继续处理资源类指标。

资源指标比时间更接近内部状态。

但它们的归属边界更不统一。

Buffers 和 WAL 可以通过差值挂到 NodeInstrumentation。

JIT 和 Memory 常常需要按查询级或节点私有实现解释。

本节刻意只处理一个问题：观测状态如何在执行器或统计体系中找到稳定边界。

凡是不能帮助解释这个边界的函数和字段，都只在后续课程或源码练习中出现。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
buffer manager 和 WAL insertion 持续累加 backend-local pgBufferUsage / pgWalUsage，Instrumentation 在节点 start / stop 之间取差值；JIT 使用 EState 中的 JitInstrumentation 汇总；planning memory 通过 MemoryContextMemConsumed 统计，节点内存则由 Sort、Hash、Agg 等节点各自提供。
```

这句话里有三层含义。

第一层是入口边界：用户看到的 SQL 选项、统计视图或扩展 hook，必须先被转换成执行期状态。

第二层是状态边界：执行路径不能在每个热函数里直接做复杂诊断，只能在少数明确对象上累计事实。

第三层是解释边界：输出或视图只是读取这些事实，不能把估算、累计、平均、延迟刷新混成一个语义。

本节后面的所有源码 walkthrough 都围绕这三层展开。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 作用 |
| --- | --- | --- |
| 1 | `src/include/executor/instrument.h` | BufferUsage、WalUsage、Instrumentation 中的 bufusage_start、walusage_start、bufusage、walusage。 |
| 2 | `src/backend/executor/instrument.c` | InstrStart / InstrStop 取 pgBufferUsage 和 pgWalUsage 差值，InstrStartParallelQuery 汇总并行 worker。 |
| 3 | `src/backend/storage/buffer/bufmgr.c` | shared buffer hit/read/dirty/write 计数更新的主要位置。 |
| 4 | `src/backend/storage/buffer/localbuf.c` | local buffer 计数更新。 |
| 5 | `src/backend/storage/file/buffile.c` | 临时文件 block 读写和 temp I/O timing 计数。 |
| 6 | `src/backend/access/transam/xlog.c` | pgWalUsage 的 wal_records、wal_fpi、wal_bytes、wal_buffers_full 更新。 |
| 7 | `src/backend/utils/activity/pgstat_io.c` | track_io_timing 打开时把 I/O 时间同时记入 pgBufferUsage。 |
| 8 | `src/backend/commands/explain.c` | show_buffer_usage、show_wal_usage、ExplainPrintJITSummary、show_memory_counters 输出资源指标。 |
| 9 | `src/include/jit/jit.h` | JitInstrumentation 和 SharedJitInstrumentation 的结构。 |
| 10 | `src/backend/utils/mmgr/mcxt.c` | MemoryContextMemConsumed 计算规划阶段内存使用。 |

阅读时不要按文件名背 API。

先找入口状态，再找运行期持有者，然后看 cleanup、fallback 和输出路径。

如果一个函数只是在表格中出现，而没有参与本节主问题，就不要把它扩展成新的主线。

## 4. 关键数据结构与状态边界

本节需要关注的不是单个字段名字，而是字段组合在生命周期中的语义。

### 4.1 `pgBufferUsage`

backend-local 持续递增的 buffer 和 temp block 计数器，不会为每个节点清零。

### 4.2 `pgWalUsage`

backend-local WAL 产生量计数器，记录 per-query 可归因的 WAL 生成事实。

### 4.3 `Instrumentation.bufusage_start`

节点 start 时保存的 buffer 计数快照。

### 4.4 `Instrumentation.walusage_start`

节点 start 时保存的 WAL 计数快照。

### 4.5 `Instrumentation.bufusage`

节点 stop 时通过全局计数差值累计出的节点资源消耗。

### 4.6 `JitInstrumentation`

JIT 编译、优化、发射和 deform 等时间与计数的汇总结构。

### 4.7 `MemoryContextCounters`

MemoryContextMemConsumed 输出的 totalspace / freespace，用于 explain memory 展示。

贯穿本节的不变量是：观测状态不应该破坏被观测对象的执行语义。

## 5. 主流程源码 walkthrough

主流程按时间推进，而不是按文件归类。

```text
ReadBuffer_common() / buffer manager 路径
  ->
XLogInsertRecord()
  ->
InstrStart()
  ->
InstrStop()
  ->
ExplainNode()
  ->
ExplainPrintJITSummary()
  ->
MemoryContextMemConsumed()
  ->
Sort / Hash / Agg 节点输出
```

### 5.1 `ReadBuffer_common() / buffer manager 路径`

时间线推进到 `ReadBuffer_common() / buffer manager 路径` 时，关键变化是：命中、读取、脏页、写出时更新 pgBufferUsage。

资源事实先进入 backend-local 累加器。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.2 `XLogInsertRecord()`

时间线推进到 `XLogInsertRecord()` 时，关键变化是：生成 WAL record、FPI 和字节数时更新 pgWalUsage。

WAL 指标来自 WAL 插入路径，而不是 ExplainNode 自己估算。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.3 `InstrStart()`

时间线推进到 `InstrStart()` 时，关键变化是：在节点开始时保存 pgBufferUsage 和 pgWalUsage 快照。

节点级归因依赖差值，不依赖全局计数清零。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.4 `InstrStop()`

时间线推进到 `InstrStop()` 时，关键变化是：用当前全局计数减 start 快照，累加到 Instrumentation。

这让同一个 backend 中嵌套节点能各自获得窗口内变化。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.5 `ExplainNode()`

时间线推进到 `ExplainNode()` 时，关键变化是：在 planstate->instrument 存在时调用 show_buffer_usage 和 show_wal_usage。

节点资源字段只有在选项打开且 instrumentation 存在时输出。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.6 `ExplainPrintJITSummary()`

时间线推进到 `ExplainPrintJITSummary()` 时，关键变化是：从 queryDesc->estate 汇总 leader 和 worker JIT instrumentation。

JIT 是查询级和 worker 级汇总，不是普通 PlanState rows 字段。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.7 `MemoryContextMemConsumed()`

时间线推进到 `MemoryContextMemConsumed()` 时，关键变化是：在规划或特定上下文上统计内存占用。

MEMORY 输出受统计对象边界限制。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.8 `Sort / Hash / Agg 节点输出`

时间线推进到 `Sort / Hash / Agg 节点输出` 时，关键变化是：节点私有 instrumentation 提供 peak memory、disk usage、batch 等。

这些不是通用 Instrumentation 字段，而是节点算法自己知道的资源状态。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

主流程结束后，用户看到的是输出或视图；源码中真正重要的是状态已经在哪个边界完成折叠。

## 6. 状态随时间推进的完整故事

把本节主问题压成一条对象生命周期，可以避免在函数清单里迷路。

阶段 1：创建或启用

观察 `pgBufferUsage` 在 `ReadBuffer_common() / buffer manager 路径` 前后的变化。

这个阶段不要急着解释用户输出。

先确认状态是否已经产生，是否仍是本地状态，是否已经被折叠成共享或输出状态。

只有完成这个判断，后面的诊断结论才不会跨层。

阶段 2：挂载或关联

观察 `pgWalUsage` 在 `XLogInsertRecord()` 前后的变化。

这个阶段不要急着解释用户输出。

先确认状态是否已经产生，是否仍是本地状态，是否已经被折叠成共享或输出状态。

只有完成这个判断，后面的诊断结论才不会跨层。

阶段 3：热路径累计

观察 `Instrumentation.bufusage_start` 在 `InstrStart()` 前后的变化。

这个阶段不要急着解释用户输出。

先确认状态是否已经产生，是否仍是本地状态，是否已经被折叠成共享或输出状态。

只有完成这个判断，后面的诊断结论才不会跨层。

阶段 4：周期折叠

观察 `Instrumentation.walusage_start` 在 `InstrStop()` 前后的变化。

这个阶段不要急着解释用户输出。

先确认状态是否已经产生，是否仍是本地状态，是否已经被折叠成共享或输出状态。

只有完成这个判断，后面的诊断结论才不会跨层。

阶段 5：输出或刷新

观察 `Instrumentation.bufusage` 在 `ExplainNode()` 前后的变化。

这个阶段不要急着解释用户输出。

先确认状态是否已经产生，是否仍是本地状态，是否已经被折叠成共享或输出状态。

只有完成这个判断，后面的诊断结论才不会跨层。

阶段 6：清理或失效

观察 `JitInstrumentation` 在 `ExplainPrintJITSummary()` 前后的变化。

这个阶段不要急着解释用户输出。

先确认状态是否已经产生，是否仍是本地状态，是否已经被折叠成共享或输出状态。

只有完成这个判断，后面的诊断结论才不会跨层。

完整故事的终点不是“函数返回”，而是用户可以稳定看到或推断这个状态。

## 7. 生命周期 / ownership / cleanup

1. pgBufferUsage 和 pgWalUsage 是 backend-local 全局计数器，进程内持续递增。

这里要同时回答谁创建、谁持有、谁在正常路径读取、谁在异常路径兜底。

2. NodeInstrumentation 只保存某个节点窗口内的差值，生命周期随 PlanState。

这里要同时回答谁创建、谁持有、谁在正常路径读取、谁在异常路径兜底。

3. JIT instrumentation 挂在 EState 或 worker 共享结构上，随查询结束释放。

这里要同时回答谁创建、谁持有、谁在正常路径读取、谁在异常路径兜底。

4. planning memory 的 MemoryContextCounters 来自 planner context，打印后上下文会释放。

这里要同时回答谁创建、谁持有、谁在正常路径读取、谁在异常路径兜底。

5. 并行 worker 的 buffer/WAL/JIT 需要在 DSM 或 leader 汇总路径中复制回来，否则 ExecutorEnd 后不可读。

这里要同时回答谁创建、谁持有、谁在正常路径读取、谁在异常路径兜底。

一个常见错误是把“能读到”误认为“拥有”。

执行器、统计系统和扩展 hook 中，这两者经常分离。

## 8. 正确性机制层次

正确性不是由单个计数器保证的。

层次 1：Buffers 表示 buffer manager 层看到的 block 访问，不等于操作系统 I/O 次数。

把这个层次和本节主流程对齐，才能知道它保证什么、不能保证什么。

层次 2：I/O Timings 依赖 track_io_timing，不打开时 block 计数仍可能存在但时间为零或不展示。

把这个层次和本节主流程对齐，才能知道它保证什么、不能保证什么。

层次 3：WAL 指标记录本 backend 产生的 WAL，不代表 WAL flush 延迟或 fsync 次数。

把这个层次和本节主流程对齐，才能知道它保证什么、不能保证什么。

层次 4：JIT 指标主要解释编译阶段和 JIT 生成代码成本，不等于表达式执行全部 CPU。

把这个层次和本节主流程对齐，才能知道它保证什么、不能保证什么。

层次 5：Memory 字段必须看输出位置：Planning Memory、Sort Memory、Hash Memory 的语义不同。

把这个层次和本节主流程对齐，才能知道它保证什么、不能保证什么。

本节涉及的是观测正确性，不是事务隔离本身。

观测正确性的最低要求是：不要因为收集指标而改变执行语义。

## 9. 错误路径 / 异常路径 / fallback

异常路径 1：并行查询的 worker 资源需要 InstrAccumParallelQuery 合并，否则 leader 只能看到自己的 pgBufferUsage。

遇到这个路径时，先判断状态是缺失、延迟、被聚合，还是被有意隐藏。

异常路径 2：临时文件读写走 Buffile，出现在 temp blocks 中，不能直接当作 shared buffer miss。

遇到这个路径时，先判断状态是缺失、延迟、被聚合，还是被有意隐藏。

异常路径 3：HashAgg、HashJoin、Sort 的 peak memory 来自节点私有算法状态，不是通用 MemoryContextCounters。

遇到这个路径时，先判断状态是缺失、延迟、被聚合，还是被有意隐藏。

异常路径 4：EXPLAIN BUFFERS 不会告诉你具体哪个 relation page 被访问，只给节点窗口内计数。

遇到这个路径时，先判断状态是缺失、延迟、被聚合，还是被有意隐藏。

异常路径 5：WAL bytes 高可能来自 FPI，也可能来自普通 record，需要同时看 wal_fpi 和 wal_fpi_bytes。

遇到这个路径时，先判断状态是缺失、延迟、被聚合，还是被有意隐藏。

异常路径的课程价值在于保护诊断结论。

看不到某个指标，不等于底层事件没有发生。

## 10. 成本、资源与跨模块传播

成本点 1：BUFFERS 和 WAL 的计数差值比 timing 便宜，但需要 Instrumentation 保存和累加结构。

它对应的跨模块传播要从执行器边界继续追到存储、WAL、JIT、MemoryContext、统计共享状态或扩展 hook。

成本点 2：track_io_timing 需要读时钟，打开后会增加 I/O 路径成本。

它对应的跨模块传播要从执行器边界继续追到存储、WAL、JIT、MemoryContext、统计共享状态或扩展 hook。

成本点 3：JIT 统计依赖 JIT 开启后的编译路径，短查询中观测成本可能超过执行收益。

它对应的跨模块传播要从执行器边界继续追到存储、WAL、JIT、MemoryContext、统计共享状态或扩展 hook。

成本点 4：MemoryContextMemConsumed 遍历 context 统计，适合解释边界，不适合无限制频繁调用。

它对应的跨模块传播要从执行器边界继续追到存储、WAL、JIT、MemoryContext、统计共享状态或扩展 hook。

成本点 5：资源指标跨 worker 汇总时要付 DSM 分配、复制和 leader 合并成本。

它对应的跨模块传播要从执行器边界继续追到存储、WAL、JIT、MemoryContext、统计共享状态或扩展 hook。

成本分析必须问两个问题。

第一，事件发生在热路径还是收尾路径。

第二，成本是每 tuple、每 node、每 query、每 flush，还是每输出字段支付。

## 11. 观测与诊断入口

入口 1：EXPLAIN (ANALYZE, BUFFERS) 能看到 shared hit/read/dirtied/written。

观察结果出来后，要回到第 3 节的源码入口确认它的来源。

入口 2：EXPLAIN (ANALYZE, WAL) 对 INSERT/UPDATE/DELETE 更容易出现 WAL records 和 bytes。

观察结果出来后，要回到第 3 节的源码入口确认它的来源。

入口 3：打开 track_io_timing 后，EXPLAIN BUFFERS 可能出现 I/O Timings。

观察结果出来后，要回到第 3 节的源码入口确认它的来源。

入口 4：打开 jit 且成本阈值足够低，可以看到 JIT summary。

观察结果出来后，要回到第 3 节的源码入口确认它的来源。

入口 5：调低 work_mem 触发 Sort 或 Hash spill，可以观察 Disk Usage 或 temp blocks。

观察结果出来后，要回到第 3 节的源码入口确认它的来源。

诊断闭环的顺序应当是：看到现象，定位状态，回到源码，再修正解释。

## 12. 课堂实验

实验 1：对同一 SELECT 先冷缓存后热缓存执行，比较 shared read 与 shared hit。

实验前先写下你预期哪个状态会变化。

实验后再用源码解释为什么变化发生在这个边界，而不是相邻模块。

实验 2：执行 INSERT INTO t SELECT generate_series，观察 WAL records 和 WAL bytes。

实验前先写下你预期哪个状态会变化。

实验后再用源码解释为什么变化发生在这个边界，而不是相邻模块。

实验 3：设置 track_io_timing=on，执行需要读盘的查询，比较 I/O Timings 是否出现。

实验前先写下你预期哪个状态会变化。

实验后再用源码解释为什么变化发生在这个边界，而不是相邻模块。

实验 4：调小 work_mem 后执行 ORDER BY，观察 Sort Method、Disk Usage 和 temp blocks。

实验前先写下你预期哪个状态会变化。

实验后再用源码解释为什么变化发生在这个边界，而不是相邻模块。

课堂实验不追求压测结果。

它只验证一个 runtime 现象能否回到源码状态。

## 13. 常见误区

误区 1：不要把 shared read 直接等同于物理磁盘读。它表示 shared buffer 需要读入 block。

修正这个误区的方法是重新确认粒度：节点、查询、对象、backend、本地 pending、共享 entry 或输出格式。

误区 2：不要把 WAL bytes 当成 commit flush 延迟。flush 由 WAL 写入和同步路径决定。

修正这个误区的方法是重新确认粒度：节点、查询、对象、backend、本地 pending、共享 entry 或输出格式。

误区 3：不要把 JIT total time 加到每个节点上。JIT summary 通常是查询级解释。

修正这个误区的方法是重新确认粒度：节点、查询、对象、backend、本地 pending、共享 entry 或输出格式。

误区 4：不要把 Memory Used 和 work_mem 上限混为一谈。一个是观测值，一个是配置约束。

修正这个误区的方法是重新确认粒度：节点、查询、对象、backend、本地 pending、共享 entry 或输出格式。

大多数误读都不是因为字段名难，而是因为把两个生命周期不同的事实放在一起比较。

## 14. 源码复盘清单

复盘 1：从 `ReadBuffer_common() / buffer manager 路径` 回到 `src/include/executor/instrument.h`。

先确认 `pgBufferUsage` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：命中、读取、脏页、写出时更新 pgBufferUsage。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：BUFFERS 和 WAL 的计数差值比 timing 便宜，但需要 Instrumentation 保存和累加结构。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 2：从 `XLogInsertRecord()` 回到 `src/backend/executor/instrument.c`。

先确认 `pgWalUsage` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：生成 WAL record、FPI 和字节数时更新 pgWalUsage。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：track_io_timing 需要读时钟，打开后会增加 I/O 路径成本。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 3：从 `InstrStart()` 回到 `src/backend/storage/buffer/bufmgr.c`。

先确认 `Instrumentation.bufusage_start` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：在节点开始时保存 pgBufferUsage 和 pgWalUsage 快照。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：JIT 统计依赖 JIT 开启后的编译路径，短查询中观测成本可能超过执行收益。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 4：从 `InstrStop()` 回到 `src/backend/storage/buffer/localbuf.c`。

先确认 `Instrumentation.walusage_start` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：用当前全局计数减 start 快照，累加到 Instrumentation。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：MemoryContextMemConsumed 遍历 context 统计，适合解释边界，不适合无限制频繁调用。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 5：从 `ExplainNode()` 回到 `src/backend/storage/file/buffile.c`。

先确认 `Instrumentation.bufusage` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：在 planstate->instrument 存在时调用 show_buffer_usage 和 show_wal_usage。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：资源指标跨 worker 汇总时要付 DSM 分配、复制和 leader 合并成本。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 6：从 `ExplainPrintJITSummary()` 回到 `src/backend/access/transam/xlog.c`。

先确认 `JitInstrumentation` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：从 queryDesc->estate 汇总 leader 和 worker JIT instrumentation。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：BUFFERS 和 WAL 的计数差值比 timing 便宜，但需要 Instrumentation 保存和累加结构。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 7：从 `MemoryContextMemConsumed()` 回到 `src/backend/utils/activity/pgstat_io.c`。

先确认 `MemoryContextCounters` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：在规划或特定上下文上统计内存占用。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：track_io_timing 需要读时钟，打开后会增加 I/O 路径成本。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 8：从 `Sort / Hash / Agg 节点输出` 回到 `src/backend/commands/explain.c`。

先确认 `pgBufferUsage` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：节点私有 instrumentation 提供 peak memory、disk usage、batch 等。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：JIT 统计依赖 JIT 开启后的编译路径，短查询中观测成本可能超过执行收益。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 9：从 `ReadBuffer_common() / buffer manager 路径` 回到 `src/include/jit/jit.h`。

先确认 `pgWalUsage` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：命中、读取、脏页、写出时更新 pgBufferUsage。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：MemoryContextMemConsumed 遍历 context 统计，适合解释边界，不适合无限制频繁调用。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 10：从 `XLogInsertRecord()` 回到 `src/backend/utils/mmgr/mcxt.c`。

先确认 `Instrumentation.bufusage_start` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：生成 WAL record、FPI 和字节数时更新 pgWalUsage。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：资源指标跨 worker 汇总时要付 DSM 分配、复制和 leader 合并成本。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 11：从 `InstrStart()` 回到 `src/include/executor/instrument.h`。

先确认 `Instrumentation.walusage_start` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：在节点开始时保存 pgBufferUsage 和 pgWalUsage 快照。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：BUFFERS 和 WAL 的计数差值比 timing 便宜，但需要 Instrumentation 保存和累加结构。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 12：从 `InstrStop()` 回到 `src/backend/executor/instrument.c`。

先确认 `Instrumentation.bufusage` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：用当前全局计数减 start 快照，累加到 Instrumentation。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：track_io_timing 需要读时钟，打开后会增加 I/O 路径成本。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 13：从 `ExplainNode()` 回到 `src/backend/storage/buffer/bufmgr.c`。

先确认 `JitInstrumentation` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：在 planstate->instrument 存在时调用 show_buffer_usage 和 show_wal_usage。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：JIT 统计依赖 JIT 开启后的编译路径，短查询中观测成本可能超过执行收益。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 14：从 `ExplainPrintJITSummary()` 回到 `src/backend/storage/buffer/localbuf.c`。

先确认 `MemoryContextCounters` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：从 queryDesc->estate 汇总 leader 和 worker JIT instrumentation。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：MemoryContextMemConsumed 遍历 context 统计，适合解释边界，不适合无限制频繁调用。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 15：从 `MemoryContextMemConsumed()` 回到 `src/backend/storage/file/buffile.c`。

先确认 `pgBufferUsage` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：在规划或特定上下文上统计内存占用。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：资源指标跨 worker 汇总时要付 DSM 分配、复制和 leader 合并成本。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 16：从 `Sort / Hash / Agg 节点输出` 回到 `src/backend/access/transam/xlog.c`。

先确认 `pgWalUsage` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：节点私有 instrumentation 提供 peak memory、disk usage、batch 等。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：BUFFERS 和 WAL 的计数差值比 timing 便宜，但需要 Instrumentation 保存和累加结构。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 17：从 `ReadBuffer_common() / buffer manager 路径` 回到 `src/backend/utils/activity/pgstat_io.c`。

先确认 `Instrumentation.bufusage_start` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：命中、读取、脏页、写出时更新 pgBufferUsage。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：track_io_timing 需要读时钟，打开后会增加 I/O 路径成本。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 18：从 `XLogInsertRecord()` 回到 `src/backend/commands/explain.c`。

先确认 `Instrumentation.walusage_start` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：生成 WAL record、FPI 和字节数时更新 pgWalUsage。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：JIT 统计依赖 JIT 开启后的编译路径，短查询中观测成本可能超过执行收益。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 19：从 `InstrStart()` 回到 `src/include/jit/jit.h`。

先确认 `Instrumentation.bufusage` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：在节点开始时保存 pgBufferUsage 和 pgWalUsage 快照。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：MemoryContextMemConsumed 遍历 context 统计，适合解释边界，不适合无限制频繁调用。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 20：从 `InstrStop()` 回到 `src/backend/utils/mmgr/mcxt.c`。

先确认 `JitInstrumentation` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：用当前全局计数减 start 快照，累加到 Instrumentation。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：资源指标跨 worker 汇总时要付 DSM 分配、复制和 leader 合并成本。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 21：从 `ExplainNode()` 回到 `src/include/executor/instrument.h`。

先确认 `MemoryContextCounters` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：在 planstate->instrument 存在时调用 show_buffer_usage 和 show_wal_usage。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：BUFFERS 和 WAL 的计数差值比 timing 便宜，但需要 Instrumentation 保存和累加结构。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 22：从 `ExplainPrintJITSummary()` 回到 `src/backend/executor/instrument.c`。

先确认 `pgBufferUsage` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：从 queryDesc->estate 汇总 leader 和 worker JIT instrumentation。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：track_io_timing 需要读时钟，打开后会增加 I/O 路径成本。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 23：从 `MemoryContextMemConsumed()` 回到 `src/backend/storage/buffer/bufmgr.c`。

先确认 `pgWalUsage` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：在规划或特定上下文上统计内存占用。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：JIT 统计依赖 JIT 开启后的编译路径，短查询中观测成本可能超过执行收益。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 24：从 `Sort / Hash / Agg 节点输出` 回到 `src/backend/storage/buffer/localbuf.c`。

先确认 `Instrumentation.bufusage_start` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：节点私有 instrumentation 提供 peak memory、disk usage、batch 等。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：MemoryContextMemConsumed 遍历 context 统计，适合解释边界，不适合无限制频繁调用。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 25：从 `ReadBuffer_common() / buffer manager 路径` 回到 `src/backend/storage/file/buffile.c`。

先确认 `Instrumentation.walusage_start` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：命中、读取、脏页、写出时更新 pgBufferUsage。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：资源指标跨 worker 汇总时要付 DSM 分配、复制和 leader 合并成本。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 26：从 `XLogInsertRecord()` 回到 `src/backend/access/transam/xlog.c`。

先确认 `Instrumentation.bufusage` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：生成 WAL record、FPI 和字节数时更新 pgWalUsage。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：BUFFERS 和 WAL 的计数差值比 timing 便宜，但需要 Instrumentation 保存和累加结构。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 27：从 `InstrStart()` 回到 `src/backend/utils/activity/pgstat_io.c`。

先确认 `JitInstrumentation` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：在节点开始时保存 pgBufferUsage 和 pgWalUsage 快照。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：track_io_timing 需要读时钟，打开后会增加 I/O 路径成本。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 28：从 `InstrStop()` 回到 `src/backend/commands/explain.c`。

先确认 `MemoryContextCounters` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：用当前全局计数减 start 快照，累加到 Instrumentation。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：JIT 统计依赖 JIT 开启后的编译路径，短查询中观测成本可能超过执行收益。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 29：从 `ExplainNode()` 回到 `src/include/jit/jit.h`。

先确认 `pgBufferUsage` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：在 planstate->instrument 存在时调用 show_buffer_usage 和 show_wal_usage。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：MemoryContextMemConsumed 遍历 context 统计，适合解释边界，不适合无限制频繁调用。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 30：从 `ExplainPrintJITSummary()` 回到 `src/backend/utils/mmgr/mcxt.c`。

先确认 `pgWalUsage` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：从 queryDesc->estate 汇总 leader 和 worker JIT instrumentation。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：资源指标跨 worker 汇总时要付 DSM 分配、复制和 leader 合并成本。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

## 15. 讨论题

1. 为什么 buffer/WAL 可以用差值挂到节点，而 JIT 更自然地按查询汇总？

回答时必须引用至少一个第 3 节源码入口。

2. EXPLAIN BUFFERS 为什么无法替代 page-level tracing？

回答时必须引用至少一个第 3 节源码入口。

3. track_io_timing 默认不开启的代价考量是什么？

回答时必须引用至少一个第 3 节源码入口。

4. 当 Hash 节点同时出现 Memory Usage 和 Buffers 时，如何区分算法内存和存储访问？

回答时必须引用至少一个第 3 节源码入口。

讨论题的目标不是扩展范围，而是检查本节唯一主问题是否已经能独立解释。

## 16. 本节小结

资源指标来自不同子系统的状态，而不是 ExplainNode 的重新计算。

Buffers 和 WAL 借助 backend-local 累加器差值进入节点级 Instrumentation。

JIT、Memory、Sort、Hash、Agg 等指标必须按各自 ownership 边界解释。

可以带走的可迁移规律是：

```text
先找到状态归属，再解释输出字段；先确认生命周期，再讨论成本和正确性。
```

下一节继续沿同一条主线，把这个边界推进到相邻观测机制。
