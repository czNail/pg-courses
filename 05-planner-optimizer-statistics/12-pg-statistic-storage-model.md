# PostgreSQL pg_statistic 存储模型

## 课程定位

前置知识：熟悉系统 catalog、syscache、Datum array、TOAST，以及 planner 通过 catalog cache 读取统计信息。

本节唯一主问题：

```text
`pg_statistic` 为什么不用一张张专用统计表，而是用 kind/operator/collation/numbers/values 的五个 slot 保存多种列分布？
```

核心矛盾：planner 需要统一、可缓存、可扩展的统计读取接口；但不同类型和不同选择率函数需要保存的统计形态差异很大。

学完后应能判断一个 `pg_statistic` slot 是否能被某个选择率函数使用，为什么不能按 slot 序号解释含义，以及为什么 `pg_stats` 只是安全视图不是完整内部模型。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

上一节说明 typanalyze 如何决定统计语义；本节看这些语义如何被压缩进一个通用 catalog。

05 目录的主线是 planner 如何把 SQL 语义、统计事实和成本模型压缩成可执行计划。本节只处理其中一个局部问题，不把相邻主题合并成百科式说明。

```text
SQL / Query 形态
  -> planner-local 状态
  -> 统计或约束参与判断
  -> Path rows / cost 改变
  -> Plan 节点体现选择结果
```

下一节会使用这个存储模型，专门分析 MCV、histogram、nullfrac 和 ndistinct 如何参与选择率估算。

因此，本节的阅读边界是明确的：只解释这一个判断如何成立、如何失败、如何被观测，不追求覆盖优化器的所有入口。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
`ANALYZE` 把每列统计写成一行 `pg_statistic`，固定字段保存 nullfrac、width、distinct，最多五个 slot 用 `stakind` 标识统计种类，再由 `staop`、`stacoll`、`stanumbers` 和 `stavalues` 补齐解释上下文。
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
| 1 | `src/include/catalog/pg_statistic.h` | `FormData_pg_statistic` 和 `STATISTIC_KIND_*` 定义内部存储模型。 |
| 2 | `src/backend/commands/analyze.c` | `update_attstats()` 把 `VacAttrStats` 序列化成 catalog tuple。 |
| 3 | `src/backend/utils/cache/lsyscache.c` | `get_attstatsslot()` 和 `free_attstatsslot()` 负责读取和释放 slot 内容。 |
| 4 | `src/backend/utils/adt/selfuncs.c` | `examine_variable()` 和选择率函数消费 `pg_statistic`。 |
| 5 | `src/backend/catalog/system_views.sql` | `pg_stats` 和 `pg_stats_ext` 展示用户可见的统计摘要。 |
| 6 | `src/include/utils/selfuncs.h` | `VariableStatData` 与 `AttStatsSlot` 描述 planner 读取后的本地状态。 |
| 7 | `src/include/catalog/pg_statistic_ext_data.h` | 表达式统计会复用 `pg_statistic` 复合类型。 |

推荐阅读方式是先从入口函数建立时间轴，再读结构体字段，最后读 fallback 和观测点。不要按文件名排序读，也不要在第一个复杂分支里停太久。

```text
入口函数
  -> 本地状态结构
  -> 判断条件
  -> fallback 或保守路径
  -> rows / cost / catalog / plan 的可见结果
```

- 读本节源码时，始终把问题压回这一句：`pg_statistic` 为什么不用一张张专用统计表，而是用 kind/operator/collation/numbers/values 的五个 slot 保存多种列分布？
- 如果一个分支不能改变这个问题的答案，只做定位提示，不展开成旁支。
- 如果一个字段看起来像结论，要继续追问它由哪个阶段写入、被哪个阶段读取。
- 如果 SQL 现象和源码注释看起来冲突，先确认当前路径是否进入 fallback。
- 如果不同版本实现细节变化，优先保留当前源码中能验证的不变量。

## 4. 关键数据结构与状态边界

`pg_statistic` 是 planner 读取列统计的底层 catalog。

它不是 `pg_stats` 视图的展开形式，也不是每列固定写满所有字段的表。

它的核心设计是：

```text
一行描述 relation 的一个统计目标
  -> 用固定的基础字段表达 nullfrac / width / ndistinct
  -> 用最多 5 个可变 slot 表达类型相关统计
  -> 读取端按 stakind + staop + stacoll 查找需要的 slot
```

