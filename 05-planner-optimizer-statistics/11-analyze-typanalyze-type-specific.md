# PostgreSQL typanalyze 与类型相关统计

## 课程定位

前置知识：熟悉 PostgreSQL 类型系统、操作符、opclass、collation，以及 planner 选择率函数会读取不同统计 kind。

本节唯一主问题：

```text
为什么 `ANALYZE` 要通过 `pg_type.typanalyze` 把统计语义交给数据类型，而不是把所有列都当成可排序标量处理？
```

核心矛盾：统一统计框架能让 planner 共享 catalog 和选择率入口；但数组、范围、tsvector、自定义类型和普通标量需要完全不同的分布摘要。

学完后应能判断一个类型为什么能生成 MCV/histogram/correlation，为什么只能生成 distinct，或者为什么需要自定义统计 kind 和选择率函数配套。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

上一节说明了 `ANALYZE` 如何拿到样本；本节看样本为什么不能被所有数据类型用同一种算法解释。

05 目录的主线是 planner 如何把 SQL 语义、统计事实和成本模型压缩成可执行计划。本节只处理其中一个局部问题，不把相邻主题合并成百科式说明。

```text
SQL / Query 形态
  -> planner-local 状态
  -> 统计或约束参与判断
  -> Path rows / cost 改变
  -> Plan 节点体现选择结果
```

下一节会把 typanalyze 产出的 `VacAttrStats` 落到 `pg_statistic` 的 slot 存储模型中。

因此，本节的阅读边界是明确的：只解释这一个判断如何成立、如何失败、如何被观测，不追求覆盖优化器的所有入口。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
`examine_attribute()` 为列建立 `VacAttrStats`，优先调用类型自己的 `typanalyze`，否则回退 `std_typanalyze()`；typanalyze 选择 `compute_stats`、样本需求和 slot 含义，后续 `ANALYZE` 只按这个契约执行。
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
| 1 | `src/backend/commands/analyze.c` | `examine_attribute()`、`std_typanalyze()`、`compute_scalar_stats()`、`compute_distinct_stats()` 是默认路径。 |
| 2 | `src/include/commands/vacuum.h` | `VacAttrStats` 说明 typanalyze 需要填哪些字段。 |
| 3 | `src/include/catalog/pg_type.h` | `typanalyze` 字段把数据类型和分析函数连接起来。 |
| 4 | `src/include/catalog/pg_proc.dat` | 声明 `array_typanalyze`、`range_typanalyze`、`multirange_typanalyze`、`ts_typanalyze`。 |
| 5 | `src/backend/utils/adt/array_typanalyze.c` | 数组列统计 most common elements 和 distinct element count。 |
| 6 | `src/backend/utils/adt/rangetypes_typanalyze.c` | 范围和 multirange 统计 bounds 与 length 分布。 |
| 7 | `src/backend/tsearch/ts_typanalyze.c` | tsvector 统计 lexeme 分布。 |
| 8 | `src/include/catalog/pg_statistic.h` | 统计 kind 编码必须与 typanalyze 和选择率函数共同解释。 |

推荐阅读方式是先从入口函数建立时间轴，再读结构体字段，最后读 fallback 和观测点。不要按文件名排序读，也不要在第一个复杂分支里停太久。

```text
入口函数
  -> 本地状态结构
  -> 判断条件
  -> fallback 或保守路径
  -> rows / cost / catalog / plan 的可见结果
```

- 读本节源码时，始终把问题压回这一句：为什么 `ANALYZE` 要通过 `pg_type.typanalyze` 把统计语义交给数据类型，而不是把所有列都当成可排序标量处理？
- 如果一个分支不能改变这个问题的答案，只做定位提示，不展开成旁支。
- 如果一个字段看起来像结论，要继续追问它由哪个阶段写入、被哪个阶段读取。
- 如果 SQL 现象和源码注释看起来冲突，先确认当前路径是否进入 fallback。
- 如果不同版本实现细节变化，优先保留当前源码中能验证的不变量。

