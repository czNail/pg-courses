# PostgreSQL Clause Selectivity Combination

## 课程定位

前置知识：已经知道 parser 和 rewriter 会把 `WHERE`、`JOIN/ON`、`HAVING` 等条件交给 planner，也已经理解 `RestrictInfo` 是 planner 为 clause 增加 relids、outer join 边界、缓存字段和移动性信息的包装。

本节唯一主问题：

```text
多个 clause 同时限制一个 relation 时，PostgreSQL 为什么默认把选择率相乘，又在哪些地方拒绝简单相乘？
```

核心矛盾：planner 必须把任意布尔表达式压缩成一个可以继续传播的 `Selectivity`，这个数字要足够便宜、稳定、可缓存；但真实数据分布常常相关，简单独立性假设会把 rows 错误放大或压小几个数量级。

学完后应能判断：一个 `EXPLAIN` 中的 base scan rows 偏差，是来自单个 clause 估算、多个 clause 组合、范围上下界配对、OR 重叠、扩展统计未命中，还是 fallback 默认值。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

前几节已经把统计信息拆开看过：

```text
ANALYZE
  -> pg_statistic / pg_statistic_ext_data
  -> MCV、histogram、ndistinct、correlation、dependencies
  -> 单个 operator 或 expression 的 selectivity estimator
```

本节把视角放到 planner 真正要消费的地方。

SQL 里很少只有一个条件。

更常见的是：

```sql
WHERE region = 'CN'
  AND city = 'Beijing'
  AND created_at >= now() - interval '7 days'
  AND deleted_at IS NULL
```

每个条件都可能有统计入口。

但 path search 需要的是一个 relation 层面的 rows：

```text
RelOptInfo.tuples
  * clause-list selectivity
  -> RelOptInfo.rows
```

这一节只追这一条主线。

它不展开所有 operator 的 estimator。

也不讲 join cardinality。

它只回答：

```text
clauselist_selectivity_ext() 如何把多个 clause 的局部估算合成一个整体估算？
```

下一节会把同样的选择率问题推到 join：

```text
base rel:
  rows = table tuples * restriction selectivity

join rel:
  rows = outer rows * inner rows * join selectivity
```

理解本节之后，再看 join rows 偏差时，才能分清：

```text
输入 rel 已经估错
  vs
join 条件本身估错
  vs
join type 公式改变了 rows 下界
```

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
clauselist_selectivity_ext() 先尝试让 extended statistics 消费同一 relation 上可联合估算的 clause；
剩余 clause 逐个调用 clause_selectivity_ext()；
普通 AND 按乘法组合；
同一变量上的范围上下界先收集到 RangeQueryClause 再合成；
OR 用 s1 + s2 - s1 * s2；
最后把这个 selectivity 交给 set_baserel_size_estimates() 形成 rel->rows。
```

这里有一个看似简单、实际很硬的 tension：

```text
如果每个 clause 都精确联合估算：
  planner 会需要任意维度统计、任意表达式分布和巨大计算成本。

如果永远简单相乘：
  相关列、互斥条件、范围条件和 MCV skew 会系统性误估。
