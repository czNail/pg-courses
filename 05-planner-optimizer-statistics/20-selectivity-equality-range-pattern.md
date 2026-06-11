# PostgreSQL 等值、范围与模式匹配 Selectivity

## 课程定位

前置知识：已经理解 selectivity API 的入口、默认值边界和 `VariableStatData` 的短生命周期 ownership。
还应知道 `pg_statistic` 中 MCV、histogram、null fraction 和 ndistinct 来自 `ANALYZE` 采样。
本节唯一主问题是：为什么等值、范围和 LIKE 或 regex 都是布尔条件，却必须使用不同统计槽位和不同 fallback。
本节围绕的核心矛盾是：planner 需要把 predicate 统一成概率，但不同谓词对数据分布提出的问题完全不同。
学完后应能判断：等值、范围、区间和文本模式匹配的 rows 偏差分别该回到哪个 estimator 和哪个统计槽。
本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

上一节讲的是 selectivity API 如何找到 estimator，以及缺信息时如何返回默认值。
本节进入最常见的三个 estimator 家族。
等值条件问的是某个具体值是否常见。
范围条件问的是常量在有序分布中的位置。
模式匹配问的是 pattern 是否有固定前缀，以及剩余模式能过滤多少值。
这三个问题不能用同一个公式处理。
如果等值条件只看 histogram，就会错过 MCV 中的热点值。
如果范围条件只看 ndistinct，就不知道常量落在分布左侧还是右侧。
如果 LIKE 条件只用固定默认值，就会忽略带固定前缀的强约束。
PostgreSQL 的做法是让每个 operator 家族登记自己的 `oprrest`。
`eqsel()` 主要消费 MCV、ndistinct 和 null fraction。
`scalarineqsel()` 主要消费 MCV、histogram 和可标量化的常量位置。
`patternsel_common()` 主要消费 fixed prefix、histogram、MCV 和模式启发式。
三者都共享 `VariableStatData`、权限检查、NULL 处理和 `CLAMP_PROBABILITY()`。
三者也都在统计缺失或表达式形状不合适时回到默认值。
本节不讲多个 clause 如何组合。
例如两个范围边界的配对在下一节的 `clausesel.c` 主线中处理。
本节只讲单个 predicate 如何形成一个 selectivity。
理解这条线后，看到 rows 偏差时才能判断是单个 estimator 误差，还是多个 estimator 组合后的误差。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：`restriction_selectivity()` 通过 `pg_operator.oprrest` 调到 `eqsel()`、`scalarltsel()` 或 `likesel()` 等 estimator；等值先查 MCV 再估长尾，范围先计 MCV 再用 histogram 插值，模式匹配先抽 fixed prefix 再混合 histogram、MCV 和启发式，失败时各自回到 `DEFAULT_*`。
等值 estimator 的核心假设是：如果常量出现在 MCV 中，MCV 频率就是最可靠的估算。
等值 estimator 的 fallback 是：如果常量不在 MCV 中，就把非 NULL、非 MCV 的剩余频率分给剩余 distinct 值。
范围 estimator 的核心假设是：MCV 可以直接判断，非 MCV 部分可由 histogram 近似代表。
范围 estimator 的 fallback 是：没有统计或无法转换标量位置时，使用 `DEFAULT_INEQ_SEL` 或 bin 中点近似。
模式匹配 estimator 的核心假设是：exact prefix 可转成等值，partial prefix 可转成范围，剩余 pattern 用启发式。
模式匹配 estimator 的 fallback 是：无法识别 variable 与常量、类型不支持或 pattern 信息不足时，回到 `DEFAULT_MATCH_SEL`。
三个 estimator 共同体现一个规律。
PostgreSQL 不追求任意 predicate 的精确概率。
它在收益高、识别便宜、语义明确的位置使用统计事实。
它在收益低或信息不足的位置使用稳定默认值。
这就是 planner selectivity 的工程性。
它的数字足够支撑 path 比较，但永远只是近似。

## 3. 核心文件分工与阅读顺序

- 第一步读 `src/backend/utils/adt/selfuncs.c` 的 `eqsel()`、`eqsel_internal()`、`var_eq_const()` 和 `var_eq_non_const()`，先建立等值估算主线。
- 第二步继续读 `src/backend/utils/adt/selfuncs.c` 的 `scalarineqsel_wrapper()`、`scalarineqsel()`、`mcv_selectivity()` 和 `ineq_histogram_selectivity()`，建立范围估算主线。
- 第三步读 `src/backend/utils/adt/like_support.c` 的 `patternsel_common()`、`pattern_fixed_prefix()`、`prefix_selectivity()` 和 `like_selectivity()`，建立模式匹配估算主线。
- 第四步读 `src/include/utils/selfuncs.h`，确认 `DEFAULT_EQ_SEL`、`DEFAULT_INEQ_SEL`、`DEFAULT_MATCH_SEL` 和相关 helper 的声明。
- 第五步读 `src/backend/optimizer/util/plancat.c`，确认这些 estimator 是通过 `restriction_selectivity()` 调进来的。
- 第六步读 `src/include/catalog/pg_operator.dat`，确认内置 equality、inequality、LIKE 和 regex operator 与 estimator 的绑定。
- 第七步读 `src/backend/optimizer/path/clausesel.c`，只看单 clause 分派和 range clause pairing 的边界，不要提前展开多 clause 组合。
- 第八步读 `src/backend/commands/analyze.c`，回忆 MCV 和 histogram 来自样本统计，不是 executor 运行时生成。
- 第九步读 `src/test/regress/sql/stats_ext.sql`，借用实验风格观察 estimated rows 随统计变化而变化。
- 推荐阅读顺序是先等值，再范围，最后模式匹配。
- 等值路径最容易看清 MCV 与 ndistinct 的关系。
- 范围路径最容易看清 histogram 插值和标量转换。
- 模式路径最容易看清 prefix、histogram、MCV 和启发式如何混合。
- 不要先背所有 selectivity 函数名。
- 要从一个 SQL predicate 出发，追踪它最终调用哪个 estimator。

## 4. 等值 selectivity：先找 MCV，再估长尾

- Walkthrough 第 1 步：进入等值 wrapper。
- 这一步的源码动作是 `eqsel()` 调用 `eqsel_internal()` 并标记这是正向等值估算。
- 这一步改变或确认的 planner 状态是 operator OID、args、varRelid 和 collation 被传入。
- 这一步的边界条件是 wrapper 本身不读取统计槽位。
- Walkthrough 第 2 步：处理不等取反。
- 这一步的源码动作是 `neqsel()` 也进入 `eqsel_internal()`，但先寻找 negator。
- 这一步改变或确认的 planner 状态是 不等估算被转成等值估算再取反。
- 这一步的边界条件是 找不到 negator 时返回 `1.0 - DEFAULT_EQ_SEL`。
- Walkthrough 第 3 步：识别变量侧。
- 这一步的源码动作是 `get_restriction_variable()` 尝试把参数拆成 variable 与另一侧。
- 这一步改变或确认的 planner 状态是 `VariableStatData` 被填充。
- 这一步的边界条件是 拆解失败时返回等值默认值。
- Walkthrough 第 4 步：区分 Const 与非 Const。
- 这一步的源码动作是 `eqsel_internal()` 根据另一侧节点类型选择 `var_eq_const()` 或 `var_eq_non_const()`。
- 这一步改变或确认的 planner 状态是 常量路径与未知值路径分离。
- 这一步的边界条件是 Param 或表达式通常不能命中具体 MCV 值。
- Walkthrough 第 5 步：处理 NULL 常量。
- 这一步的源码动作是 `var_eq_const()` 遇到 NULL 常量时返回零。
- 这一步改变或确认的 planner 状态是 严格 operator 假设生效。
- 这一步的边界条件是 这一步发生在读取 MCV 之前。
- Walkthrough 第 6 步：读取 null fraction。
- 这一步的源码动作是 `var_eq_const()` 从 stats tuple 读取 `stanullfrac`。
- 这一步改变或确认的 planner 状态是 NULL population 被单独扣除。
- 这一步的边界条件是 无统计时 null fraction 默认为零。
- Walkthrough 第 7 步：利用唯一性。
- 这一步的源码动作是 `vardata->isunique` 为真且 relation tuples 有效时估为一行比例。
- 这一步改变或确认的 planner 状态是 唯一性信息短路普通 MCV 估算。
- 这一步的边界条件是 operator 语义差异仍是源码注释承认的近似。
- Walkthrough 第 8 步：查找 MCV 命中。
- 这一步的源码动作是 函数读取 `STATISTIC_KIND_MCV` 并用 equality procedure 逐项比较。
- 这一步改变或确认的 planner 状态是 命中时直接使用 MCV frequency。
- 这一步的边界条件是 安全检查失败时不能消费该槽位。
- Walkthrough 第 9 步：估算长尾。
- 这一步的源码动作是 未命中 MCV 时计算非 NULL、非 MCV 剩余比例并除以剩余 distinct。
- 这一步改变或确认的 planner 状态是 长尾平均假设生效。
- 这一步的边界条件是 结果不能高于最小 MCV 频率。
- Walkthrough 第 10 步：处理未知值。
- 这一步的源码动作是 `var_eq_non_const()` 用非 NULL 比例除以 distinct 数。
- 这一步改变或确认的 planner 状态是 未知比较值被当成平均值。
- 这一步的边界条件是 这条路径不能利用具体常量的 MCV 命中。
- Walkthrough 第 11 步：取反修正。
- 这一步的源码动作是 不等估算用一减等值选择率再扣除 null fraction。
- 这一步改变或确认的 planner 状态是 NULL 不属于普通不等匹配。
- 这一步的边界条件是 不要把不等简单理解成一减等值。
- Walkthrough 第 12 步：释放统计状态。
- 这一步的源码动作是 `eqsel_internal()` 调用 `ReleaseVariableStats()`。
- 这一步改变或确认的 planner 状态是 统计 tuple 的短生命周期结束。
- 这一步的边界条件是 提前返回路径也要保持 ownership 正确。

