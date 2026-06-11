# PostgreSQL Predicate 下推、延迟执行与 join qual 分类

## 课程定位

前置知识：熟悉 `Query`、jointree、outer join 的 null-extension 语义，以及 `RestrictInfo` 中 relids 元数据的基本含义。

本节唯一主问题：

```text
同一个 SQL predicate 为什么有时能提前变成 base restriction，有时只能作为 join qual，有时还必须延迟到 outer join 之后执行？
```

核心矛盾：越早执行 predicate，越能减少 rows、打开索引路径和缩小 join 搜索空间；但 outer join、security barrier、LATERAL、nullable side 和参数化路径会让过早执行直接改变 SQL 语义。

学完后应能判断一个 qual 当前能挂到哪个 `RelOptInfo`，为什么 `RINFO_IS_PUSHED_DOWN()` 不能被简化成只看 `is_pushed_down`，以及哪些计划差异来自合法性边界而不是 cost 参数。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

前面课程已经把表达式 canonicalize、子查询 pullup、outer join reduction 和 `RestrictInfo` 包装讲清楚。

05 目录的主线是 planner 如何把 SQL 语义、统计事实和成本模型压缩成可执行计划。本节只处理其中一个局部问题，不把相邻主题合并成百科式说明。

```text
SQL / Query 形态
  -> planner-local 状态
  -> 统计或约束参与判断
  -> Path rows / cost 改变
  -> Plan 节点体现选择结果
```

下一节会离开 qual 分发，进入 `ANALYZE` 如何为这些 qual 提供选择率输入。

因此，本节的阅读边界是明确的：只解释这一个判断如何成立、如何失败、如何被观测，不追求覆盖优化器的所有入口。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
planner 先把表达式包装成携带 relids 和安全级别的 `RestrictInfo`，再按 required relids、outer join 作用域和可移动性把它放入 base restriction、joininfo、EC 派生列表或延迟处理列表。
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
| 1 | `src/backend/optimizer/plan/initsplan.c` | `distribute_qual_to_rels()`、`distribute_restrictinfo_to_rels()` 负责把原始 qual 放到正确的语义层级。 |
| 2 | `src/backend/optimizer/util/restrictinfo.c` | `make_restrictinfo()`、`join_clause_is_movable_to()`、`join_clause_is_movable_into()` 解释可移动性的局部规则。 |
| 3 | `src/include/nodes/pathnodes.h` | `RestrictInfo`、`SpecialJoinInfo`、`RINFO_IS_PUSHED_DOWN()` 给出状态字段和上下文判断。 |
| 4 | `src/include/optimizer/restrictinfo.h` | 声明 RestrictInfo 构造和 join clause 移动检查入口。 |
| 5 | `src/backend/optimizer/path/allpaths.c` | `qual_is_pushdown_safe()` 展示 subquery path 中另一个 pushdown 安全边界。 |
| 6 | `src/backend/optimizer/path/joinpath.c` | join path 生成时区分 outer join qual 和普通 pushed-down filter。 |
| 7 | `src/backend/optimizer/util/relnode.c` | parameterized joinrel 中判断 clause 是否能在当前 relids 上执行。 |
| 8 | `src/backend/optimizer/plan/createplan.c` | 最终 plan 生成时把 joinquals 与 otherquals 交给 executor。 |

推荐阅读方式是先从入口函数建立时间轴，再读结构体字段，最后读 fallback 和观测点。不要按文件名排序读，也不要在第一个复杂分支里停太久。

```text
入口函数
  -> 本地状态结构
  -> 判断条件
  -> fallback 或保守路径
  -> rows / cost / catalog / plan 的可见结果
```

- 读本节源码时，始终把问题压回这一句：同一个 SQL predicate 为什么有时能提前变成 base restriction，有时只能作为 join qual，有时还必须延迟到 outer join 之后执行？
- 如果一个分支不能改变这个问题的答案，只做定位提示，不展开成旁支。
- 如果一个字段看起来像结论，要继续追问它由哪个阶段写入、被哪个阶段读取。
- 如果 SQL 现象和源码注释看起来冲突，先确认当前路径是否进入 fallback。
- 如果不同版本实现细节变化，优先保留当前源码中能验证的不变量。