```

PostgreSQL 的选择不是追求完美。

它建立了几层边界：

| 层次 | 代表入口 | 作用 |
| --- | --- | --- |
| clause 层 | `clause_selectivity_ext()` | 单个布尔表达式得到 `Selectivity`。 |
| list 层 | `clauselist_selectivity_ext()` | AND 语义下组合多个 clause。 |
| OR 层 | `clauselist_selectivity_or()` | 用容斥近似处理 OR 的重叠。 |
| range 层 | `addRangeClause()` / `RangeQueryClause` | 把同一变量上下界从两个 clause 合成一个区间选择率。 |
| extended stats 层 | `statext_clauselist_selectivity()` | 在单列独立性失效时用多列统计覆盖部分 clause。 |
| rows 层 | `set_baserel_size_estimates()` | 把选择率乘到 `RelOptInfo.tuples` 上。 |

这几层共同形成一个实用模型：

```text
优先使用更了解“多个条件关系”的统计；
无法联合估算时回到独立性假设；
对非常常见且便宜的非独立模式做专门修正；
任何结果都只是 plan choice 的输入，不改变 SQL 语义。
```

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/optimizer/path/costsize.c` | `set_baserel_size_estimates()` 如何把 clause-list selectivity 变成 `rel->rows`。 |
| 2 | `src/backend/optimizer/path/clausesel.c` | `clauselist_selectivity_ext()`、`clause_selectivity_ext()`、`clauselist_selectivity_or()` 和 `addRangeClause()`。 |
| 3 | `src/backend/statistics/extended_stats.c` | `statext_clauselist_selectivity()` 如何先用多变量 MCV，再用 functional dependencies。 |
| 4 | `src/backend/statistics/mcv.c` | `mcv_clauselist_selectivity()` 如何处理多列 MCV 命中。 |
| 5 | `src/backend/statistics/dependencies.c` | `dependencies_clauselist_selectivity()` 如何修正 AND 条件中的相关性。 |
| 6 | `src/backend/utils/adt/selfuncs.c` | `boolvarsel()`、`scalararraysel()`、`nulltestsel()`、各种 operator selectivity 的基础入口。 |
| 7 | `src/include/optimizer/optimizer.h` | `clauselist_selectivity()`、`clause_selectivity()`、`clamp_row_est()` 等 planner 公共入口声明。 |
| 8 | `src/include/nodes/pathnodes.h` | `RelOptInfo.rows`、`Path.rows`、`RestrictInfo` 缓存字段所在的状态边界。 |

推荐阅读顺序不是从 `clause_selectivity_ext()` 的每个分支背起。

更好的顺序是：

```text
先看 costsize.c 的 set_baserel_size_estimates()
  -> 明确 selectivity 的消费者是谁
  -> 回到 clausesel.c 看 list 组合规则
  -> 只挑 BoolExpr、OpExpr、NullTest、ScalarArrayOpExpr 这些常见分支
  -> 再看 extended_stats.c 为什么要提前消费部分 clause
  -> 最后用 EXPLAIN rows 偏差反推是哪一层失败
```

这样读能避免一个常见误区：

```text
不要把 selectivity estimator 当成孤立函数。
它的意义来自 rows 传播和 path 成本比较。
```

## 4. 从 EXPLAIN 看到的问题

先看一个诊断现场。

假设有一张用户表：

```sql
CREATE TABLE u (
    id bigint,
    country text,
    city text,
    active boolean,
    created_at timestamptz
);
```

业务上 `country` 和 `city` 强相关。

例如 `city = 'Beijing'` 基本意味着 `country = 'CN'`。

如果只看单列统计，两个条件会被当作近似独立：

```sql
EXPLAIN SELECT *
FROM u
WHERE country = 'CN'
  AND city = 'Beijing';
```

诊断时你会看到：

```text
Seq Scan on u
  Filter: ((country = 'CN') AND (city = 'Beijing'))
  rows=...
```

这里的 `rows` 不是 executor 执行后数出来的。

它在 planner 阶段已经形成。

主链路是：

```text
make_one_rel()
  -> set_base_rel_sizes()
     -> set_rel_size()
        -> set_plain_rel_size()
           -> set_baserel_size_estimates()
              -> clauselist_selectivity()
```

本节只关注最后两步：

```text
baserel->tuples
  * clauselist_selectivity(root, baserel->baserestrictinfo, ...)
  -> baserel->rows
```

如果 `country='CN'` 估成 0.2。

如果 `city='Beijing'` 估成 0.01。

简单相乘得到 0.002。

真实选择率可能接近 0.01。

这一节要回答：

```text
PostgreSQL 什么时候会接受 0.2 * 0.01？
什么时候会避免它？
避免不了时，我们如何从 EXPLAIN 回到源码解释？
```

## 5. 关键状态与语义边界

### `Selectivity`

`Selectivity` 是 double 风格的概率数。

语义上通常在 `[0, 1]`。

但课程里要记住两点：

```text
Selectivity 不是行数；
它只有乘到输入 cardinality 上才变成 rows。
```

它也不是正确性边界。

估错不会让 PostgreSQL 返回错误结果。

它只改变：

```text
选哪个 scan path
选哪个 join order
选哪个 join algorithm
是否认为并行值得
是否认为排序、hash、memoize 的代价划算
```