## 5. 范围 selectivity：MCV 直接计数，histogram 负责位置

- Walkthrough 第 1 步：进入范围 wrapper。
- 这一步的源码动作是 `scalarltsel()`、`scalarlesel()`、`scalargtsel()` 和 `scalargesel()` 统一调用 `scalarineqsel_wrapper()`。
- 这一步改变或确认的 planner 状态是 operator 方向被转成 `isgt` 与 `iseq` 标记。
- 这一步的边界条件是 wrapper 只整理输入形状。
- Walkthrough 第 2 步：识别变量与常量。
- 这一步的源码动作是 `scalarineqsel_wrapper()` 调用 `get_restriction_variable()` 并要求另一侧是 Const。
- 这一步改变或确认的 planner 状态是 `VariableStatData` 和常量类型被确认。
- 这一步的边界条件是 失败时返回 `DEFAULT_INEQ_SEL`。
- Walkthrough 第 3 步：处理 NULL 常量。
- 这一步的源码动作是 常量为 NULL 时释放统计状态并返回零。
- 这一步改变或确认的 planner 状态是 strict operator 假设生效。
- 这一步的边界条件是 这一步不读取 histogram。
- Walkthrough 第 4 步：强制变量在左侧。
- 这一步的源码动作是 变量不在左侧时查找 commutator 并翻转方向。
- 这一步改变或确认的 planner 状态是 后续核心函数只处理 var op const。
- 这一步的边界条件是 缺 commutator 时返回默认范围估算。
- Walkthrough 第 5 步：进入核心估算。
- 这一步的源码动作是 `scalarineqsel()` 接收整理后的 operator、方向、常量和统计状态。
- 这一步改变或确认的 planner 状态是 范围估算开始消费统计槽位。
- 这一步的边界条件是 无统计时普通列返回默认范围估算。
- Walkthrough 第 6 步：处理 CTID 特例。
- 这一步的源码动作是 没有统计时 CTID 可以根据 relation pages 和 item pointer 粗估位置。
- 这一步改变或确认的 planner 状态是 表大小提供特殊位置估算。
- 这一步的边界条件是 普通列没有这条路径。
- Walkthrough 第 7 步：计算 MCV 贡献。
- 这一步的源码动作是 `mcv_selectivity()` 对每个 MCV 执行范围 operator。
- 这一步改变或确认的 planner 状态是 满足条件的 MCV 频率进入 `mcv_selec`。
- 这一步的边界条件是 所有 MCV 频率进入 `sumcommon`。
- Walkthrough 第 8 步：计算 histogram 贡献。
- 这一步的源码动作是 `ineq_histogram_selectivity()` 判断常量落在哪个 histogram bin。
- 这一步改变或确认的 planner 状态是 非 MCV population 的位置比例被估算。
- 这一步的边界条件是 histogram 不存在时返回负值。
- Walkthrough 第 9 步：执行标量转换。
- 这一步的源码动作是 `convert_to_scalar()` 把常量和 bin 边界映射到可比较尺度。
- 这一步改变或确认的 planner 状态是 bin 内线性插值成为可能。
- 这一步的边界条件是 失败时 bin fraction 退化为中点。
- Walkthrough 第 10 步：合并 population。
- 这一步的源码动作是 `scalarineqsel()` 先取 `1 - stanullfrac - sumcommon` 作为剩余 population。
- 这一步改变或确认的 planner 状态是 histogram 只代表剩余 population。
- 这一步的边界条件是 MCV 与 histogram 不能重复计数。
- Walkthrough 第 11 步：缺 histogram 退化。
- 这一步的源码动作是 如果 histogram 无效但仍有剩余 population，源码任意假设一半满足。
- 这一步改变或确认的 planner 状态是 范围估算保持可运行。
- 这一步的边界条件是 这是一种明确启发式。
- Walkthrough 第 12 步：夹紧结果。
- 这一步的源码动作是 最终结果经过 `CLAMP_PROBABILITY()`。
- 这一步改变或确认的 planner 状态是 返回合法概率。
- 这一步的边界条件是 近似估算不能越过 API 边界。

## 6. 模式匹配 selectivity：prefix 决定能否转成更强约束

- Walkthrough 第 1 步：进入 pattern wrapper。
- 这一步的源码动作是 `likesel()`、`regexeqsel()`、`iclikesel()` 和 `prefixsel()` 通过 `patternsel()` 进入公共逻辑。
- 这一步改变或确认的 planner 状态是 pattern type 和 negate 标记被传入。
- 这一步的边界条件是 join 版本当前多为默认估算。
- Walkthrough 第 2 步：初始化默认结果。
- 这一步的源码动作是 `patternsel_common()` 根据正向或反向匹配初始化为 `DEFAULT_MATCH_SEL` 或取反。
- 这一步改变或确认的 planner 状态是 默认匹配概率先被准备好。
- 这一步的边界条件是 后续任何识别失败都会返回它。
- Walkthrough 第 3 步：识别变量与常量。
- 这一步的源码动作是 函数要求形如 variable op constant 且 variable 在左侧。
- 这一步改变或确认的 planner 状态是 `VariableStatData` 和 pattern const 被确认。
- 这一步的边界条件是 失败时返回默认模式估算。
- Walkthrough 第 4 步：检查 NULL 与类型。
- 这一步的源码动作是 NULL pattern 返回零，pattern 类型必须是 text 或 bytea。
- 这一步改变或确认的 planner 状态是 unsupported type 直接 fallback。
- 这一步的边界条件是 类型边界保护后续 operator 选择。
- Walkthrough 第 5 步：选择比较 operator。
- 这一步的源码动作是 根据左侧 exposed type 选择 equality、less-than 和 greater-equal operator。
- 这一步改变或确认的 planner 状态是 prefix 范围估算获得比较工具。
- 这一步的边界条件是 未知类型不继续猜测。
- Walkthrough 第 6 步：读取 null fraction。
- 这一步的源码动作是 有 stats tuple 时读取 `stanullfrac`。
- 这一步改变或确认的 planner 状态是 NULL population 后续从匹配 population 中扣除。
- 这一步的边界条件是 无统计时 null fraction 默认为零。
- Walkthrough 第 7 步：提取固定前缀。
- 这一步的源码动作是 `pattern_fixed_prefix()` 返回 exact、partial 或 none 状态。
- 这一步改变或确认的 planner 状态是 pattern 语义被压缩成 prefix 状态和剩余选择率。
- 这一步的边界条件是 没有 prefix 时只能依赖 histogram 或启发式。
- Walkthrough 第 8 步：处理 exact pattern。
- 这一步的源码动作是 exact prefix 直接调用 `var_eq_const()`。
- 这一步改变或确认的 planner 状态是 模式匹配退化为等值估算。
- 这一步的边界条件是 这条路径可以命中 MCV。
- Walkthrough 第 9 步：处理 partial prefix。
- 这一步的源码动作是 `prefix_selectivity()` 把 prefix 转成范围选择率。
- 这一步改变或确认的 planner 状态是 固定前缀贡献与剩余 pattern 贡献相乘。
- 这一步的边界条件是 无法构造上界时 prefix 收益受限。
- Walkthrough 第 10 步：读取 histogram。
- 这一步的源码动作是 `histogram_selectivity()` 可以逐个测试 histogram entry 是否匹配 pattern。
- 这一步改变或确认的 planner 状态是 样本分布直接参与模式估算。
- 这一步的边界条件是 小 histogram 会和启发式混合。
- Walkthrough 第 11 步：读取 MCV。
- 这一步的源码动作是 `mcv_selectivity()` 对 MCV 字符串逐项执行 pattern operator。
- 这一步改变或确认的 planner 状态是 热点字符串频率被直接加回。
- 这一步的边界条件是 MCV 与 histogram population 仍然分离。
- Walkthrough 第 12 步：释放 prefix 与统计。
- 这一步的源码动作是 如果 prefix 被分配，函数释放 prefix datum 和 node，再释放 `VariableStatData`。
- 这一步改变或确认的 planner 状态是 短生命周期对象不泄漏到 planner context。
- 这一步的边界条件是 模式估算也遵守 ownership 边界。

## 7. 核心源码锚点与状态阅读

