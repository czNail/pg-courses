# PostgreSQL Join Cardinality 与 Join Selectivity

## 课程定位

前置知识：已经理解上一节的 clause-list selectivity 组合规则，也知道 `RelOptInfo.rows` 是 path search 的核心输入。

本节唯一主问题：

```text
为什么 joinrel 的 rows 不能只由 join operator 的选择率决定，而必须同时解释输入 rows、join type、外键语义、pushed-down quals 和 SEMI/ANTI 行为？
```

核心矛盾：join 是 cardinality 放大的中心。一个 base relation 的 rows 错误通常影响一个子树；一个 joinrel 的 rows 错误会改变后续 join order、join algorithm、内存节点和并行路径。但 PostgreSQL 在 planning 阶段只能拿到有限统计、约束信息和 clause 结构，因此必须把多个近似合成一个稳定数字。

学完后应能判断：`EXPLAIN` 中 join 节点 rows 偏差，究竟来自输入 relation 已经估错、join selectivity 估错、foreign key 修正未命中、outer join 下限、pushed-down qual 分类，还是 semi/anti join 的存在性语义。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

上一节的主线是：

```text
base relation tuples
  * restriction clause selectivity
  -> base RelOptInfo.rows
```

本节继续向后走一步：

```text
outer RelOptInfo.rows
  * inner RelOptInfo.rows
  * join clause selectivity
  * join type 规则
  -> join RelOptInfo.rows
```

这一步比 base restriction 更危险。

原因很简单：

```text
join rows 是两个输入 rows 的乘积再缩放。
```

如果两个输入各错 10 倍，join 结果可能错 100 倍。

如果 join selectivity 又错 10 倍，后续搜索空间会直接被误导。

本节只讲 join cardinality。

不展开 join order 枚举。

不细讲 nested loop、merge join、hash join 的完整成本。

这些会在后续课程进入。

本节要建立一个判断：

```text
join selectivity 是 join rows 的一个因子；
它不是 join rows 的全部语义。
```

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
build_join_rel() 创建 join RelOptInfo 并收集 restrictlist；
set_joinrel_size_estimates() 调用 calc_joinrel_size_estimate()；
后者先用 foreign key 语义尝试替换部分 join clause selectivity；
再按 join type 区分 joinquals 与 pushed-down quals；
最后把 outer_rows、inner_rows、fkselec、jselec、pselec 合成 joinrel->rows，并通过 clamp_row_est() 收束。
```

这个模型里有四个边界。

第一，输入 rows 不是原始表大小。

它已经包含了前面 restriction、parameterized path、append 或上层构造带来的估算。

第二，join selectivity 来自 operator 的 `oprjoin` estimator。

例如等值 join 最终会走 `src/backend/utils/adt/selfuncs.c` 中的 `eqjoinsel()`。

第三，join type 改变 rows 公式。

`INNER`、`LEFT`、`FULL`、`SEMI`、`ANTI` 不是同一个概率解释。

第四，外键和 pushed-down quals 会改变哪些 clause 被如何计入。

因此一个 join rows 偏差，不能只说：

```text
eqjoinsel() 估错了。
```

必须先还原：

```text
输入 rows 是否已经错？
restrictlist 中哪些 clause 真正在当前 join level 计入？
foreign key 是否接管了部分 clause？
join type 是否施加了输出下限？
```

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/optimizer/path/joinrels.c` | `make_join_rel()` 如何检查 join 合法性并调用 `build_join_rel()`。 |
| 2 | `src/backend/optimizer/util/relnode.c` | `build_join_rel()` 如何创建 join `RelOptInfo`、收集 restrictlist，并调用 `set_joinrel_size_estimates()`。 |
| 3 | `src/backend/optimizer/path/costsize.c` | `set_joinrel_size_estimates()`、`calc_joinrel_size_estimate()`、`get_foreign_key_join_selectivity()`、`compute_semi_anti_join_factors()`。 |
| 4 | `src/backend/optimizer/path/clausesel.c` | join clause 如何经 `clause_selectivity_ext()` 转到 `join_selectivity()`。 |
| 5 | `src/backend/optimizer/util/plancat.c` | `join_selectivity()` 如何调用 operator 的 `oprjoin` 函数。 |
| 6 | `src/backend/utils/adt/selfuncs.c` | `eqjoinsel()`、`eqjoinsel_inner()`、`eqjoinsel_semi()` 如何利用 MCV、ndistinct 和 null fraction。 |
| 7 | `src/include/nodes/pathnodes.h` | `SpecialJoinInfo`、`SemiAntiJoinFactors`、`JoinPathExtraData` 的语义边界。 |