## 4. 关键数据结构与状态边界

`typanalyze` 是 `ANALYZE` 主流程和类型语义之间的边界。

主流程负责采样、memory context、catalog 写入和进度汇报。

类型相关函数负责回答：这批样本应该怎样被解释成 planner 能消费的统计摘要。

| 状态 | 语义边界 |
| --- | --- |
| `VacAttrStats.attrtypid` | 被分析数据的实际类型；表达式索引会使用表达式类型，而不是索引存储类型。 |
| `VacAttrStats.attrcollid` | 比较、排序和选择率解释所用 collation。 |
| `VacAttrStats.attrtype` | `pg_type` 行的拷贝，包含 `typanalyze`、长度、byval、align 等信息。 |
| `VacAttrStats.compute_stats` | typanalyze 选择的第二阶段计算函数。 |
| `VacAttrStats.minrows` | 该类型希望采集的最小样本规模。 |
| `VacAttrStats.extra_data` | typanalyze 传给 compute 函数的私有状态，例如默认 scalar 统计里的等值/排序 operator。 |
| `stakind[]` / `staop[]` / `stacoll[]` | 输出 slot 的解释协议。 |
| `stavalues[]` / `stanumbers[]` | 具体统计值，必须分配在 `anl_context` 中。 |

`VacAttrStats` 在 `src/include/commands/vacuum.h` 中定义。

注释里最重要的约定是：

```text
typanalyze 返回 true 时，必须设置 compute_stats 和 minrows；
compute_stats 如果产生有用统计，要设置 stats_valid；
指针型输出必须放在 anl_context，而不是当前临时 context。
```

这把类型扩展点限制在一个明确 contract 内。

类型函数可以改变 slot 语义，例如数组统计保存元素统计，range 统计保存 bounds/length 信息。

但它不能绕过主流程的采样、catalog 写入和 cleanup。

`std_typanalyze()` 是默认实现。

它先处理 `attstattarget < 0` 的情况，回退到 `default_statistics_target`。

然后查找该类型的默认 `<` 和 `=` operator。

两者都有时，选择 `compute_scalar_stats()`。

只有 `=` 时，选择 distinct 相关统计。

都缺失时，只能走 trivial 统计，通常只能得到 nullfrac 和 width。

这个选择不是功能偏好，而是统计摘要需要操作符语义支撑。

没有排序 operator，就不能构造有意义的 scalar histogram。

没有 equality operator，就不能判断 MCV 中的“相同值”。

## 5. 从 SQL 现象进入源码

typanalyze 的外部现象常出现在 `pg_stats` 的 slot 形态里。

普通标量类型通常能看到：

```text
most_common_vals
most_common_freqs
histogram_bounds
correlation
n_distinct
null_frac
avg_width
```

数组、tsvector、range、multirange 等类型的统计形态会不同。

它们通过类型专用 typanalyze 与专用 selectivity 函数约定 slot 含义。

一个诊断路径是：

```text
查询 pg_type.typanalyze
  -> 确认是否有类型专用函数
  -> 重新 ANALYZE
  -> 查看 pg_stats / pg_stats_ext 的 slot
  -> 在 analyze.c 或类型专用文件断点
```

示例：

```sql
SELECT typname, typanalyze::regproc
FROM pg_type
WHERE typname IN ('int4', '_int4', 'tsvector', 'int4range');
```

`int4` 通常依赖 `std_typanalyze()`。

数组类型会使用 `array_typanalyze`。

range 和 multirange 类型会使用 `range_typanalyze` 或 `multirange_typanalyze`。

如果某列没有 histogram，不要立刻认为 `ANALYZE` 没运行。

可能原因包括：

```text
类型没有排序 operator；
样本中非 NULL 值太少；
MCV 已经覆盖几乎全部可表示分布；
attstattarget 为 0；
类型专用 typanalyze 选择了不同 slot 协议。
```

