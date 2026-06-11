# PostgreSQL MCV、histogram、nullfrac 与 ndistinct

## 课程定位

前置知识：熟悉 `pg_statistic` 的基础字段和 slot，知道 `eqsel`、`scalarineqsel` 等选择率函数会把统计转换成 selectivity。

本节唯一主问题：

```text
MCV、histogram、nullfrac 和 ndistinct 分别回答什么问题，planner 为什么不能用其中任何一个单独替代其它统计？
```

核心矛盾：列分布摘要必须小到能存进 catalog 并快速读取，但又要同时解释热点值、长尾范围、空值和 distinct 基数。

学完后应能判断等值、范围、不等值和 IS NULL 估算分别依赖哪些统计，以及 rows 失真来自热点值、直方图边界还是 distinct 假设。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

上一节建立了 `pg_statistic` slot 模型；本节用最常见的列统计解释选择率估算如何组合。

05 目录的主线是 planner 如何把 SQL 语义、统计事实和成本模型压缩成可执行计划。本节只处理其中一个局部问题，不把相邻主题合并成百科式说明。

```text
SQL / Query 形态
  -> planner-local 状态
  -> 统计或约束参与判断
  -> Path rows / cost 改变
  -> Plan 节点体现选择结果
```

下一节会把 correlation 单独拉出来，解释同样 rows 为什么会对应完全不同的 index scan I/O 成本。

因此，本节的阅读边界是明确的：只解释这一个判断如何成立、如何失败、如何被观测，不追求覆盖优化器的所有入口。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
`ANALYZE` 从样本中提取 nullfrac、平均宽度、ndistinct、MCV、histogram；选择率函数先处理 NULL 和 MCV，再用 histogram 或 ndistinct 推断剩余长尾。
```

这句话要同时包含状态来源、判断时机和后续消费者。缺少其中任何一环，课程就会退化成函数名索引。

| 侧面 | 本节关注点 |
| --- | --- |
| 输入事实 | SQL 形态、catalog 统计、relids、操作符语义或样本分布。 |
| planner-local 状态 | 只在一次规划过程中存在，通常挂在 `PlannerInfo`、`RelOptInfo` 或局部统计结构上。 |
| 正确性边界 | 不能为了更低成本破坏 SQL 语义、权限边界、outer join null-extension 或统计含义。 |
| 性能收益 | rows 更接近现实、path 搜索更少走弯路、cost 比较更稳定。 |
| 诊断入口 | `EXPLAIN`、`pg_stats`、catalog 查询、gdb 断点和源码断点共同还原判断过程。 |

后文只沿这条链路展开：先看状态如何形成，再看它如何被消费、失效和观测。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/commands/analyze.c` | `compute_scalar_stats()`、`compute_distinct_stats()`、`analyze_mcv_list()` 生成列统计。 |
| 2 | `src/include/catalog/pg_statistic.h` | `STATISTIC_KIND_MCV`、`STATISTIC_KIND_HISTOGRAM`、`stadistinct` 解释存储语义。 |
| 3 | `src/backend/utils/adt/selfuncs.c` | `eqsel()`、`var_eq_const()`、`scalarineqsel()`、`mcv_selectivity()`、`histogram_selectivity()` 消费统计。 |
| 4 | `src/include/utils/selfuncs.h` | `VariableStatData`、`AttStatsSlot` 和选择率辅助函数声明。 |
| 5 | `src/backend/optimizer/path/clausesel.c` | `clauselist_selectivity()` 组合多个 clause 的选择率。 |
| 6 | `src/backend/statistics/extended_stats.c` | 多列统计在部分场景下修正单列独立性假设。 |

推荐阅读方式是先从入口函数建立时间轴，再读结构体字段，最后读 fallback 和观测点。不要按文件名排序读，也不要在第一个复杂分支里停太久。

```text
入口函数
  -> 本地状态结构
  -> 判断条件
  -> fallback 或保守路径
  -> rows / cost / catalog / plan 的可见结果
```

- 读本节源码时，始终把问题压回这一句：MCV、histogram、nullfrac 和 ndistinct 分别回答什么问题，planner 为什么不能用其中任何一个单独替代其它统计？
- 如果一个分支不能改变这个问题的答案，只做定位提示，不展开成旁支。
- 如果一个字段看起来像结论，要继续追问它由哪个阶段写入、被哪个阶段读取。
- 如果 SQL 现象和源码注释看起来冲突，先确认当前路径是否进入 fallback。
- 如果不同版本实现细节变化，优先保留当前源码中能验证的不变量。

## 4. 关键数据结构与状态边界

本节关注普通列统计中最常见的三类分布摘要：MCV、histogram 和 ndistinct。