## 4. 关键数据结构与状态边界

本节的状态核心不是 `RestrictInfo` 这个节点名，而是它把一个布尔表达式从“语法树上的一段文本”变成 planner 可移动、可估算、可延迟的对象。

`RestrictInfo` 的字段要放在同一个时间轴里理解：

```text
jointree 中发现 qual
  -> 计算引用关系和 outer join 作用域
  -> 包装成 RestrictInfo
  -> 分发到 baserestrictinfo / joininfo / EC / outer join clause list
  -> join path 和 createplan 再按当前 joinrelids 消费
```

| 状态 | 源码语义 |
| --- | --- |
| `clause` | 原始表达式本体，后续选择率、执行计划和 executor qual 都会回到它。 |
| `clause_relids` | 表达式实际引用的 base rel 与可能的 nulling relid；它回答“计算这个表达式需要哪些值”。 |
| `required_relids` | `make_restrictinfo()` 给出的执行层级下界；它回答“这个 qual 最早能在哪个 relids 集合上成立”。 |
| `outer_relids` | 来自 outer join 非空侧的禁止区；命中后不能把 clause 推进该侧。 |
| `nullable_relids` / nulling relids | 变量是否可能已经被某个 outer join 置 NULL；它影响移动后表达式值是否等价。 |
| `is_pushed_down` | 记录该 qual 是否按普通 filter 语义处理；解释时必须结合 `RINFO_IS_PUSHED_DOWN(rinfo, joinrelids)`。 |
| `pseudoconstant` | 不引用 base rel 且不含 volatile 函数的表达式可成为 gating qual。 |
| `security_level` / `leakproof` | security barrier 下的执行顺序约束，防止低权限表达式借错误信息泄漏高权限行。 |
| `has_clone` / `is_clone` / `incompatible_relids` | outer join 交换和 clone clause 的安全边界，限制副本在哪些 join order 中可用。 |

`clause_relids` 和 `required_relids` 很容易被混用。

`clause_relids` 更接近表达式本体：它从 `pull_varnos()` 和 nulling relids 来。

`required_relids` 更接近执行承诺：它在 `make_restrictinfo()` 时把 outer join 作用域、pseudoconstant、clone 信息一起压进来。

诊断时如果只看 `clause_relids`，会错过 outer join 把执行层级强行抬高的情况。

`is_pushed_down` 也不是独立结论。

`pathnodes.h` 里的 `RINFO_IS_PUSHED_DOWN(rinfo, joinrelids)` 明确把布尔标记和当前 joinrelids 放在一起判断。

同一个 `RestrictInfo` 在不同 joinrel 上被解释时，可能处在不同语义上下文。

这就是为什么课程标题里同时写 pushdown 和 delay：

```text
pushdown 不是“越早越好”；
delay 也不是“优化失败”；
两者都是为了让 qual 在第一个语义合法的位置被消费。
```

这些状态全部是 planner-local 状态。

它们挂在一次规划的 memory context 中，不进入共享内存，也不会跨 backend 复用。

长期有效的是 SQL 语义和 catalog 定义，不是某次规划中产生的 `RestrictInfo *` 指针。

## 5. 从 SQL 现象进入源码

本节适合从 `EXPLAIN` 的 qual 位置开始读源码。

同一个谓词可能出现在这些位置：

```text
Index Cond
Filter
Join Filter
Hash Cond / Merge Cond
One-Time Filter
```

这些位置不是装饰文本，它们通常对应不同的 planner 分类。

`Index Cond` 说明该表达式或其派生形式进入了 index path 的可用条件。

`Filter` 出现在 base scan 上，说明它已经能在单表层级执行，但没有变成 index qual。

`Join Filter` 常见于 outer join 语义不能提前过滤的场景。

`Hash Cond` / `Merge Cond` 通常说明等值关系进入了 join method 可消费的条件集合，很多时候还经过 EquivalenceClass 处理。