### `RelOptInfo.tuples` 与 `RelOptInfo.rows`

`RelOptInfo` 定义在 `src/include/nodes/pathnodes.h`。

本节只需要分清两种数量：

| 字段 | 语义 |
| --- | --- |
| `tuples` | relation 在没有本层 restriction 之后的原始规模估计。 |
| `rows` | 本层 restriction 生效后的输出行数估计。 |

在 base relation 上：

```text
tuples:
  来自 relcache / pg_class / sampling 后的 relation 规模估计

rows:
  tuples * baserestrictinfo 的组合选择率
```

`rows` 会继续传给 path。

例如 `cost_seqscan()` 会设置：

```text
path->rows = baserel->rows
```

如果是 parameterized path，则可能使用：

```text
path->rows = param_info->ppi_rows
```

因此一个 selectivity 组合错误，不会停留在 base rel。

它会向后进入：

```text
Path.rows
  -> startup_cost / total_cost
  -> add_path() 比较
  -> joinrel rows
  -> upper path rows
```

### `RestrictInfo` 中的缓存

`clause_selectivity_ext()` 接受的可能是裸表达式，也可能是 `RestrictInfo`。

当它看到 `RestrictInfo` 时，会尝试复用缓存。

关键字段包括：

```text
norm_selec
outer_selec
orclause
pseudoconstant
clause_relids
num_base_rels
```

缓存的原因不是为了改变估算语义。

它是为了让同一个 clause 在多个 path 候选中反复被询问时，不重复查统计和执行 estimator。

缓存也有边界。

如果 `varRelid` 会影响结果，就不能随便复用。

outer join clause 还可能以实际 outer join 语义或 `JOIN_INNER` 语义被估算，因此源码区分 `norm_selec` 和 `outer_selec`。

### `RangeQueryClause`

`RangeQueryClause` 是 `clausesel.c` 内部的小结构。

它不是 parse tree 节点。

也不会流到 executor。

它只在 `clauselist_selectivity_ext()` 处理一个 clause list 的过程中存在。

它记录：

```text
同一变量的低边界选择率
同一变量的高边界选择率
变量在 operator 左边还是右边
```

它解决一个很具体的问题：

```sql
WHERE x > 10 AND x < 20
```

如果简单相乘：

```text
P(x > 10) * P(x < 20)
```

通常会高估。

更接近的组合是：

```text
P(x < 20) + P(x > 10) - 1 + P(x IS NULL)
```

源码中就是 `hibound + lobound - 1.0`，再加回 NULL 修正。

这是一条重要线索：

```text
PostgreSQL 并不是不知道独立性假设有问题；
它只在收益足够高、识别足够便宜、语义足够明确的地方专门修正。
```

## 6. 主流程源码 walkthrough

### 入口：`set_baserel_size_estimates()`

先从消费者读起。

`src/backend/optimizer/path/costsize.c` 中：

```text
set_baserel_size_estimates()
  -> nrows = rel->tuples * clauselist_selectivity(...)
  -> rel->rows = clamp_row_est(nrows)
  -> cost_qual_eval(&rel->baserestrictcost, ...)
  -> set_rel_width(root, rel)
```

这里有三个输出：

```text
rows:
  后续 path 和 join 的 cardinality 输入。

baserestrictcost:
  条件表达式自身的 CPU 代价。

width:
  后续 sort/hash/materialize 等节点估算内存和 I/O 的输入。
```

本节主角是第一项。

注意这里没有“实际执行”。

也没有访问 heap tuple。

planner 只是把统计事实压缩成一个数字。

### 第一层：`clauselist_selectivity()`

`clauselist_selectivity()` 是薄包装：

```text
clauselist_selectivity()
  -> clauselist_selectivity_ext(..., use_extended_stats = true)
```

真正逻辑在 `clauselist_selectivity_ext()`。

它先处理一个快路径：

```text
如果 clause list 只有一个元素：
  直接调用 clause_selectivity_ext()
```

这很重要。

单个 clause 不需要考虑：

```text
多个 clause 是否相关
range 上下界是否要成对
extended statistics 是否要共同消费一个 clause set
```