推荐顺序：

```text
make_join_rel()
  -> build_join_rel()
     -> build_joinrel_restrictlist()
     -> set_joinrel_size_estimates()
        -> calc_joinrel_size_estimate()
           -> get_foreign_key_join_selectivity()
           -> clauselist_selectivity()
              -> join_selectivity()
                 -> eqjoinsel()
```

读源码时要一直区分两件事：

```text
哪个函数估算 joinrel 输出 rows？
哪个函数估算某种 join path 执行它要花多少钱？
```

本节是前者。

`final_cost_hashjoin()`、`final_cost_nestloop()` 会读取这些 rows，但它们不是 join cardinality 的第一入口。

## 4. 从一个 EXPLAIN 偏差开始

考虑两张表：

```sql
CREATE TABLE customers (
    id bigint primary key,
    country text
);

CREATE TABLE orders (
    id bigint primary key,
    customer_id bigint references customers(id),
    status text
);
```

查询：

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.status = 'paid'
  AND c.country = 'CN';
```

如果 join 节点 estimated rows 与 actual rows 差很多，诊断不要直接跳到 hash join 或 nested loop。

先拆开：

```text
orders 经过 status 过滤后的 rows 是否准？
customers 经过 country 过滤后的 rows 是否准？
o.customer_id = c.id 的 join selectivity 是否准？
foreign key 是否被 planner 利用？
join order 中这一层 restrictlist 包含哪些 clause？
```

源码中的 join rows 是在 joinrel 构建时算出来的。

它早于具体 join path 的最终成本计算。

这意味着：

```text
同一个 joinrel 的多种 path 通常共享同一个 parent->rows；
具体 path 可以因为 parameterization 有不同 Path.rows；
但 joinrel cardinality 是搜索空间中“这个关系集合有多大”的公共判断。
```

## 5. 关键状态与数据结构

### `RelOptInfo.rows`

在 join rel 上，`RelOptInfo.rows` 表示：

```text
这个 relids 集合形成后，当前 join level 输出多少行。
```

它不是某个 join algorithm 专属。

hash join、merge join、nested loop path 都会引用它。

`build_join_rel()` 初始把 `joinrel->rows = 0`。

随后调用：

```text
set_joinrel_size_estimates()
```

把真实估算填进去。

### `SpecialJoinInfo`

`SpecialJoinInfo` 定义在 `src/include/nodes/pathnodes.h`。

本节重点字段：

```text
min_lefthand / min_righthand:
  join 两侧的最小 relids 边界。

syn_lefthand / syn_righthand:
  SQL 语法上左右侧边界。

jointype:
  INNER、LEFT、FULL、SEMI、ANTI 等。

ojrelid:
  outer join 的 RT index，普通 join 为 0。
```

它不是只给 join order legality 用。

它也会传给 selectivity estimator。

例如 `eqjoinsel()` 中会根据 `sjinfo->jointype` 区分 inner join 与 semi/anti join。

### `restrictlist`

join cardinality 用的 clause list 不是 SQL 文本里简单的 `ON` 文本。

`build_joinrel_restrictlist()` 会根据当前 joinrel、输入 rel 和 outer join 语义，收集这一层应该应用的 `RestrictInfo`。

`calc_joinrel_size_estimate()` 收到这个 list 后，还会继续分类：

```text
joinquals:
  当前 join 自身的 ON 语义条件。