`One-Time Filter` 则提示 pseudoconstant qual 被 `createplan.c` 拉出来做 gating。

一个可靠的观察流程是：

```text
先固定 SQL 和数据
  -> 用 EXPLAIN 看 qual 位置
  -> 改写 outer join / WHERE / ON 位置
  -> 比较 Filter 与 Join Filter 是否移动
  -> 再到 distribute_qual_to_rels() 断点核对 relids 和 is_pushed_down
```

如果关闭 `enable_hashjoin`、`enable_mergejoin` 或 `enable_nestloop` 后 qual 位置没有移动，说明问题更可能在合法性分类，而不是某个 join method 的 cost。

如果把 `LEFT JOIN ... ON` 条件移到 `WHERE` 后结果行数变化，说明这个 predicate 不是自由可移动的普通 filter。

如果 security barrier 视图下 predicate 没有穿透到更底层，不应先怀疑 cost 模型；要先检查 `security_level` 和函数 leakproof 属性。

如果 LATERAL 场景中 qual 被延迟，断点应先看 `distribute_qual_to_rels()` 中 “Vars outside syntactic scope” 的 postponed 路径。

本节的观测结论要谨慎表述：

```text
EXPLAIN 能告诉你 predicate 最终在哪个 plan 节点执行；
源码断点才能说明它为什么被允许或禁止移动到那里。
```

## 6. 主流程源码 walkthrough

主链路从 `src/backend/optimizer/plan/initsplan.c` 的 `deconstruct_jointree()` 和 `distribute_qual_to_rels()` 开始。

第一步，jointree 递归遍历时把 WHERE、JOIN/ON 和 outer join 附属条件交给 `distribute_qual_to_rels()`。

这一步的重点是语法位置还没有变成执行层级。

`qualscope` 表达该 qual 在 SQL jointree 中覆盖的范围。

`ojscope` 表达 outer join qual 如果必须留在 join 层，需要形成的最小 relids 集合。

`outerjoin_nonnullable` 表达 outer join 非空侧，是最重要的禁止下推边界。

第二步，`pull_varnos(root, clause)` 得到表达式实际引用的 relids。

如果这些 relids 不在当前 `qualscope` 中，通常意味着 LATERAL pullup 让表达式引用了外层 relation。

源码不会硬塞到当前层级，而是沿 `JoinTreeItem` parent 向上找包含全部引用的层级，并把 clause 放进 `lateral_clauses`。

这是 delay 的第一类来源：不是 cost 不想下推，而是当前层级没有参数来源。

第三步，变量为空的 qual 走 pseudoconstant 分支。

outer join 上的变量空 qual 仍然必须留在 outer join 语义层。

含 volatile 函数的变量空 qual 会留在原始 syntactic level。

只有不含 volatile 函数且处在可接受 join domain 中时，才会标记 `pseudoconstant`，后续由 plan 生成阶段形成 gating node。

第四步，源码判断 outer join 是否要求延迟。

如果 qual 命中 `outerjoin_nonnullable`，它是 non-degenerate outer join qual。

这类 qual 如果提前推入非空侧，失败的行会被删除，而不是在 outer join 之后补成 NULL 行。

源码因此把 `is_pushed_down` 置为 false，并把 `relids` 强制设为 `ojscope`。

这一步是 correctness hard boundary。

第五步，普通 WHERE、INNER JOIN qual 和 degenerate outer join qual 会被标记为 pushed-down。

这里的 degenerate 指它不引用 outer join 非空侧，因此放到 nullable side 或更低层级不会改变 null-extension 结果。

这一步之后还会调用 `check_redundant_nullability_qual()`，例如某些被 lower antijoin 覆盖的 `IS NULL` 会被丢弃，避免错误选择率继续污染 path。

第六步，`make_restrictinfo()` 创建 `RestrictInfo`。

这不是简单包一层节点。

它把 `is_pushed_down`、`pseudoconstant`、`security_level`、`required_relids`、`outer_relids`、`incompatible_relids` 等后续判断需要的信息收束到一起。