`EXPLAIN` rows 偏差要和 slot 缺失一起看。

如果类型选择率函数不读取某个 slot，即使 catalog 里有数据，也不会影响相应谓词。

## 6. 主流程源码 walkthrough

入口仍然是 `src/backend/commands/analyze.c` 的 `examine_attribute()`。

第一步，`attribute_is_analyzable()` 检查列是否应该参与分析。

dropped 列会跳过。

`attstattarget=0` 的列不会新建列统计。

显式列列表、继承场景和索引表达式会影响哪些 `VacAttrStats` 被创建。

第二步，`examine_attribute()` 创建 `VacAttrStats`。

普通列使用 `pg_attribute` 中的类型、typmod 和 collation。

表达式索引使用表达式树的 `exprType()`、`exprTypmod()` 和表达式或索引列 collation。

这一步避免把索引 opclass 的存储类型误当成被分析值的语义类型。

第三步，源码用 `SearchSysCacheCopy1(TYPEOID, ...)` 拷贝 `pg_type` 行。

这里用 copy 是为了让 `VacAttrStats.attrtype` 在分析期间稳定可用。

如果类型 OID 查不到，直接 ERROR，因为后续 slot 语义无法成立。

第四步，初始化五个统计 slot 的默认 element type 信息。

默认情况下，slot values 的元素类型等于被分析类型。

自定义 typanalyze 如果要存不同元素类型，必须覆盖 `statypid`、`statyplen`、`statypbyval` 和 `statypalign`。

第五步，调用类型专用 typanalyze。

`pg_type.typanalyze` 有效时，通过 `OidFunctionCall1()` 调用该函数。

无效时调用 `std_typanalyze(stats)`。

第六步，检查 typanalyze 的返回结果。

返回 false、没有 `compute_stats`、或 `minrows <= 0` 都表示该列不能继续分析。

源码会释放类型 tuple copy 和 `VacAttrStats`，并返回 NULL。

第七步，采样结束后，`do_analyze_rel()` 调用 `stats->compute_stats()`。

此时传入的是 fetchfunc、实际样本行数和总行数估计。

compute 函数只通过 fetchfunc 读取样本值，不需要知道样本数组的物理存放细节。

第八步，`compute_scalar_stats()` 这样的函数把结果写回 `VacAttrStats`。

它设置 `stats_valid`、`stanullfrac`、`stawidth`、`stadistinct` 和各个 slot。

第九步，`update_attstats()` 把这些结构转换成 `pg_statistic` tuple。

planner 后续读取的是 catalog tuple，不会调用 typanalyze。

typanalyze 只发生在统计刷新阶段。

## 7. 生命周期 / ownership / cleanup

`VacAttrStats` 由 `examine_attribute()` 创建。

它被 `do_analyze_rel()` 的 `vacattrstats[]` 数组持有，直到本次 analyze 命令结束。

`attrtype` 是 syscache copy，需要在跳过列时释放。

正常完成路径中，整个 analyze context 会统一回收这些对象。

`compute_stats` 的中间对象一般活在 `Analyze Column` 子 context。

例如排序数组、duplicate tracking、临时 detoast 值和 compare support 状态。

这些对象每列处理完后 reset。

要写入 `pg_statistic` 的 `stavalues` 和 `stanumbers` 必须切到 `stats->anl_context` 分配。

否则列子 context reset 后，catalog 写入阶段会拿到悬空指针。

类型专用 typanalyze 可以把私有信息放在 `extra_data`。

但它的 lifetime 也只能覆盖本次 analyze。

不要把 `extra_data` 设计成跨 analyze 或跨 backend 的缓存。

ERROR 时，memory context 会释放本地对象。

catalog 修改由事务回滚兜底。

如果 typanalyze 调用外部函数或访问系统 cache，它仍然要遵守常规 syscache release、fmgr 和内存上下文约定。

## 8. 正确性机制层次