它们不是三份互相独立的数据，而是同一次样本排序和去重过程的不同产物。

| 状态 | 写入端语义 | 读取端用途 |
| --- | --- | --- |
| `stanullfrac` | 样本中 NULL 比例 | 先把 NULL 分量从选择率中分离。 |
| `stadistinct` | distinct 个数或随 rows 缩放的比例 | `get_variable_numdistinct()`、group rows、join rows。 |
| `STATISTIC_KIND_MCV` | 高频非 NULL 值及频率 | equality、IN、部分 pattern / containment 估算。 |
| `STATISTIC_KIND_HISTOGRAM` | 去掉 MCV 后的排序边界 | range、inequality、prefix selectivity。 |
| `AttStatsSlot.values` | detoast 后的统计值数组 | 选择率函数逐项比较。 |
| `AttStatsSlot.numbers` | float4 频率或系数数组 | MCV frequency、correlation 等数值输入。 |
| `VariableStatData.statsTuple` | planner 读取到的统计 tuple | selectivity 函数的入口状态。 |
| `estimatedclauses` | 扩展统计中已被处理的 clause 位图 | 避免同一条件被重复估算。 |

写入端主要在 `compute_scalar_stats()`。

它先提取样本中的非 NULL、非过宽值。

然后按类型的 `<` operator 排序。

排序后，重复值相邻，源码可以统计 distinct、重复次数、MCV 候选和 correlation。

MCV 与 histogram 的关系很关键：

```text
MCV 保存明显高频值；
histogram 描述去掉 MCV 后的剩余非 NULL 分布；
两个 slot 不能简单相加成“完整排序数组”。
```

`stadistinct` 也有编码规则。

正数表示估计 distinct 个数。

负数表示 distinct 与总行数成比例缩放。

例如接近 unique 的列可能写成 `-1 * (1 - stanullfrac)`。

读取端从 `VariableStatData` 开始。

`examine_variable()` 找到统计 tuple。

`var_eq_const()`、`generic_restriction_selectivity()`、`ineq_histogram_selectivity()` 等函数再按需要读取 MCV 或 histogram。

`get_attstatsslot()` 返回的 `AttStatsSlot` 是调用者拥有的拷贝，用完要释放。

## 5. 从 SQL 现象进入源码

MCV 的典型现象是 equality rows 改善。

如果值在 MCV 中，`var_eq_const()` 可以直接使用对应 frequency。

如果值不在 MCV 中，选择率会用剩余质量在剩余 distinct 中分摊。

这就是为什么热点值和冷门值的估算可能差异明显。

histogram 的典型现象是 range rows 改善。

`x < const`、`x BETWEEN a AND b` 这类谓词会尝试用 histogram 边界定位 const 在剩余分布中的位置。

如果 histogram 缺失，range selectivity 只能走更粗的默认估算。

ndistinct 的典型现象出现在 aggregation、DISTINCT 和 join rows。

`n_distinct` 为负数时，表增长后 distinct 估算会随 `reltuples` 放大。

这对自增 id、近似唯一列和时间戳类数据很重要。

一个课堂实验可以这样做：

```sql
CREATE TABLE t AS
SELECT CASE WHEN g <= 50000 THEN 1 ELSE g END AS k
FROM generate_series(1, 100000) g;

ANALYZE t;
SELECT most_common_vals, most_common_freqs, histogram_bounds, n_distinct
FROM pg_stats
WHERE tablename = 't' AND attname = 'k';

EXPLAIN SELECT * FROM t WHERE k = 1;
EXPLAIN SELECT * FROM t WHERE k = 99999;
EXPLAIN SELECT * FROM t WHERE k BETWEEN 90000 AND 91000;
```

第一个 equality 命中 MCV。

第二个 equality 通常落在剩余分布。

range 谓词更多依赖 histogram。

如果你调小 statistics target，热点值可能仍保留，但 histogram 边界变少，范围估算会变粗。

## 6. 主流程源码 walkthrough

主流程从 `compute_scalar_stats()` 开始。

第一步，遍历样本，统计 NULL、非 NULL、宽度和过宽 varlena 数量。

NULL 只进入 `stanullfrac`。

过宽值可能计入 width 和 distinct 倾向，但不会参与后续完整排序比较。

第二步，把可排序值放入 `ScalarItem values[]`。

每个 item 记录 datum 和样本中的 tuple 序号。

tuple 序号后续用于 correlation：排序位置与物理样本顺序越接近，相关性越高。

第三步，用 sort support 按 `<` operator 排序。

排序后的相邻重复值形成 duplicate group。

源码利用 `tupnoLink` 避免重复调用 equality 比较。