当 list 中有多个 clause 时，才进入组合逻辑。

### 第二层：extended statistics 先尝试接管

源码先做：

```text
find_single_rel_for_clauses(root, clauses)
```

如果这些 clause 都引用同一个 base relation，并且该 relation 有 `statlist`，就调用：

```text
statext_clauselist_selectivity()
```

这个函数在 `src/backend/statistics/extended_stats.c`。

它的顺序是：

```text
statext_mcv_clauselist_selectivity()
  -> 尽量用多变量 MCV 估算一组 clause

dependencies_clauselist_selectivity()
  -> 对 AND 条件中的 functional dependency 做修正
```

它还会维护：

```text
estimatedclauses
```

这是一个 bitmapset。

语义是：

```text
哪些 clause 的选择率已经被 extended statistics 估进 s1 里了；
后续普通循环不要再估一次。
```

这就是 ownership 和正确性的交界：

```text
extended stats 可以覆盖部分 clause；
但覆盖过的 clause 必须被明确标记；
否则会重复乘一次，造成系统性低估。
```

### 第三层：剩余 clause 逐个估算

接下来 `clauselist_selectivity_ext()` 循环遍历 clause list。

每个未被 `estimatedclauses` 标记的 clause，调用：

```text
clause_selectivity_ext(root, clause, varRelid, jointype, sjinfo, use_extended_stats)
```

返回值存在局部变量 `s2`。

大多数 clause 会直接：

```text
s1 = s1 * s2
```

这就是默认独立性假设。

它背后的直觉是：

```text
如果两个条件独立：
  P(A AND B) = P(A) * P(B)
```

PostgreSQL 不声称它总是成立。

它只是把它作为：

```text
没有更好统计时的便宜近似。
```

### 第四层：范围条件不马上相乘

循环中有一个特殊识别。

如果 clause 是二元 operator，并且一侧是变量、一侧是 pseudo constant，源码会检查 operator 的 restriction estimator：

```text
F_SCALARLTSEL
F_SCALARLESEL
F_SCALARGTSEL
F_SCALARGESEL
```

如果命中，就不立即乘入 `s1`。

而是调用：

```text
addRangeClause(&rqlist, clause, varonleft, isLTsel, s2)
```

后面统一扫描 `rqlist`。

如果同一变量同时有低边界和高边界：

```text
s2 = hibound + lobound - 1.0
s2 += nulltestsel(...)
```

如果只有一侧边界：

```text
s1 *= lobound
```

或：

```text
s1 *= hibound
```

如果估算结果因为默认值或浮点误差变得不可信，源码还会 fallback 到：

```text
DEFAULT_RANGE_INEQ_SEL
```

或者极小正数：

```text
1.0e-10
```

这里体现了一个重要原则：

```text
planner 的估算代码会宁愿保留一个小的正 rows，
也不会让普通路径因为 0 或负数破坏成本插值和路径比较。
```

第 23 节会专门讲 `clamp_row_est()`。

### 第五层：`clause_selectivity_ext()` 的分派

单个 clause 的入口在同一个文件中。

它先处理 `RestrictInfo`：

```text
pseudoconstant
cacheable
norm_selec / outer_selec
orclause
```

然后根据节点类型分派。

本节只需要记住几个常见分支：

| 节点 | 源码处理 |
| --- | --- |
| `Var` | 布尔列走 `boolvarsel()`。 |
| `Const` | 布尔常量直接得到 0 或 1。 |
| `NOT` | 递归估算子条件，再用 `1.0 - s`。 |
| `AND` | 回到 `clauselist_selectivity_ext()`。 |
| `OR` | 进入 `clauselist_selectivity_or()`。 |
| `OpExpr` / `DistinctExpr` | 根据是否为 join clause 走 `join_selectivity()` 或 `restriction_selectivity()`。 |
| `FuncExpr` | 先试 support function，再 fallback 到 `boolvarsel()`。 |
| `ScalarArrayOpExpr` | 走 `scalararraysel()`。 |
| `RowCompareExpr` | 走 `rowcomparesel()`。 |
| `NullTest` | 走 `nulltestsel()`。 |
| `BooleanTest` | 走 `booltestsel()`。 |

