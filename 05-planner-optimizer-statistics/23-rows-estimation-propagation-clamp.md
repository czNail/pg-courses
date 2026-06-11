# PostgreSQL Rows 估算传播与 Clamp 边界

## 课程定位

前置知识：已经理解 base restriction selectivity 和 join cardinality 如何形成，也知道 `RelOptInfo`、`Path`、`Plan` 是 planner 从搜索空间到执行计划的三层对象。

本节唯一主问题：

```text
PostgreSQL 为什么让 rows 估算在 RelOptInfo、ParamPathInfo、Path、Plan 之间持续传播，并且在多个边界用 clamp_row_est() 把它收束成可比较数字？
```

核心矛盾：planner 需要保留行数数量级差异，才能比较 scan、join、sort、aggregate、parallel 等 path；但估算公式来自采样、默认值、浮点计算和近似独立性，可能产生 0、负数、NaN、无穷大或过大数量级。PostgreSQL 不能让这些值破坏成本插值、path pruning 和 EXPLAIN 展示。

学完后应能判断：`EXPLAIN` 中一个 rows 数字，是在哪一层生成、在哪一层被复制、在哪一层因 parameterization 或 parallelism 改写，以及 `clamp_row_est()` 是稳定性保护，不是准确性保证。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

上一节结束在 joinrel：

```text
outer rows
  * inner rows
  * join selectivity
  * join type 规则
  -> join RelOptInfo.rows
```

但 planner 并不会在这里停止。

同一个 rows 会继续进入：

```text
Path.rows
  -> startup_cost / total_cost
  -> cheapest path 选择
  -> Plan.plan_rows
  -> EXPLAIN 输出
```

还有一些 rows 会分叉。

例如 parameterized path：

```text
RelOptInfo.rows:
  不带外部参数时的 relation 规模。

ParamPathInfo.ppi_rows:
  指定 required_outer 后，这个 path 每次被外侧参数驱动时的规模。

Path.rows:
  普通 path 取 parent rows；
  parameterized path 取 ppi_rows。
```

本节不重新讲 selectivity 算法。

它关心的是：

```text
一旦 rows 被算出来，PostgreSQL 如何让它在 planner 内部保持一致、可缓存、可比较，并避免坏数值污染后续 cost。
```

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
base/join/upper rel 先把 cardinality 写入 RelOptInfo.rows；
parameterized path 用 ParamPathInfo.ppi_rows 记录特定参数化下的行数；
cost_*() 把 rows 拷入 Path.rows 并用它计算 CPU、I/O、输出投影和 LIMIT 插值；
createplan.c 再把 Path.rows 写入 Plan.plan_rows；
在这些边界上，clamp_row_est() 把异常或过细的浮点估算收束成 planner 可继续消费的行数。
```

这个模型有三个稳定判断。

第一，rows 是 planner 状态。

它不是 EXPLAIN 打印时临时计算的。

第二，不同层次的 rows 有不同语义。

```text
RelOptInfo.rows:
  relation set 的公共规模估算。

ParamPathInfo.ppi_rows:
  某种 required_outer 参数化下的规模估算。

Path.rows:
  某个候选执行路径承诺输出多少行。

Plan.plan_rows:
  最终计划节点继承的估算，用于展示和少量后续计划修饰。
