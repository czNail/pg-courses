# Executor 与可观测性

本目录约 11 节课，目标是理解 Plan 如何变成运行中的 PlanState，以及 EXPLAIN / pg_stat 指标从哪里来。

课程安排：

1. Executor 生命周期：ExecutorStart / Run / End。
2. PlanState / TupleTableSlot / ExprContext。
3. Executor MemoryContext。
4. 常见执行节点源码 walkthrough。
5. EXPLAIN ANALYZE 与 Instrumentation。
6. pg_stat 统计体系。
7. Wait Event 与等待诊断。
8. auto_explain / executor hook。
9. 并行查询观测。
10. 慢 SQL 从 EXPLAIN 到源码。
11. 轻量 Executor profiler extension。