| 状态 | 语义边界 |
| --- | --- |
| `starelid` / `staattnum` / `stainherit` | 唯一定位一条统计记录；继承统计和普通统计分开。 |
| `stanullfrac` | NULL 比例，选择率函数会先处理 NULL 分量。 |
| `stawidth` | 平均宽度，影响 width、tuple cost、hash/sort memory 估算。 |
| `stadistinct` | distinct 摘要；正数表示估计 distinct 个数，负数表示按 row count 缩放。 |
| `stakindN` | slot 类型，不保证某个 kind 固定出现在某个 N。 |
| `staopN` | 解释该 slot 所需 operator，例如 MCV 的 equality 或 histogram 的 ordering。 |
| `stacollN` | slot 使用的 collation。 |
| `stanumbersN` | float4 数组，常用于 MCV frequency 或 correlation 系数。 |
| `stavaluesN` | anyarray，元素类型可能由 typanalyze 改写。 |

`STATISTIC_NUM_SLOTS` 当前是 5。

这个限制让 catalog tuple 结构稳定，同时给 typanalyze 留出有限扩展空间。

slot 的含义由 `src/include/catalog/pg_statistic.h` 记录。

核心 kind 包括：

```text
STATISTIC_KIND_MCV = 1
STATISTIC_KIND_HISTOGRAM = 2
STATISTIC_KIND_CORRELATION = 3
```

但读取端不能假设 slot 顺序。

`get_attstatsslot()` 会遍历所有 `stakind`，寻找匹配的 kind 和可选 operator。

找到后，它会 detoast 并复制 `stavalues` / `stanumbers`，把结果放进 `AttStatsSlot`。

调用者必须用 `free_attstatsslot()` 释放这些拷贝。

这条 release 规则是本节最重要的 ownership 边界。

`pg_stats` 只是面向用户的视图。

它把 `pg_statistic` 中的内部数组转换成更容易读的列。

内核选择率代码读的是 `pg_statistic` syscache tuple 和 `AttStatsSlot`，不是从 `pg_stats` 视图解析文本。

## 5. 从 SQL 现象进入源码

本节的外部入口是比较 `pg_stats` 与 `pg_statistic`。

`pg_stats` 可以快速回答：

```text
这列有没有 MCV；
histogram bounds 是否存在；
n_distinct 是正数还是负数；
correlation 是否可用；
统计是否继承场景下可见。
```

但当你需要解释读取端行为时，要回到 `pg_statistic`。

例如 MCV slot 是否能被某个 equality 选择率函数使用，不只看 `most_common_vals` 非空。

还要看 slot 的 equality operator 是否匹配当前 operator family 或选择率函数要求。

一个观察流程是：

```sql
SELECT attname, null_frac, avg_width, n_distinct,
       most_common_vals, most_common_freqs,
       histogram_bounds, correlation
FROM pg_stats
WHERE tablename = 't';
```

然后进入源码：

```text
selfuncs.c: examine_variable()
  -> 找到 VariableStatData.statsTuple
lsyscache.c: get_attstatsslot()
  -> 按 kind/op 取 slot
selfuncs.c: free_attstatsslot()
  -> 释放 slot 拷贝
```

如果 `pg_stats` 显示 histogram 存在，但某个谓词估算没有明显变化，可能是读取函数没有使用该 kind。

如果 `n_distinct` 是负数，它不是负的 distinct 个数。

它表示按 relation rows 缩放的比例，例如 `-1` 常用于接近 unique 的非 NULL 值。

如果某列统计刚更新但计划仍使用旧估算，先确认当前 session 是否持有旧 plan、是否使用 prepared statement、以及 relcache/syscache invalidation 是否已经触发 replanning。

## 6. 主流程源码 walkthrough

写入主链路在 `src/backend/commands/analyze.c`。

`compute_stats` 填充 `VacAttrStats`。

`update_attstats()` 把 `VacAttrStats` 转成 `pg_statistic` tuple。

它会删除或替换同一 `(starelid, staattnum, stainherit)` 的旧统计。

写入基础字段时，`stanullfrac`、`stawidth` 和 `stadistinct` 直接来自 `VacAttrStats`。

写入 slot 时，源码按 `stakind[i]` 判断是否有效。

`stavalues[i]` 会被构造成 array datum。

`stanumbers[i]` 会被构造成 float4 array。

slot 元素类型、长度、byval 和 align 来自 `statypid/statyplen/statypbyval/statypalign`。

这就是自定义 typanalyze 能保存非原列元素类型的入口。

读取主链路通常在 `src/backend/utils/adt/selfuncs.c`。