typanalyze 的第一层正确性是操作符语义。

MCV 需要 equality operator。

histogram 和 correlation 需要 ordering operator。

如果类型没有这些 operator，默认代码只能降级。

第二层是 collation。

字符串等可排序类型的分布解释必须使用被分析值的 collation。

否则 histogram 边界和选择率函数使用的比较语义不一致。

第三层是 slot 协议。

`stakind`、`staop`、`stacoll`、`stavalues` 和 `stanumbers` 必须共同解释。

读取端不能假设第一个 slot 一定是 MCV，必须按 kind/op 查找。

第四层是宽值处理。

`compute_scalar_stats()` 对过宽 varlena 值有 `WIDTH_THRESHOLD` 策略。

这不是丢失数据的 bug，而是在统计成本和代表性之间选择：极宽值很少成为 MCV，也不适合反复 detoast 比较。

第五层是扩展类型的一致性。

数组、range、tsvector 的 typanalyze 和对应 selectivity 函数必须约定同一套 slot 语义。

只实现写入端或只实现读取端都不能改善 planner 行为。

## 9. 错误路径 / 异常路径 / fallback

类型没有专用 typanalyze 时，`std_typanalyze()` 是默认 fallback。

类型没有排序 operator 时，无法构造 scalar histogram。

类型没有 equality operator 时，无法构造普通 MCV。

样本中没有可排序非 NULL 值时，`compute_scalar_stats()` 只能生成 nullfrac、width 或有限 distinct 结论。

过宽 varlena 值超过阈值时，会被排除在 MCV/histogram 详细比较之外，并在 ndistinct 估算中按 distinct 倾向处理。

自定义 typanalyze 返回 false 时，该列被跳过。

这要求扩展作者明确区分：

```text
无法分析该类型；
只能生成粗统计；
可以生成自定义 slot；
需要更多样本。
```

如果 `pg_stats` 看不到预期 slot，应先核对 `pg_type.typanalyze` 和操作符支持，而不是直接调大 target。

如果 rows 偏差只出现在某个自定义类型谓词上，应同时读 typanalyze 写入端和 selectivity 读取端。

## 10. 成本、资源与跨模块传播

typanalyze 把类型语义传播到三个后续模块。

第一是 catalog。

`update_attstats()` 只负责写入 slot，不理解每种类型的全部语义。

slot 的可解释性来自 typanalyze 和 selectivity 函数的共同约定。

第二是选择率函数。

`selfuncs.c`、`array_selfuncs.c`、`rangetypes_selfuncs.c`、`ts_selfuncs.c` 等会按各自协议读取 slot。

读取端找不到合适 slot 时会走默认估算。

第三是成本模型。

选择率变成 rows，rows 再进入 scan、join、aggregation 和 sort cost。

类型统计本身不直接选择 plan，但它改变 plan 搜索的输入事实。

资源成本主要来自 analyze 阶段。

`std_typanalyze()` 的 scalar path 需要排序样本。

类型专用函数如果解析复杂 datum、展开数组元素或计算 range bounds，也会增加 CPU 和内存成本。

提高 target 会放大这些成本。

本节的诊断顺序是：

```text
先确认类型使用哪个 typanalyze；
再看 typanalyze 写了哪些 slot；
再看选择率函数读取哪些 slot；
最后看 rows 偏差如何传播到 plan。
```

可迁移规律是：

```text
统计不是“列值摘要”这么简单；
它是类型语义、操作符语义和 planner 读取协议共同形成的契约。
```

## 11. 观测与诊断入口

planner 内部很多状态没有直接 SQL 视图，但可以通过计划形态和 catalog 摘要推断。

`pg_type.typanalyze` 能看出某个类型是否走自定义分析路径。

`pg_stats` 中 `most_common_elems`、`elem_count_histogram`、`histogram_bounds` 能区分数组与标量统计。

