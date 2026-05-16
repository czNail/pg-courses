# Planner / Optimizer / Statistics

本目录约 12 节课，目标是解释 PostgreSQL 为什么选择这个执行计划。

课程安排：

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
