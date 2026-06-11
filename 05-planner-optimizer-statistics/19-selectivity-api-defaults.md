# PostgreSQL Selectivity API 与默认估算边界

## 课程定位

前置知识：已经理解 `RestrictInfo` 如何包装 clause，也知道 `ANALYZE` 会把单列统计写入 `pg_statistic`。
还应知道 planner 会把选择率乘到 relation 行数上，再把行数送入 scan path、join path 和 upper path 的成本估算。
本节唯一主问题是：为什么 PostgreSQL 在统计缺失、operator 缺少专用估算函数、表达式无法识别时仍然必须返回一个 selectivity。
本节围绕的核心矛盾是：路径搜索不能因为估算信息缺失而停止，但默认值又不能被误读成真实数据分布。
学完后应能独立判断：一个 `EXPLAIN` rows 偏差是来自默认 selectivity、operator catalog、统计 tuple、权限检查、cache 边界还是后续 clause 组合。
本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

前几节已经说明 `pg_statistic` 存储 MCV、histogram、null fraction、ndistinct 和 correlation。
那些内容回答的是有统计时 planner 可以看到什么。
本节反过来回答统计不完整时 planner 如何继续工作。
PostgreSQL 的 planner 是搜索器，不是证明器。
它必须在有限时间里给每个候选 path 一个可比较的 cost。
如果某个 predicate 没有估算函数，planner 不能停在原地等待更多事实。
如果某个表达式无法安全拆成 variable 与常量，planner 也不能放弃生成计划。
如果某个列没有 `pg_statistic` tuple，planner 仍然要给 seq scan、index scan 和 join search 一个输入行数。
这就是 selectivity API 的位置：它把局部条件压缩成概率，并且保证失败时也有退路。
本节只讲 API 契约和默认值边界。
下一节才把 `eqsel()`、`scalarineqsel()` 和 `patternsel_common()` 的具体统计消费规则展开。
再下一节会把多个 clause 的选择率如何组合接到 `clauselist_selectivity_ext()`。
因此本节不要急着讨论 `country = city` 的相关性。
也不要把所有 rows 偏差都归因到扩展统计缺失。
先把单个 estimator 如何被找出、如何退化、如何释放状态、如何返回概率讲清楚。
这条边界建立后，后续才能区分单点估算错误和组合估算错误。
一个内核工程师诊断 rows 问题时，第一步不是改 GUC。
第一步是确认这个 predicate 最终走到了哪个 estimator。
第二步是确认 estimator 得到的是统计事实还是默认假设。
第三步才是讨论成本参数或 path 选择。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：`clause_selectivity_ext()` 识别 clause 类型，operator clause 通过 `restriction_selectivity()` 或 `join_selectivity()` 找到 catalog 中登记的 estimator，estimator 尝试读取 `VariableStatData` 和统计槽位，失败时返回 `DEFAULT_*` 或 `0.5`，最终概率被乘到 rows 并进入成本搜索。
这个模型里有三个不同层次的默认值。
第一层是 `clause_selectivity_ext()` 对未处理 clause type 的默认 `0.5`。
第二层是 `restriction_selectivity()` 和 `join_selectivity()` 在缺少 `oprrest` 或 `oprjoin` 时使用的 `0.5`。
第三层是 `src/include/utils/selfuncs.h` 中按 operator 家族定义的 `DEFAULT_EQ_SEL`、`DEFAULT_INEQ_SEL`、`DEFAULT_RANGE_INEQ_SEL`、`DEFAULT_MATCH_SEL` 和 `DEFAULT_UNK_SEL`。
这三层默认值的语义不同，诊断时不能混成一个“统计差”。
`0.5` 通常说明 planner 对 clause 或 operator 缺少更具体知识。
`DEFAULT_EQ_SEL` 说明等值类 estimator 已经被调用，但缺少可用于更细估算的形状或统计。
`DEFAULT_INEQ_SEL` 说明范围类 estimator 不能把条件落到可比较的常量和有序分布上。
`DEFAULT_MATCH_SEL` 说明模式匹配估算缺少可靠 prefix、histogram 或 MCV 支撑。
`DEFAULT_UNK_SEL` 更接近 boolean 与 null test 的未知条件默认假设。
核心 tension 在于默认值必须足够便宜。
它们还必须足够稳定，不能因为缺少统计而让相同 SQL 的 plan 在同一版本里随机跳动。
但默认值如果太大，会让 index scan 看起来没有价值。
默认值如果太小，又会让 planner 过度相信过滤能力。
`selfuncs.h` 的注释说明默认值不是完全随机选择，而是和典型表密度、index scan 可用性、默认 distinct 数量有关。
本节的目标不是背这些常量。
本节的目标是知道这些常量在哪些失败边界被注入，以及它们如何传播成可见 plan 差异。

## 3. 核心文件分工与阅读顺序

- 第一步读 `src/include/utils/selfuncs.h`，先建立 `DEFAULT_*`、`CLAMP_PROBABILITY()`、`VariableStatData`、`EstimationInfo` 和 `ReleaseVariableStats()` 的状态边界。
- 第二步读 `src/backend/optimizer/path/clausesel.c`，重点看 `clause_selectivity_ext()` 如何处理节点类型、`RestrictInfo` cache、布尔表达式和 operator clause。
- 第三步读 `src/backend/optimizer/util/plancat.c`，重点看 `restriction_selectivity()` 与 `join_selectivity()` 如何通过 `pg_operator` 调用 estimator。
- 第四步读 `src/backend/utils/cache/lsyscache.c`，确认 `get_oprrest()` 和 `get_oprjoin()` 只是从 syscache 读取 `oprrest` 与 `oprjoin`。
- 第五步读 `src/backend/utils/adt/selfuncs.c`，本节只追 `examine_variable()`、`get_restriction_variable()`、`generic_restriction_selectivity()` 和 `get_variable_numdistinct()`。
- 第六步读 `src/include/catalog/pg_operator.dat`，用内置 equality、inequality 和 pattern operator 的 `oprrest` 字段反查 estimator 绑定。
- 第七步读 `src/backend/optimizer/path/costsize.c`，从 `set_baserel_size_estimates()` 看 selectivity 何时变成 `RelOptInfo.rows`。
- 第八步读 `src/include/nodes/pathnodes.h`，确认 `RestrictInfo.norm_selec`、`RestrictInfo.outer_selec`、`RelOptInfo.tuples` 和 `RelOptInfo.rows` 的承接关系。
- 第九步读 `src/backend/statistics/extended_stats.c`，只确认扩展统计会在 clause list 层抢先消费部分 clause。
- 第十步再回到 `EXPLAIN`，把 estimated rows 反推到上述某一层，而不是泛泛说 planner 估错了。
- 阅读顺序的关键是先找消费者，再找估算入口。
- 如果直接从 `selfuncs.c` 顶部线性阅读，会被几千行 estimator 细节淹没。
- 如果只看 `EXPLAIN`，又会把默认值、统计样本和成本模型混在一起。
- 本节的源码阅读应始终围绕一个问题：这个概率是在有事实的情况下算出来的，还是在缺事实时兜底出来的。

## 4. API 契约与默认值地图

Selectivity 的公开语义是概率。
在多数 planner 路径中，它应该落在 `[0, 1]`。
`CLAMP_PROBABILITY()` 的作用不是保证估算正确，而是防止越界概率继续污染行数和成本。
`restriction_selectivity()` 会检查 estimator 返回值，若小于零或大于一就报错。
这说明 PostgreSQL 区分两个问题：估算可以近似，但 API 结果必须保持有效概率形状。
Selectivity 本身不是 rows。
`set_baserel_size_estimates()` 中才会把 `rel->tuples` 乘以 `clauselist_selectivity()` 的结果。
Selectivity 本身也不是执行语义。
估错只影响 plan choice，不会让 executor 返回错误 tuple。
默认值的第一类含义是缺少结构识别。
例如 `clause_selectivity_ext()` 对无法处理的 clause type 初始化为 `0.5`。
默认值的第二类含义是缺少 operator catalog 绑定。
例如 `restriction_selectivity()` 在 `get_oprrest()` 返回无效过程时直接返回 `0.5`。
默认值的第三类含义是 estimator 已经知道 operator 家族，但缺少统计或常量形状。
例如 `eqsel_internal()` 无法识别 variable 与另一侧表达式时返回 `DEFAULT_EQ_SEL`。
默认值的第四类含义是 estimator 已经读到统计，但统计太小或不适合当前操作，只能混合启发式。
例如 `generic_restriction_selectivity()` 会在 histogram 太小时混入 caller 提供的 default selectivity。
这些情况都叫 fallback，但诊断含义不同。
第一类通常提示表达式形状或函数支持缺失。
第二类通常提示 operator 定义没有登记 selectivity procedure。
第三类通常提示统计无法绑定到变量、常量或安全检查。
第四类通常提示统计存在但质量不足或不匹配当前谓词。
课程讨论时要把 fallback 说成边界，而不是说成失败。
PostgreSQL 的目标是在边界内继续规划。
只有当 estimator 返回非法概率时，planner 才会把它当成错误。

## 5. 主流程源码 walkthrough