- 源码锚点 `eqsel_internal` 位于 `src/backend/utils/adt/selfuncs.c`，本节阅读它是为了确认 等值 estimator 如何识别 variable 与另一侧表达式。
- 进入 `eqsel_internal` 前先写下输入状态：`PlannerInfo`、operator OID、args、varRelid 和 collation。
- 离开 `eqsel_internal` 后再写下输出状态：等值或不等选择率。
- 如果 `eqsel_internal` 不能获得足够信息，它的 fallback 是 无法识别时返回 `DEFAULT_EQ_SEL`。
- 诊断 `eqsel_internal` 时不要只看最终 plan，要先观察 predicate 是否形如 variable equals something。
- 复盘 `eqsel_internal` 时要说明它返回的是概率、行数、缓存值还是错误。
- 源码锚点 `var_eq_const` 位于 `src/backend/utils/adt/selfuncs.c`，本节阅读它是为了确认 常量等值如何使用 NULL、unique、MCV 和 ndistinct。
- 进入 `var_eq_const` 前先写下输入状态：`VariableStatData`、const datum、operator function 和 stats tuple。
- 离开 `var_eq_const` 后再写下输出状态：常量等值选择率。
- 如果 `var_eq_const` 不能获得足够信息，它的 fallback 是 无统计时使用 distinct 估算。
- 诊断 `var_eq_const` 时不要只看最终 plan，要先观察 常量是否命中 MCV。
- 复盘 `var_eq_const` 时要说明它返回的是概率、行数、缓存值还是错误。
- 源码锚点 `var_eq_non_const` 位于 `src/backend/utils/adt/selfuncs.c`，本节阅读它是为了确认 未知比较值如何做平均等值估算。
- 进入 `var_eq_non_const` 前先写下输入状态：`VariableStatData`、另一侧表达式和 null fraction。
- 离开 `var_eq_non_const` 后再写下输出状态：非 Const 等值选择率。
- 如果 `var_eq_non_const` 不能获得足够信息，它的 fallback 是 无统计时使用默认 distinct 估算。
- 诊断 `var_eq_non_const` 时不要只看最终 plan，要先观察 另一侧是否是 Param 或非 Const 表达式。
- 复盘 `var_eq_non_const` 时要说明它返回的是概率、行数、缓存值还是错误。
- 源码锚点 `scalarineqsel_wrapper` 位于 `src/backend/utils/adt/selfuncs.c`，本节阅读它是为了确认 范围 wrapper 如何把表达式整理成 var op const。
- 进入 `scalarineqsel_wrapper` 前先写下输入状态：operator 方向、commutator、constant type 和 stats state。
- 离开 `scalarineqsel_wrapper` 后再写下输出状态：整理后的范围估算调用。
- 如果 `scalarineqsel_wrapper` 不能获得足够信息，它的 fallback 是 无法整理时返回 `DEFAULT_INEQ_SEL`。
- 诊断 `scalarineqsel_wrapper` 时不要只看最终 plan，要先观察 变量是否在左侧以及 commutator 是否存在。
- 复盘 `scalarineqsel_wrapper` 时要说明它返回的是概率、行数、缓存值还是错误。
- 源码锚点 `scalarineqsel` 位于 `src/backend/utils/adt/selfuncs.c`，本节阅读它是为了确认 范围核心如何合并 MCV 与 histogram。
- 进入 `scalarineqsel` 前先写下输入状态：MCV population、histogram population、null fraction 和 operator procedure。
- 离开 `scalarineqsel` 后再写下输出状态：范围条件选择率。
- 如果 `scalarineqsel` 不能获得足够信息，它的 fallback 是 无统计时返回 `DEFAULT_INEQ_SEL` 或 CTID 特殊估算。
- 诊断 `scalarineqsel` 时不要只看最终 plan，要先观察 histogram 是否存在且可解释。
- 复盘 `scalarineqsel` 时要说明它返回的是概率、行数、缓存值还是错误。
- 源码锚点 `ineq_histogram_selectivity` 位于 `src/backend/utils/adt/selfuncs.c`，本节阅读它是为了确认 常量如何落入 histogram bin 并插值。
- 进入 `ineq_histogram_selectivity` 前先写下输入状态：histogram bounds、operator、collation 和 scalar conversion。
- 离开 `ineq_histogram_selectivity` 后再写下输出状态：非 MCV population 的范围比例。
- 如果 `ineq_histogram_selectivity` 不能获得足够信息，它的 fallback 是 无法 scalar conversion 时使用 bin 中点。
- 诊断 `ineq_histogram_selectivity` 时不要只看最终 plan，要先观察 常量落在哪个 histogram 区间。
- 复盘 `ineq_histogram_selectivity` 时要说明它返回的是概率、行数、缓存值还是错误。
- 源码锚点 `patternsel_common` 位于 `src/backend/utils/adt/like_support.c`，本节阅读它是为了确认 模式匹配如何抽 prefix 并混合统计与启发式。
- 进入 `patternsel_common` 前先写下输入状态：pattern constant、vartype、null fraction、histogram 和 MCV。
- 离开 `patternsel_common` 后再写下输出状态：模式匹配选择率。
- 如果 `patternsel_common` 不能获得足够信息，它的 fallback 是 无法识别形状或类型时返回 `DEFAULT_MATCH_SEL`。
- 诊断 `patternsel_common` 时不要只看最终 plan，要先观察 prefix status 是 exact、partial 还是 none。
- 复盘 `patternsel_common` 时要说明它返回的是概率、行数、缓存值还是错误。
- 源码锚点 `prefix_selectivity` 位于 `src/backend/utils/adt/like_support.c`，本节阅读它是为了确认 partial prefix 如何转成范围选择率。
- 进入 `prefix_selectivity` 前先写下输入状态：等值、less-than、greater-equal operator 和 prefix constant。
- 离开 `prefix_selectivity` 后再写下输出状态：固定前缀选择率。
- 如果 `prefix_selectivity` 不能获得足够信息，它的 fallback 是 无有效 prefix 时不提供额外约束。
- 诊断 `prefix_selectivity` 时不要只看最终 plan，要先观察 prefix 能否形成上界字符串。
- 复盘 `prefix_selectivity` 时要说明它返回的是概率、行数、缓存值还是错误。
- 源码锚点 `like_selectivity` 位于 `src/backend/utils/adt/like_support.c`，本节阅读它是为了确认 LIKE 剩余 pattern 如何用字符启发式估算。
- 进入 `like_selectivity` 前先写下输入状态：pattern 字符、通配符和 case-insensitive 标记。
- 离开 `like_selectivity` 后再写下输出状态：剩余模式选择率。
- 如果 `like_selectivity` 不能获得足够信息，它的 fallback 是 复杂模式只给启发式过滤能力。
- 诊断 `like_selectivity` 时不要只看最终 plan，要先观察 剩余 pattern 是否主要由通配符组成。
- 复盘 `like_selectivity` 时要说明它返回的是概率、行数、缓存值还是错误。
- 源码锚点 `mcv_selectivity` 位于 `src/backend/utils/adt/selfuncs.c`，本节阅读它是为了确认 MCV population 如何被 operator 直接测试。
- 进入 `mcv_selectivity` 前先写下输入状态：统计槽位、operator function、collation 和 const datum。
- 离开 `mcv_selectivity` 后再写下输出状态：MCV 命中频率与 MCV 总频率。
- 如果 `mcv_selectivity` 不能获得足够信息，它的 fallback 是 无 MCV 时返回零并让 histogram 或 default 处理剩余。
- 诊断 `mcv_selectivity` 时不要只看最终 plan，要先观察 MCV 覆盖了多少总频率。
- 复盘 `mcv_selectivity` 时要说明它返回的是概率、行数、缓存值还是错误。

## 8. 生命周期、ownership 与统计槽位释放

三个 estimator 家族都依赖 `VariableStatData`。
`get_restriction_variable()` 或 `examine_variable()` 填充它。
调用者必须在返回前执行 `ReleaseVariableStats()`。
等值路径中，`eqsel_internal()` 在调用 `var_eq_const()` 或 `var_eq_non_const()` 后统一释放。
范围路径中，`scalarineqsel_wrapper()` 在所有提前返回前都要释放已经取得的统计状态。
模式路径中，`patternsel_common()` 在返回前释放 `VariableStatData`。
模式路径还可能分配 prefix `Const`。
如果 prefix 存在，函数会释放 prefix 的 datum 和 node。
MCV 和 histogram 槽位通过 `get_attstatsslot()` 取得。
读取完成后必须调用 `free_attstatsslot()`。
这些释放动作说明 selectivity 估算是短生命周期工作区。
它不把统计槽位长期挂到 planner structure 上。
它只把最终概率写回 `RestrictInfo` cache 或直接返回给上层。
一个常见扩展 bug 是在自定义 estimator 中提前返回但忘记释放 stats slot。
另一个常见 bug 是保存 `AttStatsSlot` 中的 Datum 指针到更长生命周期对象。
本节要建立的 ownership 规则是：统计槽位只服务当前 estimator 调用。
如果需要长期数据，必须复制到合适的 planner context，并明确生命周期。
但 PostgreSQL 内置 estimator 一般不需要长期保存这些统计值。
它们只需要向上层返回一个合法概率。

## 9. 正确性、fallback 与估算边界