第七步，mergejoinable clause 先进入 EquivalenceClass 逻辑。

如果 `process_equivalence()` 接收它，原始 restrictinfo 不会直接进入 rel 的 clause list。

如果 outer join 语义限制 EC 推导，源码可能把它放到 `left_join_clauses`、`right_join_clauses` 或 `full_join_clauses`，留给后续外连接等值推导使用。

第八步，没有被 EC 或 outer join set-aside 消费的普通条件进入 `distribute_restrictinfo_to_rels()`。

单 relation 条件进入 `RelOptInfo.baserestrictinfo`。

多 relation 条件进入相关 base rel 的 `joininfo`。

后续 `join_clause_is_movable_to()` 和 `join_clause_is_movable_into()` 会在 parameterized path 或 joinrel 构造时再次检查当前上下文是否真的具备执行条件。

第九步，`createplan.c` 生成执行计划时把 join 层的 qual 拆给 executor。

outer join 自身语义条件进入 `joinqual`。

普通 pushed-down filter 在 join 节点上表现为 `otherqual`。

这就是 `EXPLAIN` 中 `Join Filter` 与普通 `Filter` 差异的源码根源。

## 7. 生命周期 / ownership / cleanup

`RestrictInfo` 的创建者是 planner，不是 executor。

它通常由 `make_restrictinfo()` 通过 `palloc` 放在当前 planning context 中。

持有者不是单一 owner，而是多个 planner 状态列表：

```text
RelOptInfo.baserestrictinfo
RelOptInfo.joininfo
PlannerInfo.left_join_clauses / right_join_clauses / full_join_clauses
EquivalenceClass 派生结构
参数化路径的 ppi_clauses
```

这些列表共享的是一次规划内的指针。

规划成功后，plan tree 只保留 executor 需要的表达式和 qual 分类，不保留这些 planner-only 链表作为运行时协议。

规划期间 ERROR 时，普通 `palloc` 对象由 memory context 统一释放。

这里没有 buffer pin、shared refcount、WAL 或 heavyweight lock 的 cleanup 问题。

真正需要小心的是 syscache、relcache 和表达式树引用边界。

如果某个阶段拿到 catalog tuple 或 detoast 后的统计数组，必须遵守相应 release/free 约定。

`RestrictInfo` 自身的生命周期短，但它承载的结论会影响后续 rows、path 和 plan node。

因此不要把一次 gdb 里看到的 `required_relids` 当作 catalog 事实。

它只是当前 SQL、当前 rewrite 结果、当前 outer join reduction 结果和当前 planner memory context 下的局部状态。

## 8. 正确性机制层次

本节的正确性不是“估算准不准”，而是“predicate 移动后 SQL 结果是否仍等价”。

第一层是 outer join null-extension。

non-degenerate outer join qual 不能推进非空侧。

否则原本应该保留下来并补 NULL 的行会被提前过滤。

第二层是 LATERAL 参数来源。

一个依赖外层 relation 的 qual 只能在外层值已经可用的位置执行。

如果把它放到没有参数来源的 base scan 上，不只是 cost 错，而是表达式无法按语义求值。

第三层是 security barrier。

非 leakproof 条件不能越过更高安全级别的过滤条件。

这里保护的不是 rows 估算，而是错误信息、函数副作用和可观察行为可能造成的信息泄漏。

第四层是 EquivalenceClass 与 outer join 的关系。

等值条件能传播出更多可用 qual，但 outer join 下的等值推导必须保留 nulling 语义。

第五层是 parameterized path。

`join_clause_is_movable_to()` 要求 clause 实际引用目标 rel，不能进入 outer join 的 outer side，目标 rel 的 Vars 不能被低层 outer join 置 NULL，也不能违反 LATERAL 方向。

`join_clause_is_movable_into()` 还要确认当前 relids 加 outer relids 能提供表达式所有变量。

这些检查解释了一个常见现象：

```text
一个 qual 看起来只差一点就能下推，
但只要缺一个语义前提，planner 就宁愿保留更高执行层级。
```

## 9. 错误路径 / 异常路径 / fallback