- Walkthrough 第 1 步：从消费者进入。
- 这一步的源码动作是 `set_baserel_size_estimates()` 读取 base relation 的 tuples 并调用 `clauselist_selectivity()`。
- 这一步改变或确认的 planner 状态是 `rel->rows` 尚未形成。
- 这一步的边界条件是 只适用于 base relation 的 size estimate。
- Walkthrough 第 2 步：把概率变成行数。
- 这一步的源码动作是 `rel->tuples` 乘以 restriction selectivity 后交给 `clamp_row_est()`。
- 这一步改变或确认的 planner 状态是 `RelOptInfo.rows` 被写入。
- 这一步的边界条件是 行数被夹到 planner 可处理的估算形状。
- Walkthrough 第 3 步：进入 clause list。
- 这一步的源码动作是 `clauselist_selectivity()` 调用扩展版本并保留默认使用扩展统计的语义。
- 这一步改变或确认的 planner 状态是 clause list 还没有被拆成单点 estimator。
- 这一步的边界条件是 单个 clause 会有快路径。
- Walkthrough 第 4 步：处理 RestrictInfo。
- 这一步的源码动作是 `clause_selectivity_ext()` 解包 `RestrictInfo` 并检查 cache。
- 这一步改变或确认的 planner 状态是 `norm_selec` 或 `outer_selec` 可能被复用。
- 这一步的边界条件是 `varRelid` 和 join type 会影响 cache 安全性。
- Walkthrough 第 5 步：处理 pseudoconstant。
- 这一步的源码动作是 pseudoconstant gating qual 通常返回 `1.0`。
- 这一步改变或确认的 planner 状态是 该 qual 不降低 relation rows。
- 这一步的边界条件是 常量 false 可以被估为零。
- Walkthrough 第 6 步：按节点类型分派。
- 这一步的源码动作是 Var、Const、Param、NOT、AND、OR、OpExpr、FuncExpr、ScalarArrayOpExpr、RowCompareExpr、NullTest 和 BooleanTest 各自走不同分支。
- 这一步改变或确认的 planner 状态是 selectivity 从抽象 clause 变成具体 estimator 调用。
- 这一步的边界条件是 未覆盖节点保留默认 `0.5`。
- Walkthrough 第 7 步：判断 join 或 restriction。
- 这一步的源码动作是 `treat_as_join_clause()` 决定 operator clause 的估算语义。
- 这一步改变或确认的 planner 状态是 同一个 operator 可能调用 restriction 或 join estimator。
- 这一步的边界条件是 outer join 上下文不能被忽略。
- Walkthrough 第 8 步：读取 restriction estimator。
- 这一步的源码动作是 `restriction_selectivity()` 调用 `get_oprrest()`。
- 这一步改变或确认的 planner 状态是 `pg_operator.oprrest` 变成过程 OID。
- 这一步的边界条件是 缺失时返回 `0.5`。
- Walkthrough 第 9 步：读取 join estimator。
- 这一步的源码动作是 `join_selectivity()` 调用 `get_oprjoin()`。
- 这一步改变或确认的 planner 状态是 `pg_operator.oprjoin` 变成过程 OID。
- 这一步的边界条件是 缺失时返回 `0.5`。
- Walkthrough 第 10 步：调用具体 estimator。
- 这一步的源码动作是 函数管理器按 catalog 绑定调用 `eqsel()` 或其它 estimator。
- 这一步改变或确认的 planner 状态是 统计 tuple、operator OID、args 和 collation 进入 estimator。
- 这一步的边界条件是 estimator 必须返回合法概率。
- Walkthrough 第 11 步：写回缓存。
- 这一步的源码动作是 可缓存场景把结果写入 `RestrictInfo` 的 selectivity 字段。
- 这一步改变或确认的 planner 状态是 后续 path 复用该概率。
- 这一步的边界条件是 不同语义上下文不能错误共用。
- Walkthrough 第 12 步：传播到 path search。
- 这一步的源码动作是 `RelOptInfo.rows` 被 access path 和 join search 消费。
- 这一步改变或确认的 planner 状态是 单点概率变成成本排序输入。
- 这一步的边界条件是 执行期实际行数只会在后续观测中暴露。

## 6. 核心源码锚点与状态阅读

- 源码锚点 `set_baserel_size_estimates` 位于 `src/backend/optimizer/path/costsize.c`，本节阅读它是为了确认 selectivity 变成 `RelOptInfo.rows` 的消费者边界。
- 进入 `set_baserel_size_estimates` 前先写下输入状态：`RelOptInfo.tuples`、`baserestrictinfo` 和 planner root。
- 离开 `set_baserel_size_estimates` 后再写下输出状态：`RelOptInfo.rows`、qual cost 和 width。
- 如果 `set_baserel_size_estimates` 不能获得足够信息，它的 fallback 是 继续用 `clamp_row_est()` 保护行数形状。
- 诊断 `set_baserel_size_estimates` 时不要只看最终 plan，要先观察 `rel->tuples` 与 `rel->rows` 的差值。
- 复盘 `set_baserel_size_estimates` 时要说明它返回的是概率、行数、缓存值还是错误。
- 源码锚点 `clause_selectivity_ext` 位于 `src/backend/optimizer/path/clausesel.c`，本节阅读它是为了确认 单个 clause 如何分派到具体 estimator。
- 进入 `clause_selectivity_ext` 前先写下输入状态：`RestrictInfo`、`varRelid`、`jointype` 和 `sjinfo`。
- 离开 `clause_selectivity_ext` 后再写下输出状态：单个 clause 的 `Selectivity`。
- 如果 `clause_selectivity_ext` 不能获得足够信息，它的 fallback 是 未处理节点保持默认 `0.5`。
- 诊断 `clause_selectivity_ext` 时不要只看最终 plan，要先观察 `norm_selec` 与 `outer_selec` 是否命中。
- 复盘 `clause_selectivity_ext` 时要说明它返回的是概率、行数、缓存值还是错误。
- 源码锚点 `restriction_selectivity` 位于 `src/backend/optimizer/util/plancat.c`，本节阅读它是为了确认 restriction operator 如何从 catalog 找估算函数。
- 进入 `restriction_selectivity` 前先写下输入状态：`pg_operator.oprrest` 和 operator arguments。
- 离开 `restriction_selectivity` 后再写下输出状态：restriction clause 的概率。
- 如果 `restriction_selectivity` 不能获得足够信息，它的 fallback 是 缺少 `oprrest` 时返回 `0.5`。
- 诊断 `restriction_selectivity` 时不要只看最终 plan，要先观察 `pg_operator.dat` 中的 `oprrest` 绑定。
- 复盘 `restriction_selectivity` 时要说明它返回的是概率、行数、缓存值还是错误。
- 源码锚点 `join_selectivity` 位于 `src/backend/optimizer/util/plancat.c`，本节阅读它是为了确认 join operator 如何从 catalog 找估算函数。
- 进入 `join_selectivity` 前先写下输入状态：`pg_operator.oprjoin`、join type 和 special join info。
- 离开 `join_selectivity` 后再写下输出状态：join clause 的概率。
- 如果 `join_selectivity` 不能获得足够信息，它的 fallback 是 缺少 `oprjoin` 时返回 `0.5`。
- 诊断 `join_selectivity` 时不要只看最终 plan，要先观察 join clause 是否被当成 restriction 估算。
- 复盘 `join_selectivity` 时要说明它返回的是概率、行数、缓存值还是错误。
- 源码锚点 `get_oprrest` 位于 `src/backend/utils/cache/lsyscache.c`，本节阅读它是为了确认 syscache 读取 restriction estimator procedure。
- 进入 `get_oprrest` 前先写下输入状态：`OPEROID` syscache 和 `Form_pg_operator`。
- 离开 `get_oprrest` 后再写下输出状态：restriction estimator 的 RegProcedure。
- 如果 `get_oprrest` 不能获得足够信息，它的 fallback 是 operator 不存在时返回 InvalidOid。
- 诊断 `get_oprrest` 时不要只看最终 plan，要先观察 自定义 operator 是否登记了估算函数。
- 复盘 `get_oprrest` 时要说明它返回的是概率、行数、缓存值还是错误。
- 源码锚点 `get_restriction_variable` 位于 `src/backend/utils/adt/selfuncs.c`，本节阅读它是为了确认 operator arguments 是否能拆成变量和另一侧表达式。
- 进入 `get_restriction_variable` 前先写下输入状态：左右 argument 的 `VariableStatData`。
- 离开 `get_restriction_variable` 后再写下输出状态：变量侧、另一侧和变量位置。
- 如果 `get_restriction_variable` 不能获得足够信息，它的 fallback 是 无法识别时让调用者使用家族默认值。
- 诊断 `get_restriction_variable` 时不要只看最终 plan，要先观察 表达式形状是否阻断统计绑定。
- 复盘 `get_restriction_variable` 时要说明它返回的是概率、行数、缓存值还是错误。
- 源码锚点 `examine_variable` 位于 `src/backend/utils/adt/selfuncs.c`，本节阅读它是为了确认 表达式如何绑定到 relation、统计 tuple 和类型信息。
- 进入 `examine_variable` 前先写下输入状态：planner root、node、varRelid 和 stats hook。
- 离开 `examine_variable` 后再写下输出状态：`VariableStatData` 的 rel、statsTuple、类型和权限信息。
- 如果 `examine_variable` 不能获得足够信息，它的 fallback 是 找不到统计时留下无效 stats tuple。
- 诊断 `examine_variable` 时不要只看最终 plan，要先观察 `VariableStatData.statsTuple` 是否有效。
- 复盘 `examine_variable` 时要说明它返回的是概率、行数、缓存值还是错误。
- 源码锚点 `ReleaseVariableStats` 位于 `src/include/utils/selfuncs.h`，本节阅读它是为了确认 释放 `VariableStatData` 中的统计 tuple。
- 进入 `ReleaseVariableStats` 前先写下输入状态：`statsTuple` 与 `freefunc` 的配对。
- 离开 `ReleaseVariableStats` 后再写下输出状态：一次 estimator 调用的统计资源被归还。
- 如果 `ReleaseVariableStats` 不能获得足够信息，它的 fallback 是 无统计 tuple 时不做释放动作。
- 诊断 `ReleaseVariableStats` 时不要只看最终 plan，要先观察 estimator 是否在所有返回路径释放统计状态。
- 复盘 `ReleaseVariableStats` 时要说明它返回的是概率、行数、缓存值还是错误。
- 源码锚点 `generic_restriction_selectivity` 位于 `src/backend/utils/adt/selfuncs.c`，本节阅读它是为了确认 没有专门知识的 operator 如何使用 MCV 和 histogram。
- 进入 `generic_restriction_selectivity` 前先写下输入状态：operator、collation、args、varRelid 和默认选择率。
- 离开 `generic_restriction_selectivity` 后再写下输出状态：合并 MCV 与 histogram 后的概率。
- 如果 `generic_restriction_selectivity` 不能获得足够信息，它的 fallback 是 无法识别变量或统计不足时回到 caller default。
- 诊断 `generic_restriction_selectivity` 时不要只看最终 plan，要先观察 小 histogram 是否被混入默认值。
- 复盘 `generic_restriction_selectivity` 时要说明它返回的是概率、行数、缓存值还是错误。
- 源码锚点 `get_variable_numdistinct` 位于 `src/backend/utils/adt/selfuncs.c`，本节阅读它是为了确认 默认 distinct 与统计 distinct 如何支撑等值 fallback。
- 进入 `get_variable_numdistinct` 前先写下输入状态：`VariableStatData` 与统计 tuple。
- 离开 `get_variable_numdistinct` 后再写下输出状态：估算 distinct 数和是否默认的标记。
- 如果 `get_variable_numdistinct` 不能获得足够信息，它的 fallback 是 无统计时使用默认 distinct 推断。
- 诊断 `get_variable_numdistinct` 时不要只看最终 plan，要先观察 等值长尾是否只是平均假设。
- 复盘 `get_variable_numdistinct` 时要说明它返回的是概率、行数、缓存值还是错误。