- Fallback 场景 `等值变量识别失败` 的触发条件是 `get_restriction_variable()` 无法拆出变量侧。
- 该场景交给后续 planner 的估算含义是 返回 `DEFAULT_EQ_SEL` 或其取反。
- 该场景不改变 SQL 正确性，但会改变 path search 的成本排序。
- 排查该场景时优先尝试 改写表达式或创建可用表达式统计。
- Fallback 场景 `等值 NULL 常量` 的触发条件是 右侧常量为 NULL。
- 该场景交给后续 planner 的估算含义是 返回零并跳过 MCV。
- 该场景不改变 SQL 正确性，但会改变 path search 的成本排序。
- 排查该场景时优先尝试 确认 SQL 是否应使用 IS NULL 语义。
- Fallback 场景 `等值无统计` 的触发条件是 `statsTuple` 无效。
- 该场景交给后续 planner 的估算含义是 通过默认 distinct 或 `get_variable_numdistinct()` 估算。
- 该场景不改变 SQL 正确性，但会改变 path search 的成本排序。
- 排查该场景时优先尝试 运行 `ANALYZE` 并检查统计目标。
- Fallback 场景 `等值未知值` 的触发条件是 另一侧不是 Const。
- 该场景交给后续 planner 的估算含义是 使用非 NULL 比例除以 distinct 数。
- 该场景不改变 SQL 正确性，但会改变 path search 的成本排序。
- 排查该场景时优先尝试 确认 Param 或表达式是否能常量折叠。
- Fallback 场景 `范围变量识别失败` 的触发条件是 范围 wrapper 无法拆成 variable 与另一侧。
- 该场景交给后续 planner 的估算含义是 返回 `DEFAULT_INEQ_SEL`。
- 该场景不改变 SQL 正确性，但会改变 path search 的成本排序。
- 排查该场景时优先尝试 改写 predicate 让变量和常量可识别。
- Fallback 场景 `范围非 Const` 的触发条件是 范围比较另一侧不是 Const。
- 该场景交给后续 planner 的估算含义是 返回 `DEFAULT_INEQ_SEL`。
- 该场景不改变 SQL 正确性，但会改变 path search 的成本排序。
- 排查该场景时优先尝试 检查 prepared statement 和表达式折叠时机。
- Fallback 场景 `范围缺 commutator` 的触发条件是 变量在右侧且 operator 没有 commutator。
- 该场景交给后续 planner 的估算含义是 返回 `DEFAULT_INEQ_SEL`。
- 该场景不改变 SQL 正确性，但会改变 path search 的成本排序。
- 排查该场景时优先尝试 为 operator 定义 commutator 或调整 SQL 方向。
- Fallback 场景 `范围无 histogram` 的触发条件是 统计 tuple 存在但 histogram 无效。
- 该场景交给后续 planner 的估算含义是 对剩余 population 任意假设一半满足。
- 该场景不改变 SQL 正确性，但会改变 path search 的成本排序。
- 排查该场景时优先尝试 提高统计目标或检查类型统计支持。
- Fallback 场景 `范围 scalar conversion 失败` 的触发条件是 `convert_to_scalar()` 不能映射值和边界。
- 该场景交给后续 planner 的估算含义是 bin 内位置退化为中点。
- 该场景不改变 SQL 正确性，但会改变 path search 的成本排序。
- 排查该场景时优先尝试 为类型提供合适比较和统计支持。
- Fallback 场景 `pattern 形状失败` 的触发条件是 模式匹配不是 variable op constant。
- 该场景交给后续 planner 的估算含义是 返回 `DEFAULT_MATCH_SEL`。
- 该场景不改变 SQL 正确性，但会改变 path search 的成本排序。
- 排查该场景时优先尝试 让 pattern 变成可见常量或使用表达式统计。
- Fallback 场景 `pattern 类型失败` 的触发条件是 pattern 或变量 exposed type 不在支持集合内。
- 该场景交给后续 planner 的估算含义是 返回默认模式估算。
- 该场景不改变 SQL 正确性，但会改变 path search 的成本排序。
- 排查该场景时优先尝试 确认 operator 是否适合 pattern estimator。
- Fallback 场景 `pattern 无 prefix` 的触发条件是 leading wildcard 或复杂 regex 无法提取固定前缀。
- 该场景交给后续 planner 的估算含义是 主要依赖 histogram、MCV 或启发式。
- 该场景不改变 SQL 正确性，但会改变 path search 的成本排序。
- 排查该场景时优先尝试 调整查询形状或建立适配的索引策略。

## 10. 成本传播：一个 predicate 的误差如何放大

等值条件估小，会让 base relation rows 偏低。
base rows 偏低，会让 nested loop 的内层重复访问看起来更便宜。
如果实际 rows 很高，执行时可能出现大量随机访问。
等值条件估大，会让高选择性 index scan 看起来不划算。
planner 可能选择 seq scan 或更晚过滤。
范围条件估小，会让 index range scan 看起来更窄。
如果 histogram 过旧，实际扫描页数可能远高于估算。
范围条件估大，会让排序前过滤效果被低估。
后续 sort、hash aggregate 或 join 的输入行数会被抬高。
模式匹配估小，会让带固定前缀的路径看起来很便宜。
如果实际 prefix 很常见，后续 heap fetch 或 join 会放大成本。
模式匹配估大，会让可用前缀索引被低估。
一个单点 selectivity 误差会沿 `RelOptInfo.rows` 进入 `Path.rows`。
再进入 join cardinality、hash table size、merge sort cost 和 parallel divisor 判断。
因此诊断时不要只问这个 predicate 的估算是否准确。
还要问它是不是 join order 的早期输入。
早期 base rel 误差比 late filter 误差更容易放大。
这也是为什么 planner rows 诊断通常先看最底层 scan node。
如果 base scan 已经偏差十倍，join node 的百倍偏差可能只是传播结果。
如果 base scan 准确但 join 偏差大，才转向 join selectivity 或 join dependency。

## 11. 观测与诊断入口

- 诊断卡片 `等值热点误估` 的可见现象是 某个值非常常见但 estimated rows 接近平均值。
- 第一反查入口是 `var_eq_const`，因为这里最早决定是否仍有真实统计输入。
- 第二反查入口是 `mcv_selectivity`，因为这里能确认 fallback 是否已经发生。
- 如果两个入口都没有解释偏差，再继续看 clause 组合、join cardinality 或成本参数。
- 这个诊断的结论应写成：常量没有命中可用 MCV 或统计已经过旧。
- 诊断卡片 `等值长尾误估` 的可见现象是 某个罕见值估得比实际高或低。
- 第一反查入口是 `get_variable_numdistinct`，因为这里最早决定是否仍有真实统计输入。
- 第二反查入口是 `var_eq_const`，因为这里能确认 fallback 是否已经发生。
- 如果两个入口都没有解释偏差，再继续看 clause 组合、join cardinality 或成本参数。
- 这个诊断的结论应写成：非 MCV 剩余 distinct 数只是近似假设。
- 诊断卡片 `唯一性修正` 的可见现象是 唯一列等值估算接近一行。
- 第一反查入口是 `var_eq_const`，因为这里最早决定是否仍有真实统计输入。
- 第二反查入口是 `examine_variable`，因为这里能确认 fallback 是否已经发生。
- 如果两个入口都没有解释偏差，再继续看 clause 组合、join cardinality 或成本参数。
- 这个诊断的结论应写成：`vardata->isunique` 被识别并短路普通统计。
- 诊断卡片 `范围边界过旧` 的可见现象是 时间列范围估算明显偏离近期数据。
- 第一反查入口是 `ineq_histogram_selectivity`，因为这里最早决定是否仍有真实统计输入。
- 第二反查入口是 `get_actual_variable_range`，因为这里能确认 fallback 是否已经发生。
- 如果两个入口都没有解释偏差，再继续看 clause 组合、join cardinality 或成本参数。
- 这个诊断的结论应写成：histogram endpoint 不能完全代表最新数据。
- 诊断卡片 `范围类型不可标量化` 的可见现象是 自定义类型范围条件估算粗糙。
- 第一反查入口是 `convert_to_scalar`，因为这里最早决定是否仍有真实统计输入。
- 第二反查入口是 `scalarineqsel`，因为这里能确认 fallback 是否已经发生。
- 如果两个入口都没有解释偏差，再继续看 clause 组合、join cardinality 或成本参数。
- 这个诊断的结论应写成：类型无法可靠映射到线性尺度。
- 诊断卡片 `LIKE 前缀有效` 的可见现象是 带固定前缀的模式估算明显优于无前缀模式。
- 第一反查入口是 `pattern_fixed_prefix`，因为这里最早决定是否仍有真实统计输入。
- 第二反查入口是 `prefix_selectivity`，因为这里能确认 fallback 是否已经发生。
- 如果两个入口都没有解释偏差，再继续看 clause 组合、join cardinality 或成本参数。
- 这个诊断的结论应写成：partial prefix 被转成范围选择率。
- 诊断卡片 `LIKE 完全匹配` 的可见现象是 无通配的模式估算接近等值估算。
- 第一反查入口是 `patternsel_common`，因为这里最早决定是否仍有真实统计输入。
- 第二反查入口是 `var_eq_const`，因为这里能确认 fallback 是否已经发生。
- 如果两个入口都没有解释偏差，再继续看 clause 组合、join cardinality 或成本参数。
- 这个诊断的结论应写成：exact pattern 复用等值估算。
- 诊断卡片 `LIKE 前缀失败` 的可见现象是 leading wildcard 模式估算回到启发式或默认。
- 第一反查入口是 `like_selectivity`，因为这里最早决定是否仍有真实统计输入。
- 第二反查入口是 `pattern_fixed_prefix`，因为这里能确认 fallback 是否已经发生。
- 如果两个入口都没有解释偏差，再继续看 clause 组合、join cardinality 或成本参数。
- 这个诊断的结论应写成：pattern 缺少可用于范围的固定前缀。
- 诊断卡片 `MCV 与 histogram 重复误读` 的可见现象是 手工估算时把 MCV 频率和 histogram 全表频率重复计算。
- 第一反查入口是 `scalarineqsel`，因为这里最早决定是否仍有真实统计输入。
- 第二反查入口是 `mcv_selectivity`，因为这里能确认 fallback 是否已经发生。
- 如果两个入口都没有解释偏差，再继续看 clause 组合、join cardinality 或成本参数。
- 这个诊断的结论应写成：histogram population 已扣除 NULL 与 MCV。
- 诊断卡片 `取反误读` 的可见现象是 不等或 not-match 的 rows 没有简单等于一减原估算。
- 第一反查入口是 `eqsel_internal`，因为这里最早决定是否仍有真实统计输入。
- 第二反查入口是 `patternsel_common`，因为这里能确认 fallback 是否已经发生。
- 如果两个入口都没有解释偏差，再继续看 clause 组合、join cardinality 或成本参数。
- 这个诊断的结论应写成：取反时还要扣除 null fraction。