这张表不要背成 API。

诊断时要问：

```text
我的 SQL clause 在 parse tree 中属于哪种节点？
它会进入哪个 estimator？
这个 estimator 是否能看到我以为它能看到的统计？
```

### 第六层：OR 的组合

`clauselist_selectivity_or()` 用 `s1 + s2 - s1 * s2` 近似 `P(A OR B)`。这仍然带有独立性假设，只是把它放到 OR 的容斥形式里。

因此 OR 的 rows 偏差通常来自两类情况：

```text
条件互斥，但源码按可能重叠处理；
条件高度重叠，但源码低估了重叠部分。
```

extended statistics 可以先尝试处理单 relation 上的一部分 OR；但 `statext_clauselist_selectivity()` 中 functional dependencies 只服务 AND 条件，`is_or` 为 true 时不会继续应用 dependencies。

## 7. 生命周期 / ownership / cleanup

这一节没有 shared memory。

也没有 executor 运行期资源。

所有核心状态都是 planner 生命周期内的 backend-local 对象。

### 谁创建

`PlannerInfo` 和各个 `RelOptInfo` 在一次 planning 过程中创建。

base relation 的 `baserestrictinfo` 来自前面 qual 分发阶段。

`RestrictInfo` 通常在 planner context 中分配。

`clauselist_selectivity_ext()` 内部临时创建：

```text
RangeQueryClause
estimatedclauses bitmapset
临时 ListCell
```

### 谁持有

`RelOptInfo` 持有 `baserestrictinfo`。

`RestrictInfo` 持有原始 clause、relids、缓存字段。

`RangeQueryClause` 只由 `clauselist_selectivity_ext()` 当前调用持有。

它不是长期状态。

### 谁释放

`RangeQueryClause` 在处理完后显式 `pfree()`。

planner 里许多长期对象由 planner memory context 统一释放。

`RestrictInfo` 上的 selectivity 缓存跟随 `RestrictInfo` 生命周期。

这解释了为什么源码愿意把 `norm_selec`、`outer_selec` 放到 `RestrictInfo` 上：

```text
同一个 planner run 中，它能被反复复用；
planner 结束后，整个 context 回收；
不需要 executor 看到这些估算缓存。
```

### ERROR 路径

如果 estimator 中 `elog(ERROR)`，不会靠每个函数逐层清理普通 C 对象。

PostgreSQL 的 MemoryContext 机制会在上层错误恢复边界清理本次 planning 相关内存。

因此这里的 ownership 不是：

```text
每个分支都必须 free 所有 planner 对象。
```

而是：

```text
短期临时链表能显式释放就释放；
planner 生命周期对象依赖 planner_cxt；
ERROR 由上层 context cleanup 兜底。
```

## 8. 正确性机制与 fallback

selectivity 代码的“正确性”不是 SQL 结果正确性；SQL 结果仍由 executor 按 qual 真正过滤保证。这里的正确性是让 planner 在统计缺失、表达式复杂或版本近似不够时仍能继续搜索，并且不重复计算同一 clause。

几个 fallback 要记牢：

| 场景 | 源码行为 | 诊断含义 |
| --- | --- | --- |
| 无法识别的 clause | `clause_selectivity_ext()` 初值是 `0.5`，部分表达式最终回到 `boolvarsel()` 或默认值。 | 复杂函数或表达式 rows 偏差，先查 support function 和表达式统计，不要先调 cost。 |
| pseudoconstant | 大多数 gating qual 返回 `1.0`，constant false 可以返回 `0.0`。 | 不引用 relation 的条件不一定作为普通行过滤选择率。 |
| range 默认值 | 范围一侧像是默认不等式选择率时，成对范围回到 `DEFAULT_RANGE_INEQ_SEL`；轻微负数按浮点误差处理。 | `x > a AND x < b` 的估算异常，要看 operator estimator 是否 punt。 |
| extended stats 未命中 | clause 不在单 relation、统计对象不覆盖、形态不匹配时回到逐 clause 估算。 | 创建统计对象后仍要确认 `ANALYZE`、列/表达式、operator 和 AND/OR 结构都匹配。 |

这些 fallback 都不会改变查询结果，只会改变 path choice 的输入数字。