## 7. 关键状态与生命周期 ownership

`VariableStatData` 是本节最重要的短生命周期状态。
它不是 catalog tuple 的永久引用。
它只是 estimator 在一次估算中携带变量、relation、统计 tuple、类型和权限信息的局部对象。
`statsTuple` 可能来自 syscache，也可能来自复制出来的 tuple。
因此 `VariableStatData` 保存了 `freefunc`，用于释放 `statsTuple`。
`ReleaseVariableStats()` 宏体现了这个 ownership。
每个调用 `examine_variable()` 或 `get_restriction_variable()` 的 estimator 都必须在退出前释放统计状态。
这和 planner context 中长期存在的 `RelOptInfo` 不一样。
`RelOptInfo` 持有 base relation 的估算结果和 path 列表。
`RestrictInfo` 持有 clause 包装、relids、移动性信息和 selectivity cache。
`VariableStatData` 只活在估算函数的局部调用链中。
默认常量没有 ownership。
`DEFAULT_EQ_SEL` 等宏在编译期定义，不需要释放，也不会记住某次估算上下文。
`EstimationInfo` 是另一类轻量状态。
它当前主要允许部分估算函数通过 `SELFLAG_USED_DEFAULT` 把使用默认值的信息传回调用者。
这类标记不是所有 selectivity 调用都会暴露到 `EXPLAIN`。
它的价值在于源码诊断时提醒你：估算器知道自己走过默认路径。
生命周期问题的常见误区是把统计 tuple 看成 `RelOptInfo` 的一部分。
实际上 estimator 应该在函数返回前释放它。
另一个误区是把 `RestrictInfo` cache 看成统计缓存。
它只缓存某个 clause 在特定 planner 上下文里的估算结果。
统计源变更和 catalog invalidation 是更外层的问题，本节不展开。

## 8. 默认值不是失败，而是显式 fallback

`selfuncs.h` 对默认值的注释直接说明它们不是随机常数。
`DEFAULT_EQ_SEL` 是 `0.005`，对应默认 distinct 数约为二百。
`DEFAULT_INEQ_SEL` 是三分之一，用于普通范围不等式。
`DEFAULT_RANGE_INEQ_SEL` 是 `0.005`，用于两个范围边界合成后的默认估算。
`DEFAULT_MATCH_SEL` 是 `0.005`，用于 LIKE 和 regex 之类模式匹配。
`DEFAULT_MATCHING_SEL` 是 `0.010`，用于其它 matching operator。
`DEFAULT_UNK_SEL` 是 `0.005`，用于 boolean 与 null test 的未知情况。
这些值的选择要让典型表密度下的 index scan 仍然可能被考虑。
如果默认等值选择率过大，index scan 的页访问收益会被低估。
如果默认等值选择率过小，planner 会过度相信任意未知等值谓词。
因此默认值是 PostgreSQL 在规划阶段接受的工程折中。
它们把“不知道”变成“可继续搜索但带风险的数字”。
fallback 的正确性含义是 SQL 结果仍然正确。
fallback 的性能含义是 plan choice 可能偏离真实最优。
fallback 的诊断含义是要把问题定位到信息缺口。
如果缺口是 operator catalog，就补 `oprrest` 或调整 operator 定义。
如果缺口是统计 tuple，就检查 `ANALYZE`、统计目标、表达式统计或权限。
如果缺口是表达式形状，就检查是否被函数包裹、collation 是否可比较、常量是否能折叠。
如果缺口是多列相关性，就不该在单个 estimator 里解决，而要回到扩展统计或 clause list 组合。
这就是默认值边界的课程价值。

## 9. 正确性机制层次与权限边界

selectivity 估算不参与 MVCC 可见性判断。
它也不决定 tuple 是否满足 predicate。
正确性由 executor、operator 函数、快照和访问方法保证。
selectivity 只影响选择哪个计划更便宜。
但 selectivity API 仍然有自己的正确性边界。
第一条正确性边界是返回值必须保持合法概率范围。
estimator 返回非法概率时，`restriction_selectivity()` 和 `join_selectivity()` 会报错。
第二条正确性边界是统计调用必须通过安全检查。
`statistic_proc_security_check()` 会限制某些统计调用，避免通过 estimator 绕过权限或执行不应执行的函数。
第三条边界是 strict operator 对 NULL 的约定。
许多 estimator 在右侧常量为 NULL 时直接返回 `0.0`。
这是估算层的常见假设，不是 executor 对所有自定义 operator 的证明。
第四条边界是 join 与 restriction 的语义区分。
`treat_as_join_clause()` 判断同一个 operator clause 应走 join estimator 还是 restriction estimator。
第五条边界是 outer join 语义。
`RestrictInfo.outer_selec` 与 `norm_selec` 分开缓存，避免把 outer join 上下文下的估算误用到 inner 语义。
第六条边界是 `varRelid`。
带 `varRelid` 的估算可能只针对某个 relation 视角，不能随意复用到所有上下文。
这些边界让 planner 能在不完美信息下保持可解释。
它们不能保证估算总是接近真实行数。
它们保证的是 estimator 的输入、返回值和缓存不会越过明显语义界限。

## 10. 错误路径、异常路径与 fallback 分类