`examine_variable()` 尝试把表达式匹配到 base column、expression index 或统计对象。

成功时会在 `VariableStatData.statsTuple` 中保存统计 tuple，并设置释放函数。

具体选择率函数再调用 `get_attstatsslot()`。

`get_attstatsslot()` 位于 `src/backend/utils/cache/lsyscache.c`。

它先按 kind 和 operator 找 slot。

如果需要 values，它复制并 detoast array，再 `deconstruct_array()` 得到 Datum 列表。

如果需要 numbers，它复制 float4 array，并让 `sslot->numbers` 指向 array 内部数据。

读取端用完后必须调用 `free_attstatsslot()`。

最后，`ReleaseVariableStats()` 释放 `VariableStatData` 中的 stats tuple。

这条链路说明 `pg_statistic` 的 runtime 读取有两个 cleanup 点：

```text
统计 tuple 的 release；
slot detoast/copy 后的 free_attstatsslot。
```

## 7. 生命周期 / ownership / cleanup

`pg_statistic` tuple 的 owner 是 system catalog。

它由 `ANALYZE` 在当前事务中写入或替换。

事务提交后，其他 backend 才能通过 catalog snapshot 看到新统计。

planner 读取时拿到的是 syscache tuple。

这类 tuple 不能跨越释放边界长期保存。

`VariableStatData` 记录释放方式，调用者需要在选择率函数结束前释放。

`AttStatsSlot` 中的 arrays 是 `get_attstatsslot()` 为调用者复制出来的。

它们不属于 syscache tuple 本体。

`free_attstatsslot()` 需要释放：

```text
values[] 指针数组；
values_arr detoast/copy 后的 array；
numbers_arr detoast/copy 后的 array。
```

`numbers` 指向 `numbers_arr` 内部，不能单独 pfree。

如果把 `AttStatsSlot.values` 指针保存到更长生命周期结构里，会在 free 后变成悬空引用。

如果忘记 free，则选择率函数的内存会在当前 planner context 中堆积。

ERROR 路径下，memory context 会兜底释放 palloc 内存，但正常路径仍应遵守 release 约定，避免长规划过程中的峰值放大。

catalog 统计失效来自新的 `ANALYZE`、DDL、relation rewrite 或 catalog invalidation。

一次规划过程中读取到的 stats tuple 是该规划 snapshot 下的事实，不会在中途自动换成更新后的统计。

## 8. 正确性机制层次

第一层是 slot 查找规则。

读取端必须按 kind 和 operator 查找，而不是按 slot 序号。

这允许 typanalyze 改变 slot 排布，也允许某些 slot 缺失。

第二层是 operator 和 collation 一致性。

MCV 的 equality、histogram 的 ordering 和 correlation 的 ordering 都记录在 `staop` / `stacoll` 中。

如果选择率函数使用的 operator 语义不匹配，就不应强行消费该 slot。

第三层是 NULL 分量。

`stanullfrac` 和 MCV/histogram 的非 NULL 数据要分开解释。

histogram 通常描述去掉 NULL、并且去掉 MCV 后的剩余分布。

第四层是 `stadistinct` 编码。

正数和负数代表不同语义。

负数的价值在于让 unique-like 列随 `reltuples` 缩放，而不是把 analyze 当天的绝对 distinct 个数永久写死。

第五层是权限。

`examine_variable()` 会记录 `acl_ok`，选择率函数通过 `statistic_proc_security_check()` 避免用户借不可读列统计和非 leakproof 函数获得信息。

这说明统计读取也有安全边界，不只是性能辅助数据。

## 9. 错误路径 / 异常路径 / fallback

找不到 `pg_statistic` tuple 时，选择率函数走默认估算。

找不到匹配 kind/op 的 slot 时，也走该谓词类型的 fallback。

slot 存在但 array 格式不符合预期，`get_attstatsslot()` 会 ERROR，因为 catalog 数据已经不满足内部协议。

`pg_stats` 显示 NULL 不代表 catalog 不存在。

可能只是该列没有生成该种 slot，或者该类型的 typanalyze 不使用通用 slot。

`attstattarget=0` 时，`ANALYZE` 不重建该列统计。

旧统计可能仍在 catalog 中，诊断时要结合最后 analyze 时间和列设置。

对于继承统计，`stainherit` 不匹配时，planner 不能混用父表汇总和单表统计。

如果 prepared statement 复用旧 plan，更新 `pg_statistic` 后也未必立刻看到计划变化。

