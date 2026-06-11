# PostgreSQL ANALYZE 采样与统计目标

## 课程定位

前置知识：熟悉 planner 依赖 `pg_class.reltuples`、`pg_statistic` 和选择率函数估算 rows，知道 `ANALYZE` 会刷新统计信息。

本节唯一主问题：

```text
`ANALYZE` 为什么不全表精确统计，而是用 statistics target 驱动采样规模，并把采样误差交给 planner 的成本模型承受？
```

核心矛盾：planner 需要足够准确的分布信息来选择 plan，但生产库不能为了每次统计刷新都付出全表精确计数、全列排序和全部组合分布的代价。

学完后应能判断统计目标、表大小、列类型、扩展统计对象和继承/分区表如何共同影响采样行数，以及为什么 rows 估错不能简单归咎于 planner 代码。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

上一节解释了 predicate 被放到哪个层级；本节追问这些 predicate 的 rows 判断从哪里来。

05 目录的主线是 planner 如何把 SQL 语义、统计事实和成本模型压缩成可执行计划。本节只处理其中一个局部问题，不把相邻主题合并成百科式说明。

```text
SQL / Query 形态
  -> planner-local 状态
  -> 统计或约束参与判断
  -> Path rows / cost 改变
  -> Plan 节点体现选择结果
```

下一节会把采样结果交给类型相关的 `typanalyze`，看不同数据类型如何解释同一批样本。

因此，本节的阅读边界是明确的：只解释这一个判断如何成立、如何失败、如何被观测，不追求覆盖优化器的所有入口。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
`ANALYZE` 先为每个可分析列计算 `minrows`，取所有列、索引表达式和扩展统计对象需求的最大值作为样本目标，再通过 block sampling 与 reservoir sampling 得到样本行，最后把样本投影成 catalog 统计。
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
| 1 | `src/backend/commands/analyze.c` | `analyze_rel()`、`do_analyze_rel()`、`acquire_sample_rows()`、`update_attstats()` 是主链路。 |
| 2 | `src/include/commands/vacuum.h` | `VacAttrStats`、`AnalyzeAttrFetchFunc`、`AcquireSampleRowsFunc` 定义分析阶段的状态契约。 |
| 3 | `src/include/access/tableam.h` | `table_scan_analyze_next_block()` 和 `table_scan_analyze_next_tuple()` 是 table AM 的采样回调边界。 |
| 4 | `src/backend/statistics/extended_stats.c` | `ComputeExtStatisticsRows()` 让扩展统计对象参与样本行数决策。 |
| 5 | `src/include/commands/progress.h` | `PROGRESS_ANALYZE_*` 描述可观测的 analyze 阶段。 |
| 6 | `src/backend/commands/vacuum.c` | VACUUM/ANALYZE 命令入口和参数传递。 |
| 7 | `src/backend/postmaster/autovacuum.c` | autovacuum 根据修改计数触发自动 analyze。 |

推荐阅读方式是先从入口函数建立时间轴，再读结构体字段，最后读 fallback 和观测点。不要按文件名排序读，也不要在第一个复杂分支里停太久。

```text
入口函数
  -> 本地状态结构
  -> 判断条件
  -> fallback 或保守路径
  -> rows / cost / catalog / plan 的可见结果
```

- 读本节源码时，始终把问题压回这一句：`ANALYZE` 为什么不全表精确统计，而是用 statistics target 驱动采样规模，并把采样误差交给 planner 的成本模型承受？
- 如果一个分支不能改变这个问题的答案，只做定位提示，不展开成旁支。
- 如果一个字段看起来像结论，要继续追问它由哪个阶段写入、被哪个阶段读取。
- 如果 SQL 现象和源码注释看起来冲突，先确认当前路径是否进入 fallback。
- 如果不同版本实现细节变化，优先保留当前源码中能验证的不变量。

## 4. 关键数据结构与状态边界

`ANALYZE` 的核心状态分成两层。

第一层是一次 analyze 命令里的工作状态。

第二层是写入 catalog 后供 planner 长期读取的统计摘要。