```

第三，clamp 保护的是 planner 数值系统。

它不会让估算更真实。

它只保证：

```text
普通 rows 不变成 NaN / infinite / 负数 / 过大值；
普通非空路径不因为估算接近 0 而破坏除法或展示；
cost 公式能继续运行。
```

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/optimizer/path/costsize.c` | `clamp_row_est()`、`set_baserel_size_estimates()`、`set_joinrel_size_estimates()`、`get_parameterized_*_size()`、`cost_seqscan()`、`cost_gather()`、`compute_gather_rows()`。 |
| 2 | `src/backend/optimizer/util/relnode.c` | `get_baserel_parampathinfo()`、`get_joinrel_parampathinfo()` 如何缓存 `ppi_rows`。 |
| 3 | `src/include/nodes/pathnodes.h` | `RelOptInfo.rows`、`ParamPathInfo.ppi_rows`、`Path.rows` 的字段语义。 |
| 4 | `src/backend/optimizer/util/pathnode.c` | 各类 `create_*_path()` 如何设置 path、调用 cost 函数并保存 rows。 |
| 5 | `src/backend/optimizer/plan/createplan.c` | `copy_generic_path_info()` 等路径如何把 `Path.rows` 复制成 `Plan.plan_rows`。 |
| 6 | `src/include/nodes/plannodes.h` | `Plan.plan_rows`、`startup_cost`、`total_cost` 的最终计划字段。 |
| 7 | `src/backend/commands/explain.c` | EXPLAIN 如何展示 `Plan Rows` 与实际行数。 |

推荐顺序：

```text
先读 clamp_row_est()
  -> 再读 set_baserel_size_estimates()
  -> 再读 get_baserel_parampathinfo()
  -> 再读 cost_seqscan() 看 Path.rows
  -> 再读 createplan.c 看 Plan.plan_rows
  -> 最后回到 EXPLAIN 对照 estimated rows / actual rows
```

不要从 EXPLAIN 输出倒背字段名。

要把它当成一个状态传播链的末端。

## 4. 先看 `clamp_row_est()`

`clamp_row_est()` 在 `src/backend/optimizer/path/costsize.c`。

源码逻辑非常短：

```text
如果 nrows > MAXIMUM_ROWCOUNT 或 isnan(nrows):
  nrows = MAXIMUM_ROWCOUNT
否则如果 nrows <= 1.0:
  nrows = 1.0
否则:
  nrows = rint(nrows)
返回 nrows
```

注释说明了动机：

```text
避免 infinite 和 NaN；
避免成本公式变得 useless；
普通估算至少为一行；
避免 cost 插值时除以零；
让 EXPLAIN 看起来合理；
把行数取整。
```

这个函数有两个容易误解的地方。

第一，它把 `0.01` 变成 `1`，不是说真实会返回一行。

这是 planner 对普通非空路径的稳定性处理。

第二，它不处理所有“零行”场景。

PostgreSQL 仍然有 dummy rel 和 provably-empty path 的概念。

那些路径可以有 0 rows。

但普通估算链路中，接近 0 的概率通常会被收束到 1。

因此诊断时看到：

```text
rows=1
actual rows=0
```

不能立刻说 estimator 太乐观。

它可能只是 clamp 后的最小展示值。

## 5. `RelOptInfo.rows`：relation 层承诺

在 `src/include/nodes/pathnodes.h` 中，`RelOptInfo` 有字段：

```text
Cardinality rows;
```

对 base relation：

```text
set_baserel_size_estimates()
  -> nrows = rel->tuples * clauselist_selectivity(...)
  -> rel->rows = clamp_row_est(nrows)
```

对 join relation：

```text
set_joinrel_size_estimates()
  -> rel->rows = calc_joinrel_size_estimate(...)
```

而 `calc_joinrel_size_estimate()` 末尾也会：

```text
return clamp_row_est(nrows)
```

这说明：

```text
RelOptInfo.rows 是 relation set 的公共规模估算。
```

它服务于：

```text
生成 path 前的基本判断；
path 成本函数的默认 row 输入；
更高层 joinrel 的输入 rows；
upper planning 的 group/sort/limit 等估算。
```

它不是某个具体 path 的执行结果。

如果同一个 relation 有多个 path：

```text
SeqScan path
IndexScan path
BitmapHeapScan path
Parallel path
```

普通情况下它们都从同一个 parent rel 拿 rows。

但 path 的成本仍可能不同。

因为它们的：

```text
pages fetched
startup work
rescan cost
qual placement
parallel workers
ordering
```

不同。

## 6. `ParamPathInfo.ppi_rows`：参数化路径的行数缓存

parameterized path 是 rows 传播中最容易混淆的地方。

`ParamPathInfo` 定义在 `pathnodes.h`。