## 12. 课堂实验：比较三类 predicate 的退化路径

- 课堂实验 `等值 MCV 命中` 的准备动作是 插入一个频率很高的状态值并运行 `ANALYZE`。
- 实验 SQL 关注的 predicate 是 状态列等值过滤。
- `EXPLAIN` 观察点是 estimated rows 是否接近 MCV 频率。
- 回到源码时先读 `var_eq_const`，确认估算数字从哪里进入 planner。
- 实验报告必须同时写出可见现象、源码入口、fallback 条件和可迁移规律。
- 课堂实验 `等值长尾平均` 的准备动作是 查询一个存在但不在 MCV 中的状态值。
- 实验 SQL 关注的 predicate 是 状态列罕见值过滤。
- `EXPLAIN` 观察点是 estimated rows 是否接近剩余 population 除以剩余 distinct。
- 回到源码时先读 `get_variable_numdistinct`，确认估算数字从哪里进入 planner。
- 实验报告必须同时写出可见现象、源码入口、fallback 条件和可迁移规律。
- 课堂实验 `未知参数等值` 的准备动作是 使用 prepared statement 或不可折叠表达式作为比较值。
- 实验 SQL 关注的 predicate 是 参数化等值过滤。
- `EXPLAIN` 观察点是 estimated rows 是否更接近平均等值选择率。
- 回到源码时先读 `var_eq_non_const`，确认估算数字从哪里进入 planner。
- 实验报告必须同时写出可见现象、源码入口、fallback 条件和可迁移规律。
- 课堂实验 `范围 histogram 插值` 的准备动作是 在时间列上查询一个中间区间边界。
- 实验 SQL 关注的 predicate 是 时间列范围过滤。
- `EXPLAIN` 观察点是 estimated rows 是否随边界位置连续变化。
- 回到源码时先读 `ineq_histogram_selectivity`，确认估算数字从哪里进入 planner。
- 实验报告必须同时写出可见现象、源码入口、fallback 条件和可迁移规律。
- 课堂实验 `范围无统计退化` 的准备动作是 新建表后立即跑范围查询或清空统计影响。
- 实验 SQL 关注的 predicate 是 范围过滤。
- `EXPLAIN` 观察点是 estimated rows 是否接近三分之一输入行数。
- 回到源码时先读 `scalarineqsel`，确认估算数字从哪里进入 planner。
- 实验报告必须同时写出可见现象、源码入口、fallback 条件和可迁移规律。
- 课堂实验 `LIKE exact prefix` 的准备动作是 比较无通配模式与等值过滤。
- 实验 SQL 关注的 predicate 是 文本列 exact pattern。
- `EXPLAIN` 观察点是 两个 estimated rows 是否接近。
- 回到源码时先读 `patternsel_common`，确认估算数字从哪里进入 planner。
- 实验报告必须同时写出可见现象、源码入口、fallback 条件和可迁移规律。
- 课堂实验 `LIKE partial prefix` 的准备动作是 比较固定前缀模式与 leading wildcard 模式。
- 实验 SQL 关注的 predicate 是 文本列 prefix pattern。
- `EXPLAIN` 观察点是 前者 estimated rows 是否更受 prefix 影响。
- 回到源码时先读 `prefix_selectivity`，确认估算数字从哪里进入 planner。
- 实验报告必须同时写出可见现象、源码入口、fallback 条件和可迁移规律。
- 课堂实验 `regex 启发式` 的准备动作是 使用简单 regex 与复杂 regex 对比。
- 实验 SQL 关注的 predicate 是 正则模式过滤。
- `EXPLAIN` 观察点是 estimated rows 是否随固定字符和通配结构变化。
- 回到源码时先读 `regex_selectivity_sub`，确认估算数字从哪里进入 planner。
- 实验报告必须同时写出可见现象、源码入口、fallback 条件和可迁移规律。
- 课堂实验 `提高统计目标` 的准备动作是 提高文本列和数值列的 statistics target 后重新 `ANALYZE`。
- 实验 SQL 关注的 predicate 是 偏斜列和文本列过滤。
- `EXPLAIN` 观察点是 MCV 和 histogram 变化是否改善 rows。
- 回到源码时先读 `analyze.c`，确认估算数字从哪里进入 planner。
- 实验报告必须同时写出可见现象、源码入口、fallback 条件和可迁移规律。
- 课堂实验 `NULL 比例影响` 的准备动作是 插入大量 NULL 并比较等值、范围和模式查询。
- 实验 SQL 关注的 predicate 是 含 NULL 列过滤。
- `EXPLAIN` 观察点是 estimated rows 是否扣除了 null fraction。
- 回到源码时先读 `selfuncs.c`，确认估算数字从哪里进入 planner。
- 实验报告必须同时写出可见现象、源码入口、fallback 条件和可迁移规律。

## 13. 源码练习：把每个估算数字拆回统计输入

- 源码练习 `等值 MCV 命中路径` 从 `src/backend/utils/adt/selfuncs.c` 开始。
- 练习动作是 在 `var_eq_const()` 中跟踪 MCV loop 的 match 变量。
- 完成标准是能解释 命中时为什么直接返回 `sslot.numbers[i]`。
- 如果只能背函数名，说明还没有把状态生命周期和 fallback 边界串起来。
- 源码练习 `等值长尾路径` 从 `src/backend/utils/adt/selfuncs.c` 开始。
- 练习动作是 观察 `sumcommon`、`nullfrac` 和 `otherdistinct` 的计算。
- 完成标准是能解释 非 MCV 值为什么不能比最小 MCV 更常见。
- 如果只能背函数名，说明还没有把状态生命周期和 fallback 边界串起来。
- 源码练习 `非 Const 等值路径` 从 `src/backend/utils/adt/selfuncs.c` 开始。
- 练习动作是 让比较值变成 Param 并跟踪 `var_eq_non_const()`。
- 完成标准是能解释 未知值为什么只能按 distinct 平均。
- 如果只能背函数名，说明还没有把状态生命周期和 fallback 边界串起来。
- 源码练习 `范围 wrapper 形状检查` 从 `src/backend/utils/adt/selfuncs.c` 开始。
- 练习动作是 在 `scalarineqsel_wrapper()` 中观察 commutator 处理。
- 完成标准是能解释 右侧变量或缺 commutator 为什么触发默认范围估算。
- 如果只能背函数名，说明还没有把状态生命周期和 fallback 边界串起来。
- 源码练习 `范围 MCV 直接计数` 从 `src/backend/utils/adt/selfuncs.c` 开始。
- 练习动作是 在 `mcv_selectivity()` 中观察 operator 对 MCV 的逐项测试。
- 完成标准是能解释 MCV 频率为什么不进入 histogram population。
- 如果只能背函数名，说明还没有把状态生命周期和 fallback 边界串起来。
- 源码练习 `范围 histogram 插值` 从 `src/backend/utils/adt/selfuncs.c` 开始。
- 练习动作是 在 `ineq_histogram_selectivity()` 中观察 binary search 和 bin fraction。
- 完成标准是能解释 常量位置如何变成 histogram fraction。
- 如果只能背函数名，说明还没有把状态生命周期和 fallback 边界串起来。
- 源码练习 `标量转换失败路径` 从 `src/backend/utils/adt/selfuncs.c` 开始。
- 练习动作是 寻找 `convert_to_scalar()` 返回 false 后的处理。
- 完成标准是能解释 bin fraction 为什么退化为 `0.5`。
- 如果只能背函数名，说明还没有把状态生命周期和 fallback 边界串起来。
- 源码练习 `LIKE exact 路径` 从 `src/backend/utils/adt/like_support.c` 开始。
- 练习动作是 在 `patternsel_common()` 中观察 `Pattern_Prefix_Exact` 分支。
- 完成标准是能解释 exact pattern 为什么复用 `var_eq_const()`。
- 如果只能背函数名，说明还没有把状态生命周期和 fallback 边界串起来。
- 源码练习 `LIKE partial 路径` 从 `src/backend/utils/adt/like_support.c` 开始。
- 练习动作是 在 `prefix_selectivity()` 中观察前缀范围估算。
- 完成标准是能解释 固定前缀如何变成范围约束。
- 如果只能背函数名，说明还没有把状态生命周期和 fallback 边界串起来。
- 源码练习 `LIKE heuristic 混合` 从 `src/backend/utils/adt/like_support.c` 开始。
- 练习动作是 观察 histogram size 小于一百时的权重混合。
- 完成标准是能解释 为什么小 histogram 不应被完全相信。
- 如果只能背函数名，说明还没有把状态生命周期和 fallback 边界串起来。
- 源码练习 `pattern MCV 加回` 从 `src/backend/utils/adt/like_support.c` 开始。
- 练习动作是 跟踪 pattern operator 对 MCV 的逐项匹配。
- 完成标准是能解释 热点字符串为什么可以比启发式更准确。
- 如果只能背函数名，说明还没有把状态生命周期和 fallback 边界串起来。
- 源码练习 `取反与 NULL` 从 `src/backend/utils/adt/selfuncs.c` 开始。
- 练习动作是 比较 `neqsel()` 和 `nlikesel()` 的取反逻辑。
- 完成标准是能解释 为什么取反要扣除 null fraction。
- 如果只能背函数名，说明还没有把状态生命周期和 fallback 边界串起来。