这时问题在 plan cache，而不是 catalog 写入失败。

## 10. 成本、资源与跨模块传播

`pg_statistic` 的设计把统计成本分散到两个阶段。

ANALYZE 阶段支付采样、排序、slot 构造和 catalog 写入成本。

planner 阶段支付 syscache lookup、detoast、array copy 和选择率计算成本。

slot 数固定为 5，限制了 catalog 行宽和读取复杂度。

但每个 slot 内数组大小仍受 statistics target 影响。

target 过大时，MCV/histogram 数组更大，catalog 可能 TOAST，planner 读取也更重。

这些成本换来的收益是 rows 输入更稳定。

`selfuncs.c` 用它估算 restriction selectivity。

join selectivity、group distinct、index correlation 和 width 估算也会间接受益。

诊断时可以按这个顺序：

```text
先看 pg_stats 是否有目标 slot；
再看 pg_statistic 中 kind/op/collation 是否匹配；
再看选择率函数是否读取该 slot；
再看 ReleaseVariableStats 和 free_attstatsslot 是否在正常路径执行。
```

本节的可迁移规律是：

```text
catalog 里的 raw field 不是统计语义；
slot kind、operator、collation、生命周期和读取函数合起来才是语义。
```

## 11. 观测与诊断入口

planner 内部很多状态没有直接 SQL 视图，但可以通过计划形态和 catalog 摘要推断。

`pg_stats` 能快速确认 null_frac、n_distinct、most_common_vals、histogram_bounds。

需要确认 slot kind/op/collation 时，应查询 `pg_statistic` 或在 gdb 中看 `Form_pg_statistic`。

`EXPLAIN` 不显示统计 slot，但 rows 估算变化能间接反映 planner 是否读到了统计。

同一列在 inherited 与 non-inherited 场景下可能有两行统计。

| 入口 | 能看到什么 | 看不到什么 |
| --- | --- | --- |
| `EXPLAIN` | 估算 rows、cost、Filter 位置和选中 path。 | 未入选的多数候选 path 和中间判断细节。 |
| `EXPLAIN (ANALYZE, BUFFERS)` | 估算与实际的偏差、buffer 访问形态。 | planner 当时的所有统计 slot 读取过程。 |
| `pg_stats` / catalog | 统计摘要、目标、对象定义和部分扩展统计。 | 选择率函数如何组合这些摘要。 |
| gdb 断点 | 当前 `PlannerInfo`、`RelOptInfo`、slot 或依赖结构。 | 没有复现实验时难以判断业务代表性。 |

日常诊断通常先用 SQL 和 EXPLAIN 缩小范围，再用源码断点验证关键状态。

## 12. 课堂实验一：从计划反推状态

第一个实验只改变查询形态或统计状态，目标是让计划差异能回到本节主问题。

创建含 NULL、热点值和长尾值的表，analyze 后对照 `pg_stats` 与 `pg_statistic`。

```text
EXPLAIN (ANALYZE, BUFFERS)
  -> 记录 estimated rows 与 actual rows
  -> 记录 Filter / Index Cond / Join Filter 的位置
  -> 对照 pg_stats 或 catalog 状态
```

改变列级 statistics target，观察 slot 中数组长度是否变化。

```text
EXPLAIN (ANALYZE, BUFFERS)
  -> 记录 estimated rows 与 actual rows
  -> 记录 Filter / Index Cond / Join Filter 的位置
  -> 对照 pg_stats 或 catalog 状态
```

实验结果不要求某个固定计划，因为硬件、数据量、GUC 和缓存都会影响 cost；要求是解释计划变化时能回到同一个源码判断。

## 13. 课堂实验二：改变输入验证边界

第二个实验改变输入分布或配置，用来验证 fallback 和成本传播。

查询 `pg_statistic` 的 stakind 字段，验证 MCV 不一定总在同一个 slot 序号。

- 记录变更前的 catalog 摘要。
- 执行一次明确的 `ANALYZE`，避免把统计陈旧误认为源码行为。
- 只改变一个变量，保留其它 GUC 和 SQL 形态。
- 把计划差异写成“输入变化 -> planner 状态变化 -> path 变化”。

删除统计或设置 target 为 0，观察选择率函数如何退回默认估算。

- 记录变更前的 catalog 摘要。
- 执行一次明确的 `ANALYZE`，避免把统计陈旧误认为源码行为。
- 只改变一个变量，保留其它 GUC 和 SQL 形态。
- 把计划差异写成“输入变化 -> planner 状态变化 -> path 变化”。

