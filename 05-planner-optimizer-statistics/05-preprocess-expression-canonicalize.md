# PostgreSQL 表达式 canonicalize 与隐式 AND 拆分

## 课程定位

前置知识：已经理解 planner 主流程和 hook/GUC 边界。本节开始进入 preprocessing：在 Path 搜索之前先整理表达式形态。

本节唯一主问题：

```text
为什么 planner 要先把 where / join qual / having 等表达式做常量折叠、函数内联、布尔表达式规范化和 AND clause 拆分，后续选择率、索引匹配和等价类推导依赖哪些形态？
```

核心矛盾：表达式保留原始 SQL 形态有利于对应用户输入，但优化器需要可比较、可拆分、可下推、可估算的规范形态；过早改写会有语义风险，过晚改写又会让后续模块重复处理复杂树形。

学完后应能判断一个 qual 还处于 explicit boolean tree、已经 canonicalize、已经 implicit-AND，还是已经被包装成 RestrictInfo。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

本节是 preprocessing 组的入口。后面三节会说明 canonicalize 之后的 Query 还会经历子查询 pullup、outer join 约束保护和 RestrictInfo 包装。

```text
SQL text
  -> parse / analyze / rewrite
  -> Query
  -> planner preprocessing
  -> RelOptInfo / Path search
  -> Plan
  -> PlannedStmt
  -> executor
```

05 目录关注 planner 如何把语义树转成可比较的候选实现，并最终压缩成执行器契约。本节只围绕自己的唯一主问题展开，相邻主题只在解释当前状态推进时出现。