第四步，扫描排序结果，计算 `ndistinct`、`nmultiple`、duplicate count 和 MCV 候选。

`track[]` 只保留最可能进入 MCV 的若干值。

是否最终写入 MCV，还要通过 `analyze_mcv_list()` 判断它们是否显著高于剩余值。

第五步，估算 `stadistinct`。

没有重复非 NULL 值时，源码倾向认为列接近 unique，并按非 NULL 比例写负数。

所有样本值都重复且没有过宽值时，可能认为 distinct 是固定小集合。

普通情况使用 Haas-Stokes Duj1 estimator，并对结果做 clamp。

第六步，生成 MCV slot。

源码切换到 `stats->anl_context`，复制 MCV 值和 frequency。

`stakind` 写 `STATISTIC_KIND_MCV`。

`staop` 写 equality operator。

`stacoll` 写列 collation。

第七步，生成 histogram slot。

源码先把 MCV item 从排序数组中折叠出去。

剩余值按等人口间隔抽取边界。

至少需要两个 distinct 边界，否则 histogram 省略。

第八步，生成 correlation slot。

`corr_xysum` 在排序扫描中累计。

源码用排序位置和原始样本位置计算相关系数。

这个 slot 后续会影响 btree index scan 的 heap page I/O cost。

第九步，选择率读取这些 slot。

`var_eq_const()` 优先看 MCV。

`mcv_selectivity()` 遍历 MCV values。

`histogram_selectivity()` 和 `ineq_histogram_selectivity()` 使用 histogram。

`get_variable_numdistinct()` 解释 `stadistinct`。

## 7. 生命周期 / ownership / cleanup

写入端的中间数组在列计算 context 中。

`values[]`、`track[]`、`tupnoLink[]`、sort support 相关对象都在处理一列时存在。

它们不会写入 catalog。

真正写入 catalog 的 MCV values、MCV freqs、histogram bounds 和 correlation numbers 会复制到 `stats->anl_context`。

这一点保证 `Analyze Column` reset 后，`update_attstats()` 仍能安全读取 `VacAttrStats`。

读取端的生命周期更短。

`get_attstatsslot()` 返回的数组只为当前选择率函数服务。

选择率函数完成后调用 `free_attstatsslot()`。

`VariableStatData` 的 stats tuple 由 `ReleaseVariableStats()` 释放。

ERROR 路径由 memory context 兜底，但正常路径仍要释放，避免复杂 query 规划时占用过多内存。

统计本身的失效由新的 `ANALYZE` 或 DDL 触发。

某次 planner 读取到的 MCV/histogram 是 catalog snapshot 下的事实，不会反映查询执行期间发生的数据变化。

## 8. 正确性机制层次

MCV 的正确性来自 equality operator。

没有与谓词兼容的 equality 语义，不能把某个值当成 MCV 命中。

histogram 的正确性来自 ordering operator 和 collation。

边界只在相同排序语义下有意义。

MCV 与 histogram 的分工也必须保持。

histogram 描述的是去掉 MCV 后的剩余分布。

如果读取端把 histogram 当成全量分布，会双重计算热点值。

ndistinct 的正确性来自编码解释。

负数不是错误数据，而是告诉 planner 随 relation rows 缩放。

NULL 分量必须单独处理。

MCV/histogram 通常只描述非 NULL 值，选择率函数要把 NULL 比例从总质量中扣除。

宽值 fallback 也是语义选择。

过宽值被认为很难成为 MCV，通常按 distinct 倾向处理。

这会牺牲某些极端数据集的精度，但避免统计阶段被大 datum 拖垮。

## 9. 错误路径 / 异常路径 / fallback

没有 MCV slot 时，equality selectivity 会退回 ndistinct 或默认值。

有 MCV 但 const 不在 MCV 中时，源码使用剩余 frequency 和剩余 distinct 估算。

没有 histogram 时，range selectivity 走默认或 operator-specific fallback。

histogram 边界太少时，估算会更粗。

没有重复值时，MCV slot 可能为空。

这对 unique-like 列是正常现象。

样本太小或 target 太低时，热点值可能没有进入 MCV。

过宽 varlena 值太多时，详细分布会受限。

统计陈旧时，所有 slot 都可能精确描述旧分布而不是当前数据。

诊断时要把这些情况分开：

```text
slot 缺失；
slot 存在但谓词没有命中；
slot 命中但样本陈旧；
slot 本身无法表达多列相关性。
```

## 10. 成本、资源与跨模块传播

MCV、histogram 和 ndistinct 最先影响 restriction rows。

restriction rows 再传播到 index path、bitmap path、join order、join method、aggregate rows 和 sort cost。

MCV 对热点 equality 最敏感。