pushedquals:
  已经被推到 outer join 之后应用的条件。
```

对 inner join，这个区分通常不重要。

对 outer join，它直接影响 rows 下限。

### `SemiAntiJoinFactors`

`SemiAntiJoinFactors` 也在 `pathnodes.h`。

它包含：

```text
outer_match_frac:
  外侧行中至少有一个匹配的比例。

match_count:
  对有匹配的外侧行，平均匹配多少内侧行。
```

它由 `compute_semi_anti_join_factors()` 填充。

这些字段服务的是 cost。

原因是 SEMI/ANTI 或 `inner_unique` join 的 executor 可能在找到第一个 match 后停止扫描内侧。

但它也帮助我们理解：

```text
SEMI/ANTI 的 selectivity 不是 Cartesian product 的比例；
它是外侧行是否存在匹配的比例。
```

## 6. 主流程源码 walkthrough

### 第一步：`make_join_rel()`

`src/backend/optimizer/path/joinrels.c` 中的 `make_join_rel()` 做三件事：

```text
构造 joinrelids；
调用 join_is_legal() 判断这个 join 是否允许；
调用 build_join_rel() 找到或创建 join RelOptInfo。
```

如果是普通 inner join，源码会构造一个 dummy `SpecialJoinInfo`：

```text
init_dummy_sjinfo()
```

这样后续 selectivity 代码仍然能统一使用 `sjinfo`。

这是一个源码阅读中的小细节：

```text
即使 SQL 没有 outer join，join selectivity API 仍然带 SpecialJoinInfo。
```

因为 estimator 需要知道 join 两侧 relids 和 join type。

### 第二步：`build_join_rel()`

`build_join_rel()` 在 `src/backend/optimizer/util/relnode.c`。

如果 joinrel 已经存在，它不会重新建 rel。

但仍会为当前输入 pair 计算 restrictlist。

如果 joinrel 不存在，它会分配 `RelOptInfo`，初始化：

```text
reloptkind = RELOPT_JOINREL
relids
consider_startup
reltarget
pathlist
ppilist
lateral_relids
```

然后构造输出 targetlist、PlaceHolderVars、restrictlist、joininfo。

在本节主线上，关键调用是：

```text
set_joinrel_size_estimates(root, joinrel, outer_rel, inner_rel, sjinfo, restrictlist)
```

到这里，planner 还没有决定使用 hash join 还是 nested loop。

它只是先回答：

```text
这个 relation set 如果形成了，大概有多少行？
```

### 第三步：`set_joinrel_size_estimates()`

`src/backend/optimizer/path/costsize.c` 中：

```text
set_joinrel_size_estimates()
  -> rel->rows = calc_joinrel_size_estimate(...)