predicate 下推失败时，通常不会报错。

优化器的 fallback 是生成语义正确但候选路径较少的计划。

LATERAL 引用超出当前 `qualscope` 时，clause 会被挂到 parent `JoinTreeItem.lateral_clauses`，等包含所有引用的层级再处理。

non-degenerate outer join qual 如果调用者要求 postponing，会进入 `postponed_oj_qual_list`，稍后在合适的 outer join 层重新分发。

变量空但含 volatile 函数的 qual 不会变成全局 one-time filter，因为提前求值会改变函数调用次数或副作用位置。

security barrier 不满足时，qual 保持在较高安全级别之后执行。

subquery pushdown 在 `allpaths.c` 的 `qual_is_pushdown_safe()` 里还有一套安全判断。

那条路径处理的是 subquery scan 外部 filter 能否进入子查询内部，和本节的 join qual 分类相邻但不能混为一谈。

冗余 `IS NULL` 被 `check_redundant_nullability_qual()` 丢弃时，目标是避免后续 selectivity 被一个语义上已由 antijoin 保证的条件再次扭曲。

这些 fallback 的共同特征是：

```text
少用一个优化机会可以接受；
提前执行导致结果语义变化不可接受。
```

## 10. 成本、资源与跨模块传播

predicate 分类本身不在 `costsize.c` 中完成，但它改变 cost 的输入。

下推到 base rel 后，`set_baserel_size_estimates()` 会在更低层级计算 rows。

如果进入 index qual，`indxpath.c` 能生成更多 index path 或 bitmap path。

如果留在 join 层，base rel 的 rows 不会提前缩小，join search 的候选规模和成本比较都会不同。

如果进入 EquivalenceClass，可能派生出新的等值约束，从而打开 merge join、hash join 或索引匹配机会。

如果进入 parameterized path 的 `ppi_clauses`，nested loop 内侧 path 的 rows 和成本会随 outer 参数变化。

资源成本主要来自 planner 时间和搜索空间。

更早过滤通常减少后续 rows，但也可能增加 path 组合、参数化路径和 EC 推导成本。

outer join clone clause 通过 `has_clone`、`is_clone` 和 `incompatible_relids` 限制爆炸，不让每个交换形态都生成等价但无意义的候选。

诊断时可以按这个顺序定位：

```text
先看 qual 最终出现在 EXPLAIN 的哪个节点；
再看 RestrictInfo 的 relids 与 pushed-down 判断；
再看 rows 是否在第一个可执行层级发生变化；
最后才调整 cost GUC 或统计目标。
```

本节的可迁移规律是：

```text
优化器里的“移动”首先是语义问题，其次才是成本问题。
```

## 11. 观测与诊断入口

planner 内部很多状态没有直接 SQL 视图，但可以通过计划形态和 catalog 摘要推断。

`EXPLAIN` 中 base scan 的 `Filter` 或 `Index Cond` 说明 qual 已进入 base rel 层级。

outer join 节点上的 `Join Filter` 往往提示 predicate 被保留在 join 层。

`Rows Removed by Filter` 的位置比数值本身更重要，它显示 predicate 的执行层级。

关闭某类 join GUC 只能改变候选路径，不会让不合法 qual 突然下推。

| 入口 | 能看到什么 | 看不到什么 |
| --- | --- | --- |
| `EXPLAIN` | 估算 rows、cost、Filter 位置和选中 path。 | 未入选的多数候选 path 和中间判断细节。 |
| `EXPLAIN (ANALYZE, BUFFERS)` | 估算与实际的偏差、buffer 访问形态。 | planner 当时的所有统计 slot 读取过程。 |
| `pg_stats` / catalog | 统计摘要、目标、对象定义和部分扩展统计。 | 选择率函数如何组合这些摘要。 |
| gdb 断点 | 当前 `PlannerInfo`、`RelOptInfo`、slot 或依赖结构。 | 没有复现实验时难以判断业务代表性。 |

日常诊断通常先用 SQL 和 EXPLAIN 缩小范围，再用源码断点验证关键状态。