histogram 对 range selectivity 最敏感。

ndistinct 对 group、distinct、join 和非 MCV equality fallback 很关键。

提高 target 会增加 analyze 阶段排序、比较、MCV tracking 和 catalog 数组大小。

读取阶段也会有更多 detoast/copy 和比较成本。

这些成本通常小于错误计划带来的执行成本，但在宽表、大量列和频繁 auto-analyze 环境下不能忽视。

本节诊断顺序是：

```text
先看 pg_stats 中三类摘要是否存在；
再确认谓词类型会读取哪类摘要；
再看 const 是否命中 MCV 或落入 histogram 区间；
最后判断是否需要提高 target 或创建扩展统计。
```

可迁移规律是：

```text
单列统计用有限摘要表达分布；
每个摘要只擅长一类问题，不能指望一个 slot 解释所有 rows 偏差。
```

## 11. 观测与诊断入口

planner 内部很多状态没有直接 SQL 视图，但可以通过计划形态和 catalog 摘要推断。

`pg_stats.most_common_vals` 和 `most_common_freqs` 能直接看到热点值。

`pg_stats.histogram_bounds` 可以判断查询常量落在哪个区间。

`pg_stats.null_frac` 能解释 `IS NULL` 与普通比较 rows 的差异。

`pg_stats.n_distinct` 为负时，估算 distinct 会随当前 relation rows 变化。

| 入口 | 能看到什么 | 看不到什么 |
| --- | --- | --- |
| `EXPLAIN` | 估算 rows、cost、Filter 位置和选中 path。 | 未入选的多数候选 path 和中间判断细节。 |
| `EXPLAIN (ANALYZE, BUFFERS)` | 估算与实际的偏差、buffer 访问形态。 | planner 当时的所有统计 slot 读取过程。 |
| `pg_stats` / catalog | 统计摘要、目标、对象定义和部分扩展统计。 | 选择率函数如何组合这些摘要。 |
| gdb 断点 | 当前 `PlannerInfo`、`RelOptInfo`、slot 或依赖结构。 | 没有复现实验时难以判断业务代表性。 |

日常诊断通常先用 SQL 和 EXPLAIN 缩小范围，再用源码断点验证关键状态。

## 12. 课堂实验一：从计划反推状态

第一个实验只改变查询形态或统计状态，目标是让计划差异能回到本节主问题。

构造一个 90% 热点值加 10% 长尾的列，比较热点等值和长尾等值的 rows。

```text
EXPLAIN (ANALYZE, BUFFERS)
  -> 记录 estimated rows 与 actual rows
  -> 记录 Filter / Index Cond / Join Filter 的位置
  -> 对照 pg_stats 或 catalog 状态
```

在均匀分布列上删除 MCV 命中机会，观察 histogram 对范围查询的估算。

```text
EXPLAIN (ANALYZE, BUFFERS)
  -> 记录 estimated rows 与 actual rows
  -> 记录 Filter / Index Cond / Join Filter 的位置
  -> 对照 pg_stats 或 catalog 状态
```

实验结果不要求某个固定计划，因为硬件、数据量、GUC 和缓存都会影响 cost；要求是解释计划变化时能回到同一个源码判断。

## 13. 课堂实验二：改变输入验证边界

第二个实验改变输入分布或配置，用来验证 fallback 和成本传播。

把大量 NULL 加入列，比较 `col = value` 与 `col IS NULL` 的估算变化。

- 记录变更前的 catalog 摘要。
- 执行一次明确的 `ANALYZE`，避免把统计陈旧误认为源码行为。
- 只改变一个变量，保留其它 GUC 和 SQL 形态。
- 把计划差异写成“输入变化 -> planner 状态变化 -> path 变化”。

修改 statistics target，观察 MCV 截断如何改变长尾值估算。

- 记录变更前的 catalog 摘要。
- 执行一次明确的 `ANALYZE`，避免把统计陈旧误认为源码行为。
- 只改变一个变量，保留其它 GUC 和 SQL 形态。
- 把计划差异写成“输入变化 -> planner 状态变化 -> path 变化”。

如果实验没有出现预期差异，先检查数据规模是否太小、统计是否刷新、GUC 是否强行禁用了候选路径。

## 14. 源码阅读练习

源码练习以断点和变量观察为主，不要求修改代码。

- 在 `src/backend/commands/analyze.c` 的入口函数附近设置断点，确认本节主路径是否进入。
- 在构造 planner-local 状态的位置打印关键字段，确认它们来自 SQL、catalog 还是样本。
- 在 fallback 分支设置断点，验证输入条件不满足时系统如何继续生成计划。
- 在选择率或 cost 消费点打印 rows/cost，观察误差从哪里开始放大。
- 在 `EXPLAIN` 结果中只解释能回到源码状态的差异，不把缓存波动当成 planner 结论。