```

传入的 `outer_rows` 和 `inner_rows` 来自：

```text
outer_rel->rows
inner_rel->rows
```

注意这里是 relation rows。

不是 path rows。

parameterized path 会走另一个入口：

```text
get_parameterized_joinrel_size()
```

它传入：

```text
outer_path->rows
inner_path->rows
```

然后同样调用 `calc_joinrel_size_estimate()`。

这说明 PostgreSQL 把“公式”集中在一个 workhorse 里。

不同入口只负责提供不同输入规模。

### 第四步：foreign key 修正

`calc_joinrel_size_estimate()` 先调用：

```text
get_foreign_key_join_selectivity()
```

这个函数会尝试识别 join clause 是否能匹配已知外键约束。

如果能，它会：

```text
从 restrictlist 中移除这些 FK-matching clauses；
返回一个替代 selectivity fkselec；
```

这样做的原因在源码注释里很直接：

```text
多列 FK 或单列统计缺失时，普通 clauselist_selectivity() 的独立性假设可能很差；
FK 语义能提供更可靠的估算。
```

这是 join cardinality 的第一条非统计通道。

它说明 planner 不只看 `pg_statistic`。

约束也会进入估算。

### 第五步：outer join 下的 clause 分类

如果 `jointype` 是 outer join，源码会把 restrictlist 分成：

```text
joinquals
pushedquals
```

判断条件是：

```text
RINFO_IS_PUSHED_DOWN(rinfo, joinrel->relids)
```

然后分别估算：

```text
jselec = clauselist_selectivity(joinquals, ...)
pselec = clauselist_selectivity(pushedquals, ...)
```

为什么要分开？

因为 outer join 的 ON 条件不能随便把 preserved side 的行过滤掉。

例如 LEFT JOIN：

```text
joinquals 影响匹配成功概率；
但输出至少要保留 outer_rows。
```

pushed-down quals 则是在 outer join 结果形成后再过滤。

所以它们可以继续缩小 rows。

### 第六步：按 join type 计算 rows

核心公式在 `calc_joinrel_size_estimate()` 的 switch 中。

INNER：

```text
nrows = outer_rows * inner_rows * fkselec * jselec
```

LEFT：

```text
nrows = outer_rows * inner_rows * fkselec * jselec
if nrows < outer_rows:
    nrows = outer_rows
nrows *= pselec
```

FULL：

```text
nrows 至少为 outer_rows；
nrows 也至少为 inner_rows；
再乘 pushed-down selectivity。
```

SEMI：

```text
nrows = outer_rows * fkselec * jselec
```

ANTI：

```text
nrows = outer_rows * (1.0 - fkselec * jselec)
nrows *= pselec
```

最后：

```text
return clamp_row_est(nrows)
```

这一段是本节最重要的源码。

它把 join selectivity 的概率解释变成了 rows 语义。

## 7. `join_selectivity()` 与 operator estimator

`clause_selectivity_ext()` 遇到 `OpExpr` 时，会判断它是 restriction 还是 join clause。

如果是 join clause，就调用：

```text
join_selectivity()
```

这个函数在 `src/backend/optimizer/util/plancat.c`。

它从 operator catalog 中取：

```text
oprjoin
```

然后通过 `OidFunctionCall5Coll()` 调用。

如果 operator 没有 join selectivity function：

```text
return 0.5
```

这就是一个重要 fallback。

等值 join 常见入口是：

```text
selfuncs.c: eqjoinsel()
```

`eqjoinsel()` 会读取两侧变量统计：

```text
get_join_variables()
get_variable_numdistinct()
get_attstatsslot(... STATISTIC_KIND_MCV ...)
```

如果两侧都有 MCV，它会尝试精确匹配 MCV 对。

如果没有足够 MCV，则使用 ndistinct、null fraction 的近似。

对 SEMI/ANTI，它会进入：

```text
eqjoinsel_semi()
```

这再次说明：

```text
同一个 operator，在不同 join type 下 selectivity 的含义不同。
```

## 8. 生命周期 / ownership / cleanup

join cardinality 估算发生在一次 planner run 内。

核心对象都是 backend-local。

### 谁创建

`make_join_rel()` 创建 joinrelids。

`build_join_rel()` 创建或查找 `RelOptInfo`。

`build_joinrel_restrictlist()` 构造当前 pair 的 restrictlist。

`calc_joinrel_size_estimate()` 使用局部变量：

```text
fkselec
jselec
pselec
nrows
joinquals
pushedquals
```

### 谁持有

`RelOptInfo` 持有 joinrel 的公共估算状态。

`restrictlist` 中的 `RestrictInfo` 仍归 planner context。

`joinquals`、`pushedquals` 是临时 list。

它们引用 `RestrictInfo`，不拥有 clause 本身。

### 谁释放

outer join 分支里，源码会：

```text
list_free(joinquals)
list_free(pushedquals)
```

释放的是 list cell。

不是 `RestrictInfo`。

joinrel、path、restrictinfo 等对象随 planner memory context 一起回收。

`eqjoinsel()` 中临时申请的 MCV match 数组会显式 `pfree()`。

统计 slot 通过：

```text
free_attstatsslot()
ReleaseVariableStats()
```

释放。

### ERROR 路径

如果 join type 不合法或 estimator 返回非法概率，源码可能 `elog(ERROR)`。

MemoryContext 会清理 planner 生命周期内的对象。

因此这里的 cleanup 模型是：

```text
局部临时资源尽量显式释放；
planner 长生命周期对象归 planner context；
ERROR 由上层 context cleanup 兜底。
```

## 9. 正确性与 fallback

join rows 估算不会改变 SQL 语义。

executor 仍按 join qual 真正匹配。

但估算必须满足几个 planner 正确性要求。

第一，概率必须在有效范围。

`join_selectivity()` 会检查结果是否在 `[0, 1]`。

越界直接报错。

第二，不能让普通 joinrel rows 变成不可用数值。

所以最终调用：

```text
clamp_row_est()
```

第三，outer join preserved side 下限不能丢。

LEFT JOIN 输出不能少于 outer rows，除非后续 pushed-down quals 再过滤。

第四，SEMI/ANTI 不能使用 inner join 的 Cartesian product 解释。

SEMI 输出最多与 outer side 同数量级。

ANTI 输出也是 outer side 的未匹配部分。

第五，没有 estimator 时必须 fallback。

operator 没有 `oprjoin` 时，`join_selectivity()` 返回 0.5。

这个值不准，但能让 path search 继续。

诊断时看到 0.5 风格的可疑估算，应检查：

```sql
SELECT oprname, oprjoin::regproc
FROM pg_operator
WHERE oid = '<operator_oid>'::oid;
```

## 10. 成本传播

joinrel rows 会进入 path cost。

以 hash join 为例：

```text
initial_cost_hashjoin()
  -> 用 outer_path->rows / inner_path->rows 估算 hash table、batch 和输入工作。