## 12. 课堂实验一：从计划反推状态

第一个实验只改变查询形态或统计状态，目标是让计划差异能回到本节主问题。

构造 `left join`，把右表条件分别写在 `ON` 和 `WHERE` 中，观察结果行数和 Filter 位置差异。

```text
EXPLAIN (ANALYZE, BUFFERS)
  -> 记录 estimated rows 与 actual rows
  -> 记录 Filter / Index Cond / Join Filter 的位置
  -> 对照 pg_stats 或 catalog 状态
```

给右表列建索引，比较 degenerate outer join qual 是否能进入 index condition。

```text
EXPLAIN (ANALYZE, BUFFERS)
  -> 记录 estimated rows 与 actual rows
  -> 记录 Filter / Index Cond / Join Filter 的位置
  -> 对照 pg_stats 或 catalog 状态
```

实验结果不要求某个固定计划，因为硬件、数据量、GUC 和缓存都会影响 cost；要求是解释计划变化时能回到同一个源码判断。

## 13. 课堂实验二：改变输入验证边界

第二个实验改变输入分布或配置，用来验证 fallback 和成本传播。

写一个 LATERAL 子查询，让内层条件引用外层列，观察 parameterized nested loop 的 required_outer。

- 记录变更前的 catalog 摘要。
- 执行一次明确的 `ANALYZE`，避免把统计陈旧误认为源码行为。
- 只改变一个变量，保留其它 GUC 和 SQL 形态。
- 把计划差异写成“输入变化 -> planner 状态变化 -> path 变化”。

用 security barrier view 加非 leakproof 函数，观察 qual 是否被阻止提前执行。

- 记录变更前的 catalog 摘要。
- 执行一次明确的 `ANALYZE`，避免把统计陈旧误认为源码行为。
- 只改变一个变量，保留其它 GUC 和 SQL 形态。
- 把计划差异写成“输入变化 -> planner 状态变化 -> path 变化”。

如果实验没有出现预期差异，先检查数据规模是否太小、统计是否刷新、GUC 是否强行禁用了候选路径。

## 14. 源码阅读练习

源码练习以断点和变量观察为主，不要求修改代码。

- 在 `src/backend/optimizer/plan/initsplan.c` 的入口函数附近设置断点，确认本节主路径是否进入。
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

如果一个 predicate 下推会带来巨大收益，但只在某些 join order 下安全，planner 应该如何表达这种局部机会？

回答时要给出一个可复现实验或一个源码入口，避免只给抽象观点。

为什么 `is_pushed_down` 这种名字容易误导源码阅读者？

回答时要给出一个可复现实验或一个源码入口，避免只给抽象观点。

怎样从计划中的 Filter 位置反推出 `RestrictInfo` 被挂到了哪个层级？

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

断点入口 1：`src/backend/optimizer/plan/initsplan.c`。`distribute_qual_to_rels()`、`distribute_restrictinfo_to_rels()` 负责把原始 qual 放到正确的语义层级。

在这个位置不要一次性展开整份文件。先确认当前调用栈是否服务本节唯一主问题，再决定是否继续向下单步。

如果这里没有被命中，优先检查 SQL 形态、统计是否存在、GUC 是否关闭了相关路径，以及本节机制是否适用于当前 relation 类型。

如果这里被命中但状态为空，通常说明前置 catalog、样本、relids 或 clause 形态没有满足条件。这个结论比“planner 没优化”更有诊断价值。

断点入口 2：`src/backend/optimizer/util/restrictinfo.c`。`make_restrictinfo()`、`join_clause_is_movable_to()`、`join_clause_is_movable_into()` 解释可移动性的局部规则。

在这个位置不要一次性展开整份文件。先确认当前调用栈是否服务本节唯一主问题，再决定是否继续向下单步。

如果这里没有被命中，优先检查 SQL 形态、统计是否存在、GUC 是否关闭了相关路径，以及本节机制是否适用于当前 relation 类型。

如果这里被命中但状态为空，通常说明前置 catalog、样本、relids 或 clause 形态没有满足条件。这个结论比“planner 没优化”更有诊断价值。