先确认 SQL 现象，再回到 planner 中保存状态的位置。

先看状态字段由谁写入，再看后续阶段如何消费。

不要把单个布尔字段当成完整语义，必须连同 relids、生命周期和调用场景一起解释。

如果执行计划看起来反直觉，优先检查 rows、width、cost 和统计信息是否一致。

如果源码路径里出现 fallback，先问这个 fallback 是保护 correctness，还是保护 planner 可以继续给出计划。

如果一个判断依赖 catalog tuple，要区分 catalog 中保存的事实和 planner 从事实推导出的估算。

阅读源码时保留真实实现的历史痕迹，不要把多个分支整理成想象中的干净架构。

## 15. 常见误区

把 planner 的估算当成真实执行承诺。

修正方式是把判断链写完整：输入是什么，状态在哪里，源码如何分支，计划如何体现。

只看最终 plan，不找第一个 rows 或 cost 失真点。

修正方式是把判断链写完整：输入是什么，状态在哪里，源码如何分支，计划如何体现。

把 catalog 统计字段孤立解释，不看操作符、collation、relids 或生命周期。

修正方式是把判断链写完整：输入是什么，状态在哪里，源码如何分支，计划如何体现。

看到 fallback 就认为是 bug，而不是先确认前置条件是否满足。

修正方式是把判断链写完整：输入是什么，状态在哪里，源码如何分支，计划如何体现。

把提高 statistics target 当成所有慢 SQL 的通用修复。

修正方式是把判断链写完整：输入是什么，状态在哪里，源码如何分支，计划如何体现。

在没有复现实验的情况下，只凭函数名推断优化器行为。

修正方式是把判断链写完整：输入是什么，状态在哪里，源码如何分支，计划如何体现。

误区本身不是问题，问题是停在误区上继续调参。优化器诊断需要把每个猜测压回可验证状态。

## 16. 讨论题

为什么 MCV 和 histogram 同时存在时，histogram 要排除 MCV？

回答时要给出一个可复现实验或一个源码入口，避免只给抽象观点。

一列的 ndistinct 正负号对长期运行表有什么不同含义？

回答时要给出一个可复现实验或一个源码入口，避免只给抽象观点。

如果热点值变化很快，增加 statistics target 和提高 analyze 频率哪个更有效？

回答时要给出一个可复现实验或一个源码入口，避免只给抽象观点。

- 哪些结论是当前源码版本的实现细节？
- 哪些结论是 SQL 语义或 catalog 语义带来的长期边界？
- 哪些判断会受数据分布、硬件、缓存或 GUC 影响？
- 如果要给生产库建议，需要先收集哪些证据？

## 17. 源码断点矩阵

这一节课到这里已经有了主流程，但真正做内核诊断时，还需要知道断点应该落在哪里。断点矩阵不是为了覆盖所有函数，而是为了让 SQL 现象能回到一个有限的状态集合。

| 断点层次 | 目的 | 退出条件 |
| --- | --- | --- |
| 入口断点 | 确认本次 SQL 是否进入本节主路径。 | 能看到目标 relation、clause、统计对象或 cost path。 |
| 状态写入断点 | 确认关键字段第一次被赋值的位置。 | 能说明字段来自 SQL、catalog、样本还是默认值。 |
| 判断分支断点 | 确认为什么选择精确路径或 fallback。 | 能列出至少一个导致分支的布尔条件。 |
| 消费断点 | 确认 rows、cost、path 或 plan 节点如何使用前面状态。 | 能把状态变化映射到 `EXPLAIN` 中的一个可见差异。 |

断点入口 1：`src/backend/commands/analyze.c`。`compute_scalar_stats()`、`compute_distinct_stats()`、`analyze_mcv_list()` 生成列统计。

在这个位置不要一次性展开整份文件。先确认当前调用栈是否服务本节唯一主问题，再决定是否继续向下单步。

如果这里没有被命中，优先检查 SQL 形态、统计是否存在、GUC 是否关闭了相关路径，以及本节机制是否适用于当前 relation 类型。

如果这里被命中但状态为空，通常说明前置 catalog、样本、relids 或 clause 形态没有满足条件。这个结论比“planner 没优化”更有诊断价值。

断点入口 2：`src/include/catalog/pg_statistic.h`。`STATISTIC_KIND_MCV`、`STATISTIC_KIND_HISTOGRAM`、`stadistinct` 解释存储语义。

在这个位置不要一次性展开整份文件。先确认当前调用栈是否服务本节唯一主问题，再决定是否继续向下单步。