这两层的生命周期不同，诊断 rows 估错时必须分开看。

| 状态 | 所在位置 | 语义边界 |
| --- | --- | --- |
| `default_statistics_target` | GUC | 列级目标缺省值，只影响之后执行的 analyze 或规划读取逻辑，不 retroactively 修复旧统计。 |
| `attstattarget` | `pg_attribute` / `VacAttrStats` | 列级目标，负数回退默认值，0 表示该列不重新生成列统计。 |
| `VacAttrStats.minrows` | `src/include/commands/vacuum.h` | typanalyze 提出的最小样本需求，主 analyze 代码取所有需求的最大值。 |
| `targrows` | `do_analyze_rel()` 局部变量 | 本次采样 reservoir 的目标容量，不保证表里一定有这么多活行。 |
| `rows[]` | Analyze memory context | `HeapTuple *` 样本数组，后续列统计、索引表达式统计和扩展统计共享它。 |
| `totalrows` / `totaldeadrows` | acquirefunc 输出 | 从采样扫描估算出的表级活行/死行数量。 |
| `pg_class.reltuples` / `relpages` | catalog | planner 估算 base relation size 的粗粒度输入。 |
| `pg_statistic` | catalog | 列分布的压缩表示，不保存全量数据。 |

`do_analyze_rel()` 先为目标表创建名为 `Analyze` 的 memory context。

这个 context 持有本次 analyze 里较长寿的对象，例如 `VacAttrStats`、样本 tuple 和最终要写入 catalog 的 slot 数组。

每列计算阶段还有一个 `Analyze Column` 子 context。

`compute_stats` 每处理完一列后 reset 子 context，而真正要写入 `VacAttrStats` 的结果必须分配在 `stats->anl_context` 中。

这条 ownership 规则解释了为什么 `VacAttrStats` 结构中有 `anl_context` 字段。

`minrows` 不是最终样本数。

它是各个 typanalyze、索引表达式和扩展统计对象对样本规模提出的下限。

`do_analyze_rel()` 最终取：

```text
100
  与所有普通列 minrows 的最大值
  与所有索引表达式 minrows 的最大值
  与 ComputeExtStatisticsRows() 返回值的最大值
```

`rows[]` 也不是 planner 直接读取的数据。

它只在 analyze 命令内存在。

命令结束后，planner 只能看到 `pg_statistic`、`pg_statistic_ext_data`、`pg_class` 和统计视图中的摘要。

所以提高 statistics target 的效果要经过新的 `ANALYZE` 才能进入 planner。

## 5. 从 SQL 现象进入源码

本节可以用三个外部现象进入源码。

第一类现象是 `pg_stats` 中 slot 变多。

提高某列 statistics target 后重新 `ANALYZE`，常见结果是 MCV 数量、histogram 边界数量或 correlation 可用性变化。

这对应 `compute_scalar_stats()` 中 `num_mcv = stats->attstattarget` 和 `num_bins = stats->attstattarget`。

第二类现象是 `pg_stat_progress_analyze` 阶段变化。

采样时能看到 acquire sample rows 阶段。

构建扩展统计时能看到 ext stats 总数和已完成数量。

这些进度字段不能显示每个 slot 的内部数值，但能判断耗时卡在扫描、列统计还是扩展统计。

第三类现象是 `EXPLAIN` 的首个 rows 偏差节点。

如果 base scan 的 estimated rows 已经明显错，问题通常在 relation size、列统计、predicate 形态或样本代表性。

如果 base scan 准而上层 join 错，问题更可能在相关性、join selectivity 或误差传播。

一个可复现的课堂路径是：

```sql
CREATE TABLE t AS
SELECT g AS id, (g % 10) AS k, repeat('x', 20) AS pad
FROM generate_series(1, 100000) g;

ALTER TABLE t ALTER COLUMN k SET STATISTICS 1;
ANALYZE t;
EXPLAIN SELECT * FROM t WHERE k = 1;

ALTER TABLE t ALTER COLUMN k SET STATISTICS 1000;
ANALYZE t;
EXPLAIN SELECT * FROM t WHERE k = 1;
```