断点入口 3：`src/include/nodes/pathnodes.h`。`RestrictInfo`、`SpecialJoinInfo`、`RINFO_IS_PUSHED_DOWN()` 给出状态字段和上下文判断。

在这个位置不要一次性展开整份文件。先确认当前调用栈是否服务本节唯一主问题，再决定是否继续向下单步。

如果这里没有被命中，优先检查 SQL 形态、统计是否存在、GUC 是否关闭了相关路径，以及本节机制是否适用于当前 relation 类型。

如果这里被命中但状态为空，通常说明前置 catalog、样本、relids 或 clause 形态没有满足条件。这个结论比“planner 没优化”更有诊断价值。

断点入口 4：`src/include/optimizer/restrictinfo.h`。声明 RestrictInfo 构造和 join clause 移动检查入口。

在这个位置不要一次性展开整份文件。先确认当前调用栈是否服务本节唯一主问题，再决定是否继续向下单步。

如果这里没有被命中，优先检查 SQL 形态、统计是否存在、GUC 是否关闭了相关路径，以及本节机制是否适用于当前 relation 类型。

如果这里被命中但状态为空，通常说明前置 catalog、样本、relids 或 clause 形态没有满足条件。这个结论比“planner 没优化”更有诊断价值。

断点入口 5：`src/backend/optimizer/path/allpaths.c`。`qual_is_pushdown_safe()` 展示 subquery path 中另一个 pushdown 安全边界。

在这个位置不要一次性展开整份文件。先确认当前调用栈是否服务本节唯一主问题，再决定是否继续向下单步。

如果这里没有被命中，优先检查 SQL 形态、统计是否存在、GUC 是否关闭了相关路径，以及本节机制是否适用于当前 relation 类型。

如果这里被命中但状态为空，通常说明前置 catalog、样本、relids 或 clause 形态没有满足条件。这个结论比“planner 没优化”更有诊断价值。

断点入口 6：`src/backend/optimizer/path/joinpath.c`。join path 生成时区分 outer join qual 和普通 pushed-down filter。

在这个位置不要一次性展开整份文件。先确认当前调用栈是否服务本节唯一主问题，再决定是否继续向下单步。

如果这里没有被命中，优先检查 SQL 形态、统计是否存在、GUC 是否关闭了相关路径，以及本节机制是否适用于当前 relation 类型。

如果这里被命中但状态为空，通常说明前置 catalog、样本、relids 或 clause 形态没有满足条件。这个结论比“planner 没优化”更有诊断价值。

断点入口 7：`src/backend/optimizer/util/relnode.c`。parameterized joinrel 中判断 clause 是否能在当前 relids 上执行。

在这个位置不要一次性展开整份文件。先确认当前调用栈是否服务本节唯一主问题，再决定是否继续向下单步。

如果这里没有被命中，优先检查 SQL 形态、统计是否存在、GUC 是否关闭了相关路径，以及本节机制是否适用于当前 relation 类型。

如果这里被命中但状态为空，通常说明前置 catalog、样本、relids 或 clause 形态没有满足条件。这个结论比“planner 没优化”更有诊断价值。

断点入口 8：`src/backend/optimizer/plan/createplan.c`。最终 plan 生成时把 joinquals 与 otherquals 交给 executor。

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

反向检查 `required_relids`：语义上必须具备哪些 relation 才能计算这个 qual。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

反向检查 `clause_relids`：表达式实际引用哪些 base rel 与 nulling rel。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

反向检查 `outer_relids`：不能被移动进去的 outer join 非空侧边界。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

反向检查 `nullable_relids` / nulling relids：变量可能被哪些 outer join null-extend。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

反向检查 `is_pushed_down`：记录 qual 是否可能处在原始语法层级以外，但解释时还要结合 joinrelids。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

反向检查 `pseudoconstant`：表达式不引用 base rel 时可能成为 gating qual。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

反向检查 `security_level` / `leakproof`：security barrier 视图下避免低级 qual 泄漏高权限信息。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