- Fallback 场景 `空 clause` 的触发条件是 调用者传入 NULL clause。
- 该场景交给后续 planner 的估算含义是 返回 `0.5` 并让规划继续。
- 该场景不改变 SQL 正确性，但会改变 path search 的成本排序。
- 排查该场景时优先尝试 检查调用链是否仍可能产生空 clause。
- Fallback 场景 `未识别节点` 的触发条件是 `clause_selectivity_ext()` 没有覆盖该 Node tag。
- 该场景交给后续 planner 的估算含义是 保留初始化的 `0.5`。
- 该场景不改变 SQL 正确性，但会改变 path search 的成本排序。
- 排查该场景时优先尝试 确认表达式是否需要 support function 或专用 estimator。
- Fallback 场景 `缺 `oprrest`` 的触发条件是 `get_oprrest()` 返回 InvalidOid。
- 该场景交给后续 planner 的估算含义是 restriction selectivity 使用 `0.5`。
- 该场景不改变 SQL 正确性，但会改变 path search 的成本排序。
- 排查该场景时优先尝试 检查 `pg_operator` 定义和 `pg_operator.dat` 绑定。
- Fallback 场景 `缺 `oprjoin`` 的触发条件是 `get_oprjoin()` 返回 InvalidOid。
- 该场景交给后续 planner 的估算含义是 join selectivity 使用 `0.5`。
- 该场景不改变 SQL 正确性，但会改变 path search 的成本排序。
- 排查该场景时优先尝试 检查 join operator 是否登记 join estimator。
- Fallback 场景 `变量识别失败` 的触发条件是 `get_restriction_variable()` 无法拆出变量侧。
- 该场景交给后续 planner 的估算含义是 家族 estimator 返回自己的 `DEFAULT_*`。
- 该场景不改变 SQL 正确性，但会改变 path search 的成本排序。
- 排查该场景时优先尝试 改写表达式或创建表达式统计。
- Fallback 场景 `统计 tuple 缺失` 的触发条件是 `VariableStatData.statsTuple` 无效。
- 该场景交给后续 planner 的估算含义是 等值和范围进入默认 distinct 或默认范围估算。
- 该场景不改变 SQL 正确性，但会改变 path search 的成本排序。
- 排查该场景时优先尝试 运行 `ANALYZE` 并检查统计目标。
- Fallback 场景 `权限安全检查失败` 的触发条件是 `statistic_proc_security_check()` 不允许调用统计 operator。
- 该场景交给后续 planner 的估算含义是 MCV 或 histogram 槽位不能被正常消费。
- 该场景不改变 SQL 正确性，但会改变 path search 的成本排序。
- 排查该场景时优先尝试 确认当前用户权限和 operator 安全性。
- Fallback 场景 `小 histogram` 的触发条件是 histogram entry 数量不足以完全信任。
- 该场景交给后续 planner 的估算含义是 结果会与默认值或启发式混合。
- 该场景不改变 SQL 正确性，但会改变 path search 的成本排序。
- 排查该场景时优先尝试 提高统计目标并重新 `ANALYZE`。
- Fallback 场景 `非法概率` 的触发条件是 自定义 estimator 返回小于零或大于一。
- 该场景交给后续 planner 的估算含义是 planner 报错而不是继续传播。
- 该场景不改变 SQL 正确性，但会改变 path search 的成本排序。
- 排查该场景时优先尝试 修复 estimator 并使用合法概率返回值。
- Fallback 场景 `outer join 上下文` 的触发条件是 同一 clause 以 outer join 语义被估算。
- 该场景交给后续 planner 的估算含义是 使用 `outer_selec` 而不是 `norm_selec`。
- 该场景不改变 SQL 正确性，但会改变 path search 的成本排序。
- 排查该场景时优先尝试 确认 join type 与 SpecialJoinInfo。
- Fallback 场景 `函数支持缺失` 的触发条件是 `FuncExpr` 没有 support function 返回估算。
- 该场景交给后续 planner 的估算含义是 退回 boolean 估算路径。
- 该场景不改变 SQL 正确性，但会改变 path search 的成本排序。
- 排查该场景时优先尝试 为函数提供 planner support 或改写 predicate。
- Fallback 场景 `扩展统计未覆盖` 的触发条件是 clause list 不满足单 relation 或统计对象不匹配。
- 该场景交给后续 planner 的估算含义是 单个 clause 继续按普通 estimator 处理。
- 该场景不改变 SQL 正确性，但会改变 path search 的成本排序。
- 排查该场景时优先尝试 检查 `statext_clauselist_selectivity()` 的 estimated clauses。

## 11. 成本、资源与跨模块传播

selectivity 一旦变成 `RelOptInfo.rows`，影响就不再局限于单个 predicate。
base relation 的 `rows` 会进入 `cost_seqscan()`、`cost_index()` 和其它 access path 估算。
index path 还会使用 index condition 的 selectivity 估算访问多少 index tuple 和 heap page。
如果 restriction selectivity 过低，planner 可能过度偏向 index scan 或 nested loop。
如果 restriction selectivity 过高，planner 可能忽略本来很有效的 index path。
base rows 还会进入 join cardinality。
join cardinality 再影响 join order、join method、hash table size 和 sort 代价。
一个默认等值选择率可能从 base rel 传播到 join rel，再传播到 upper rel。
上层 aggregate 的 group 数、sort 的输入行数、limit 的启动成本判断都会间接受影响。
这就是为什么默认值不是无害细节。
它们虽然不影响 SQL correctness，却会改变整个 path search 的代价排序。
资源层面也要区分估算成本和运行成本。
planner 调用 estimator 的成本发生在 planning 阶段。
估错导致的额外 I/O、CPU、内存和临时文件发生在 execution 阶段。
诊断时要把 planning 阶段成本和 execution 阶段成本分开。
如果 planning time 很高，要看复杂表达式、扩展统计、partition 和 path explosion。
如果 execution time 很高但 planning time 正常，要看估算数字如何引导了错误 path。
selectivity default 的典型影响是第二类。
它很快返回一个数字，却可能让后续执行选到更贵路径。
因此课堂实验要同时观察 estimated rows、actual rows、chosen path 和 buffers。

## 12. 观测与诊断入口

- 诊断卡片 `operator 缺估算函数` 的可见现象是 estimated rows 接近输入行数的一半或 join 选择率异常粗糙。
- 第一反查入口是 `restriction_selectivity`，因为这里最早决定是否仍有真实统计输入。
- 第二反查入口是 `get_oprrest`，因为这里能确认 fallback 是否已经发生。
- 如果两个入口都没有解释偏差，再继续看 clause 组合、join cardinality 或成本参数。
- 这个诊断的结论应写成：operator catalog 缺少 restriction estimator。
- 诊断卡片 `表达式无法拆变量` 的可见现象是 自定义函数包裹列后 rows 从统计驱动退化成默认值。
- 第一反查入口是 `get_restriction_variable`，因为这里最早决定是否仍有真实统计输入。
- 第二反查入口是 `examine_variable`，因为这里能确认 fallback 是否已经发生。
- 如果两个入口都没有解释偏差，再继续看 clause 组合、join cardinality 或成本参数。
- 这个诊断的结论应写成：表达式形状阻断了统计绑定。
- 诊断卡片 `统计 tuple 缺失` 的可见现象是 刚建表或长时间未 ANALYZE 后等值条件估算粗糙。
- 第一反查入口是 `examine_variable`，因为这里最早决定是否仍有真实统计输入。
- 第二反查入口是 `get_variable_numdistinct`，因为这里能确认 fallback 是否已经发生。
- 如果两个入口都没有解释偏差，再继续看 clause 组合、join cardinality 或成本参数。
- 这个诊断的结论应写成：估算依赖默认 distinct 或默认选择率。
- 诊断卡片 `权限检查阻断` 的可见现象是 统计存在但 estimator 没有使用某些统计槽位。
- 第一反查入口是 `statistic_proc_security_check`，因为这里最早决定是否仍有真实统计输入。
- 第二反查入口是 `mcv_selectivity`，因为这里能确认 fallback 是否已经发生。
- 如果两个入口都没有解释偏差，再继续看 clause 组合、join cardinality 或成本参数。
- 这个诊断的结论应写成：安全边界阻断了统计槽位消费。
- 诊断卡片 `cache 语义误读` 的可见现象是 同一 clause 在不同 join 语义下估算看似重复。
- 第一反查入口是 `clause_selectivity_ext`，因为这里最早决定是否仍有真实统计输入。
- 第二反查入口是 `RestrictInfo`，因为这里能确认 fallback 是否已经发生。
- 如果两个入口都没有解释偏差，再继续看 clause 组合、join cardinality 或成本参数。
- 这个诊断的结论应写成：selectivity cache 受 varRelid 和 join type 限制。
- 诊断卡片 `默认值传播` 的可见现象是 base scan rows 小偏差在 join 后被放大。
- 第一反查入口是 `set_baserel_size_estimates`，因为这里最早决定是否仍有真实统计输入。
- 第二反查入口是 `RelOptInfo.rows`，因为这里能确认 fallback 是否已经发生。
- 如果两个入口都没有解释偏差，再继续看 clause 组合、join cardinality 或成本参数。
- 这个诊断的结论应写成：base relation rows 已经带着默认假设进入 join search。
- 诊断卡片 `扩展统计未接管` 的可见现象是 多列条件仍按独立性传播到后续 rows。
- 第一反查入口是 `clauselist_selectivity_ext`，因为这里最早决定是否仍有真实统计输入。
- 第二反查入口是 `statext_clauselist_selectivity`，因为这里能确认 fallback 是否已经发生。
- 如果两个入口都没有解释偏差，再继续看 clause 组合、join cardinality 或成本参数。
- 这个诊断的结论应写成：单点 estimator 不能解决多列相关性。
- 诊断卡片 `默认标记有限` 的可见现象是 源码可见 fallback，但 EXPLAIN 不直接说明。
- 第一反查入口是 `EstimationInfo`，因为这里最早决定是否仍有真实统计输入。
- 第二反查入口是 `SELFLAG_USED_DEFAULT`，因为这里能确认 fallback 是否已经发生。
- 如果两个入口都没有解释偏差，再继续看 clause 组合、join cardinality 或成本参数。
- 这个诊断的结论应写成：默认路径并不总会在用户界面显式展示。
- 诊断卡片 `函数支持缺失` 的可见现象是 布尔函数过滤条件估算接近粗略默认值。
- 第一反查入口是 `clause_selectivity_ext`，因为这里最早决定是否仍有真实统计输入。
- 第二反查入口是 `boolvarsel`，因为这里能确认 fallback 是否已经发生。
- 如果两个入口都没有解释偏差，再继续看 clause 组合、join cardinality 或成本参数。
- 这个诊断的结论应写成：缺少 planner support function 导致 fallback。
- 诊断卡片 `非法 estimator` 的可见现象是 自定义 estimator 导致 planning 时报错。
- 第一反查入口是 `restriction_selectivity`，因为这里最早决定是否仍有真实统计输入。
- 第二反查入口是 `join_selectivity`，因为这里能确认 fallback 是否已经发生。
- 如果两个入口都没有解释偏差，再继续看 clause 组合、join cardinality 或成本参数。
- 这个诊断的结论应写成：返回值越界违反 selectivity API 契约。