这个实验不证明 target 越大越好。

它只展示 target 如何通过样本规模和 slot 容量改变 planner 可用事实。

如果数据分布在 analyze 后迅速变化，target 再高也只能描述旧样本。

如果谓词使用表达式而没有表达式统计或扩展统计，普通列统计也不会自动变成表达式分布。

## 6. 主流程源码 walkthrough

主流程从 `src/backend/commands/vacuum.c` 调用 `analyze_rel()` 开始。

`analyze_rel()` 打开 relation，选择普通表、分区表、继承树或外部表路径。

普通 heap 表最终走 `do_analyze_rel()`，采样函数通常是 `acquire_sample_rows()`。

外部表可以通过 FDW 提供自己的 acquirefunc。

进入 `do_analyze_rel()` 后，源码先创建 `Analyze` memory context。

随后切换到 relation owner，并进入 `SECURITY_RESTRICTED_OPERATION`。

这个身份切换是因为 analyze 可能执行索引表达式或统计表达式，不能用调用者身份随意运行。

接着源码确定要分析哪些列。

显式列列表只处理指定列。

没有显式列列表时，会遍历所有普通列，并额外检查索引表达式。

`examine_attribute()` 是列级状态创建入口。

它调用 `attribute_is_analyzable()` 检查 dropped 列、系统列和 `attstattarget`。

随后拷贝 `pg_type` 行，设置 `attrtypid`、`attrcollid`、`anl_context` 和 slot 默认类型信息。

然后调用类型自己的 `typanalyze`，没有类型专用函数时使用 `std_typanalyze()`。

`typanalyze` 返回 true 后必须填好 `compute_stats` 和 `minrows`。

如果返回 false、没有 compute 函数或 minrows 非正，该列会被跳过。

列检查完成后，`do_analyze_rel()` 计算 `targrows`。

普通列、索引表达式和扩展统计对象都可能把它抬高。

扩展统计对象通过 `ComputeExtStatisticsRows()` 查看可构建的统计对象，并基于对象 target、属性 target 和默认 target 决定需求。

采样阶段调用 acquirefunc。

heap 的 `acquire_sample_rows()` 先用 `BlockSampler_Init()` 选择要扫描的 block。

然后用 `table_scan_analyze_next_block()` 和 `table_scan_analyze_next_tuple()` 走 table AM 回调。

前 `targrows` 个活 tuple 直接进入 reservoir。

之后使用 Vitter reservoir sampling 计算跳过数量，并随机替换 reservoir 中已有样本。

这保证样本是扫描过的活 tuple 的随机样本，而不是只偏向前几个页面。

采样完成后，源码进入 `PROGRESS_ANALYZE_PHASE_COMPUTE_STATS`。

每列的 `compute_stats` 在同一批样本上计算 nullfrac、width、ndistinct、MCV、histogram 或 correlation。

索引表达式统计通过 `compute_index_stats()` 使用样本 tuple 先求表达式值，再调用对应 compute 函数。

列统计由 `update_attstats()` 写入 `pg_statistic`。

扩展统计由 `BuildRelationExtStatistics()` 写入 `pg_statistic_ext_data`。

最后 `vac_update_relstats()` 更新 `pg_class` 的 pages、tuples、all-visible 等 relation 级信息。

`pgstat_report_analyze()` 更新累计统计，autovacuum 日志也在这个阶段拿到耗时和资源信息。

## 7. 生命周期 / ownership / cleanup

本节最容易混淆的是样本 tuple 和 catalog 统计。

样本 tuple 只活在 analyze 命令内。

`rows[]` 中的 `HeapTuple` 由采样阶段复制出来，后续多个统计计算共享。

命令结束后这些 tuple 随 `Analyze` memory context 一起释放。

列级计算里的中间排序数组、MCV tracking 数组和临时 detoast 数据活在 `Analyze Column` 子 context 中。

每列处理完后 reset 子 context，避免宽列或大 target 让内存随列数线性累积。