范围列在 `pg_stats` 中暴露的信息有限，必要时要读 `pg_statistic` slot。

同一列改成不同类型后，`ANALYZE` 产出的 slot kind 可能完全不同。

| 入口 | 能看到什么 | 看不到什么 |
| --- | --- | --- |
| `EXPLAIN` | 估算 rows、cost、Filter 位置和选中 path。 | 未入选的多数候选 path 和中间判断细节。 |
| `EXPLAIN (ANALYZE, BUFFERS)` | 估算与实际的偏差、buffer 访问形态。 | planner 当时的所有统计 slot 读取过程。 |
| `pg_stats` / catalog | 统计摘要、目标、对象定义和部分扩展统计。 | 选择率函数如何组合这些摘要。 |
| gdb 断点 | 当前 `PlannerInfo`、`RelOptInfo`、slot 或依赖结构。 | 没有复现实验时难以判断业务代表性。 |

日常诊断通常先用 SQL 和 EXPLAIN 缩小范围，再用源码断点验证关键状态。

## 12. 课堂实验一：从计划反推状态

第一个实验只改变查询形态或统计状态，目标是让计划差异能回到本节主问题。

分别创建 int、text、int[]、int4range、tsvector 列，执行 analyze 后比较 `pg_stats` 字段。

```text
EXPLAIN (ANALYZE, BUFFERS)
  -> 记录 estimated rows 与 actual rows
  -> 记录 Filter / Index Cond / Join Filter 的位置
  -> 对照 pg_stats 或 catalog 状态
```

创建一个只有等值语义明显、范围语义弱的数据类型，思考默认路径会如何退化。

```text
EXPLAIN (ANALYZE, BUFFERS)
  -> 记录 estimated rows 与 actual rows
  -> 记录 Filter / Index Cond / Join Filter 的位置
  -> 对照 pg_stats 或 catalog 状态
```

实验结果不要求某个固定计划，因为硬件、数据量、GUC 和缓存都会影响 cost；要求是解释计划变化时能回到同一个源码判断。

## 13. 课堂实验二：改变输入验证边界

第二个实验改变输入分布或配置，用来验证 fallback 和成本传播。

提高数组列 statistics target，观察 most_common_elems 数量和元素频率变化。

- 记录变更前的 catalog 摘要。
- 执行一次明确的 `ANALYZE`，避免把统计陈旧误认为源码行为。
- 只改变一个变量，保留其它 GUC 和 SQL 形态。
- 把计划差异写成“输入变化 -> planner 状态变化 -> path 变化”。

对 range 列构造大量 empty range，观察 empty fraction 对范围选择率的影响。

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

一个扩展类型什么时候应该写 typanalyze，而不是只写 selectivity 函数？

回答时要给出一个可复现实验或一个源码入口，避免只给抽象观点。

为什么 slot kind 编号需要社区级约束，不能随意复用？

回答时要给出一个可复现实验或一个源码入口，避免只给抽象观点。

如果类型排序操作符和业务查询使用的操作符不一致，统计还能帮助 planner 吗？

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

断点入口 1：`src/backend/commands/analyze.c`。`examine_attribute()`、`std_typanalyze()`、`compute_scalar_stats()`、`compute_distinct_stats()` 是默认路径。

在这个位置不要一次性展开整份文件。先确认当前调用栈是否服务本节唯一主问题，再决定是否继续向下单步。

如果这里没有被命中，优先检查 SQL 形态、统计是否存在、GUC 是否关闭了相关路径，以及本节机制是否适用于当前 relation 类型。

如果这里被命中但状态为空，通常说明前置 catalog、样本、relids 或 clause 形态没有满足条件。这个结论比“planner 没优化”更有诊断价值。

断点入口 2：`src/include/commands/vacuum.h`。`VacAttrStats` 说明 typanalyze 需要填哪些字段。

在这个位置不要一次性展开整份文件。先确认当前调用栈是否服务本节唯一主问题，再决定是否继续向下单步。