## 14. 常见误区：同一个概率接口不代表同一种统计问题

- 误区一是认为等值、范围和 LIKE 都只是调用 `clause_selectivity_ext()`，所以估算逻辑差不多。
- 误区二是认为 MCV 只服务等值；范围和模式匹配也会直接测试 MCV，只是合并方式不同。
- 误区三是认为 histogram 覆盖全表；在这些 estimator 中，histogram 通常代表非 NULL、非 MCV 的剩余 population。
- 误区四是认为带固定前缀的模式只能靠 `DEFAULT_MATCH_SEL`；固定前缀可以让模式匹配转成范围估算。
- 误区五是认为 leading wildcard 模式提高统计目标一定能解决；没有可用前缀时，统计目标不能凭空创造范围约束。
- 误区六是认为不等估算等于一减等值选择率；NULL population 不满足普通不等比较，需要扣除。
- 误区七是认为 histogram 插值总是精确；它依赖样本边界、标量转换和线性位置近似。
- 误区八是认为自定义类型只要有比较 operator 就能得到可靠范围估算；如果不能被 `convert_to_scalar()` 合理映射，估算会退化。
- 误区九是把 pattern estimator 的启发式当成执行匹配算法；执行时仍由真实 LIKE 或 regex 函数判断。
- 误区十是把单个 predicate 的估算误差和多个 clause 的组合误差混在一起；单点 estimator 先形成概率，组合误差在 clause list 层继续发生。

## 15. 讨论题与本节小结

- 讨论题一：为什么等值常量命中 MCV 时可以直接采用 MCV 频率。
- 讨论题二：为什么等值常量没有命中 MCV 时不能简单使用 `DEFAULT_EQ_SEL`。
- 讨论题三：`var_eq_non_const()` 对 Param 使用平均估算有什么风险。
- 讨论题四：范围条件为什么需要先把 MCV population 单独拿出来。
- 讨论题五：histogram bin 内线性插值隐含了哪些数据分布假设。
- 讨论题六：`convert_to_scalar()` 失败时使用 bin 中点有什么工程含义。
- 讨论题七：为什么 exact pattern 可以复用等值估算。
- 讨论题八：partial prefix 为什么能构造范围约束，而 leading wildcard 不行。
- 讨论题九：为什么 pattern estimator 要混合 histogram 和 heuristic，而不是只信其中一个。
- 本节小结一：等值条件关注具体值是否常见，所以它优先使用 MCV，随后用 ndistinct 分摊长尾。
- 本节小结二：范围条件关注常量在有序分布中的位置，所以它直接计数 MCV，再用 histogram 对剩余 population 插值。
- 本节小结三：模式匹配关注 pattern 是否能提供固定前缀，所以它先抽 prefix，再把 exact 转成等值、partial 转成范围，剩余部分用启发式。
- 本节小结四：三者都使用 `VariableStatData`，也都要释放 stats tuple 和 stats slot。
- 本节小结五：三者都在统计缺失、表达式不合形状或类型不支持时 fallback。
- 本节小结六：三者返回的都只是概率，这个概率会进入 `RelOptInfo.rows` 并影响后续成本搜索。
## 17. 运行案例拆解：把可见现象落到源码入口