核心字段：

```text
ppi_req_outer:
  这个 path 需要哪些外侧 rel 提供参数。

ppi_rows:
  在这些参数约束下，预计输出多少行。

ppi_clauses:
  base relation 参数化路径负责执行的 movable join clauses。

ppi_serials:
  对应 RestrictInfo serial 集合。
```

`get_baserel_parampathinfo()` 在 `relnode.c` 中做这些事：

```text
检查 required_outer 是否为空；
复用已有 PPI；
找出 movable join clauses；
加入 EquivalenceClass 推导出来的 clauses；
调用 get_parameterized_baserel_size() 计算 rows；
创建 ParamPathInfo 并保存 ppi_rows；
挂到 baserel->ppilist。
```

对 join relation，则是：

```text
get_joinrel_parampathinfo()
```

它也会缓存 `ppi_rows`。

但 join 的 `ppi_clauses` 通常是 NIL。

原因在源码注释里：

```text
join relation 的 relevant clauses 会随输入 path pair 变化；
不能像 base relation 那样简单保存在 PPI 中。
```

这里的核心不变量是：

```text
同一个 relation + 同一个 required_outer，应使用同一个 ppi_rows。
```

这避免不同 path 因重复估算得到不一致行数。

## 7. `Path.rows`：候选路径承诺

`Path` 也在 `pathnodes.h`。

字段：

```text
Cardinality rows;
Cost startup_cost;
Cost total_cost;
```

`cost_seqscan()` 展示了最简单的规则：

```text
如果 param_info 存在:
  path->rows = param_info->ppi_rows
否则:
  path->rows = baserel->rows
```

然后它继续计算：

```text
disk_run_cost = spc_seq_page_cost * baserel->pages
cpu_per_tuple = cpu_tuple_cost + qpqual_cost.per_tuple
cpu_run_cost = cpu_per_tuple * baserel->tuples
tlist cost = path->pathtarget->cost.per_tuple * path->rows
```

这显示两个行数语义：

```text
扫描了多少 tuple:
  可能用 baserel->tuples。

输出了多少 tuple:
  用 path->rows。
```

index path、bitmap path、join path、sort path、aggregate path 都有各自的 rows 传递方式。

但大原则一样：

```text
Path.rows 是 cost 函数对这个候选 path 输出规模的承诺。
```

## 8. Parallel rows 的特殊处理

并行路径让 rows 语义更细。

`cost_seqscan()` 中，如果 `path->parallel_workers > 0`，会：

```text
parallel_divisor = get_parallel_divisor(path)
path->rows = clamp_row_est(path->rows / parallel_divisor)
```

这意味着 partial path 的 rows 表示：

```text
每个 worker 预计处理或输出的行数。
```

当生成 Gather 或 Gather Merge 时，又要把它合回总行数。

`compute_gather_rows()` 的注释说明：

```text
partial rows 已经除过 parallel_divisor；
Gather 估算行数时要乘回去。
```

源码：

```text
return clamp_row_est(path->rows * get_parallel_divisor(path))
```

诊断并行计划时要小心：

```text
worker 内部节点 rows
  与
Gather 节点 rows
不是同一层语义。
```

这不是 EXPLAIN 打印错误。

这是 planner 对 partial path 与 gathered path 的不同承诺。

## 9. `Plan.plan_rows`：最终计划中的继承值

当 planner 选定 best path 后，进入 create plan 阶段。

`src/backend/optimizer/plan/createplan.c` 中会把 path 信息复制给 plan。

常见逻辑是：

```text
Plan.plan_rows = Path.rows
Plan.startup_cost = Path.startup_cost
Plan.total_cost = Path.total_cost
```

`Plan` 结构在 `src/include/nodes/plannodes.h`。

字段名包括：

```text
plan_rows
startup_cost
total_cost
plan_width
```

EXPLAIN 展示的 estimated rows，主要来自 `plan_rows`。

所以从 EXPLAIN 反推源码时，要沿着：