final_cost_hashjoin()
  -> path->jpath.path.rows = parent->rows 或 param_info->ppi_rows
  -> tlist cost 按输出 rows 计费
  -> hashjointuples 用 approx_tuple_count() 估算进入 qpqual 的 tuple 数
```

以 nested loop 为例：

```text
final_cost_nestloop()
  -> 普通 join 的 processed tuples 近似 outer_path_rows * inner_path_rows
  -> SEMI/ANTI 使用 semifactors 调整内侧扫描比例
  -> 输出 tlist 成本按 path rows 计费
```

所以 join cardinality 错误会产生两类后果：

```text
输出 rows 错:
  影响上层 join、sort、aggregate、limit、parallel gather。

处理 tuple 数错:
  影响 join algorithm 自身 CPU/I/O 成本。
```

这也是为什么 join rows 偏差比 base scan 偏差更容易放大。

## 11. 观测与诊断入口

诊断 join rows，按顺序做。

第一步，看输入节点。

```text
如果 join 两侧 scan 的 estimated rows 已经错很多，
先回到上一节的 restriction selectivity。
```

第二步，看 join 节点。

```text
Hash Join  (rows=...)
Merge Join (rows=...)
Nested Loop (rows=...)
```

把 join rows 与输入 rows 相乘关系对照。

第三步，区分 join type。

```text
INNER:
  是否接近 outer * inner * selectivity？

LEFT:
  是否至少保留 outer rows，再受 pushed-down quals 影响？

SEMI:
  是否最多按 outer rows 的比例输出？

ANTI:
  是否输出 outer rows 中未匹配部分？