如果实验没有出现预期差异，先检查数据规模是否太小、统计是否刷新、GUC 是否强行禁用了候选路径。

## 14. 源码阅读练习

源码练习以断点和变量观察为主，不要求修改代码。

- 在 `src/include/catalog/pg_statistic.h` 的入口函数附近设置断点，确认本节主路径是否进入。
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

为什么 `pg_statistic` 不把 MCV、histogram、correlation 拆成多张规范化表？

回答时要给出一个可复现实验或一个源码入口，避免只给抽象观点。

如果自定义类型使用私有 kind，选择率函数如何避免误读别人的 slot？

回答时要给出一个可复现实验或一个源码入口，避免只给抽象观点。

`pg_stats` 暴露的信息为什么足够日常诊断，但不足以解释所有 planner 行为？

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

断点入口 1：`src/include/catalog/pg_statistic.h`。`FormData_pg_statistic` 和 `STATISTIC_KIND_*` 定义内部存储模型。

在这个位置不要一次性展开整份文件。先确认当前调用栈是否服务本节唯一主问题，再决定是否继续向下单步。

如果这里没有被命中，优先检查 SQL 形态、统计是否存在、GUC 是否关闭了相关路径，以及本节机制是否适用于当前 relation 类型。

如果这里被命中但状态为空，通常说明前置 catalog、样本、relids 或 clause 形态没有满足条件。这个结论比“planner 没优化”更有诊断价值。

断点入口 2：`src/backend/commands/analyze.c`。`update_attstats()` 把 `VacAttrStats` 序列化成 catalog tuple。

在这个位置不要一次性展开整份文件。先确认当前调用栈是否服务本节唯一主问题，再决定是否继续向下单步。

如果这里没有被命中，优先检查 SQL 形态、统计是否存在、GUC 是否关闭了相关路径，以及本节机制是否适用于当前 relation 类型。

如果这里被命中但状态为空，通常说明前置 catalog、样本、relids 或 clause 形态没有满足条件。这个结论比“planner 没优化”更有诊断价值。

断点入口 3：`src/backend/utils/cache/lsyscache.c`。`get_attstatsslot()` 和 `free_attstatsslot()` 负责读取和释放 slot 内容。

在这个位置不要一次性展开整份文件。先确认当前调用栈是否服务本节唯一主问题，再决定是否继续向下单步。

如果这里没有被命中，优先检查 SQL 形态、统计是否存在、GUC 是否关闭了相关路径，以及本节机制是否适用于当前 relation 类型。

如果这里被命中但状态为空，通常说明前置 catalog、样本、relids 或 clause 形态没有满足条件。这个结论比“planner 没优化”更有诊断价值。

断点入口 4：`src/backend/utils/adt/selfuncs.c`。`examine_variable()` 和选择率函数消费 `pg_statistic`。

在这个位置不要一次性展开整份文件。先确认当前调用栈是否服务本节唯一主问题，再决定是否继续向下单步。

如果这里没有被命中，优先检查 SQL 形态、统计是否存在、GUC 是否关闭了相关路径，以及本节机制是否适用于当前 relation 类型。

如果这里被命中但状态为空，通常说明前置 catalog、样本、relids 或 clause 形态没有满足条件。这个结论比“planner 没优化”更有诊断价值。

断点入口 5：`src/backend/catalog/system_views.sql`。`pg_stats` 和 `pg_stats_ext` 展示用户可见的统计摘要。

在这个位置不要一次性展开整份文件。先确认当前调用栈是否服务本节唯一主问题，再决定是否继续向下单步。

如果这里没有被命中，优先检查 SQL 形态、统计是否存在、GUC 是否关闭了相关路径，以及本节机制是否适用于当前 relation 类型。

如果这里被命中但状态为空，通常说明前置 catalog、样本、relids 或 clause 形态没有满足条件。这个结论比“planner 没优化”更有诊断价值。

断点入口 6：`src/include/utils/selfuncs.h`。`VariableStatData` 与 `AttStatsSlot` 描述 planner 读取后的本地状态。

在这个位置不要一次性展开整份文件。先确认当前调用栈是否服务本节唯一主问题，再决定是否继续向下单步。

如果这里没有被命中，优先检查 SQL 形态、统计是否存在、GUC 是否关闭了相关路径，以及本节机制是否适用于当前 relation 类型。

如果这里被命中但状态为空，通常说明前置 catalog、样本、relids 或 clause 形态没有满足条件。这个结论比“planner 没优化”更有诊断价值。