## 13. 课堂实验：让默认值暴露出来

- 课堂实验 `未分析的等值条件` 的准备动作是 新建表、插入偏斜数据但暂不运行 `ANALYZE`。
- 实验 SQL 关注的 predicate 是 普通列等值过滤。
- `EXPLAIN` 观察点是 base scan estimated rows 是否接近默认 distinct 假设。
- 回到源码时先读 `selfuncs.c`，确认估算数字从哪里进入 planner。
- 实验报告必须同时写出可见现象、源码入口、fallback 条件和可迁移规律。
- 课堂实验 `分析后的等值条件` 的准备动作是 对同一张表运行 `ANALYZE` 并保持 SQL 不变。
- 实验 SQL 关注的 predicate 是 普通列等值过滤。
- `EXPLAIN` 观察点是 estimated rows 是否向 MCV 频率靠近。
- 回到源码时先读 `eqsel`，确认估算数字从哪里进入 planner。
- 实验报告必须同时写出可见现象、源码入口、fallback 条件和可迁移规律。
- 课堂实验 `函数包裹列` 的准备动作是 比较普通列过滤与函数包裹列过滤。
- 实验 SQL 关注的 predicate 是 函数结果与常量比较。
- `EXPLAIN` 观察点是 函数包裹后 rows 是否退化。
- 回到源码时先读 `get_restriction_variable`，确认估算数字从哪里进入 planner。
- 实验报告必须同时写出可见现象、源码入口、fallback 条件和可迁移规律。
- 课堂实验 `表达式统计修复` 的准备动作是 为表达式创建统计或改写为可识别表达式后重新分析。
- 实验 SQL 关注的 predicate 是 表达式过滤。
- `EXPLAIN` 观察点是 estimated rows 是否改善。
- 回到源码时先读 `examine_variable`，确认估算数字从哪里进入 planner。
- 实验报告必须同时写出可见现象、源码入口、fallback 条件和可迁移规律。
- 课堂实验 `自定义 operator 缺绑定` 的准备动作是 使用没有登记 restriction estimator 的 operator 做过滤。
- 实验 SQL 关注的 predicate 是 自定义 operator restriction。
- `EXPLAIN` 观察点是 estimated rows 是否表现为 `0.5` 级别粗估。
- 回到源码时先读 `restriction_selectivity`，确认估算数字从哪里进入 planner。
- 实验报告必须同时写出可见现象、源码入口、fallback 条件和可迁移规律。
- 课堂实验 `布尔列未知选择率` 的准备动作是 构造布尔列并比较不同布尔测试写法。
- 实验 SQL 关注的 predicate 是 布尔列过滤。
- `EXPLAIN` 观察点是 estimated rows 是否受 boolean estimator 和统计影响。
- 回到源码时先读 `boolvarsel`，确认估算数字从哪里进入 planner。
- 实验报告必须同时写出可见现象、源码入口、fallback 条件和可迁移规律。
- 课堂实验 `多列相关条件` 的准备动作是 构造两个强相关列并只建单列统计。
- 实验 SQL 关注的 predicate 是 两个等值条件同时过滤。
- `EXPLAIN` 观察点是 组合 rows 是否明显低估。
- 回到源码时先读 `clauselist_selectivity_ext`，确认估算数字从哪里进入 planner。
- 实验报告必须同时写出可见现象、源码入口、fallback 条件和可迁移规律。
- 课堂实验 `统计目标变化` 的准备动作是 提高列统计目标后重新 `ANALYZE`。
- 实验 SQL 关注的 predicate 是 偏斜列过滤。
- `EXPLAIN` 观察点是 MCV 或 histogram 是否更能覆盖偏斜分布。
- 回到源码时先读 `analyze.c`，确认估算数字从哪里进入 planner。
- 实验报告必须同时写出可见现象、源码入口、fallback 条件和可迁移规律。
- 课堂实验 `outer join 条件` 的准备动作是 构造 outer join 并观察 join qual 与 pushed qual。
- 实验 SQL 关注的 predicate 是 outer join 条件。
- `EXPLAIN` 观察点是 `norm_selec` 与 `outer_selec` 是否可能分开缓存。
- 回到源码时先读 `clausesel.c`，确认估算数字从哪里进入 planner。
- 实验报告必须同时写出可见现象、源码入口、fallback 条件和可迁移规律。
- 课堂实验 `默认路径断点` 的准备动作是 在本地调试环境给 estimator 打断点。
- 实验 SQL 关注的 predicate 是 任意过滤条件。
- `EXPLAIN` 观察点是 是否真的进入 `DEFAULT_*` 返回路径。
- 回到源码时先读 `selfuncs.h`，确认估算数字从哪里进入 planner。
- 实验报告必须同时写出可见现象、源码入口、fallback 条件和可迁移规律。

## 14. 源码练习：从 rows 偏差反推 fallback

- 源码练习 `确认 operator catalog 入口` 从 `src/backend/optimizer/util/plancat.c` 开始。
- 练习动作是 在 `restriction_selectivity()` 处跟踪 `operatorid` 与 `oprrest`。
- 完成标准是能解释 缺少 `oprrest` 为什么直接得到 `0.5`。
- 如果只能背函数名，说明还没有把状态生命周期和 fallback 边界串起来。
- 源码练习 `确认 clause 分派` 从 `src/backend/optimizer/path/clausesel.c` 开始。
- 练习动作是 在 `clause_selectivity_ext()` 中记录每种节点类型走向。
- 完成标准是能解释 同一个 SQL 条件为何可能不是普通 `OpExpr`。
- 如果只能背函数名，说明还没有把状态生命周期和 fallback 边界串起来。
- 源码练习 `确认统计绑定` 从 `src/backend/utils/adt/selfuncs.c` 开始。
- 练习动作是 跟踪 `examine_variable()` 返回的 `VariableStatData`。
- 完成标准是能解释 `statsTuple`、`rel`、`vartype` 与 `acl_ok` 如何共同决定估算质量。
- 如果只能背函数名，说明还没有把状态生命周期和 fallback 边界串起来。
- 源码练习 `确认释放路径` 从 `src/include/utils/selfuncs.h` 开始。
- 练习动作是 检查调用 `ReleaseVariableStats()` 的所有提前返回路径。
- 完成标准是能解释 短生命周期统计 tuple 为什么不能挂在 planner 结构里。
- 如果只能背函数名，说明还没有把状态生命周期和 fallback 边界串起来。
- 源码练习 `确认默认常量` 从 `src/include/utils/selfuncs.h` 开始。
- 练习动作是 把 `DEFAULT_*` 与调用它们的 estimator 一一对应。
- 完成标准是能解释 `DEFAULT_EQ_SEL` 与 catalog 层 `0.5` 的不同含义。
- 如果只能背函数名，说明还没有把状态生命周期和 fallback 边界串起来。
- 源码练习 `确认 rows 消费者` 从 `src/backend/optimizer/path/costsize.c` 开始。
- 练习动作是 从 `set_baserel_size_estimates()` 反向找到 `clauselist_selectivity()`。
- 完成标准是能解释 selectivity 何时变成 `rel->rows`。
- 如果只能背函数名，说明还没有把状态生命周期和 fallback 边界串起来。
- 源码练习 `确认 cache 条件` 从 `src/include/nodes/pathnodes.h` 开始。
- 练习动作是 查找 `RestrictInfo` 的 selectivity 缓存字段。
- 完成标准是能解释 为什么 outer join 与 inner 语义不能共用一个缓存。
- 如果只能背函数名，说明还没有把状态生命周期和 fallback 边界串起来。
- 源码练习 `确认扩展统计边界` 从 `src/backend/statistics/extended_stats.c` 开始。
- 练习动作是 只看 `statext_clauselist_selectivity()` 如何标记已估算 clause。
- 完成标准是能解释 为什么单个 clause API 不负责多列相关性。
- 如果只能背函数名，说明还没有把状态生命周期和 fallback 边界串起来。
- 源码练习 `确认函数支持路径` 从 `src/backend/optimizer/path/clausesel.c` 开始。
- 练习动作是 查看 `FuncExpr` 如何调用函数选择率路径。
- 完成标准是能解释 support function 缺失时为什么会退回 boolean 估算。
- 如果只能背函数名，说明还没有把状态生命周期和 fallback 边界串起来。
- 源码练习 `确认非法概率处理` 从 `src/backend/optimizer/util/plancat.c` 开始。
- 练习动作是 让自定义 estimator 返回越界值并观察错误。
- 完成标准是能解释 近似估算和非法 API 返回值之间的边界。
- 如果只能背函数名，说明还没有把状态生命周期和 fallback 边界串起来。

## 15. 常见误区：不要把默认值看成统计事实