要写入 catalog 的数组必须复制到 `stats->anl_context`。

`compute_scalar_stats()` 生成 MCV、histogram 和 correlation slot 时会显式切回 `stats->anl_context`。

`pg_statistic` 和 `pg_class` 的 tuple 更新是 catalog 生命周期。

它们不是 `ANALYZE` 进程私有状态，后续 planner 通过 syscache/relcache 读取。

ERROR 路径下，memory context 负责释放本地分配。

catalog 更新如果在事务中失败，会随事务 abort 回滚。

外部资源方面，relation、index relation 和 catalog relation 由相应 open/close 路径管理，不依赖 `VacAttrStats` 指针本身。

## 8. 正确性机制层次

`ANALYZE` 的统计结果允许近似，但采样过程不能随意偏置。

block sampling 控制页面层面的覆盖。

reservoir sampling 控制 tuple 层面的均匀性。

如果只扫描前 N 个 block，histogram 会反映物理聚集，而不是表总体。

权限层面，analyze 会切换到表 owner 并限制安全上下文。

这保证索引表达式、统计表达式和类型函数按表定义者语义运行，同时限制危险操作。

类型层面，`typanalyze` 必须用正确的类型、collation 和 operator 来解释样本。

如果一个类型没有合适的 `<` 或 `=`，`std_typanalyze()` 会降级到 distinct 或 trivial 统计，而不是伪造 scalar histogram。

catalog 层面，`attstattarget=0` 表示不为该列重新构建统计。

它不是把 nullfrac、ndistinct 或 histogram 写成零。

继承/分区层面，普通统计和 inherited 统计分开写。

planner 在父表继承查询和单表查询中需要不同语义。

这些机制共同表达一个边界：

```text
统计可以近似；
采样和写入必须按明确语义近似。
```

## 9. 错误路径 / 异常路径 / fallback

没有可分析列时，`do_analyze_rel()` 仍然可以更新 relation 级统计。

这能让 planner 至少拥有 pages 和 tuples 输入。

某列 `typanalyze` 返回 false 时，该列不会写入新的列统计。

这不是规划失败，而是后续选择率函数会走默认估算或已有统计。

样本行数少于 `targrows` 时，compute 函数只能基于实际 `numrows` 工作。

小表常见这种情况，不能把它理解为采样失败。

样本为空时，不生成列统计，但 relation 级信息仍可能更新。

外部表的 acquirefunc 可以采用与 heap 不同的采样策略。

读源码时不要把 `acquire_sample_rows()` 的 heap 细节推广到所有 table AM 或 FDW。

扩展统计对象如果缺少本次 analyze 覆盖的列，会在 `ComputeExtStatisticsRows()` 中被忽略，在实际 build 阶段可能给出 warning。

自动 analyze 的日志只展示总耗时和部分资源，不展示每个统计 slot 的细节。

需要解释具体 rows 偏差时，还是要回到 `pg_stats`、`pg_stats_ext` 和源码断点。

## 10. 成本、资源与跨模块传播

提高 statistics target 有两类成本。

第一类发生在 analyze 时。

样本数组更大。

每列排序、比较、MCV 统计和 histogram 生成更贵。

宽 varlena 值还可能触发更多 detoast 或被 `WIDTH_THRESHOLD` 策略跳过。

扩展统计对象会放大这个成本，因为同一批样本还要被用于多列组合、表达式、MCV 或 dependencies。

第二类发生在 planner 和 catalog 层。

更大的 MCV/histogram 数组会让 `pg_statistic` 行更大，必要时进入 TOAST。

选择率函数读取 slot 时需要 detoast 和复制数组。

不过 target 提高的收益也很具体：

```text
更大的 MCV 能覆盖更多热点值；
更细的 histogram 能改善范围谓词估算；
更好的 ndistinct 能改善 group/join/aggregation rows；
更稳定的 correlation 能影响 index scan I/O 成本。
```

诊断时不要先把 `default_statistics_target` 当万能旋钮。

先确认偏差来自哪一层：