- 案例 `热点等值命中` 的 SQL 形状是 状态列等于高频常量。
- 可观察现象是 estimated rows 接近 MCV frequency 乘以表大小。
- 第一源码入口是 `var_eq_const`，因为这里最早把输入事实压缩成选择率。
- 诊断时要确认该入口拿到的是统计事实、catalog 绑定、表达式形状还是默认假设。
- 可迁移结论是 等值热点首先看 MCV，而不是看 histogram。
- 案例 `长尾等值未命中` 的 SQL 形状是 状态列等于低频常量。
- 可观察现象是 estimated rows 接近剩余 population 的平均分摊。
- 第一源码入口是 `get_variable_numdistinct`，因为这里最早把输入事实压缩成选择率。
- 诊断时要确认该入口拿到的是统计事实、catalog 绑定、表达式形状还是默认假设。
- 可迁移结论是 非 MCV 长尾是平均假设，不是逐值事实。
- 案例 `唯一列等值` 的 SQL 形状是 唯一键列等于常量。
- 可观察现象是 estimated rows 接近一行。
- 第一源码入口是 `var_eq_const`，因为这里最早把输入事实压缩成选择率。
- 诊断时要确认该入口拿到的是统计事实、catalog 绑定、表达式形状还是默认假设。
- 可迁移结论是 唯一性可以短路普通 MCV 路径。
- 案例 `未知参数等值` 的 SQL 形状是 参数化等值过滤。
- 可观察现象是 estimated rows 更接近平均值。
- 第一源码入口是 `var_eq_non_const`，因为这里最早把输入事实压缩成选择率。
- 诊断时要确认该入口拿到的是统计事实、catalog 绑定、表达式形状还是默认假设。
- 可迁移结论是 planner 看不到具体值时无法命中 MCV。
- 案例 `不等取反` 的 SQL 形状是 列不等于常量。
- 可观察现象是 结果不是简单一减等值频率。
- 第一源码入口是 `eqsel_internal`，因为这里最早把输入事实压缩成选择率。
- 诊断时要确认该入口拿到的是统计事实、catalog 绑定、表达式形状还是默认假设。
- 可迁移结论是 NULL population 必须从取反结果中扣除。
- 案例 `范围中间边界` 的 SQL 形状是 数值列小于中间常量。
- 可观察现象是 estimated rows 随 histogram 位置变化。
- 第一源码入口是 `ineq_histogram_selectivity`，因为这里最早把输入事实压缩成选择率。
- 诊断时要确认该入口拿到的是统计事实、catalog 绑定、表达式形状还是默认假设。
- 可迁移结论是 范围条件问的是常量在有序分布中的位置。
- 案例 `范围极端边界` 的 SQL 形状是 数值列小于极小或极大常量。
- 可观察现象是 estimated rows 被限制在合理边界。
- 第一源码入口是 `ineq_histogram_selectivity`，因为这里最早把输入事实压缩成选择率。
- 诊断时要确认该入口拿到的是统计事实、catalog 绑定、表达式形状还是默认假设。
- 可迁移结论是 histogram endpoint 和 clamp 防止极端估算过度自信。
- 案例 `范围无统计` 的 SQL 形状是 新表范围过滤。
- 可观察现象是 estimated rows 接近默认范围选择率。
- 第一源码入口是 `scalarineqsel`，因为这里最早把输入事实压缩成选择率。
- 诊断时要确认该入口拿到的是统计事实、catalog 绑定、表达式形状还是默认假设。
- 可迁移结论是 没有统计时范围 estimator 不能推断分布位置。
- 案例 `范围无法 commutate` 的 SQL 形状是 常量在左侧且 operator 无 commutator。
- 可观察现象是 估算退回默认范围值。
- 第一源码入口是 `scalarineqsel_wrapper`，因为这里最早把输入事实压缩成选择率。
- 诊断时要确认该入口拿到的是统计事实、catalog 绑定、表达式形状还是默认假设。
- 可迁移结论是 operator 元数据决定是否能规范化为 var op const。
- 案例 `自定义类型范围` 的 SQL 形状是 自定义类型使用大小比较。
- 可观察现象是 histogram 存在但估算仍粗糙。
- 第一源码入口是 `convert_to_scalar`，因为这里最早把输入事实压缩成选择率。
- 诊断时要确认该入口拿到的是统计事实、catalog 绑定、表达式形状还是默认假设。
- 可迁移结论是 可排序不等于可可靠映射到标量尺度。
- 案例 `CTID 范围特例` 的 SQL 形状是 ctid 与 item pointer 比较。
- 可观察现象是 无普通统计时仍可根据页面位置粗估。
- 第一源码入口是 `scalarineqsel`，因为这里最早把输入事实压缩成选择率。
- 诊断时要确认该入口拿到的是统计事实、catalog 绑定、表达式形状还是默认假设。
- 可迁移结论是 CTID 是范围 estimator 中少见的物理位置特例。
- 案例 `exact pattern` 的 SQL 形状是 文本列匹配无通配模式。
- 可观察现象是 estimated rows 接近等值估算。
- 第一源码入口是 `patternsel_common`，因为这里最早把输入事实压缩成选择率。
- 诊断时要确认该入口拿到的是统计事实、catalog 绑定、表达式形状还是默认假设。
- 可迁移结论是 exact prefix 可以直接退化成等值。
- 案例 `partial prefix` 的 SQL 形状是 文本列匹配固定前缀模式。
- 可观察现象是 estimated rows 明显受 prefix 约束影响。
- 第一源码入口是 `prefix_selectivity`，因为这里最早把输入事实压缩成选择率。
- 诊断时要确认该入口拿到的是统计事实、catalog 绑定、表达式形状还是默认假设。
- 可迁移结论是 partial prefix 可以转成范围约束。
- 案例 `leading wildcard` 的 SQL 形状是 文本列匹配前导通配模式。
- 可观察现象是 estimated rows 更依赖启发式或 MCV。
- 第一源码入口是 `pattern_fixed_prefix`，因为这里最早把输入事实压缩成选择率。
- 诊断时要确认该入口拿到的是统计事实、catalog 绑定、表达式形状还是默认假设。
- 可迁移结论是 没有固定前缀就不能构造前缀范围。
- 案例 `小 histogram 模式` 的 SQL 形状是 文本列统计样本较少。
- 可观察现象是 模式估算混合 histogram 与启发式。
- 第一源码入口是 `patternsel_common`，因为这里最早把输入事实压缩成选择率。
- 诊断时要确认该入口拿到的是统计事实、catalog 绑定、表达式形状还是默认假设。
- 可迁移结论是 小样本不应被完全信任。
- 案例 `pattern MCV 覆盖` 的 SQL 形状是 热点字符串出现在 MCV 中。
- 可观察现象是 模式估算能直接加回热点频率。
- 第一源码入口是 `mcv_selectivity`，因为这里最早把输入事实压缩成选择率。
- 诊断时要确认该入口拿到的是统计事实、catalog 绑定、表达式形状还是默认假设。
- 可迁移结论是 pattern operator 也可以直接测试 MCV。
- 案例 `regex 固定字符` 的 SQL 形状是 正则包含较多固定字符。
- 可观察现象是 estimated rows 随固定字符增多而降低。
- 第一源码入口是 `regex_selectivity_sub`，因为这里最早把输入事实压缩成选择率。
- 诊断时要确认该入口拿到的是统计事实、catalog 绑定、表达式形状还是默认假设。
- 可迁移结论是 regex 剩余部分使用启发式概率模型。
- 案例 `ILIKE 差异` 的 SQL 形状是 大小写不敏感模式过滤。
- 可观察现象是 估算仍走 pattern 公共框架。
- 第一源码入口是 `iclikesel`，因为这里最早把输入事实压缩成选择率。
- 诊断时要确认该入口拿到的是统计事实、catalog 绑定、表达式形状还是默认假设。
- 可迁移结论是 case-insensitive 影响 pattern 处理但不改变总体状态链。
- 案例 `NULL 大量存在` 的 SQL 形状是 含大量 NULL 的列做三类谓词。
- 可观察现象是 estimated rows 都扣除了 NULL population。
- 第一源码入口是 `selfuncs.c`，因为这里最早把输入事实压缩成选择率。
- 诊断时要确认该入口拿到的是统计事实、catalog 绑定、表达式形状还是默认假设。
- 可迁移结论是 NULL fraction 是三类 estimator 共享的基础事实。
- 案例 `统计目标提高` 的 SQL 形状是 提高 statistics target 后重新分析。
- 可观察现象是 MCV 或 histogram 覆盖改善。
- 第一源码入口是 `analyze.c`，因为这里最早把输入事实压缩成选择率。
- 诊断时要确认该入口拿到的是统计事实、catalog 绑定、表达式形状还是默认假设。
- 可迁移结论是 统计分辨率直接影响 estimator 可见事实。
- 案例 `模式无常量` 的 SQL 形状是 pattern 来自运行时参数。
- 可观察现象是 估算不能使用具体 prefix。
- 第一源码入口是 `patternsel_common`，因为这里最早把输入事实压缩成选择率。
- 诊断时要确认该入口拿到的是统计事实、catalog 绑定、表达式形状还是默认假设。
- 可迁移结论是 pattern 常量可见性决定是否能提取 prefix。
- 案例 `模式类型不支持` 的 SQL 形状是 非支持类型使用 pattern estimator。
- 可观察现象是 估算退回默认匹配值。
- 第一源码入口是 `patternsel_common`，因为这里最早把输入事实压缩成选择率。
- 诊断时要确认该入口拿到的是统计事实、catalog 绑定、表达式形状还是默认假设。
- 可迁移结论是 类型边界防止错误解释统计槽位。
- 案例 `范围与 MCV 重叠` 的 SQL 形状是 范围包含多个热点值。
- 可观察现象是 MCV contribution 直接加回。
- 第一源码入口是 `mcv_selectivity`，因为这里最早把输入事实压缩成选择率。
- 诊断时要确认该入口拿到的是统计事实、catalog 绑定、表达式形状还是默认假设。
- 可迁移结论是 热点 population 与 histogram population 必须分开合并。
- 案例 `估算传播到索引` 的 SQL 形状是 选择性变化导致 index range scan 与 seq scan 切换。
- 可观察现象是 scan path 选择变化。
- 第一源码入口是 `cost_index`，因为这里最早把输入事实压缩成选择率。
- 诊断时要确认该入口拿到的是统计事实、catalog 绑定、表达式形状还是默认假设。
- 可迁移结论是 单点 estimator 的误差会改变 access path 排序。

## 18. 源码决策点：逐层确认是否进入 fallback

- 决策点 `等值入口`：先检查 operator 是否绑定到 `eqsel` 或等价 estimator。
- 对应源码入口是 `pg_operator.dat`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 operator 绑定问题。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。
- 决策点 `等值变量识别`：先检查 参数是否能拆成 variable 与另一侧。
- 对应源码入口是 `get_restriction_variable`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 表达式形状 fallback。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。
- 决策点 `等值常量可见`：先检查 另一侧是否为 Const。
- 对应源码入口是 `eqsel_internal`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 未知值平均估算。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。
- 决策点 `等值 NULL`：先检查 常量是否为 NULL。
- 对应源码入口是 `var_eq_const`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 NULL strict 边界。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。
- 决策点 `等值唯一性`：先检查 `vardata->isunique` 是否为真。
- 对应源码入口是 `var_eq_const`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 唯一性短路估算。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。
- 决策点 `等值 MCV`：先检查 常量是否命中 MCV 槽位。
- 对应源码入口是 `var_eq_const`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 MCV 命中估算。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。
- 决策点 `等值长尾`：先检查 未命中 MCV 后 distinct 是否默认。
- 对应源码入口是 `get_variable_numdistinct`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 长尾平均估算。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。
- 决策点 `不等取反`：先检查 negator 是否存在。
- 对应源码入口是 `eqsel_internal`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 取反 fallback。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。
- 决策点 `范围入口`：先检查 operator 是否绑定到 scalar inequality estimator。
- 对应源码入口是 `pg_operator.dat`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 范围绑定问题。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。
- 决策点 `范围 Const`：先检查 另一侧是否为非 NULL Const。
- 对应源码入口是 `scalarineqsel_wrapper`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 默认范围 fallback。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。
- 决策点 `范围方向`：先检查 变量是否在左侧或能否 commutate。
- 对应源码入口是 `scalarineqsel_wrapper`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 方向规范化问题。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。
- 决策点 `范围统计`：先检查 stats tuple 是否有效。
- 对应源码入口是 `scalarineqsel`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 默认范围估算。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。
- 决策点 `范围 MCV`：先检查 MCV contribution 是否先被单独计算。
- 对应源码入口是 `mcv_selectivity`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 热点直接计数问题。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。
- 决策点 `范围 histogram`：先检查 histogram 是否存在且足够排序。
- 对应源码入口是 `ineq_histogram_selectivity`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 histogram fallback。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。
- 决策点 `范围 scalar`：先检查 常量与边界是否能 scalar conversion。
- 对应源码入口是 `convert_to_scalar`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 bin 中点近似。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。
- 决策点 `模式入口`：先检查 operator 是否绑定到 `likesel`、`regexeqsel` 或 `prefixsel`。
- 对应源码入口是 `pg_operator.dat`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 模式绑定问题。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。
- 决策点 `模式常量`：先检查 pattern 是否为 Const。
- 对应源码入口是 `patternsel_common`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 默认模式 fallback。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。
- 决策点 `模式类型`：先检查 pattern 和变量 exposed type 是否受支持。
- 对应源码入口是 `patternsel_common`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 类型 fallback。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。
- 决策点 `模式 exact`：先检查 prefix status 是否为 exact。
- 对应源码入口是 `pattern_fixed_prefix`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 等值复用路径。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。
- 决策点 `模式 partial`：先检查 prefix status 是否为 partial。
- 对应源码入口是 `prefix_selectivity`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 前缀范围路径。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。
- 决策点 `模式 histogram`：先检查 histogram size 是否足够信任。
- 对应源码入口是 `histogram_selectivity`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 启发式混合。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。
- 决策点 `模式 MCV`：先检查 热点字符串是否直接匹配 pattern。
- 对应源码入口是 `mcv_selectivity`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 MCV 加回路径。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。
- 决策点 `模式取反`：先检查 not-match 是否扣除 null fraction。
- 对应源码入口是 `patternsel_common`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 取反 NULL 问题。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。
- 决策点 `统计释放`：先检查 stats slot 是否在返回前释放。
- 对应源码入口是 `free_attstatsslot`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 ownership 问题。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。
- 决策点 `前缀释放`：先检查 分配出的 prefix Const 是否被释放。
- 对应源码入口是 `patternsel_common`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 短生命周期内存问题。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。
- 决策点 `成本传播`：先检查 base scan rows 是否因 estimator 变化而改变。
- 对应源码入口是 `set_baserel_size_estimates`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 下游成本问题。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。
- 决策点 `索引影响`：先检查 选择率是否改变 index path 成本。
- 对应源码入口是 `cost_index`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 access path 排序问题。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。
- 决策点 `组合边界`：先检查 多个范围边界是否已进入 list pairing。
- 对应源码入口是 `clausesel.c`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 单点与组合混淆问题。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。
- 决策点 `版本基线`：先检查 函数行为是否来自指定源码提交。
- 对应源码入口是 `git rev-parse`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 版本漂移问题。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。
- 决策点 `统计新鲜度`：先检查 统计是否反映当前数据分布。
- 对应源码入口是 `pg_stats`，不要跳过这一层直接解释最终 plan。
- 如果这一层失败，当前估算应归类为 过期统计问题。
- 如果这一层成功，再进入下一层统计槽位或成本传播分析。