## 9. 成本传播：从 selectivity 到 path choice

本节的 selectivity 最终进入：

```text
rel->rows
```

然后进入 path 成本。

`cost_seqscan()` 中：

```text
path->rows = baserel->rows
disk_run_cost = spc_seq_page_cost * baserel->pages
cpu_per_tuple = cpu_tuple_cost + qpqual_cost.per_tuple
cpu_run_cost = cpu_per_tuple * baserel->tuples
tlist cost = pathtarget cost * path->rows
```

注意这里有两个不同数量级：

```text
过滤条件评估通常对扫描到的 baserel->tuples 收费；
targetlist 输出成本对 path->rows 收费。
```

所以 rows 估错不一定等比例改变 seqscan 所有成本。

但它会明显影响：

```text
index scan 预计 fetch 的 tuple/page 数
join 输入大小
sort/hash/materialize 的规模
LIMIT 下 startup/total 插值
parallel gather 传输成本
```

`cost_index()` 中有更直接的影响：

```text
tuples_fetched = clamp_row_est(indexSelectivity * baserel->tuples)
```

如果多 clause 组合把选择率估得过小，planner 可能过度偏向 index path。

如果估得过大，可能错过 index path 或 nested loop path。

因此不要把 `rows` 偏差看成显示层问题。

它是 path search 的输入。

## 10. 观测与诊断入口

### 先从 EXPLAIN 看三类 rows

建议总是用：

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT ...
```

看三组信息：

```text
estimated rows:
  planner 输出。

actual rows:
  executor 实际返回。

loops:
  节点被重复执行次数，尤其 nested loop inner scan。
```

对 base scan，诊断顺序是：

```text
estimated rows 是否远小于 actual rows？
  -> 可能 selectivity 过低。

estimated rows 是否远大于 actual rows？
  -> 可能 selectivity 过高。

偏差是否在 base scan 就出现？
  -> 先查 restriction clause 组合。

偏差是否到 join 才出现？
  -> 下一节再查 join cardinality。
```

### 再查统计输入

基础检查：

```sql
SELECT attname, n_distinct, null_frac, most_common_vals, most_common_freqs
FROM pg_stats
WHERE schemaname = 'public'
  AND tablename = 'u';
```

如果怀疑多列相关：

```sql
SELECT *
FROM pg_statistic_ext
WHERE stxrelid = 'u'::regclass;
```

诊断目标很明确：是否存在覆盖 `country, city` 的统计对象，是否已经 `ANALYZE`，统计类型是否包含 `mcv` 或 `dependencies`。随后把现场映射回 `clausesel.c` 的 list/OR/range 入口、`extended_stats.c` 的多列统计入口，以及 `costsize.c` 的 `set_baserel_size_estimates()` 落点。

## 11. 课堂实验

### 实验一：独立性假设失效

准备数据：

```sql
DROP TABLE IF EXISTS s21_users;

CREATE TABLE s21_users AS
SELECT g AS id,
       CASE WHEN g <= 90000 THEN 'CN' ELSE 'US' END AS country,
       CASE WHEN g <= 90000 THEN 'Beijing' ELSE 'Boston' END AS city,
       (g % 10 = 0) AS active
FROM generate_series(1, 100000) AS g;

ANALYZE s21_users;
```

观察：

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM s21_users
WHERE country = 'CN'
  AND city = 'Beijing';
```

预期讨论：

```text
单列统计会知道 country='CN' 很常见；
也会知道 city='Beijing' 很常见；
但没有多列统计时，组合层可能仍按独立性乘法处理。
```

创建 extended statistics：

```sql
CREATE STATISTICS s21_users_country_city_mcv (mcv, dependencies)
ON country, city
FROM s21_users;

ANALYZE s21_users;
```

再执行同一个 `EXPLAIN`。

把 rows 变化映射回：

```text
statext_clauselist_selectivity()
  -> statext_mcv_clauselist_selectivity()
  -> dependencies_clauselist_selectivity()
```

### 实验二：范围上下界配对

准备数据：

```sql
DROP TABLE IF EXISTS s21_range;

CREATE TABLE s21_range AS
SELECT g AS x
FROM generate_series(1, 100000) AS g;

ANALYZE s21_range;
```