```text
统计是否陈旧；
谓词是否能命中列或表达式统计；
数据分布是否被单列摘要表达；
是否需要扩展统计；
analyze 成本是否能接受更高 target。
```

本节的可迁移规律是：

```text
统计目标不是精确度开关，而是采样成本、catalog 体积和 planner 事实密度之间的预算。
```

## 11. 观测与诊断入口

planner 内部很多状态没有直接 SQL 视图，但可以通过计划形态和 catalog 摘要推断。

`pg_stat_progress_analyze` 能看到当前阶段、总对象数和已完成扩展统计对象数。

`pg_stat_all_tables.last_analyze` 和 `last_autoanalyze` 能判断统计是否陈旧。

`pg_stats` 中 MCV 数量和 histogram 边界数量通常随 statistics target 改变。

`EXPLAIN` 的首个 rows 失真节点常常能反推是采样精度、统计目标还是分布变化问题。

| 入口 | 能看到什么 | 看不到什么 |
| --- | --- | --- |
| `EXPLAIN` | 估算 rows、cost、Filter 位置和选中 path。 | 未入选的多数候选 path 和中间判断细节。 |
| `EXPLAIN (ANALYZE, BUFFERS)` | 估算与实际的偏差、buffer 访问形态。 | planner 当时的所有统计 slot 读取过程。 |
| `pg_stats` / catalog | 统计摘要、目标、对象定义和部分扩展统计。 | 选择率函数如何组合这些摘要。 |
| gdb 断点 | 当前 `PlannerInfo`、`RelOptInfo`、slot 或依赖结构。 | 没有复现实验时难以判断业务代表性。 |

日常诊断通常先用 SQL 和 EXPLAIN 缩小范围，再用源码断点验证关键状态。

## 12. 课堂实验一：从计划反推状态

第一个实验只改变查询形态或统计状态，目标是让计划差异能回到本节主问题。

在同一张倾斜表上分别设置 `ALTER TABLE ... ALTER COLUMN ... SET STATISTICS 10` 和 1000，比较 `pg_stats`。

```text
EXPLAIN (ANALYZE, BUFFERS)
  -> 记录 estimated rows 与 actual rows
  -> 记录 Filter / Index Cond / Join Filter 的位置
  -> 对照 pg_stats 或 catalog 状态
```

对大表做批量插入后先不 analyze，观察 `EXPLAIN` rows，再执行 analyze 后对比。

```text
EXPLAIN (ANALYZE, BUFFERS)
  -> 记录 estimated rows 与 actual rows
  -> 记录 Filter / Index Cond / Join Filter 的位置
  -> 对照 pg_stats 或 catalog 状态
```

实验结果不要求某个固定计划，因为硬件、数据量、GUC 和缓存都会影响 cost；要求是解释计划变化时能回到同一个源码判断。

## 13. 课堂实验二：改变输入验证边界

第二个实验改变输入分布或配置，用来验证 fallback 和成本传播。

创建表达式索引并执行 analyze，检查表达式统计是否影响相关谓词估算。

- 记录变更前的 catalog 摘要。
- 执行一次明确的 `ANALYZE`，避免把统计陈旧误认为源码行为。
- 只改变一个变量，保留其它 GUC 和 SQL 形态。
- 把计划差异写成“输入变化 -> planner 状态变化 -> path 变化”。

在分区表父表和单个分区上分别 analyze，比较 inherited stats 与普通 stats 的区别。

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

为什么生产库通常不应该把所有列的 statistics target 一起调高？

回答时要给出一个可复现实验或一个源码入口，避免只给抽象观点。

当 rows 估错来自抽样误差时，增加 target 与重写 SQL 哪个更直接？

回答时要给出一个可复现实验或一个源码入口，避免只给抽象观点。

autovacuum analyze 的触发频率如何影响 planner 对热点表的判断？

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

断点入口 1：`src/backend/commands/analyze.c`。`analyze_rel()`、`do_analyze_rel()`、`acquire_sample_rows()`、`update_attstats()` 是主链路。