阅读时要避免从文件顶部线性背函数。更好的顺序是先确认对象生命周期，再看哪些字段被创建、填充、冻结、转移或丢弃。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
`preprocess_expression()` 先按表达式种类处理 join alias、常量折叠、qual canonicalize、SAOP hash 优化、SubLink 转 SubPlan、相关变量替换，最后把 qual 转成 implicit-AND list，供 jointree 分解和 RestrictInfo 分发使用。
```

这个模型里有三个稳定判断。

- 语义状态和搜索状态必须分开。
- planner-local 状态可以很丰富，但不能泄漏给 executor。
- 每一次改写都要能回到后续可验证的 Path、Plan 或 EXPLAIN 现象。

如果读源码时只记函数名，很容易把当前阶段的临时状态误认为最终语义。正确的读法是沿着一个状态对象追踪：它在哪里创建，谁写入关键字段，什么时候被下游消费，什么时候被抛弃。

## 3. 核心文件分工与阅读顺序

| 项 | 说明 |
| --- | --- |
| 1 | `src/backend/optimizer/plan/planner.c`：`preprocess_expression()`、`preprocess_qual_conditions()`，表达式预处理入口。 |
| 2 | `src/backend/optimizer/prep/prepqual.c`：`canonicalize_qual()`、`find_duplicate_ors()`，布尔 qual 规范化。 |
| 3 | `src/backend/optimizer/util/clauses.c`：`eval_const_expressions()`、`inline_function()`、`contain_volatile_functions()`。 |
| 4 | `src/backend/optimizer/plan/subselect.c`：`SS_process_sublinks()`、`SS_replace_correlation_vars()`。 |
| 5 | `src/backend/optimizer/util/orclauses.c`：OR clause 抽取和后续 RestrictInfo 相关处理。 |
| 6 | `src/backend/optimizer/util/predtest.c`：规范化后的 predicate implication 判断。 |
| 7 | `src/include/optimizer/optimizer.h`：canonicalize、constant folding、clause 工具函数声明。 |
| 8 | `src/include/nodes/primnodes.h`：BoolExpr、SubLink、SubPlan、ScalarArrayOpExpr 等表达式节点。 |

源码核对入口：

```text
cd /home/highgo/postgres
git rev-parse --abbrev-ref HEAD
git rev-parse HEAD
```

期望分支是 `master`，提交是本课课程定位中写明的完整提交号。

读这些文件时优先看状态边界，不按文件名排序。先看入口，再看状态结构，随后看 ownership 和 cleanup，最后看观测入口。

## 4. 关键数据结构与状态边界

本节只讲帮助解释主问题的状态，不把结构体字段当百科。

| 项 | 说明 |
| --- | --- |
| EXPRKIND_QUAL | WHERE、JOIN/ON、HAVING 等 qual 处理路径，最终会转 implicit-AND。 |
| EXPRKIND_TARGET | targetlist 处理路径，常量折叠但不会转 implicit-AND。 |
| join alias var | Join RTE 产生的别名 Var，在当前层需要尽早替换成 base relation Var。 |
| explicit AND / OR | 普通 BoolExpr 树，适合表达 SQL 语义但不方便逐 clause 分发。 |
| canonical qual | 经过布尔规范化的表达式，重复 OR、嵌套 AND/OR 等被整理。 |
| implicit-AND list | 顶层 AND 被拆成 List，每个元素可独立进入 RestrictInfo 分发。 |
| SubPlan / Param | SubLink 和相关变量处理后的 planner/executor 传参形态。 |

判断这些状态时有一个通用规则：

```text
raw field 不是语义；
field + 生命周期 + 阶段边界 + 访问者，才是 planner 中可诊断的语义。
```

planner 的很多对象都在普通 backend-local memory context 中分配。它们不是 shared memory，不跨 backend 可见，也不应该被保存到执行期长期结构里。

### 状态切片 1：EXPRKIND_QUAL

WHERE、JOIN/ON、HAVING 等 qual 处理路径，最终会转 implicit-AND。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

### 状态切片 2：EXPRKIND_TARGET

targetlist 处理路径，常量折叠但不会转 implicit-AND。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

### 状态切片 3：join alias var

Join RTE 产生的别名 Var，在当前层需要尽早替换成 base relation Var。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

### 状态切片 4：explicit AND / OR

普通 BoolExpr 树，适合表达 SQL 语义但不方便逐 clause 分发。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

### 状态切片 5：canonical qual

经过布尔规范化的表达式，重复 OR、嵌套 AND/OR 等被整理。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

### 状态切片 6：implicit-AND list

顶层 AND 被拆成 List，每个元素可独立进入 RestrictInfo 分发。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

### 状态切片 7：SubPlan / Param

SubLink 和相关变量处理后的 planner/executor 传参形态。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

## 5. 主流程源码 walkthrough

下面按时间顺序跟主链路。每个小段都只回答一个问题：当前状态发生了什么变化。

| 项 | 说明 |
| --- | --- |
| 空表达式快返回 | `preprocess_expression()` 先处理 NULL，保留 implicit-AND 的 NULL 语义。 |
| flatten join alias var | 有 JOIN RTE 时，当前层表达式中的 join alias 被替换成 base relation 表达式。 |
| `eval_const_expressions()` | 常量折叠、函数默认参数展开、简单函数内联、布尔树扁平化在这里发生。 |
| `canonicalize_qual()` | 仅 qual 路径执行，把布尔表达式整理成后续 predicate / selectivity 更容易消费的形态。 |
| `convert_saop_to_hashed_saop()` | 常量数组的 ANY/IN 可标记 hashfuncid，降低执行期线性查找成本。 |
| `SS_process_sublinks()` | SubLink 被转成 SubPlan 或可规划的参数结构。 |
| `SS_replace_correlation_vars()` | 上层 Var 替换成 PARAM_EXEC，保留 query level 边界。 |
| `make_ands_implicit()` | qual 最终转成顶层 List，供 `deconstruct_jointree()` 分发。 |

### 5.1. 空表达式快返回

`preprocess_expression()` 先处理 NULL，保留 implicit-AND 的 NULL 语义。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

### 5.2. flatten join alias var

有 JOIN RTE 时，当前层表达式中的 join alias 被替换成 base relation 表达式。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

### 5.3. `eval_const_expressions()`

常量折叠、函数默认参数展开、简单函数内联、布尔树扁平化在这里发生。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

这一段常见的误读是把“现在能看到的字段”当成“最终执行器会看到的字段”。多数情况下，当前字段只是下一阶段搜索或整理的材料。

### 5.4. `canonicalize_qual()`

仅 qual 路径执行，把布尔表达式整理成后续 predicate / selectivity 更容易消费的形态。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

### 5.5. `convert_saop_to_hashed_saop()`

常量数组的 ANY/IN 可标记 hashfuncid，降低执行期线性查找成本。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

### 5.6. `SS_process_sublinks()`

SubLink 被转成 SubPlan 或可规划的参数结构。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

这一段常见的误读是把“现在能看到的字段”当成“最终执行器会看到的字段”。多数情况下，当前字段只是下一阶段搜索或整理的材料。

### 5.7. `SS_replace_correlation_vars()`

上层 Var 替换成 PARAM_EXEC，保留 query level 边界。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

### 5.8. `make_ands_implicit()`

qual 最终转成顶层 List，供 `deconstruct_jointree()` 分发。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

## 6. 生命周期 / ownership / cleanup

planner 的 ownership 边界比 executor 更短，但更容易被误用。

| 项 | 说明 |
| --- | --- |
| 创建者 | 当前 Query level 的主要工作状态通常由 `subquery_planner()` 或它调用的初始化函数创建。 |
| 持有者 | `PlannerInfo root` 持有本层 planner 状态，`PlannerGlobal glob` 持有跨层结果。 |
| 释放边界 | 多数对象位于 planner 所在 memory context，规划完成后整体释放，不需要 executor 逐个清理。 |
| 转移边界 | 只有进入 `Plan`、`PlannedStmt`、flat rtable、subplans、rowmarks、invalItems 的状态会越过规划阶段。 |
| 异常路径 | planner 中途 ERROR 时依赖 backend 的 memory context 和 ResourceOwner 清理，不依赖手工释放每个 List 或 Node。 |

这也是为什么扩展 hook 不能把 root 内部指针保存到长期结构里。规划结束后，指针地址可能还像是有效 C 指针，但语义和内存所有权已经结束。

## 7. 正确性机制层次

优化器正确性不是一个函数单独保证的，而是多层边界共同保证。

| 项 | 说明 |
| --- | --- |
| 1 | 不能在 SubLink 展开之前把 explicit AND 过早拆成 implicit list，因为 SubLink 处理还需要表达式树语义。 |
| 2 | 常量折叠要考虑 volatility，volatile 函数不能被当成普通常量提前执行。 |
| 3 | join alias var 要早于 SubLink 处理，否则从 alias 中展开的子表达式可能漏处理。 |
| 4 | qual 和 target 的处理不同，不能把 targetlist 误拆成 implicit-AND。 |
| 5 | canonicalize 改变表达式形态，但必须保持三值逻辑和 outer join 语义。 |

可以把本节正确性压缩成下面的判断链：

```text
语义是否等价？
状态是否在正确阶段产生？
候选是否只影响搜索空间？
最终 Plan 是否脱离 planner-local 指针？
PlannedStmt 是否包含 executor 和 plan cache 需要的依赖？
```

## 8. 错误路径 / 异常路径 / fallback

planner 的 fallback 往往不是抛错，而是保守地缩小改写、缩小搜索空间，或保留较笨但语义可靠的计划形态。

| 项 | 说明 |
| --- | --- |
| 1 | 如果表达式包含 volatile 函数，优化器必须保守，不能把它提前折叠为常量。 |
| 2 | 如果 named-argument function call 未经常量简化路径处理，默认参数和位置参数可能在后续阶段不一致。 |
| 3 | 如果 correlated SubLink 没有替换为 Param，子计划无法从外层 executor 取得值。 |
| 4 | 如果 nested AND/OR 没有保持扁平，后续 clause 遍历会漏掉可下推条件或重复估算。 |

遇到 fallback 时要先问它保护的是语义、生命周期、资源成本还是扩展契约。不同原因对应的修复方向完全不同。

### 异常观察 1

如果表达式包含 volatile 函数，优化器必须保守，不能把它提前折叠为常量。

验证时不要只看最终节点名，还要回到触发保守路径的源码条件。

### 异常观察 2

如果 named-argument function call 未经常量简化路径处理，默认参数和位置参数可能在后续阶段不一致。

验证时不要只看最终节点名，还要回到触发保守路径的源码条件。

### 异常观察 3

如果 correlated SubLink 没有替换为 Param，子计划无法从外层 executor 取得值。

验证时不要只看最终节点名，还要回到触发保守路径的源码条件。

### 异常观察 4

如果 nested AND/OR 没有保持扁平，后续 clause 遍历会漏掉可下推条件或重复估算。

验证时不要只看最终节点名，还要回到触发保守路径的源码条件。

## 9. 成本、资源与跨模块传播

planner 成本不只来自最终计划，也来自规划期搜索空间和状态传播。

| 项 | 说明 |
| --- | --- |
| 1 | preprocess_expression 是一次性 planner CPU 成本，换来后续 selectivity、index matching 和 EC 推导的简化。 |
| 2 | 常量折叠可能减少执行期表达式成本，也可能让更多 clause 成为伪常量 gating qual。 |
| 3 | SAOP hash 标记会增加规划判断，但可降低执行期大 IN 列表的 per-tuple 成本。 |
| 4 | 规范化让后续模块少做重复树遍历，尤其是多表 join 下每个 clause 会被多次检查。 |

这些成本最后会在三个地方表现出来：planner CPU 时间、规划期间内存占用、以及最终 executor 计划的 runtime 成本。诊断时要把三者分开。

## 10. 观测与诊断入口

优化器内部状态多数不可直接从 SQL 看到，但可以通过最终 Plan、日志和断点间接定位。

| 项 | 说明 |
| --- | --- |
| 1 | `debug_print_rewritten` 与 `debug_print_plan` 的差异能间接观察 preprocessing 效果。 |
| 2 | gdb 断在 `preprocess_expression()` 前后，打印同一个 qual 可以看到 explicit AND 到 implicit list 的变化。 |
| 3 | 对 `WHERE 1=1 AND a=1` 类查询观察最终 Plan，可看到常量真条件被消除。 |
| 4 | 对大 IN 常量列表断在 `convert_saop_to_hashed_saop()`，可观察 `ScalarArrayOpExpr` 的 hashfuncid。 |

推荐的最小诊断路径：

```text
EXPLAIN (ANALYZE, VERBOSE, BUFFERS)
  -> 找到第一个估算或节点选择异常的位置
  -> 回到对应 planner 阶段
  -> 在源码入口断点观察 root / rel / path / rinfo
  -> 用 GUC 或 SQL 最小化验证假设