```

第四步，查 operator estimator。

```sql
SELECT oprname, oprjoin::regproc
FROM pg_operator
WHERE oprname = '='
LIMIT 10;
```

第五步，查统计。

```sql
SELECT attname, n_distinct, null_frac, most_common_vals, most_common_freqs
FROM pg_stats
WHERE tablename IN ('orders', 'customers');
```

第六步，查约束。

```sql
SELECT conname, contype, conrelid::regclass, confrelid::regclass
FROM pg_constraint
WHERE conrelid IN ('orders'::regclass, 'customers'::regclass);
```

最后映射源码：

```text
joinrel 创建:
  joinrels.c: make_join_rel()
  relnode.c: build_join_rel()

join rows:
  costsize.c: set_joinrel_size_estimates()
  costsize.c: calc_joinrel_size_estimate()

operator selectivity:
  clausesel.c: clause_selectivity_ext()
  plancat.c: join_selectivity()
  selfuncs.c: eqjoinsel()

semi/anti cost factors:
  costsize.c: compute_semi_anti_join_factors()
```

## 12. 课堂实验

### 实验一：输入 rows 与 join rows 分离

```sql
DROP TABLE IF EXISTS s22_c;
DROP TABLE IF EXISTS s22_o;

CREATE TABLE s22_c AS
SELECT g AS id,
       CASE WHEN g <= 1000 THEN 'CN' ELSE 'US' END AS country
FROM generate_series(1, 10000) AS g;

CREATE TABLE s22_o AS
SELECT g AS id,
       ((g - 1) % 10000 + 1) AS customer_id,
       CASE WHEN g % 10 = 0 THEN 'paid' ELSE 'open' END AS status
FROM generate_series(1, 200000) AS g;

ANALYZE s22_c;
ANALYZE s22_o;
```

执行：

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM s22_o o
JOIN s22_c c ON c.id = o.customer_id
WHERE o.status = 'paid'
  AND c.country = 'CN';
```

记录：

```text
orders scan estimated/actual rows
customers scan estimated/actual rows
join estimated/actual rows
```

讨论：

```text
如果输入 rows 很准但 join rows 偏差大，焦点转向 join selectivity。
如果输入 rows 已经偏差大，join rows 偏差可能只是传播结果。
```

### 实验二：外键信息

给实验表增加约束和索引：

```sql
ALTER TABLE s22_c ADD PRIMARY KEY (id);
ALTER TABLE s22_o ADD FOREIGN KEY (customer_id) REFERENCES s22_c(id);
ANALYZE s22_c;
ANALYZE s22_o;
```

再次执行同一个 `EXPLAIN`。

源码问题：

```text
get_foreign_key_join_selectivity() 是否可能接管 c.id = o.customer_id？
它接管后，普通 clauselist_selectivity() 是否还会重复计算该 clause？
```

### 实验三：SEMI 与 INNER 的语义差异

比较：

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM s22_c c
WHERE EXISTS (
    SELECT 1
    FROM s22_o o
    WHERE o.customer_id = c.id
      AND o.status = 'paid'
);
```

与：

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT c.*
FROM s22_c c
JOIN s22_o o ON o.customer_id = c.id
WHERE o.status = 'paid';
```

讨论：

```text
EXISTS 的输出是外侧 customer 是否有匹配；
INNER JOIN 的输出是匹配 pair 数；
两者不能用同一个 rows 公式。
```

## 13. 源码练习

练习一：

在 `relnode.c` 中从 `build_join_rel()` 找到 `set_joinrel_size_estimates()`。

写出：

```text
joinrel 是何时创建的；
restrictlist 是何时构造的；
rows 是何时填入的。
```

练习二：

阅读 `calc_joinrel_size_estimate()` 的 switch。

用自己的话解释：

```text
为什么 LEFT JOIN 要把 nrows clamp 到 outer_rows？
为什么 FULL JOIN 同时 clamp 到 outer_rows 和 inner_rows？
为什么 SEMI 不乘 inner_rows？
```