反向检查 `incompatible_relids`：clone outer join clause 与部分 join order 的不兼容集合。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

## 19. 主流程中的可验证边界

下面把主流程拆成可验证边界。每个边界都应能用一个 SQL 变体、一个 catalog 查询或一个断点确认。

边界 1：`deconstruct_jointree()` 展开 jointree，递归遇到 WHERE、JOIN/ON、outer join 附加条件。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 2：`distribute_qual_to_rels()` 计算 qual 引用的 relids、nullable side、join domain 和安全级别。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 3：如果 qual 是 non-degenerate outer join qual，源码强制它在对应 outer join 层级执行。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 4：如果 qual 不触碰 outer join 非空侧，它可以被标记为 pushed-down，并尝试进入更低层级。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 5：`make_restrictinfo()` 写入 required relids、outer relids、pseudoconstant、security_level 等上下文。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 6：mergejoinable expression 先进入 EquivalenceClass；如果 EC 接收它，就不会直接挂到 rel 的 clause list。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 7：普通 base restriction 进入 `baserestrictinfo`，join clause 进入相关 base rel 的 `joininfo`。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 8：后续 `join_clause_is_movable_to()` 与 `join_clause_is_movable_into()` 决定参数化路径能否提前消费 join qual。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 9：join plan 生成时，outer join qual 与 pushed-down filter 被拆成 `joinqual` 和 `otherqual`。

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

记录对象：PostgreSQL Predicate 下推、延迟执行与 join qual 分类。

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

课后复习 PostgreSQL Predicate 下推、延迟执行与 join qual 分类 时，可以把源码索引压缩成三轮阅读。

第一轮只看入口，确认本节主问题在哪里被提出。

第二轮只看状态，确认字段、catalog tuple 或局部结构在哪里被写入。

第三轮只看消费点，确认 rows、cost、path 或 plan 如何体现这个状态。

- `src/backend/optimizer/plan/initsplan.c`：`distribute_qual_to_rels()`、`distribute_restrictinfo_to_rels()` 负责把原始 qual 放到正确的语义层级。

- `src/backend/optimizer/util/restrictinfo.c`：`make_restrictinfo()`、`join_clause_is_movable_to()`、`join_clause_is_movable_into()` 解释可移动性的局部规则。

- `src/include/nodes/pathnodes.h`：`RestrictInfo`、`SpecialJoinInfo`、`RINFO_IS_PUSHED_DOWN()` 给出状态字段和上下文判断。

- `src/include/optimizer/restrictinfo.h`：声明 RestrictInfo 构造和 join clause 移动检查入口。

- `src/backend/optimizer/path/allpaths.c`：`qual_is_pushdown_safe()` 展示 subquery path 中另一个 pushdown 安全边界。

- `src/backend/optimizer/path/joinpath.c`：join path 生成时区分 outer join qual 和普通 pushed-down filter。

- `src/backend/optimizer/util/relnode.c`：parameterized joinrel 中判断 clause 是否能在当前 relids 上执行。

- `src/backend/optimizer/plan/createplan.c`：最终 plan 生成时把 joinquals 与 otherquals 交给 executor。

复习时不要把这些源码入口平均用力。入口、状态和消费点能串起来，才算读完本节。

如果只能记住一个动作，就是从 `EXPLAIN` 中找第一个异常节点，再回到这里选最小源码入口。

## 25. 本节小结

predicate 下推不是单纯的性能优化，而是语义层级选择。

`RestrictInfo` 把表达式、relids、outer join 约束、安全级别和可移动性绑在一起。

看 qual 是否能提前执行，必须同时看它引用什么、会被谁 null-extend、在哪个 join domain 中、以及当前路径是否能提供参数。

```text
runtime 现象
  -> planner 状态
  -> 源码判断
  -> rows / cost / plan 表现
  -> 可迁移诊断规律
```

本节结束时，应该能把一个计划现象翻译成源码中的状态判断，而不是只记住某个函数名。

下一节继续沿这个方向推进：用同样方法把相邻主题拆成一个主问题、一条状态链和一个可验证实验。