## 19. 复盘清单：从输入事实到成本传播

- 复盘项：把等值、范围、模式匹配先分到不同 estimator 家族，再讨论统计槽位。。
- 复盘项：把 MCV 是否命中作为等值热点诊断的第一分水岭。。
- 复盘项：把 non-MCV 长尾看成平均假设，而不是每个值的真实频率。。
- 复盘项：把 histogram 看成剩余 population 的有序样本，而不是全表事实。。
- 复盘项：把 `sumcommon`、`stanullfrac` 和 histogram population 的关系写清楚。。
- 复盘项：把 `convert_to_scalar()` 是否成功作为范围插值质量的关键边界。。
- 复盘项：把 exact pattern 和 partial prefix pattern 分开解释。。
- 复盘项：把 leading wildcard 模式归类为缺少范围前缀，而不是简单统计目标不足。。
- 复盘项：把 pattern MCV 直接测试和 pattern heuristic 分开记录。。
- 复盘项：把 NULL fraction 对等值、不等、范围和模式取反的影响单独写出。。
- 复盘项：把 `ReleaseVariableStats()`、`free_attstatsslot()` 和 prefix `pfree()` 作为源码练习的一部分。。
- 复盘项：把单个 predicate 的估算误差和 clause list 组合误差分开定位。。
- 复盘项：把统计目标提高后的改善范围限定在 MCV 与 histogram 分辨率。。
- 复盘项：把自定义类型范围估算是否可靠绑定到 scalar conversion 能力。。
- 复盘项：把最终 plan 变化回扣到 `RelOptInfo.rows` 和 path cost，而不是停在 estimator 函数名。。

## 20. 诊断记录模板：等值范围模式估算记录

- 记录项 `等值 MCV` 的现象字段写作：目标常量是否出现在 `most_common_vals` 且频率是否解释 rows。
- 记录项 `等值 MCV` 的源码字段写作：先从 `var_eq_const` 进入，不直接跳到最终 plan。
- 记录项 `等值 MCV` 的边界字段写作：明确这是统计事实、默认假设、catalog 绑定还是成本传播。
- 记录项 `等值 MCV` 的结论字段写作：命中 MCV 时不应再用平均长尾解释。
- 记录项 `等值长尾` 的现象字段写作：目标常量不在 MCV 时剩余 population 和 distinct 如何分摊。
- 记录项 `等值长尾` 的源码字段写作：先从 `get_variable_numdistinct` 进入，不直接跳到最终 plan。
- 记录项 `等值长尾` 的边界字段写作：明确这是统计事实、默认假设、catalog 绑定还是成本传播。
- 记录项 `等值长尾` 的结论字段写作：长尾估算是平均假设，误差通常来自分布不均。
- 记录项 `非 Const 等值` 的现象字段写作：比较值在 planning 阶段是否可见为常量。
- 记录项 `非 Const 等值` 的源码字段写作：先从 `var_eq_non_const` 进入，不直接跳到最终 plan。
- 记录项 `非 Const 等值` 的边界字段写作：明确这是统计事实、默认假设、catalog 绑定还是成本传播。
- 记录项 `非 Const 等值` 的结论字段写作：不可见值不能命中 MCV，只能按平均选择率解释。
- 记录项 `范围 MCV` 的现象字段写作：范围条件包含哪些热点值以及热点频率是否被直接加回。
- 记录项 `范围 MCV` 的源码字段写作：先从 `mcv_selectivity` 进入，不直接跳到最终 plan。
- 记录项 `范围 MCV` 的边界字段写作：明确这是统计事实、默认假设、catalog 绑定还是成本传播。
- 记录项 `范围 MCV` 的结论字段写作：范围估算也要先分离 MCV population。
- 记录项 `范围 histogram` 的现象字段写作：常量落在哪两个 histogram bounds 之间。
- 记录项 `范围 histogram` 的源码字段写作：先从 `ineq_histogram_selectivity` 进入，不直接跳到最终 plan。
- 记录项 `范围 histogram` 的边界字段写作：明确这是统计事实、默认假设、catalog 绑定还是成本传播。
- 记录项 `范围 histogram` 的结论字段写作：范围 rows 偏差要先检查 histogram 位置和样本新鲜度。
- 记录项 `标量转换` 的现象字段写作：常量和边界是否能映射到线性比较尺度。
- 记录项 `标量转换` 的源码字段写作：先从 `convert_to_scalar` 进入，不直接跳到最终 plan。
- 记录项 `标量转换` 的边界字段写作：明确这是统计事实、默认假设、catalog 绑定还是成本传播。
- 记录项 `标量转换` 的结论字段写作：转换失败时 bin 中点近似会降低估算可信度。
- 记录项 `exact pattern` 的现象字段写作：模式是否无通配并被识别为 exact prefix。
- 记录项 `exact pattern` 的源码字段写作：先从 `patternsel_common` 进入，不直接跳到最终 plan。
- 记录项 `exact pattern` 的边界字段写作：明确这是统计事实、默认假设、catalog 绑定还是成本传播。
- 记录项 `exact pattern` 的结论字段写作：exact pattern 应按等值路径解释。
- 记录项 `partial prefix` 的现象字段写作：模式是否提供可构造范围上界的固定前缀。
- 记录项 `partial prefix` 的源码字段写作：先从 `prefix_selectivity` 进入，不直接跳到最终 plan。
- 记录项 `partial prefix` 的边界字段写作：明确这是统计事实、默认假设、catalog 绑定还是成本传播。
- 记录项 `partial prefix` 的结论字段写作：partial prefix 的收益来自范围估算而不是纯默认值。
- 记录项 `leading wildcard` 的现象字段写作：模式是否以前导通配符开始并缺少固定前缀。
- 记录项 `leading wildcard` 的源码字段写作：先从 `pattern_fixed_prefix` 进入，不直接跳到最终 plan。
- 记录项 `leading wildcard` 的边界字段写作：明确这是统计事实、默认假设、catalog 绑定还是成本传播。
- 记录项 `leading wildcard` 的结论字段写作：缺 prefix 时提高统计目标只能改善 MCV 或样本部分。
- 记录项 `pattern heuristic` 的现象字段写作：histogram 太小时是否混合启发式结果。
- 记录项 `pattern heuristic` 的源码字段写作：先从 `like_selectivity` 进入，不直接跳到最终 plan。
- 记录项 `pattern heuristic` 的边界字段写作：明确这是统计事实、默认假设、catalog 绑定还是成本传播。
- 记录项 `pattern heuristic` 的结论字段写作：小样本模式估算不能被当作精确统计。
- 记录项 `pattern MCV` 的现象字段写作：热点字符串是否被 pattern operator 直接测试。
- 记录项 `pattern MCV` 的源码字段写作：先从 `mcv_selectivity` 进入，不直接跳到最终 plan。
- 记录项 `pattern MCV` 的边界字段写作：明确这是统计事实、默认假设、catalog 绑定还是成本传播。
- 记录项 `pattern MCV` 的结论字段写作：热点字符串可绕过启发式并直接贡献频率。
- 记录项 `取反 NULL` 的现象字段写作：不等或 not-match 估算是否扣除了 null fraction。
- 记录项 `取反 NULL` 的源码字段写作：先从 `eqsel_internal` 进入，不直接跳到最终 plan。
- 记录项 `取反 NULL` 的边界字段写作：明确这是统计事实、默认假设、catalog 绑定还是成本传播。
- 记录项 `取反 NULL` 的结论字段写作：取反估算不能写成简单的一减原概率。