```text
Plan.plan_rows
  <- Path.rows
     <- RelOptInfo.rows 或 ParamPathInfo.ppi_rows
        <- selectivity / grouping / append / upper planning
```

不要把 `Plan.plan_rows` 当成孤立估算。

它已经是多轮传播后的结果。

## 10. 成本传播：为什么坏 rows 会污染很多公式

`costsize.c` 顶部注释描述了 cost 插值：

```text
actual_cost = startup_cost
  + (total_cost - startup_cost) * tuples_to_fetch / path->rows
```

这解释了 `clamp_row_est()` 为什么避免普通 `path->rows` 为 0。

如果 rows 是 0，LIMIT、EXISTS、cursor 等 partial fetch 成本就可能除零。

rows 还进入：

```text
tlist per-tuple cost
qual per-tuple cost
sort tuple count
hash table size
aggregate group estimate
parallel tuple transfer cost
join processed tuple estimate
materialize spill estimate
```

例如 `cost_gather()`：

```text
startup_cost += parallel_setup_cost
run_cost += parallel_tuple_cost * path->path.rows
```

如果 rows 过大，parallel communication 被高估。

如果 rows 过小，Gather 看起来更便宜。

再例如 hash join：

```text
inner_path_rows
  -> ExecChooseHashTableSize()
  -> numbuckets / numbatches
  -> spill cost
```

因此 rows 偏差会从 cardinality 问题变成：

```text
路径选择问题
内存估算问题
并行收益问题
I/O 成本问题
```

## 11. 生命周期 / ownership / cleanup

rows 本身不是独立分配的对象。

它是多个 planner 节点上的字段。

### 谁创建

base relation rows 由 `set_baserel_size_estimates()` 创建。

join relation rows 由 `set_joinrel_size_estimates()` 创建。

parameterized rows 由 `get_parameterized_baserel_size()` 或 `get_parameterized_joinrel_size()` 创建。

Path rows 由具体 `cost_*()` 或 `create_*_path()` 例程设置。

Plan rows 由 create plan 阶段复制。

### 谁持有

`RelOptInfo` 持有 relation 公共 rows。

`ParamPathInfo` 持有 required_outer 维度下的 rows。

`Path` 持有候选路径 rows。

`Plan` 持有最终计划 rows。

### 谁释放

这些对象都在 planner memory context 生命周期内。

planning 完成后，最终 `Plan` 树会被保留给 executor。

未选中的 `Path`、`RelOptInfo`、`ParamPathInfo` 等 planner 搜索结构会随 planner context 回收。

这解释了一个边界：

```text
executor 不需要知道所有备选 path 的 rows；
它只拿到最终 Plan 节点上的 plan_rows 和成本信息。
```

### ERROR 路径

如果估算过程中出现 ERROR，上层 MemoryContext cleanup 会释放 planner 对象。

`clamp_row_est()` 本身不是 ERROR 处理。

它处理的是正常估算链路中的坏数值。

## 12. 正确性与 fallback

rows 估算的正确性边界是：

```text
不能影响 SQL 结果；
必须能让 planner 继续搜索；
必须保持成本公式数值稳定；
必须在相同语义的 path 之间尽量一致。
```

### fallback 1：最小 1 行

`nrows <= 1.0` 时返回 1。

它保护：

```text
EXPLAIN 展示
LIMIT 插值
per-row 成本除法
join cost 中的保护性假设
```

但它可能让实际 0 行的查询显示 estimated rows=1。

这不是错误结果。

### fallback 2：最大行数

`MAXIMUM_ROWCOUNT` 是 `1e100`。

如果估算大于它或变成 NaN，rows 被设到这个最大值。

这避免 cost 变成 infinite/NaN。

### fallback 3：取整

`rint()` 把普通 rows 变成整数。

行数本来是离散概念。

更重要的是，它避免 EXPLAIN 和 cost 插值中出现过细浮点噪音。

### fallback 4：parameterized rows 不超过 base rows

`get_parameterized_baserel_size()` 会在计算后做：

```text
if nrows > rel->rows:
  nrows = rel->rows
```