如果这里没有被命中，优先检查 SQL 形态、统计是否存在、GUC 是否关闭了相关路径，以及本节机制是否适用于当前 relation 类型。

如果这里被命中但状态为空，通常说明前置 catalog、样本、relids 或 clause 形态没有满足条件。这个结论比“planner 没优化”更有诊断价值。

断点入口 3：`src/backend/utils/adt/selfuncs.c`。`eqsel()`、`var_eq_const()`、`scalarineqsel()`、`mcv_selectivity()`、`histogram_selectivity()` 消费统计。

在这个位置不要一次性展开整份文件。先确认当前调用栈是否服务本节唯一主问题，再决定是否继续向下单步。

如果这里没有被命中，优先检查 SQL 形态、统计是否存在、GUC 是否关闭了相关路径，以及本节机制是否适用于当前 relation 类型。

如果这里被命中但状态为空，通常说明前置 catalog、样本、relids 或 clause 形态没有满足条件。这个结论比“planner 没优化”更有诊断价值。

断点入口 4：`src/include/utils/selfuncs.h`。`VariableStatData`、`AttStatsSlot` 和选择率辅助函数声明。

在这个位置不要一次性展开整份文件。先确认当前调用栈是否服务本节唯一主问题，再决定是否继续向下单步。

如果这里没有被命中，优先检查 SQL 形态、统计是否存在、GUC 是否关闭了相关路径，以及本节机制是否适用于当前 relation 类型。

如果这里被命中但状态为空，通常说明前置 catalog、样本、relids 或 clause 形态没有满足条件。这个结论比“planner 没优化”更有诊断价值。

断点入口 5：`src/backend/optimizer/path/clausesel.c`。`clauselist_selectivity()` 组合多个 clause 的选择率。

在这个位置不要一次性展开整份文件。先确认当前调用栈是否服务本节唯一主问题，再决定是否继续向下单步。

如果这里没有被命中，优先检查 SQL 形态、统计是否存在、GUC 是否关闭了相关路径，以及本节机制是否适用于当前 relation 类型。

如果这里被命中但状态为空，通常说明前置 catalog、样本、relids 或 clause 形态没有满足条件。这个结论比“planner 没优化”更有诊断价值。

断点入口 6：`src/backend/statistics/extended_stats.c`。多列统计在部分场景下修正单列独立性假设。

在这个位置不要一次性展开整份文件。先确认当前调用栈是否服务本节唯一主问题，再决定是否继续向下单步。

如果这里没有被命中，优先检查 SQL 形态、统计是否存在、GUC 是否关闭了相关路径，以及本节机制是否适用于当前 relation 类型。

如果这里被命中但状态为空，通常说明前置 catalog、样本、relids 或 clause 形态没有满足条件。这个结论比“planner 没优化”更有诊断价值。

## 18. 状态到计划的反向诊断

正向读源码是从入口走向 plan；排查慢 SQL 时经常要反过来，从 `EXPLAIN` 的异常节点倒推哪个状态不可信。

| 反向线索 | 优先怀疑 |
| --- | --- |
| 叶子 scan rows 已经偏差很大 | 单列统计、表达式统计、predicate 分类或统计陈旧。 |
| join rows 在第一个 join 后突然放大 | join selectivity、列相关性、join order 约束或外连接语义。 |
| rows 近似正确但 scan 成本离谱 | correlation、成本参数、缓存状态或 page 估算模型。 |
| 计划缺少预期 index path | qual 形态、操作符族、collation、下推安全性或 parameterized path。 |
| 扩展统计没有生效 | 统计对象未 analyze、clause 不兼容、表达式树不匹配或 inherit 标记不一致。 |

反向检查 `stanullfrac`：列值为 NULL 的行比例。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

反向检查 `stadistinct`：非空 distinct 数或随行数缩放的 distinct 比例。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

反向检查 `STATISTIC_KIND_MCV`：热点非空值与频率，适合等值命中。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

反向检查 `STATISTIC_KIND_HISTOGRAM`：去掉 MCV 后的非空值分布边界，适合范围估算。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

反向检查 `AttStatsSlot.numbers`：MCV 频率数组或其它数值摘要。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

反向检查 `AttStatsSlot.values`：MCV 值或 histogram bounds。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

反向检查 `VariableStatData.vartype`：选择率函数判断比较函数和 Datum 解释方式的类型信息。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

反向检查 `estimatedclauses`：扩展统计介入后，标记哪些 clause 已被特殊估算。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

## 19. 主流程中的可验证边界

下面把主流程拆成可验证边界。每个边界都应能用一个 SQL 变体、一个 catalog 查询或一个断点确认。