比较：

```sql
EXPLAIN SELECT *
FROM s21_range
WHERE x > 10000
  AND x < 20000;
```

源码映射：

```text
x > 10000:
  scalar greater-than estimator 得到低边界选择率。

x < 20000:
  scalar less-than estimator 得到高边界选择率。

clauselist_selectivity_ext():
  不直接相乘；
  用 RangeQueryClause 合成区间。
```

可选变化：把 `x` 改成 `x + 0` 再执行同类范围查询，观察表达式形态变化是否影响 range pair 识别。

## 12. 源码练习

练习一：

从 `set_baserel_size_estimates()` 开始，画出下面 SQL 的调用链：

```sql
SELECT *
FROM s21_users
WHERE country = 'CN'
  AND city = 'Beijing'
  AND active;
```

要求标出：

```text
哪个函数计算 clause list selectivity
哪个函数处理 active 这种 boolean Var
哪个状态字段接收最终 rows
```

练习二：

在 `clausesel.c` 中阅读 `clauselist_selectivity_ext()`。

回答：

```text
estimatedclauses 是怎么避免重复计算的？
为什么 range clause 先进入 rqlist？
为什么 pseudoconstant 不进入 range pair 逻辑？
```

练习三：

阅读 `statext_clauselist_selectivity()`。

回答：

```text
为什么 multivariate MCV 先于 dependencies？
为什么 OR 条件下 dependencies 不继续执行？
```

练习四：

阅读 `clause_selectivity_ext()` 中 `RestrictInfo` 缓存逻辑。

回答：

```text
什么情况下 selectivity 结果可以缓存？
为什么 outer join 需要单独的 outer_selec？
```

## 13. 讨论题

问题一：

```text
一个查询 WHERE a = 1 AND b = 1 估算过小。
你如何判断它需要 dependencies、MCV，还是只是 ANALYZE 太旧？
```

回答时必须同时引用：

```text
EXPLAIN rows/actual rows
pg_stats 或 pg_statistic_ext 输入
clausesel.c / extended_stats.c 中的入口
```

问题二：

```text
为什么 PostgreSQL 不默认为所有列组合维护联合分布？
```

讨论时从三个角度回答：

```text
ANALYZE 成本
catalog 体积
planner 搜索时的匹配成本
```

问题三：

```text
range pair 的 hi + lo - 1 为什么要加回 nulltestsel()？
```

提示：

```text
单边不等式选择率通常排除了 NULL；
两个边界简单相加时可能把 NULL 的排除重复计算。
```

问题四：

```text
如果一个复杂 FuncExpr 没有 support function，为什么 planner 仍然必须给它一个 selectivity？
```

重点不是准确。

重点是：

```text
path search 不能因为一个未知表达式停止。
```

## 14. 本节小结

本节的可迁移模型是：

```text
选择率组合不是统计函数清单；
它是把多个局部、不完美、成本受限的统计判断压缩成一个可传播 cardinality 的过程。
```

源码主链路是：

```text
set_baserel_size_estimates()
  -> clauselist_selectivity()
     -> clauselist_selectivity_ext()
        -> statext_clauselist_selectivity()
        -> clause_selectivity_ext()
        -> addRangeClause()
        -> clauselist_selectivity_or()
  -> rel->rows
```

诊断主链路是：

```text
EXPLAIN base scan rows 偏差
  -> 找到对应 WHERE clause
  -> 判断 clause 节点形态
  -> 检查单列统计和扩展统计
  -> 回到 clausesel.c 看组合规则
  -> 再观察 path choice 如何变化
```

本节最重要的边界是：

```text
独立性乘法是 fallback，不是事实；
extended statistics 是覆盖部分 clause 的修正，不是全局 oracle；
range、OR、NOT 等特殊规则服务于便宜且常见的可识别形态；
最终 rows 只是成本模型输入，真正结果仍由 executor qual 保证。
```

下一节会把这个模型推到 join：

```text
当两个 relation 相遇时，选择率不再只影响一个 base scan；
它会乘上两个输入 rows，并被 join type、foreign key、outer join 下限和 semi/anti 语义重新解释。
```