`get_parameterized_joinrel_size()` 也有类似保护。

因为额外参数化 clause 不应该让同一 relation 输出比非参数化估算更多。

这也是 rows 传播中的一个语义 clamp。

## 13. 观测与诊断入口

### 先看 EXPLAIN 层级

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT ...
```

对每个节点记录：

```text
estimated rows
actual rows
loops
```

诊断时要把 `actual rows * loops` 与语义对应起来。

Nested Loop inner 节点尤其要小心。

它的 actual rows 是每次 loop 的输出。

### 再判断偏差起点

如果第一个 base scan 已经偏差很大：

```text
回到 restriction selectivity。
```

如果 base scan 准，join 节点开始偏：

```text
回到 join cardinality。
```

如果 lower path 准，Aggregate 或 Sort 后偏：

```text
看 group estimate、distinct estimate、limit estimate。
```

如果 parallel 子节点与 Gather 行数看似不一致：

```text
检查 partial path rows 与 compute_gather_rows()。
```

### 用源码断点定位

可在调试构建中设置断点：

```text
break set_baserel_size_estimates
break set_joinrel_size_estimates
break get_baserel_parampathinfo
break get_joinrel_parampathinfo
break clamp_row_est
break cost_seqscan
break cost_gather
```

观察：

```text
rel->rows
param_info->ppi_rows
path->rows
plan->plan_rows
```

不要只看最后的 EXPLAIN。

最后的 rows 已经经过多次复制和改写。

## 14. 课堂实验

### 实验一：接近 0 的选择率为什么显示 1

```sql
DROP TABLE IF EXISTS s23_t;

CREATE TABLE s23_t AS
SELECT g AS id
FROM generate_series(1, 100000) AS g;

ANALYZE s23_t;
```

执行：

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM s23_t
WHERE id = -1;
```

观察：

```text
estimated rows 是否为 1；
actual rows 是否为 0。
```

源码映射：

```text
restriction selectivity 很小
  -> set_baserel_size_estimates()
  -> clamp_row_est()
  -> rows=1
```

### 实验二：parameterized path 的 rows

准备两张表：

```sql
DROP TABLE IF EXISTS s23_a;
DROP TABLE IF EXISTS s23_b;

CREATE TABLE s23_a AS
SELECT g AS id
FROM generate_series(1, 10000) AS g;

CREATE TABLE s23_b AS
SELECT g AS id, g AS a_id
FROM generate_series(1, 10000) AS g;

CREATE INDEX ON s23_b(a_id);
ANALYZE s23_a;
ANALYZE s23_b;
```

执行：

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM s23_a a
JOIN s23_b b ON b.a_id = a.id
WHERE a.id < 10;
```

观察 nested loop inner index scan 的 rows。

讨论：

```text
inner path 的 rows 是否代表整个 s23_b？
还是每个 outer 参数下的估算？
```

源码映射：

```text
get_baserel_parampathinfo()
  -> get_parameterized_baserel_size()
  -> ParamPathInfo.ppi_rows
  -> cost_index()
  -> Path.rows
```

### 实验三：并行 rows

调低并行门槛：

```sql
SET max_parallel_workers_per_gather = 2;
SET min_parallel_table_scan_size = 0;
SET parallel_setup_cost = 0;
SET parallel_tuple_cost = 0;