如果这里没有被命中，优先检查 SQL 形态、统计是否存在、GUC 是否关闭了相关路径，以及本节机制是否适用于当前 relation 类型。

如果这里被命中但状态为空，通常说明前置 catalog、样本、relids 或 clause 形态没有满足条件。这个结论比“planner 没优化”更有诊断价值。

断点入口 3：`src/include/catalog/pg_type.h`。`typanalyze` 字段把数据类型和分析函数连接起来。

在这个位置不要一次性展开整份文件。先确认当前调用栈是否服务本节唯一主问题，再决定是否继续向下单步。

如果这里没有被命中，优先检查 SQL 形态、统计是否存在、GUC 是否关闭了相关路径，以及本节机制是否适用于当前 relation 类型。

如果这里被命中但状态为空，通常说明前置 catalog、样本、relids 或 clause 形态没有满足条件。这个结论比“planner 没优化”更有诊断价值。

断点入口 4：`src/include/catalog/pg_proc.dat`。声明 `array_typanalyze`、`range_typanalyze`、`multirange_typanalyze`、`ts_typanalyze`。

在这个位置不要一次性展开整份文件。先确认当前调用栈是否服务本节唯一主问题，再决定是否继续向下单步。

如果这里没有被命中，优先检查 SQL 形态、统计是否存在、GUC 是否关闭了相关路径，以及本节机制是否适用于当前 relation 类型。

如果这里被命中但状态为空，通常说明前置 catalog、样本、relids 或 clause 形态没有满足条件。这个结论比“planner 没优化”更有诊断价值。

断点入口 5：`src/backend/utils/adt/array_typanalyze.c`。数组列统计 most common elements 和 distinct element count。

在这个位置不要一次性展开整份文件。先确认当前调用栈是否服务本节唯一主问题，再决定是否继续向下单步。

如果这里没有被命中，优先检查 SQL 形态、统计是否存在、GUC 是否关闭了相关路径，以及本节机制是否适用于当前 relation 类型。

如果这里被命中但状态为空，通常说明前置 catalog、样本、relids 或 clause 形态没有满足条件。这个结论比“planner 没优化”更有诊断价值。

断点入口 6：`src/backend/utils/adt/rangetypes_typanalyze.c`。范围和 multirange 统计 bounds 与 length 分布。

在这个位置不要一次性展开整份文件。先确认当前调用栈是否服务本节唯一主问题，再决定是否继续向下单步。

如果这里没有被命中，优先检查 SQL 形态、统计是否存在、GUC 是否关闭了相关路径，以及本节机制是否适用于当前 relation 类型。

如果这里被命中但状态为空，通常说明前置 catalog、样本、relids 或 clause 形态没有满足条件。这个结论比“planner 没优化”更有诊断价值。

断点入口 7：`src/backend/tsearch/ts_typanalyze.c`。tsvector 统计 lexeme 分布。

在这个位置不要一次性展开整份文件。先确认当前调用栈是否服务本节唯一主问题，再决定是否继续向下单步。

如果这里没有被命中，优先检查 SQL 形态、统计是否存在、GUC 是否关闭了相关路径，以及本节机制是否适用于当前 relation 类型。

如果这里被命中但状态为空，通常说明前置 catalog、样本、relids 或 clause 形态没有满足条件。这个结论比“planner 没优化”更有诊断价值。

断点入口 8：`src/include/catalog/pg_statistic.h`。统计 kind 编码必须与 typanalyze 和选择率函数共同解释。

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

反向检查 `VacAttrStats.attrtypid`：当前分析值的类型，不一定等同于表列物理类型。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

反向检查 `VacAttrStats.attrcollid`：排序、等值比较和 pattern 估算都可能依赖 collation。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

反向检查 `VacAttrStats.compute_stats`：类型分析函数选择的统计计算入口。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

反向检查 `VacAttrStats.minrows`：该类型认为达到目标精度至少需要多少样本行。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