- 误区一是把 estimated rows 当成 executor 采样结果；planner 没有执行这个 predicate，它只在 planning 阶段估算。
- 误区二是看到 `0.5` 就说统计目标太低；`0.5` 更常见地说明 clause 或 operator 层缺少具体 estimator。
- 误区三是把 `DEFAULT_EQ_SEL` 当成所有等值条件的默认命运；等值 estimator 还有 MCV、unique、ndistinct 和 null fraction 等改善路径。
- 误区四是把 `pg_stats` 有数据等同于 estimator 一定用到了这些数据；表达式形状、权限、安全检查和 operator 类型都可能阻断使用。
- 误区五是把 selectivity cache 当成跨查询缓存；`RestrictInfo` cache 是当前 planner 生命周期内的局部缓存。
- 误区六是把 fallback 归咎于 executor；fallback 发生在 planner 阶段，executor 只执行已经选出的计划。
- 误区七是只改成本 GUC；如果 rows 已经偏差几个数量级，成本 GUC 通常只是在错误输入上重新排序。
- 误区八是把多列相关性塞进单列 estimator；相关性应优先看扩展统计和 clause list 层。
- 误区九是忽略 NULL；很多 estimator 都会先处理 `stanullfrac` 或 NULL 常量。
- 误区十是把估算默认值看成版本无关常识；默认值和 estimator 细节可能随 PostgreSQL 版本演进。

## 16. 讨论题与本节小结

- 讨论题一：如果一个自定义 operator 没有 `oprrest`，planner 为什么选择返回 `0.5` 而不是报错。
- 讨论题二：`DEFAULT_EQ_SEL` 为什么要和 `DEFAULT_NUM_DISTINCT` 保持直觉上的对应关系。
- 讨论题三：为什么 `VariableStatData` 的 `statsTuple` 要由 estimator 释放，而不是交给 `RelOptInfo` 持有。
- 讨论题四：一个函数包裹列的 predicate 为什么可能失去普通列统计。
- 讨论题五：`RestrictInfo` 的 `norm_selec` 和 `outer_selec` 分开缓存解决了什么错误复用风险。
- 讨论题六：为什么默认值不能通过 `EXPLAIN` 一眼全部看出来。
- 讨论题七：当 estimated rows 与 actual rows 偏差很大时，如何区分统计缺失、operator estimator 缺失和多列相关性。
- 本节小结一：信息不足时 PostgreSQL 仍然必须返回 selectivity，因为 path search 需要稳定概率。
- 本节小结二：`clause_selectivity_ext()` 建立 clause 到 estimator 的分派边界。
- 本节小结三：`restriction_selectivity()` 和 `join_selectivity()` 建立 operator catalog 到 estimator 的调用边界。
- 本节小结四：`VariableStatData` 建立一次估算中统计 tuple 的短生命周期 ownership。
- 本节小结五：`DEFAULT_*` 建立 estimator 无法获得足够信息时的 fallback 边界。
- 本节小结六：`set_baserel_size_estimates()` 建立概率进入 rows 和 cost 的传播边界。
- 本节小结七：诊断 rows 偏差时，要先找到 fallback 发生的层次。
- 本节小结八：下一节会进入等值、范围和模式匹配三个 estimator 家族。
## 17. 运行案例拆解：把可见现象落到源码入口

- 案例 `新表未分析` 的 SQL 形状是 普通列等值过滤。
- 可观察现象是 estimated rows 与真实行数相差很大，但 plan 仍能生成。
- 第一源码入口是 `examine_variable`，因为这里最早把输入事实压缩成选择率。
- 诊断时要确认该入口拿到的是统计事实、catalog 绑定、表达式形状还是默认假设。
- 可迁移结论是 没有统计不等于 planner 停止工作，而是进入默认估算边界。
- 案例 `operator 没有 oprrest` 的 SQL 形状是 自定义 operator 用在 WHERE 中。
- 可观察现象是 选择率接近 catalog 层粗估。
- 第一源码入口是 `restriction_selectivity`，因为这里最早把输入事实压缩成选择率。
- 诊断时要确认该入口拿到的是统计事实、catalog 绑定、表达式形状还是默认假设。
- 可迁移结论是 operator catalog 绑定是进入 estimator 家族之前的硬边界。
- 案例 `operator 有 oprrest 但统计缺失` 的 SQL 形状是 内置等值 operator 用在未分析列上。
- 可观察现象是 估算不再是 catalog 层 `0.5`，但仍可能很粗。
- 第一源码入口是 `eqsel_internal`，因为这里最早把输入事实压缩成选择率。
- 诊断时要确认该入口拿到的是统计事实、catalog 绑定、表达式形状还是默认假设。
- 可迁移结论是 进入 estimator 不代表已经拥有真实统计。
- 案例 `函数包裹列` 的 SQL 形状是 函数结果与常量比较。
- 可观察现象是 列本身有统计但估算没有明显利用列统计。
- 第一源码入口是 `get_restriction_variable`，因为这里最早把输入事实压缩成选择率。
- 诊断时要确认该入口拿到的是统计事实、catalog 绑定、表达式形状还是默认假设。
- 可迁移结论是 表达式形状决定统计能否绑定到变量侧。
- 案例 `表达式统计修复` 的 SQL 形状是 同一函数表达式创建统计后重新分析。
- 可观察现象是 estimated rows 明显靠近真实值。
- 第一源码入口是 `examine_variable`，因为这里最早把输入事实压缩成选择率。
- 诊断时要确认该入口拿到的是统计事实、catalog 绑定、表达式形状还是默认假设。
- 可迁移结论是 修复信息缺口要对准表达式形状，而不是只提高表级统计目标。
- 案例 `outer join qual` 的 SQL 形状是 outer join 的过滤条件在不同语义下被估算。
- 可观察现象是 同一 clause 可能有不同 cache 字段。
- 第一源码入口是 `clause_selectivity_ext`，因为这里最早把输入事实压缩成选择率。
- 诊断时要确认该入口拿到的是统计事实、catalog 绑定、表达式形状还是默认假设。
- 可迁移结论是 selectivity cache 必须带着 join 语义解释。
- 案例 `Param 不能常量折叠` 的 SQL 形状是 prepared statement 使用参数过滤。
- 可观察现象是 估算更接近平均选择率。
- 第一源码入口是 `estimate_expression_value`，因为这里最早把输入事实压缩成选择率。
- 诊断时要确认该入口拿到的是统计事实、catalog 绑定、表达式形状还是默认假设。
- 可迁移结论是 planning 时不可见的值只能使用平均或默认假设。
- 案例 `布尔函数过滤` 的 SQL 形状是 返回 boolean 的函数直接出现在 WHERE 中。
- 可观察现象是 估算无法像普通列等值那样使用 MCV。
- 第一源码入口是 `clause_selectivity_ext`，因为这里最早把输入事实压缩成选择率。
- 诊断时要确认该入口拿到的是统计事实、catalog 绑定、表达式形状还是默认假设。
- 可迁移结论是 FuncExpr 需要 support function 才能提供更具体估算。
- 案例 `NULL 测试` 的 SQL 形状是 IS NULL 或 IS NOT NULL 过滤。
- 可观察现象是 估算与 null fraction 相关。
- 第一源码入口是 `nulltestsel`，因为这里最早把输入事实压缩成选择率。
- 诊断时要确认该入口拿到的是统计事实、catalog 绑定、表达式形状还是默认假设。
- 可迁移结论是 Boolean 与 null test 有自己的 estimator，不应按普通等值解释。
- 案例 `ScalarArray 条件` 的 SQL 形状是 列与常量数组比较。
- 可观察现象是 估算不等同于单个等值选择率。
- 第一源码入口是 `scalararraysel`，因为这里最早把输入事实压缩成选择率。
- 诊断时要确认该入口拿到的是统计事实、catalog 绑定、表达式形状还是默认假设。
- 可迁移结论是 数组谓词有节点级 estimator，默认值传播方式不同。
- 案例 `RowCompare 条件` 的 SQL 形状是 多列行比较表达式。
- 可观察现象是 估算来源不在单个 operator family 内完全解释。
- 第一源码入口是 `rowcomparesel`，因为这里最早把输入事实压缩成选择率。
- 诊断时要确认该入口拿到的是统计事实、catalog 绑定、表达式形状还是默认假设。
- 可迁移结论是 复杂表达式要先确认节点分派再讨论统计槽位。
- 案例 `OR 条件` 的 SQL 形状是 多个 predicate 以 OR 连接。
- 可观察现象是 estimated rows 不等于各选择率简单相加。
- 第一源码入口是 `clauselist_selectivity_or`，因为这里最早把输入事实压缩成选择率。
- 诊断时要确认该入口拿到的是统计事实、catalog 绑定、表达式形状还是默认假设。
- 可迁移结论是 OR 组合属于 clause list 层，不属于单个 estimator 的责任。
- 案例 `AND 条件` 的 SQL 形状是 多个 predicate 同时限制一个 relation。
- 可观察现象是 单点估算正常但组合 rows 偏差大。
- 第一源码入口是 `clauselist_selectivity_ext`，因为这里最早把输入事实压缩成选择率。
- 诊断时要确认该入口拿到的是统计事实、catalog 绑定、表达式形状还是默认假设。
- 可迁移结论是 多条件相关性要回到组合层和扩展统计。
- 案例 `扩展统计命中` 的 SQL 形状是 同一 relation 的多列条件有统计对象。
- 可观察现象是 部分 clause 被提前消费。
- 第一源码入口是 `statext_clauselist_selectivity`，因为这里最早把输入事实压缩成选择率。
- 诊断时要确认该入口拿到的是统计事实、catalog 绑定、表达式形状还是默认假设。
- 可迁移结论是 扩展统计接管后不能再重复乘同一 clause。
- 案例 `权限导致统计不可用` 的 SQL 形状是 低权限用户查询有敏感 operator 的列。
- 可观察现象是 统计槽位存在但 estimator 行为保守。
- 第一源码入口是 `statistic_proc_security_check`，因为这里最早把输入事实压缩成选择率。
- 诊断时要确认该入口拿到的是统计事实、catalog 绑定、表达式形状还是默认假设。
- 可迁移结论是 安全边界可能比统计存在性更早决定 fallback。
- 案例 `默认值被成本放大` 的 SQL 形状是 base scan rows 只偏差几倍但 join plan 大幅变化。
- 可观察现象是 join order 和 join method 变化明显。
- 第一源码入口是 `set_baserel_size_estimates`，因为这里最早把输入事实压缩成选择率。
- 诊断时要确认该入口拿到的是统计事实、catalog 绑定、表达式形状还是默认假设。
- 可迁移结论是 早期 rows 偏差会通过 path search 放大。
- 案例 `自定义 estimator 越界` 的 SQL 形状是 扩展返回非法 selectivity。
- 可观察现象是 planning 阶段直接报错。
- 第一源码入口是 `restriction_selectivity`，因为这里最早把输入事实压缩成选择率。
- 诊断时要确认该入口拿到的是统计事实、catalog 绑定、表达式形状还是默认假设。
- 可迁移结论是 估算可以近似，但 API 返回值必须是合法概率。
- 案例 `统计目标过低` 的 SQL 形状是 偏斜列 MCV 覆盖不足。
- 可观察现象是 默认路径减少但长尾估算仍粗糙。
- 第一源码入口是 `get_variable_numdistinct`，因为这里最早把输入事实压缩成选择率。
- 诊断时要确认该入口拿到的是统计事实、catalog 绑定、表达式形状还是默认假设。
- 可迁移结论是 提高统计目标改善的是可见事实，不是 estimator 的所有假设。
- 案例 `索引选择改变` 的 SQL 形状是 同一过滤条件在 ANALYZE 前后选择不同 scan path。
- 可观察现象是 path 成本排序发生变化。
- 第一源码入口是 `cost_index`，因为这里最早把输入事实压缩成选择率。
- 诊断时要确认该入口拿到的是统计事实、catalog 绑定、表达式形状还是默认假设。
- 可迁移结论是 selectivity 是成本输入，不是 executor 结果。
- 案例 `join 误判归因` 的 SQL 形状是 join node actual rows 偏差很大。
- 可观察现象是 base scan rows 已经偏差。
- 第一源码入口是 `RelOptInfo.rows`，因为这里最早把输入事实压缩成选择率。
- 诊断时要确认该入口拿到的是统计事实、catalog 绑定、表达式形状还是默认假设。
- 可迁移结论是 先修正 base 输入，再判断 join estimator 是否有问题。
- 案例 `默认值不可见` 的 SQL 形状是 EXPLAIN 没有说明某个 DEFAULT 被使用。
- 可观察现象是 只能从源码或断点确认 fallback。
- 第一源码入口是 `selfuncs.h`，因为这里最早把输入事实压缩成选择率。
- 诊断时要确认该入口拿到的是统计事实、catalog 绑定、表达式形状还是默认假设。
- 可迁移结论是 诊断默认值需要源码路径，不要期待所有信息都在 plan 文本中。
- 案例 `catalog 绑定差异` 的 SQL 形状是 两个语义相似 operator 的估算行为不同。
- 可观察现象是 一个走专用 estimator，一个走 `0.5`。
- 第一源码入口是 `get_oprrest`，因为这里最早把输入事实压缩成选择率。
- 诊断时要确认该入口拿到的是统计事实、catalog 绑定、表达式形状还是默认假设。
- 可迁移结论是 operator 语义相似不代表 catalog estimator 相同。
- 案例 `统计过旧` 的 SQL 形状是 数据分布大变后未重新分析。
- 可观察现象是 estimator 使用了真实统计但真实统计已不代表当前表。
- 第一源码入口是 `pg_stats`，因为这里最早把输入事实压缩成选择率。
- 诊断时要确认该入口拿到的是统计事实、catalog 绑定、表达式形状还是默认假设。
- 可迁移结论是 有统计不等于统计新鲜，rows 偏差仍可能来自事实过期。
- 案例 `成本 GUC 误修` 的 SQL 形状是 调整 random_page_cost 后 plan 暂时改变。
- 可观察现象是 estimated rows 仍明显错误。
- 第一源码入口是 `costsize.c`，因为这里最早把输入事实压缩成选择率。
- 诊断时要确认该入口拿到的是统计事实、catalog 绑定、表达式形状还是默认假设。
- 可迁移结论是 成本参数不能替代统计输入和 estimator 边界诊断。