断点入口 7：`src/include/catalog/pg_statistic_ext_data.h`。表达式统计会复用 `pg_statistic` 复合类型。

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

反向检查 `starelid` / `staattnum` / `stainherit`：一行统计归属哪张 relation、哪个属性、是否继承统计。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

反向检查 `stanullfrac`：空值比例，直接参与 `IS NULL` 与普通选择率补偿。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

反向检查 `stawidth`：非空值平均宽度，影响 tuple width、hash table 和 sort 估算。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

反向检查 `stadistinct`：正数表示固定 distinct 数，负数表示随 rows 缩放的比例。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

反向检查 `stakindN`：slot 类型，不保证固定出现在某个 N。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

反向检查 `staopN`：解释 values 的操作符，例如等值或排序操作符。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

反向检查 `stacollN`：文本等可排序类型的 collation 语义。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

反向检查 `stanumbersN` / `stavaluesN`：频率、correlation、histogram bounds 或类型自定义数组。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

## 19. 主流程中的可验证边界

下面把主流程拆成可验证边界。每个边界都应能用一个 SQL 变体、一个 catalog 查询或一个断点确认。

边界 1：`compute_stats` 在 `VacAttrStats` 中填入基础字段和若干 slot。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 2：`update_attstats()` 删除或替换目标列旧统计，构造新的 `pg_statistic` tuple。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 3：slot 数组被展开成 `stakind1..5`、`staop1..5`、`stacoll1..5`、`stanumbers1..5`、`stavalues1..5`。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 4：planner 通过 syscache 读取 `(starelid, staattnum, stainherit)` 对应 tuple。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 5：`examine_variable()` 把 catalog tuple 包装到 `VariableStatData`，并记录是否来自表列、表达式或 stats hook。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 6：`get_attstatsslot()` 按 kind 和 op 查找 slot，复制 detoast 后的数组到调用者内存。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 7：选择率函数读取完成后调用 `free_attstatsslot()`，避免把 detoasted array 泄漏到长期上下文。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 8：`pg_stats` 视图只投影一部分核心 kind，内部和扩展 kind 仍要按 catalog 解释。

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

记录对象：PostgreSQL pg_statistic 存储模型。

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

课后复习 PostgreSQL pg_statistic 存储模型 时，可以把源码索引压缩成三轮阅读。

第一轮只看入口，确认本节主问题在哪里被提出。

第二轮只看状态，确认字段、catalog tuple 或局部结构在哪里被写入。

第三轮只看消费点，确认 rows、cost、path 或 plan 如何体现这个状态。

- `src/include/catalog/pg_statistic.h`：`FormData_pg_statistic` 和 `STATISTIC_KIND_*` 定义内部存储模型。

- `src/backend/commands/analyze.c`：`update_attstats()` 把 `VacAttrStats` 序列化成 catalog tuple。

- `src/backend/utils/cache/lsyscache.c`：`get_attstatsslot()` 和 `free_attstatsslot()` 负责读取和释放 slot 内容。

- `src/backend/utils/adt/selfuncs.c`：`examine_variable()` 和选择率函数消费 `pg_statistic`。

- `src/backend/catalog/system_views.sql`：`pg_stats` 和 `pg_stats_ext` 展示用户可见的统计摘要。

- `src/include/utils/selfuncs.h`：`VariableStatData` 与 `AttStatsSlot` 描述 planner 读取后的本地状态。

- `src/include/catalog/pg_statistic_ext_data.h`：表达式统计会复用 `pg_statistic` 复合类型。

复习时不要把这些源码入口平均用力。入口、状态和消费点能串起来，才算读完本节。

如果只能记住一个动作，就是从 `EXPLAIN` 中找第一个异常节点，再回到这里选最小源码入口。

## 25. 本节小结

`pg_statistic` 是一个通用分布摘要容器，不是固定列含义清单。

slot 的真实语义来自 kind、operator、collation、numbers、values 和调用者选择率函数的组合。

调试统计问题时，要从 SQL 现象进入 slot 查找，再回到对应 selectivity 函数。

```text
runtime 现象
  -> planner 状态
  -> 源码判断
  -> rows / cost / plan 表现
  -> 可迁移诊断规律
```

本节结束时，应该能把一个计划现象翻译成源码中的状态判断，而不是只记住某个函数名。

下一节继续沿这个方向推进：用同样方法把相邻主题拆成一个主问题、一条状态链和一个可验证实验。