反向检查 `VacAttrStats.stakind[]`：slot 中保存的统计种类，由类型语义决定。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

反向检查 `VacAttrStats.staop[]`：解释 slot values 时使用的操作符。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

反向检查 `VacAttrStats.stavalues[]`：MCV、histogram、bounds 或 element values 的 Datum 数组。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

反向检查 `VacAttrStats.stanumbers[]`：频率、correlation、empty range fraction 或 element count 摘要。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

## 19. 主流程中的可验证边界

下面把主流程拆成可验证边界。每个边界都应能用一个 SQL 变体、一个 catalog 查询或一个断点确认。

边界 1：`do_analyze_rel()` 为每列调用 `examine_attribute()`。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 2：`examine_attribute()` 读取 `pg_type`，初始化类型、宽度、collation 和 attribute options。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 3：如果 `pg_type.typanalyze` 有效，源码调用该函数；否则进入 `std_typanalyze()`。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 4：`std_typanalyze()` 查找默认 `<` 和 `=` 操作符，决定标量、仅 distinct 或 trivial 统计路径。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 5：数组 typanalyze 先调用标准路径保留普通列统计，再追加 element-level 统计。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 6：范围 typanalyze 保存 bounds histogram 与 length histogram，而不是把 range 当普通 varlena 排序。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 7：tsvector typanalyze 收集 lexeme 频率，供全文检索选择率使用。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 8：自定义类型如果定义 typanalyze，还必须让对应操作符选择率函数知道 slot kind 含义。

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

记录对象：PostgreSQL typanalyze 与类型相关统计。

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

课后复习 PostgreSQL typanalyze 与类型相关统计 时，可以把源码索引压缩成三轮阅读。

第一轮只看入口，确认本节主问题在哪里被提出。

第二轮只看状态，确认字段、catalog tuple 或局部结构在哪里被写入。

第三轮只看消费点，确认 rows、cost、path 或 plan 如何体现这个状态。

- `src/backend/commands/analyze.c`：`examine_attribute()`、`std_typanalyze()`、`compute_scalar_stats()`、`compute_distinct_stats()` 是默认路径。

- `src/include/commands/vacuum.h`：`VacAttrStats` 说明 typanalyze 需要填哪些字段。

- `src/include/catalog/pg_type.h`：`typanalyze` 字段把数据类型和分析函数连接起来。

- `src/include/catalog/pg_proc.dat`：声明 `array_typanalyze`、`range_typanalyze`、`multirange_typanalyze`、`ts_typanalyze`。

- `src/backend/utils/adt/array_typanalyze.c`：数组列统计 most common elements 和 distinct element count。

- `src/backend/utils/adt/rangetypes_typanalyze.c`：范围和 multirange 统计 bounds 与 length 分布。

- `src/backend/tsearch/ts_typanalyze.c`：tsvector 统计 lexeme 分布。

- `src/include/catalog/pg_statistic.h`：统计 kind 编码必须与 typanalyze 和选择率函数共同解释。

复习时不要把这些源码入口平均用力。入口、状态和消费点能串起来，才算读完本节。

如果只能记住一个动作，就是从 `EXPLAIN` 中找第一个异常节点，再回到这里选最小源码入口。

## 25. 本节小结

`typanalyze` 是类型系统插入统计框架的入口。

默认路径覆盖标量和可比较类型，数组、范围、全文等类型用自定义统计表达自己的查询语义。

读统计问题时要同时读类型、slot kind 和选择率函数，单看 `pg_statistic` 行无法解释完整语义。

```text
runtime 现象
  -> planner 状态
  -> 源码判断
  -> rows / cost / plan 表现
  -> 可迁移诊断规律
```

本节结束时，应该能把一个计划现象翻译成源码中的状态判断，而不是只记住某个函数名。

下一节继续沿这个方向推进：用同样方法把相邻主题拆成一个主问题、一条状态链和一个可验证实验。