边界 1：`compute_scalar_stats()` 从样本中跳过 NULL，处理过宽 varlena，按排序操作符排序。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 2：排序后扫描重复值，提取最值得保存的 MCV，并估算 distinct。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 3：MCV 从值数组中被折叠出去，剩余值再构造等深 histogram。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 4：correlation 也在这个排序结果上计算，但本节只把它当作下节入口。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 5：`eqsel()` 遇到 `var = const` 时进入 `var_eq_const()`。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 6：`var_eq_const()` 先排除 NULL，再检查 const 是否命中 MCV。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 7：如果命中 MCV，selectivity 直接使用对应频率。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 8：如果未命中 MCV，源码用剩余非空比例和剩余 distinct 数估算长尾单值频率。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 9：`scalarineqsel()` 对范围比较先处理 MCV，再用 histogram 估算非 MCV 部分。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

## 20. 复盘案例：从一个估算偏差回到源码

课堂复盘不要求固定数据集，而是要求路径完整。下面四类案例可以套用到本节不同主题。

案例：估算明显偏小。常见原因是多个条件被当成独立、热点值没有进入 MCV、predicate 没有在预期层级执行，或者扩展统计没有命中。

- 先保存 `EXPLAIN (ANALYZE, BUFFERS)`，标记第一个估算失真节点。
- 查询 `pg_stats`、`pg_statistic` 或扩展统计视图，确认统计是否存在且新鲜。
- 在本节源码入口和消费点各设一个断点，确认状态是否被写入和读取。
- 只做一个最小变更，再观察 plan 是否朝预期方向移动。

复盘结论应该写成因果链，而不是写成单句调参建议。

案例：估算明显偏大。常见原因是 histogram 太粗、ndistinct 过低、NULL 比例变化、过滤条件被延迟到更高层级，或者 fallback 选择率过于保守。

- 先保存 `EXPLAIN (ANALYZE, BUFFERS)`，标记第一个估算失真节点。
- 查询 `pg_stats`、`pg_statistic` 或扩展统计视图，确认统计是否存在且新鲜。
- 在本节源码入口和消费点各设一个断点，确认状态是否被写入和读取。
- 只做一个最小变更，再观察 plan 是否朝预期方向移动。

复盘结论应该写成因果链，而不是写成单句调参建议。

案例：rows 接近但 plan 仍慢。常见原因是物理访问模型、correlation、缓存、并行成本、work_mem 或上层节点资源消耗没有被真实反映。

- 先保存 `EXPLAIN (ANALYZE, BUFFERS)`，标记第一个估算失真节点。
- 查询 `pg_stats`、`pg_statistic` 或扩展统计视图，确认统计是否存在且新鲜。
- 在本节源码入口和消费点各设一个断点，确认状态是否被写入和读取。
- 只做一个最小变更，再观察 plan 是否朝预期方向移动。

复盘结论应该写成因果链，而不是写成单句调参建议。

案例：创建对象后计划不变。常见原因是没有重新 analyze、表达式树不匹配、统计对象 kind 不适用于当前 clause，或者 planner 已经由更强信息完成估算。

- 先保存 `EXPLAIN (ANALYZE, BUFFERS)`，标记第一个估算失真节点。
- 查询 `pg_stats`、`pg_statistic` 或扩展统计视图，确认统计是否存在且新鲜。
- 在本节源码入口和消费点各设一个断点，确认状态是否被写入和读取。
- 只做一个最小变更，再观察 plan 是否朝预期方向移动。

复盘结论应该写成因果链，而不是写成单句调参建议。

## 21. 版本边界与源码阅读陷阱

本课基于指定的本地源码提交。优化器代码会持续演进，因此课程强调的是当前源码可验证的不变量，而不是把某个局部分支当作永久接口。

注释里提到的历史限制可能已经被附近代码部分修正，读注释时要结合当前调用者。

把这类问题写入诊断报告时，要说明它影响的是源码判断、统计输入、成本模型还是运行时环境。

同名字段在不同上下文中的解释可能不同，尤其是 relids、inherit、kind、op 和 collation。

把这类问题写入诊断报告时，要说明它影响的是源码判断、统计输入、成本模型还是运行时环境。

planner 里的默认估算不是错误处理，而是缺少精确信息时继续规划的机制。

把这类问题写入诊断报告时，要说明它影响的是源码判断、统计输入、成本模型还是运行时环境。

EXPLAIN 不显示大多数中间候选 path，不能只凭最终 plan 判断搜索空间中发生了什么。

把这类问题写入诊断报告时，要说明它影响的是源码判断、统计输入、成本模型还是运行时环境。

测试数据太小会让 seq scan、index scan、join path 的成本差异被启动成本淹没。