EXPLAIN
SELECT count(*)
FROM s23_t;
```

观察：

```text
parallel worker 下方节点 rows
Gather / Finalize Aggregate 上方 rows
```

讨论：

```text
partial path rows 和 gathered rows 为什么不是同一个数字？
compute_gather_rows() 做了什么？
```

## 15. 源码练习

练习一：

阅读 `clamp_row_est()`。

回答：

```text
为什么 NaN 会变成 MAXIMUM_ROWCOUNT？
为什么 nrows <= 1.0 会变成 1？
为什么要 rint()？
```

练习二：

从 `set_baserel_size_estimates()` 追到 `cost_seqscan()`。

写出：

```text
rel->rows 在哪里产生；
path->rows 在哪里赋值；
baserel->tuples 和 path->rows 分别用于哪些成本。
```

练习三：

阅读 `get_baserel_parampathinfo()`。

回答：

```text
required_outer 为空时为什么返回 NULL？
为什么同一个 parameterization 要复用 PPI？
ppi_rows 如何计算？
```

练习四：

阅读 `compute_gather_rows()` 和 `cost_gather()`。

回答：

```text
partial path rows 为什么要除 parallel_divisor？
Gather rows 为什么要乘回来？
parallel_tuple_cost 用的是哪一层 rows？
```

练习五：

在 `createplan.c` 中找一个把 path 信息复制到 plan 的位置。

说明：

```text
Plan.plan_rows 从哪里来；
EXPLAIN 看到的是哪个层次的 rows；
未选中 path 的 rows 为什么不会出现在最终 Plan 树里。
```

## 16. 常见误区

误区一：

```text
rows=1 表示 planner 真的认为一定有一行。
```

很多时候它只是 clamp 最小值。

特别是实际 0 行时，要看是否走了普通估算路径。

误区二：

```text
RelOptInfo.rows 和 Path.rows 总是相同。
```

parameterized path、parallel path、upper path 都可能改变 `Path.rows`。

误区三：

```text
EXPLAIN rows 是执行时采样得来的。
```

不是。

它来自 planner 阶段写入的 `Plan.plan_rows`。

`EXPLAIN ANALYZE` 的 actual rows 才来自执行。

误区四：

```text
clamp 修复了估算错误。
```

clamp 只修复数值可用性。

它不会理解数据分布。

误区五：

```text
调 cost 参数可以修复 rows 偏差。
```

cost 参数改变的是 rows 到成本的映射。

rows 本身要回到统计、selectivity、join cardinality 或 upper estimate。

## 17. 讨论题

问题一：

```text
为什么普通估算把 0.2 行 clamp 成 1 行，比保留 0.2 更适合 planner？
```

回答要覆盖：

```text
成本插值
EXPLAIN 展示
除零保护
普通 path 与 dummy path 的区别
```

问题二：

```text
Parameterized path 为什么不能直接复用 parent RelOptInfo.rows？
```

提示：

```text
outer 参数会带来额外 join clauses；
这些 clauses 会改变每次 inner scan 的行数。
```

问题三：

```text
并行计划里，为什么 worker 下方 rows 与 Gather rows 的语义不同？
```

必须引用：

```text
cost_seqscan()
compute_gather_rows()
cost_gather()
```

问题四：

```text
如果 EXPLAIN 中某个上层 Sort rows 很准，但下层 scan rows 不准，这可能吗？
```

讨论：

```text
上层 rows 往往继承下层输出；
除非中间有聚合、distinct、limit 或其他重新估算边界。
```

## 18. 本节小结

本节的核心模型是：

```text
rows 是 planner 内部沿对象生命周期传播的 cardinality 状态；
不是 EXPLAIN 临时显示，也不是 executor 真实计数。
```

源码主链路：

```text
set_baserel_size_estimates()
  -> RelOptInfo.rows
  -> cost_*()
     -> Path.rows
     -> startup_cost / total_cost
  -> createplan.c
     -> Plan.plan_rows
  -> EXPLAIN
```

参数化路径的旁路：

```text
get_baserel_parampathinfo()
  -> ParamPathInfo.ppi_rows
  -> Path.rows
```

并行路径的旁路：

```text
partial path rows
  -> 除 parallel_divisor
  -> compute_gather_rows()
  -> 乘 parallel_divisor 回 gathered rows
```

最重要的诊断原则：

```text
看到 rows 偏差时，先定位它在哪一层开始偏；
再决定查 restriction、join、parameterization、parallel、upper estimate 还是 clamp。
```

下一节进入 cost model：

```text
rows 一旦稳定下来，就会与 seq_page_cost、random_page_cost、cpu_tuple_cost、parallel_setup_cost 等相对单位结合，成为 path 比较的成本。
```