在这个位置不要一次性展开整份文件。先确认当前调用栈是否服务本节唯一主问题，再决定是否继续向下单步。

如果这里没有被命中，优先检查 SQL 形态、统计是否存在、GUC 是否关闭了相关路径，以及本节机制是否适用于当前 relation 类型。

如果这里被命中但状态为空，通常说明前置 catalog、样本、relids 或 clause 形态没有满足条件。这个结论比“planner 没优化”更有诊断价值。

断点入口 2：`src/include/commands/vacuum.h`。`VacAttrStats`、`AnalyzeAttrFetchFunc`、`AcquireSampleRowsFunc` 定义分析阶段的状态契约。

在这个位置不要一次性展开整份文件。先确认当前调用栈是否服务本节唯一主问题，再决定是否继续向下单步。

如果这里没有被命中，优先检查 SQL 形态、统计是否存在、GUC 是否关闭了相关路径，以及本节机制是否适用于当前 relation 类型。

如果这里被命中但状态为空，通常说明前置 catalog、样本、relids 或 clause 形态没有满足条件。这个结论比“planner 没优化”更有诊断价值。

断点入口 3：`src/include/access/tableam.h`。`table_scan_analyze_next_block()` 和 `table_scan_analyze_next_tuple()` 是 table AM 的采样回调边界。

在这个位置不要一次性展开整份文件。先确认当前调用栈是否服务本节唯一主问题，再决定是否继续向下单步。

如果这里没有被命中，优先检查 SQL 形态、统计是否存在、GUC 是否关闭了相关路径，以及本节机制是否适用于当前 relation 类型。

如果这里被命中但状态为空，通常说明前置 catalog、样本、relids 或 clause 形态没有满足条件。这个结论比“planner 没优化”更有诊断价值。

断点入口 4：`src/backend/statistics/extended_stats.c`。`ComputeExtStatisticsRows()` 让扩展统计对象参与样本行数决策。

在这个位置不要一次性展开整份文件。先确认当前调用栈是否服务本节唯一主问题，再决定是否继续向下单步。

如果这里没有被命中，优先检查 SQL 形态、统计是否存在、GUC 是否关闭了相关路径，以及本节机制是否适用于当前 relation 类型。

如果这里被命中但状态为空，通常说明前置 catalog、样本、relids 或 clause 形态没有满足条件。这个结论比“planner 没优化”更有诊断价值。

断点入口 5：`src/include/commands/progress.h`。`PROGRESS_ANALYZE_*` 描述可观测的 analyze 阶段。

在这个位置不要一次性展开整份文件。先确认当前调用栈是否服务本节唯一主问题，再决定是否继续向下单步。

如果这里没有被命中，优先检查 SQL 形态、统计是否存在、GUC 是否关闭了相关路径，以及本节机制是否适用于当前 relation 类型。

如果这里被命中但状态为空，通常说明前置 catalog、样本、relids 或 clause 形态没有满足条件。这个结论比“planner 没优化”更有诊断价值。

断点入口 6：`src/backend/commands/vacuum.c`。VACUUM/ANALYZE 命令入口和参数传递。

在这个位置不要一次性展开整份文件。先确认当前调用栈是否服务本节唯一主问题，再决定是否继续向下单步。

如果这里没有被命中，优先检查 SQL 形态、统计是否存在、GUC 是否关闭了相关路径，以及本节机制是否适用于当前 relation 类型。

如果这里被命中但状态为空，通常说明前置 catalog、样本、relids 或 clause 形态没有满足条件。这个结论比“planner 没优化”更有诊断价值。

断点入口 7：`src/backend/postmaster/autovacuum.c`。autovacuum 根据修改计数触发自动 analyze。

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

反向检查 `default_statistics_target`：没有列级目标时使用的默认统计目标。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

反向检查 `attstattarget`：列级统计目标，负数表示回退默认值，零表示不构建该列统计。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

反向检查 `VacAttrStats.minrows`：类型分析函数要求的最小样本行数。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

反向检查 `targrows`：本次 analyze 真实希望采集的样本行上限。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