```

## 11. 常见误区

- 误区 1：把最终 Plan 当成 planner 搜索过程的完整记录。
- 误区 2：把 enable_* GUC 当成绝对语义开关。
- 误区 3：把 cost 数字当成毫秒。
- 误区 4：把 Query、Path、Plan、PlannedStmt 混成同一个生命周期。
- 误区 5：看到某个函数名就认为它承担全部阶段职责。
- 误区 6：忽略 outer join、security barrier、volatile、lateral 这类语义边界。

这些误区的共同原因是只看静态结构，不看状态随阶段推进的时间线。

## 12. 课堂实验

### 实验 1：常量折叠

执行 `WHERE a = 1 AND 2 + 2 = 4`，跟踪 qual 在 `eval_const_expressions()` 后的变化。

实验记录至少包含三项：SQL、EXPLAIN 输出中的关键差异、对应源码断点或函数入口。

### 实验 2：volatile 对比

比较 `WHERE random() < 0.5` 和 `WHERE immutable_func(1)=...`，观察折叠差异。

实验记录至少包含三项：SQL、EXPLAIN 输出中的关键差异、对应源码断点或函数入口。

### 实验 3：implicit AND

断在 `preprocess_qual_conditions()` 后打印 `FromExpr->quals`，确认顶层 AND 已变成 List。

实验记录至少包含三项：SQL、EXPLAIN 输出中的关键差异、对应源码断点或函数入口。

### 实验 4：SubLink

用 `WHERE EXISTS (...)`，观察 `SS_process_sublinks()` 是否生成 SubPlan 或 join 化路径。

实验记录至少包含三项：SQL、EXPLAIN 输出中的关键差异、对应源码断点或函数入口。

实验不追求覆盖所有路径，只追求把一个可见现象解释回源码中的状态变化。

## 13. 讨论题

1. 为什么 canonicalize 是 planner 阶段而不是 parser 阶段完成？

2. 常量折叠越激进越好吗？volatile 和 security barrier 会限制哪些场景？

3. 如果一个 OR clause 既能帮助索引又很难估算，应该优先保留原形还是拆分？

回答这些问题时，需要明确引用源码阶段和状态对象，避免只用抽象概念解释抽象概念。

## 14. 源码分段阅读

这一节把第 3 节的源码顺序再拆成可执行的阅读任务。每个任务都要能回到本节唯一主问题，而不是停留在“看过这个文件”。

### 14.1. `src/backend/optimizer/plan/planner.c`

`preprocess_expression()`、`preprocess_qual_conditions()`，表达式预处理入口。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

### 14.2. `src/backend/optimizer/prep/prepqual.c`

`canonicalize_qual()`、`find_duplicate_ors()`，布尔 qual 规范化。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

### 14.3. `src/backend/optimizer/util/clauses.c`

`eval_const_expressions()`、`inline_function()`、`contain_volatile_functions()`。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

### 14.4. `src/backend/optimizer/plan/subselect.c`

`SS_process_sublinks()`、`SS_replace_correlation_vars()`。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

### 14.5. `src/backend/optimizer/util/orclauses.c`

OR clause 抽取和后续 RestrictInfo 相关处理。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

### 14.6. `src/backend/optimizer/util/predtest.c`

规范化后的 predicate implication 判断。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

### 14.7. `src/include/optimizer/optimizer.h`

canonicalize、constant folding、clause 工具函数声明。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

### 14.8. `src/include/nodes/primnodes.h`

BoolExpr、SubLink、SubPlan、ScalarArrayOpExpr 等表达式节点。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

## 15. 状态迁移检查

下面把主流程再拆成状态迁移。它们不是新的知识点，而是防止把函数调用链读成 API 清单。

### 15.1. 空表达式快返回

`preprocess_expression()` 先处理 NULL，保留 implicit-AND 的 NULL 语义。

进入前先确认 `EXPRKIND_QUAL` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `EXPRKIND_TARGET` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.2. flatten join alias var

有 JOIN RTE 时，当前层表达式中的 join alias 被替换成 base relation 表达式。

进入前先确认 `EXPRKIND_TARGET` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `join alias var` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.3. `eval_const_expressions()`

常量折叠、函数默认参数展开、简单函数内联、布尔树扁平化在这里发生。

进入前先确认 `join alias var` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `explicit AND / OR` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.4. `canonicalize_qual()`

仅 qual 路径执行，把布尔表达式整理成后续 predicate / selectivity 更容易消费的形态。

进入前先确认 `explicit AND / OR` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `canonical qual` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.5. `convert_saop_to_hashed_saop()`

常量数组的 ANY/IN 可标记 hashfuncid，降低执行期线性查找成本。

进入前先确认 `canonical qual` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `implicit-AND list` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.6. `SS_process_sublinks()`

SubLink 被转成 SubPlan 或可规划的参数结构。

进入前先确认 `implicit-AND list` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `SubPlan / Param` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.7. `SS_replace_correlation_vars()`

上层 Var 替换成 PARAM_EXEC，保留 query level 边界。

进入前先确认 `SubPlan / Param` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `EXPRKIND_QUAL` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.8. `make_ands_implicit()`

qual 最终转成顶层 List，供 `deconstruct_jointree()` 分发。

进入前先确认 `EXPRKIND_QUAL` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `EXPRKIND_TARGET` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

## 16. 现象到源码的回溯路径

面对慢 SQL 或计划异常时，可以从可观察现象倒推源码阶段。

| 项 | 说明 |
| --- | --- |
| 最终节点类型变化 | 先比较 enable_* 和 cost GUC，再回到 Path 生成与 add_path 剪枝。 |
| 估算行数偏差 | 先定位第一个 rows 失真节点，再回到 RestrictInfo、统计信息和 selectivity。 |
| Subquery Scan 出现 | 先检查 pullup 阻止条件，再看 security barrier、limit、聚合或 lateral。 |
| Join Filter 没有下推 | 先检查 outer join 边界、required_relids、security_level 和 is_pushed_down。 |
| 计划无法并行 | 先看 parallel hazard、Path 的 parallel_safe 和最终 Gather 边界。 |
| 索引没有使用 | 先确认 qual 是否进入 baserestrictinfo，再看 operator family、collation 和 index path 匹配。 |
| 计划缓存失效频繁 | 先看 PlannedStmt 中 relationOids、invalItems、dependsOnRole。 |
| 扩展 hook 改变计划 | 先确认 hook 插入的是 Path 候选、join search，还是替换了 whole planner。 |

### 16.1. 最终节点类型变化

先比较 enable_* 和 cost GUC，再回到 Path 生成与 add_path 剪枝。

回溯时保留一个最小 SQL，不要同时修改统计信息、GUC、索引和 SQL 结构，否则很难判断触发条件。

确认源码阶段以后，再决定是修 SQL、修统计信息、调成本参数、加索引，还是接受该语义边界。

### 16.2. 估算行数偏差

先定位第一个 rows 失真节点，再回到 RestrictInfo、统计信息和 selectivity。

回溯时保留一个最小 SQL，不要同时修改统计信息、GUC、索引和 SQL 结构，否则很难判断触发条件。

确认源码阶段以后，再决定是修 SQL、修统计信息、调成本参数、加索引，还是接受该语义边界。

### 16.3. Subquery Scan 出现

先检查 pullup 阻止条件，再看 security barrier、limit、聚合或 lateral。

回溯时保留一个最小 SQL，不要同时修改统计信息、GUC、索引和 SQL 结构，否则很难判断触发条件。

确认源码阶段以后，再决定是修 SQL、修统计信息、调成本参数、加索引，还是接受该语义边界。

### 16.4. Join Filter 没有下推

先检查 outer join 边界、required_relids、security_level 和 is_pushed_down。

回溯时保留一个最小 SQL，不要同时修改统计信息、GUC、索引和 SQL 结构，否则很难判断触发条件。

确认源码阶段以后，再决定是修 SQL、修统计信息、调成本参数、加索引，还是接受该语义边界。

### 16.5. 计划无法并行

先看 parallel hazard、Path 的 parallel_safe 和最终 Gather 边界。

回溯时保留一个最小 SQL，不要同时修改统计信息、GUC、索引和 SQL 结构，否则很难判断触发条件。

确认源码阶段以后，再决定是修 SQL、修统计信息、调成本参数、加索引，还是接受该语义边界。

### 16.6. 索引没有使用

先确认 qual 是否进入 baserestrictinfo，再看 operator family、collation 和 index path 匹配。

回溯时保留一个最小 SQL，不要同时修改统计信息、GUC、索引和 SQL 结构，否则很难判断触发条件。

确认源码阶段以后，再决定是修 SQL、修统计信息、调成本参数、加索引，还是接受该语义边界。

### 16.7. 计划缓存失效频繁

先看 PlannedStmt 中 relationOids、invalItems、dependsOnRole。

回溯时保留一个最小 SQL，不要同时修改统计信息、GUC、索引和 SQL 结构，否则很难判断触发条件。

确认源码阶段以后，再决定是修 SQL、修统计信息、调成本参数、加索引，还是接受该语义边界。

### 16.8. 扩展 hook 改变计划

先确认 hook 插入的是 Path 候选、join search，还是替换了 whole planner。

回溯时保留一个最小 SQL，不要同时修改统计信息、GUC、索引和 SQL 结构，否则很难判断触发条件。

确认源码阶段以后，再决定是修 SQL、修统计信息、调成本参数、加索引，还是接受该语义边界。

## 17. 源码阅读检查清单

- 检查点 1：在 `src/backend/optimizer/plan/planner.c` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 2：在 `src/backend/optimizer/prep/prepqual.c` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 3：在 `src/backend/optimizer/util/clauses.c` 中验证：字段是否被后续阶段继续改写。
- 检查点 4：在 `src/backend/optimizer/plan/subselect.c` 中验证：字段是否会越过 planner 生命周期。
- 检查点 5：在 `src/backend/optimizer/util/orclauses.c` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 6：在 `src/backend/optimizer/util/predtest.c` 中验证：最终计划是否能从源码中的同一条状态线解释。
- 检查点 7：在 `src/include/optimizer/optimizer.h` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 8：在 `src/include/nodes/primnodes.h` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 9：在 `src/backend/optimizer/plan/planner.c` 中验证：字段是否被后续阶段继续改写。
- 检查点 10：在 `src/backend/optimizer/prep/prepqual.c` 中验证：字段是否会越过 planner 生命周期。
- 检查点 11：在 `src/backend/optimizer/util/clauses.c` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 12：在 `src/backend/optimizer/plan/subselect.c` 中验证：最终计划是否能从源码中的同一条状态线解释。
- 检查点 13：在 `src/backend/optimizer/util/orclauses.c` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 14：在 `src/backend/optimizer/util/predtest.c` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 15：在 `src/include/optimizer/optimizer.h` 中验证：字段是否被后续阶段继续改写。
- 检查点 16：在 `src/include/nodes/primnodes.h` 中验证：字段是否会越过 planner 生命周期。
- 检查点 17：在 `src/backend/optimizer/plan/planner.c` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 18：在 `src/backend/optimizer/prep/prepqual.c` 中验证：最终计划是否能从源码中的同一条状态线解释。
- 检查点 19：在 `src/backend/optimizer/util/clauses.c` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 20：在 `src/backend/optimizer/plan/subselect.c` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 21：在 `src/backend/optimizer/util/orclauses.c` 中验证：字段是否被后续阶段继续改写。
- 检查点 22：在 `src/backend/optimizer/util/predtest.c` 中验证：字段是否会越过 planner 生命周期。
- 检查点 23：在 `src/include/optimizer/optimizer.h` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 24：在 `src/include/nodes/primnodes.h` 中验证：最终计划是否能从源码中的同一条状态线解释。
- 检查点 25：在 `src/backend/optimizer/plan/planner.c` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 26：在 `src/backend/optimizer/prep/prepqual.c` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 27：在 `src/backend/optimizer/util/clauses.c` 中验证：字段是否被后续阶段继续改写。
- 检查点 28：在 `src/backend/optimizer/plan/subselect.c` 中验证：字段是否会越过 planner 生命周期。
- 检查点 29：在 `src/backend/optimizer/util/orclauses.c` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 30：在 `src/backend/optimizer/util/predtest.c` 中验证：最终计划是否能从源码中的同一条状态线解释。
- 检查点 31：在 `src/include/optimizer/optimizer.h` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 32：在 `src/include/nodes/primnodes.h` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 33：在 `src/backend/optimizer/plan/planner.c` 中验证：字段是否被后续阶段继续改写。
- 检查点 34：在 `src/backend/optimizer/prep/prepqual.c` 中验证：字段是否会越过 planner 生命周期。
- 检查点 35：在 `src/backend/optimizer/util/clauses.c` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 36：在 `src/backend/optimizer/plan/subselect.c` 中验证：最终计划是否能从源码中的同一条状态线解释。
- 检查点 37：在 `src/backend/optimizer/util/orclauses.c` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 38：在 `src/backend/optimizer/util/predtest.c` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 39：在 `src/include/optimizer/optimizer.h` 中验证：字段是否被后续阶段继续改写。
- 检查点 40：在 `src/include/nodes/primnodes.h` 中验证：字段是否会越过 planner 生命周期。
- 检查点 41：在 `src/backend/optimizer/plan/planner.c` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 42：在 `src/backend/optimizer/prep/prepqual.c` 中验证：最终计划是否能从源码中的同一条状态线解释。
- 检查点 43：在 `src/backend/optimizer/util/clauses.c` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 44：在 `src/backend/optimizer/plan/subselect.c` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 45：在 `src/backend/optimizer/util/orclauses.c` 中验证：字段是否被后续阶段继续改写。
- 检查点 46：在 `src/backend/optimizer/util/predtest.c` 中验证：字段是否会越过 planner 生命周期。
- 检查点 47：在 `src/include/optimizer/optimizer.h` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 48：在 `src/include/nodes/primnodes.h` 中验证：最终计划是否能从源码中的同一条状态线解释。

## 18. 误判与校正

下面这些条目用于把课堂讨论落到排查动作上。每一条都从一个容易误判的现象开始，再给出回到源码的校正路径。

### 18.1. 正确性边界

不能在 SubLink 展开之前把 explicit AND 过早拆成 implicit list，因为 SubLink 处理还需要表达式树语义。

先确认语义是否保持，再确认候选路径是否只是搜索空间变化。

源码回看点：`src/backend/optimizer/plan/planner.c`。

### 18.2. 正确性边界

常量折叠要考虑 volatility，volatile 函数不能被当成普通常量提前执行。

先确认语义是否保持，再确认候选路径是否只是搜索空间变化。

源码回看点：`src/backend/optimizer/prep/prepqual.c`。

### 18.3. 正确性边界

join alias var 要早于 SubLink 处理，否则从 alias 中展开的子表达式可能漏处理。

先确认语义是否保持，再确认候选路径是否只是搜索空间变化。

源码回看点：`src/backend/optimizer/util/clauses.c`。

### 18.4. 正确性边界

qual 和 target 的处理不同，不能把 targetlist 误拆成 implicit-AND。

先确认语义是否保持，再确认候选路径是否只是搜索空间变化。

源码回看点：`src/backend/optimizer/plan/subselect.c`。

### 18.5. 正确性边界

canonicalize 改变表达式形态，但必须保持三值逻辑和 outer join 语义。

先确认语义是否保持，再确认候选路径是否只是搜索空间变化。

源码回看点：`src/backend/optimizer/util/orclauses.c`。

### 18.6. 异常或 fallback

如果表达式包含 volatile 函数，优化器必须保守，不能把它提前折叠为常量。

先找到保守分支的源码条件，再判断它保护的是语义、资源还是生命周期。

源码回看点：`src/backend/optimizer/util/predtest.c`。

### 18.7. 异常或 fallback

如果 named-argument function call 未经常量简化路径处理，默认参数和位置参数可能在后续阶段不一致。

先找到保守分支的源码条件，再判断它保护的是语义、资源还是生命周期。

源码回看点：`src/include/optimizer/optimizer.h`。

### 18.8. 异常或 fallback

如果 correlated SubLink 没有替换为 Param，子计划无法从外层 executor 取得值。

先找到保守分支的源码条件，再判断它保护的是语义、资源还是生命周期。

源码回看点：`src/include/nodes/primnodes.h`。

### 18.9. 异常或 fallback

如果 nested AND/OR 没有保持扁平，后续 clause 遍历会漏掉可下推条件或重复估算。

先找到保守分支的源码条件，再判断它保护的是语义、资源还是生命周期。

源码回看点：`src/backend/optimizer/plan/planner.c`。

### 18.10. 成本传播

preprocess_expression 是一次性 planner CPU 成本，换来后续 selectivity、index matching 和 EC 推导的简化。

同时记录 planner CPU、候选数量和 executor runtime，避免把三种成本混在一起。

源码回看点：`src/backend/optimizer/prep/prepqual.c`。

### 18.11. 成本传播

常量折叠可能减少执行期表达式成本，也可能让更多 clause 成为伪常量 gating qual。

同时记录 planner CPU、候选数量和 executor runtime，避免把三种成本混在一起。

源码回看点：`src/backend/optimizer/util/clauses.c`。

### 18.12. 成本传播

SAOP hash 标记会增加规划判断，但可降低执行期大 IN 列表的 per-tuple 成本。

同时记录 planner CPU、候选数量和 executor runtime，避免把三种成本混在一起。

源码回看点：`src/backend/optimizer/plan/subselect.c`。

### 18.13. 成本传播

规范化让后续模块少做重复树遍历，尤其是多表 join 下每个 clause 会被多次检查。

同时记录 planner CPU、候选数量和 executor runtime，避免把三种成本混在一起。

源码回看点：`src/backend/optimizer/util/orclauses.c`。

### 18.14. 观测入口

`debug_print_rewritten` 与 `debug_print_plan` 的差异能间接观察 preprocessing 效果。

用 EXPLAIN 或 gdb 只确认一个假设，确认后再扩大到完整 SQL。

源码回看点：`src/backend/optimizer/util/predtest.c`。

### 18.15. 观测入口

gdb 断在 `preprocess_expression()` 前后，打印同一个 qual 可以看到 explicit AND 到 implicit list 的变化。

用 EXPLAIN 或 gdb 只确认一个假设，确认后再扩大到完整 SQL。

源码回看点：`src/include/optimizer/optimizer.h`。

### 18.16. 观测入口

对 `WHERE 1=1 AND a=1` 类查询观察最终 Plan，可看到常量真条件被消除。

用 EXPLAIN 或 gdb 只确认一个假设，确认后再扩大到完整 SQL。

源码回看点：`src/include/nodes/primnodes.h`。

### 18.17. 观测入口

对大 IN 常量列表断在 `convert_saop_to_hashed_saop()`，可观察 `ScalarArrayOpExpr` 的 hashfuncid。

用 EXPLAIN 或 gdb 只确认一个假设，确认后再扩大到完整 SQL。

源码回看点：`src/backend/optimizer/plan/planner.c`。

## 19. 本节小结

- 表达式 preprocessing 的目标不是美化语法树，而是把 qual 变成后续估算、下推和索引匹配能稳定消费的形态。
- `preprocess_expression()` 的顺序本身就是正确性边界。
- 看到最终计划少了某个条件时，先判断它是被折叠、变成 gating qual、进入 SubPlan，还是被包装到 RestrictInfo 后下推。

把本节内容压缩成一个可迁移规律：

```text
复杂优化器的关键不是把所有规则塞进一个入口，
而是把语义改写、候选搜索、成本比较和执行契约放在不同生命周期里。
```