## 18. 源码决策点：逐层确认是否进入 fallback

- 决策点 `clause 是否被包装`：先检查 输入节点是否为 `RestrictInfo` 并是否可缓存。
- 对应源码入口是 `clause_selectivity_ext`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 cache 边界问题。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。
- 决策点 `是否伪常量`：先检查 pseudoconstant 标记是否导致选择率为一。
- 对应源码入口是 `clause_selectivity_ext`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 gating qual 语义问题。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。
- 决策点 `是否布尔变量`：先检查 裸 `Var` 是否被当成 boolean 条件。
- 对应源码入口是 `boolvarsel`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 boolean estimator 问题。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。
- 决策点 `是否常量条件`：先检查 Const 是否直接变成零或一。
- 对应源码入口是 `clause_selectivity_ext`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 常量折叠问题。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。
- 决策点 `Param 是否可估值`：先检查 `estimate_expression_value()` 是否得到常量。
- 对应源码入口是 `clause_selectivity_ext`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 参数不可见问题。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。
- 决策点 `NOT 是否取反`：先检查 内部 clause 估算是否再被一减。
- 对应源码入口是 `clause_selectivity_ext`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 布尔组合问题。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。
- 决策点 `AND 是否进入 list 组合`：先检查 AND 子句是否交给 `clauselist_selectivity_ext()`。
- 对应源码入口是 `clausesel.c`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 多 clause 组合问题。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。
- 决策点 `OR 是否用容斥近似`：先检查 OR 子句是否交给 OR 组合函数。
- 对应源码入口是 `clauselist_selectivity_or`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 OR 重叠估算问题。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。
- 决策点 `operator 是否为 join clause`：先检查 `treat_as_join_clause()` 是否返回 true。
- 对应源码入口是 `clausesel.c`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 join 与 restriction 混淆问题。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。
- 决策点 `restriction estimator 是否存在`：先检查 `get_oprrest()` 是否返回有效过程。
- 对应源码入口是 `plancat.c`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 catalog fallback 问题。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。
- 决策点 `join estimator 是否存在`：先检查 `get_oprjoin()` 是否返回有效过程。
- 对应源码入口是 `plancat.c`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 join catalog fallback 问题。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。
- 决策点 `函数 support 是否存在`：先检查 FuncExpr 是否能通过 support function 得到估算。
- 对应源码入口是 `clause_selectivity_ext`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 函数默认估算问题。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。
- 决策点 `数组条件是否专门处理`：先检查 ScalarArrayOpExpr 是否进入 `scalararraysel()`。
- 对应源码入口是 `selfuncs.c`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 节点级 estimator 问题。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。
- 决策点 `行比较是否专门处理`：先检查 RowCompareExpr 是否进入 `rowcomparesel()`。
- 对应源码入口是 `selfuncs.c`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 复合谓词问题。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。
- 决策点 `NULL 测试是否专门处理`：先检查 NullTest 是否进入 `nulltestsel()`。
- 对应源码入口是 `selfuncs.c`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 NULL fraction 问题。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。
- 决策点 `布尔测试是否专门处理`：先检查 BooleanTest 是否进入 `booltestsel()`。
- 对应源码入口是 `selfuncs.c`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 boolean test 问题。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。
- 决策点 `变量侧是否可识别`：先检查 `get_restriction_variable()` 是否返回 true。
- 对应源码入口是 `selfuncs.c`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 表达式形状 fallback。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。
- 决策点 `统计 tuple 是否有效`：先检查 `VariableStatData.statsTuple` 是否有效。
- 对应源码入口是 `selfuncs.c`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 统计缺失 fallback。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。
- 决策点 `统计释放是否完整`：先检查 所有提前返回路径是否调用释放宏。
- 对应源码入口是 `ReleaseVariableStats`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 ownership 问题。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。
- 决策点 `安全检查是否通过`：先检查 统计 operator 是否通过权限安全检查。
- 对应源码入口是 `statistic_proc_security_check`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 权限 fallback。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。
- 决策点 `distinct 是否默认`：先检查 `get_variable_numdistinct()` 是否标记默认。
- 对应源码入口是 `selfuncs.c`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 默认 distinct 问题。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。
- 决策点 `概率是否越界`：先检查 estimator 返回值是否小于零或大于一。
- 对应源码入口是 `restriction_selectivity`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 非法 estimator 问题。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。
- 决策点 `行数是否被夹紧`：先检查 `clamp_row_est()` 是否改变极端估算。
- 对应源码入口是 `costsize.c`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 行数形状问题。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。
- 决策点 `扩展统计是否接管`：先检查 estimated clauses bitmap 是否标记部分 clause。
- 对应源码入口是 `extended_stats.c`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 重复估算问题。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。
- 决策点 `缓存是否复用错误`：先检查 `varRelid` 是否影响 cache 安全性。
- 对应源码入口是 `RestrictInfo`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 上下文复用问题。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。
- 决策点 `outer join 是否特殊`：先检查 `outer_selec` 是否和 `norm_selec` 分离。
- 对应源码入口是 `pathnodes.h`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 outer join 语义问题。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。
- 决策点 `成本传播是否过早`：先检查 base rel rows 是否已经错误。
- 对应源码入口是 `set_baserel_size_estimates`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 下游归因问题。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。
- 决策点 `统计是否新鲜`：先检查 表数据变化后是否重新 ANALYZE。
- 对应源码入口是 `pg_stats`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 过期统计问题。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。
- 决策点 `统计目标是否足够`：先检查 MCV 和 histogram 是否覆盖偏斜分布。
- 对应源码入口是 `analyze.c`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 统计分辨率问题。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。
- 决策点 `版本基线是否一致`：先检查 本地源码是否为指定 master 提交。
- 对应源码入口是 `git rev-parse`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 版本漂移问题。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。