反向检查 `rows[]`：保存在 analyze memory context 中的样本 tuple 数组。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

反向检查 `totalrows` / `totaldeadrows`：采样过程中估计出的总活行与死行数量。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

反向检查 `pg_class.reltuples` / `relpages`：planner 计算 base relation size 的粗粒度输入。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

反向检查 `pg_statistic` rows：列分布被压缩后的 catalog 表示。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

## 19. 主流程中的可验证边界

下面把主流程拆成可验证边界。每个边界都应能用一个 SQL 变体、一个 catalog 查询或一个断点确认。

边界 1：`analyze_rel()` 打开目标 relation，选择普通表、分区表、继承树或外部表的 analyze 路径。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 2：`do_analyze_rel()` 建立 `Analyze` memory context，切换到表 owner，并收集可分析列。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 3：`examine_attribute()` 调用类型相关 `typanalyze`，每列给出 `compute_stats` 与 `minrows`。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 4：索引表达式在没有显式列列表时也会创建 `VacAttrStats`，因为 planner 可能需要表达式分布。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 5：`ComputeExtStatisticsRows()` 查看扩展统计对象，可能把样本需求进一步抬高。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 6：`acquire_sample_rows()` 使用 block sampler 选择页面，再用 reservoir sampling 维护样本行。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 7：每列的 `compute_stats` 在样本上计算 nullfrac、width、ndistinct、MCV、histogram 或 correlation。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 8：`update_attstats()` 写入 `pg_statistic`，`BuildRelationExtStatistics()` 写入扩展统计数据。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 9：`vac_update_relstats()` 更新 `pg_class`，`pgstat_report_analyze()` 更新累计统计。

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

记录对象：PostgreSQL ANALYZE 采样与统计目标。

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

课后复习 PostgreSQL ANALYZE 采样与统计目标 时，可以把源码索引压缩成三轮阅读。

第一轮只看入口，确认本节主问题在哪里被提出。

第二轮只看状态，确认字段、catalog tuple 或局部结构在哪里被写入。

第三轮只看消费点，确认 rows、cost、path 或 plan 如何体现这个状态。

- `src/backend/commands/analyze.c`：`analyze_rel()`、`do_analyze_rel()`、`acquire_sample_rows()`、`update_attstats()` 是主链路。

- `src/include/commands/vacuum.h`：`VacAttrStats`、`AnalyzeAttrFetchFunc`、`AcquireSampleRowsFunc` 定义分析阶段的状态契约。

- `src/include/access/tableam.h`：`table_scan_analyze_next_block()` 和 `table_scan_analyze_next_tuple()` 是 table AM 的采样回调边界。

- `src/backend/statistics/extended_stats.c`：`ComputeExtStatisticsRows()` 让扩展统计对象参与样本行数决策。

- `src/include/commands/progress.h`：`PROGRESS_ANALYZE_*` 描述可观测的 analyze 阶段。

- `src/backend/commands/vacuum.c`：VACUUM/ANALYZE 命令入口和参数传递。

- `src/backend/postmaster/autovacuum.c`：autovacuum 根据修改计数触发自动 analyze。

复习时不要把这些源码入口平均用力。入口、状态和消费点能串起来，才算读完本节。

如果只能记住一个动作，就是从 `EXPLAIN` 中找第一个异常节点，再回到这里选最小源码入口。

## 25. 本节小结

`ANALYZE` 的目标不是获得精确事实，而是用可接受成本获得 planner 足够可用的分布摘要。

statistics target 先影响样本行数，再影响 MCV、histogram、correlation 和扩展统计精度。

诊断 rows 估错时，要把采样规模、统计新鲜度、列类型和 selectivity 模型分开看。

```text
runtime 现象
  -> planner 状态
  -> 源码判断
  -> rows / cost / plan 表现
  -> 可迁移诊断规律
```

本节结束时，应该能把一个计划现象翻译成源码中的状态判断，而不是只记住某个函数名。

下一节继续沿这个方向推进：用同样方法把相邻主题拆成一个主问题、一条状态链和一个可验证实验。