把这类问题写入诊断报告时，要说明它影响的是源码判断、统计输入、成本模型还是运行时环境。

缓存状态和硬件延迟会影响实际耗时，但不会反向改变 planner 已经做出的估算。

把这类问题写入诊断报告时，要说明它影响的是源码判断、统计输入、成本模型还是运行时环境。

## 22. 课堂练习检查清单

完成本节练习时，可以用下面的清单检查材料是否闭环。

- 是否明确本节唯一主问题，没有把相邻主题混成一个大问题。
- 是否给出至少一个 SQL 或 catalog 现象，并能回到源码解释。
- 是否指出状态由谁创建、谁读取、何时失效。
- 是否区分 correctness fallback 和 cost fallback。
- 是否说明哪些结论依赖数据分布、统计新鲜度、GUC 或硬件。
- 是否能在计划中指出一个可观察结果，而不是只描述内部函数。
- 是否避免把默认估算、保守路径或未命中扩展统计直接称为 bug。
- 是否能给出下一步验证实验，而不是直接给生产修复建议。

如果清单中有任一项无法回答，说明还需要回到源码入口或实验设计补证据。

## 23. 实验记录模板

这一节的实验记录建议固定成同一种格式，便于后续课程继续复用。

记录对象：PostgreSQL MCV、histogram、nullfrac 与 ndistinct。

记录第一项：SQL 形态。

写清楚谓词、统计对象、索引、数据分布和相关 GUC，不要只贴最终计划。

记录第二项：统计状态。

至少保存 `pg_stats` 或相关 catalog 查询结果，并注明 analyze 发生在实验前还是实验后。

记录第三项：估算位置。

在计划树中标出第一个 estimated rows 与 actual rows 明显分离的节点。

记录第四项：源码入口。

把本节第 3 节中的入口函数映射到这次实验，不要临时跳到无关模块。

记录第五项：状态字段。

只记录能解释本节主问题的字段，避免把 gdb 输出变成无边界日志。

记录第六项：变更动作。

一次只改变一个因素：统计目标、数据分布、SQL 写法、统计对象或成本参数。

记录第七项：计划变化。

说明变化体现在 rows、cost、Filter 位置、path 类型还是最终节点选择。

记录第八项：结论边界。

把结论限定在当前数据分布和源码版本内；不能从一个小表实验直接推出生产库规则。

这份记录模板的目的，是把 runtime 现象压回源码可验证的状态链。

## 24. 课后源码索引

课后复习 PostgreSQL MCV、histogram、nullfrac 与 ndistinct 时，可以把源码索引压缩成三轮阅读。

第一轮只看入口，确认本节主问题在哪里被提出。

第二轮只看状态，确认字段、catalog tuple 或局部结构在哪里被写入。

第三轮只看消费点，确认 rows、cost、path 或 plan 如何体现这个状态。

- `src/backend/commands/analyze.c`：`compute_scalar_stats()`、`compute_distinct_stats()`、`analyze_mcv_list()` 生成列统计。

- `src/include/catalog/pg_statistic.h`：`STATISTIC_KIND_MCV`、`STATISTIC_KIND_HISTOGRAM`、`stadistinct` 解释存储语义。

- `src/backend/utils/adt/selfuncs.c`：`eqsel()`、`var_eq_const()`、`scalarineqsel()`、`mcv_selectivity()`、`histogram_selectivity()` 消费统计。

- `src/include/utils/selfuncs.h`：`VariableStatData`、`AttStatsSlot` 和选择率辅助函数声明。

- `src/backend/optimizer/path/clausesel.c`：`clauselist_selectivity()` 组合多个 clause 的选择率。

- `src/backend/statistics/extended_stats.c`：多列统计在部分场景下修正单列独立性假设。

复习时不要把这些源码入口平均用力。入口、状态和消费点能串起来，才算读完本节。

如果只能记住一个动作，就是从 `EXPLAIN` 中找第一个异常节点，再回到这里选最小源码入口。

## 25. 本节小结

MCV 回答热点值，histogram 回答非热点范围，nullfrac 回答空值比例，ndistinct 回答长尾基数。

选择率函数会按语义组合这些摘要，而不是把它们平均成一个万能数字。

定位 rows 估错时，先看谓词类型，再看对应统计是否存在、是否新鲜、是否覆盖该常量。

```text
runtime 现象
  -> planner 状态
  -> 源码判断
  -> rows / cost / plan 表现
  -> 可迁移诊断规律
```

本节结束时，应该能把一个计划现象翻译成源码中的状态判断，而不是只记住某个函数名。

下一节继续沿这个方向推进：用同样方法把相邻主题拆成一个主问题、一条状态链和一个可验证实验。