## 19. 复盘清单：从输入事实到成本传播

- 复盘项：把 rows 偏差先定位到 plan tree 的最底层节点，再决定是否进入 selectivity 源码。。
- 复盘项：把 catalog 层 `0.5` 和 estimator 家族的 `DEFAULT_*` 分开记录。。
- 复盘项：把 `VariableStatData` 的 `statsTuple` 有无作为诊断分水岭。。
- 复盘项：把 `acl_ok` 和安全检查看成统计可用性的组成部分。。
- 复盘项：把表达式形状、operator 绑定、统计新鲜度和成本参数分成四类原因。。
- 复盘项：把 `RelOptInfo.tuples` 与 `RelOptInfo.rows` 分开写在诊断记录里。。
- 复盘项：把 `RestrictInfo` cache 的命中条件写清楚，尤其是 `varRelid` 和 join type。。
- 复盘项：把 NULL 常量、NULL fraction 和 strict operator 假设单独列出。。
- 复盘项：把自定义 operator 的 `oprrest` 和 `oprjoin` 当成 schema 设计的一部分检查。。
- 复盘项：把扩展统计是否已消费 clause 作为多列误差诊断的第一步。。
- 复盘项：把默认值看成系统可运行性的边界，而不是简单错误。。
- 复盘项：把提高统计目标和改写 SQL 形状当成两类不同修复手段。。
- 复盘项：把 planner 阶段的估算成本和 executor 阶段的运行成本分开讨论。。
- 复盘项：把 `EXPLAIN` 看不到的默认路径通过源码断点或 catalog 查询补齐。。
- 复盘项：把所有结论绑定到本课指定的源码提交，避免把旧版本行为混进来。。

## 20. 诊断记录模板：selectivity API fallback 记录

- 记录项 `base rows 偏差` 的现象字段写作：最底层 scan 的 estimated rows 与 actual rows 的倍数差异。
- 记录项 `base rows 偏差` 的源码字段写作：先从 `set_baserel_size_estimates` 进入，不直接跳到最终 plan。
- 记录项 `base rows 偏差` 的边界字段写作：明确这是统计事实、默认假设、catalog 绑定还是成本传播。
- 记录项 `base rows 偏差` 的结论字段写作：先判断 base relation 输入是否已经错误，再讨论 join path。
- 记录项 `catalog 绑定` 的现象字段写作：operator 名称和 OID 对应的 `oprrest` 或 `oprjoin` 是否存在。
- 记录项 `catalog 绑定` 的源码字段写作：先从 `get_oprrest` 进入，不直接跳到最终 plan。
- 记录项 `catalog 绑定` 的边界字段写作：明确这是统计事实、默认假设、catalog 绑定还是成本传播。
- 记录项 `catalog 绑定` 的结论字段写作：缺绑定时按 catalog fallback 解释，不按统计缺失解释。
- 记录项 `表达式形状` 的现象字段写作：predicate 是否仍能看成变量与另一侧表达式的比较。
- 记录项 `表达式形状` 的源码字段写作：先从 `get_restriction_variable` 进入，不直接跳到最终 plan。
- 记录项 `表达式形状` 的边界字段写作：明确这是统计事实、默认假设、catalog 绑定还是成本传播。
- 记录项 `表达式形状` 的结论字段写作：形状失败时默认值来自识别边界，而不是统计目标。
- 记录项 `统计 tuple` 的现象字段写作：`pg_stats` 是否有数据以及源码中的 `statsTuple` 是否有效。
- 记录项 `统计 tuple` 的源码字段写作：先从 `examine_variable` 进入，不直接跳到最终 plan。
- 记录项 `统计 tuple` 的边界字段写作：明确这是统计事实、默认假设、catalog 绑定还是成本传播。
- 记录项 `统计 tuple` 的结论字段写作：统计不存在和统计存在但未使用要分开记录。
- 记录项 `默认常量` 的现象字段写作：最终估算是否落入 `DEFAULT_EQ_SEL`、`DEFAULT_INEQ_SEL` 或 `DEFAULT_MATCH_SEL`。
- 记录项 `默认常量` 的源码字段写作：先从 `selfuncs.h` 进入，不直接跳到最终 plan。
- 记录项 `默认常量` 的边界字段写作：明确这是统计事实、默认假设、catalog 绑定还是成本传播。
- 记录项 `默认常量` 的结论字段写作：家族默认值说明 estimator 已进入但事实不足。
- 记录项 `cache 上下文` 的现象字段写作：同一 clause 是否在不同 varRelid 或 join type 下被重复询问。
- 记录项 `cache 上下文` 的源码字段写作：先从 `clause_selectivity_ext` 进入，不直接跳到最终 plan。
- 记录项 `cache 上下文` 的边界字段写作：明确这是统计事实、默认假设、catalog 绑定还是成本传播。
- 记录项 `cache 上下文` 的结论字段写作：缓存命中必须带着上下文解释。
- 记录项 `outer join 语义` 的现象字段写作：outer join qual 的 rows 偏差是否只在特定 join type 下出现。
- 记录项 `outer join 语义` 的源码字段写作：先从 `RestrictInfo` 进入，不直接跳到最终 plan。
- 记录项 `outer join 语义` 的边界字段写作：明确这是统计事实、默认假设、catalog 绑定还是成本传播。
- 记录项 `outer join 语义` 的结论字段写作：外连接选择率不能和 inner 语义混用。
- 记录项 `扩展统计接管` 的现象字段写作：多个 clause 中哪些已经被 extended statistics 消费。
- 记录项 `扩展统计接管` 的源码字段写作：先从 `statext_clauselist_selectivity` 进入，不直接跳到最终 plan。
- 记录项 `扩展统计接管` 的边界字段写作：明确这是统计事实、默认假设、catalog 绑定还是成本传播。
- 记录项 `扩展统计接管` 的结论字段写作：已估算 clause 不能在普通循环中重复乘。
- 记录项 `权限安全` 的现象字段写作：当前用户是否可能被安全检查阻止消费统计槽位。
- 记录项 `权限安全` 的源码字段写作：先从 `statistic_proc_security_check` 进入，不直接跳到最终 plan。
- 记录项 `权限安全` 的边界字段写作：明确这是统计事实、默认假设、catalog 绑定还是成本传播。
- 记录项 `权限安全` 的结论字段写作：安全 fallback 是正确性边界，不是统计损坏。
- 记录项 `非法估算` 的现象字段写作：自定义 estimator 是否返回越界概率并触发 planning error。
- 记录项 `非法估算` 的源码字段写作：先从 `restriction_selectivity` 进入，不直接跳到最终 plan。
- 记录项 `非法估算` 的边界字段写作：明确这是统计事实、默认假设、catalog 绑定还是成本传播。
- 记录项 `非法估算` 的结论字段写作：合法概率是 API 契约，近似不等于任意返回。
- 记录项 `成本传播` 的现象字段写作：rows 偏差是否改变 scan path、join order 或 join method。
- 记录项 `成本传播` 的源码字段写作：先从 `costsize.c` 进入，不直接跳到最终 plan。
- 记录项 `成本传播` 的边界字段写作：明确这是统计事实、默认假设、catalog 绑定还是成本传播。
- 记录项 `成本传播` 的结论字段写作：selectivity 的最终影响要落到成本排序上。
- 记录项 `修复验证` 的现象字段写作：重新 ANALYZE、改写表达式或补 estimator 后 estimated rows 是否变化。
- 记录项 `修复验证` 的源码字段写作：先从 `EXPLAIN` 进入，不直接跳到最终 plan。
- 记录项 `修复验证` 的边界字段写作：明确这是统计事实、默认假设、catalog 绑定还是成本传播。
- 记录项 `修复验证` 的结论字段写作：修复动作必须能改变对应源码边界才算有效。