练习三：

阅读 `join_selectivity()`。

回答：

```text
operator 没有 oprjoin 时返回什么？
返回值越界时发生什么？
```

练习四：

阅读 `eqjoinsel()`。

找出：

```text
MCV 统计在哪里读取；
ndistinct 在哪里使用；
JOIN_SEMI / JOIN_ANTI 从哪里分支。
```

练习五：

阅读 `compute_semi_anti_join_factors()`。

解释：

```text
为什么它同时计算 JOIN_SEMI/ANTI selectivity 和 JOIN_INNER selectivity？
match_count 如何服务后续 join path costing？
```

## 14. 常见误区

误区一：

```text
join rows 错了，一定是 join operator estimator 错。
```

不一定。

输入 rows、outer join 语义、外键、pushed-down quals 都可能是主因。

误区二：

```text
LEFT JOIN 的 rows 应该等于 inner join rows。
```

LEFT JOIN 保留左侧行。

源码中明确把 `nrows` 提升到至少 `outer_rows`。

误区三：

```text
SEMI JOIN 是 INNER JOIN 后再 DISTINCT 外侧。
```

执行结果可以这样类比。

但估算语义不同。

源码中 SEMI rows 是 `outer_rows * selectivity`。

误区四：

```text
foreign key 只影响 executor correctness。
```

外键约束也能参与 planner 的 join selectivity 修正。

误区五：

```text
同一个 joinrel 的所有 path rows 都一样。
```

普通 path 通常使用 parent joinrel rows。

Parameterized path 会使用 `ParamPathInfo.ppi_rows`。

## 15. 讨论题

问题一：

```text
一个 INNER JOIN 的 estimated rows 比 actual rows 小 100 倍。
你如何证明问题来自输入 rows，而不是 eqjoinsel()？
```

要求回答中引用：

```text
EXPLAIN 中两个输入节点的 rows/actual rows
calc_joinrel_size_estimate() 的 INNER 公式
至少一个统计 catalog 查询
```

问题二：

```text
为什么 outer join 下要区分 joinquals 和 pushedquals？
```

必须说明：

```text
preserved side 输出下限
RINFO_IS_PUSHED_DOWN()
pselec 的后置过滤语义
```

问题三：

```text
为什么 SEMI/ANTI join 的 selectivity 可以影响 cost，但 rows 解释与 INNER JOIN 不同？
```

提示：

```text
SEMI/ANTI 关注外侧是否存在匹配；
INNER 关注匹配 pair 数。
```

问题四：

```text
如果 operator 没有 oprjoin，PostgreSQL 为什么不直接放弃这个 join path？
```

重点：

```text
估算缺失不能破坏合法计划生成；
fallback 可能选差计划，但仍保持 planner 可继续。
```

## 16. 本节小结

本节的核心模型是：

```text
join cardinality = 输入 rows 传播 + join clause selectivity + join type 语义 + 约束修正 + 数值收束。
```

源码主链路是：

```text
make_join_rel()
  -> build_join_rel()
     -> build_joinrel_restrictlist()
     -> set_joinrel_size_estimates()
        -> calc_joinrel_size_estimate()
           -> get_foreign_key_join_selectivity()
           -> clauselist_selectivity()
              -> join_selectivity()
                 -> eqjoinsel()
        -> clamp_row_est()
```

诊断主链路是：

```text
先看输入 rows；
再看 join type；
再看 join clause estimator；
再看 FK 与 pushed-down quals；
最后看具体 join path cost。
```

本节最重要的工程规律：

```text
join rows 不是一个 estimator 的输出；
它是多个边界条件共同塑形的 planner 承诺。
```

下一节会继续追踪这个承诺如何在 `RelOptInfo`、`Path`、`ParamPathInfo`、`Plan` 中传播，以及为什么 PostgreSQL 要反复使用 `clamp_row_est()` 保持数值稳定。
