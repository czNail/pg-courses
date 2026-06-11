# PostgreSQL 外连接约束与 join tree 语义保护

## 课程定位

前置知识：已经理解 simple subquery 可以被拉平，但 pullup 必须尊重 outer join、lateral 和 security barrier 等边界。

本节唯一主问题：

```text
planner 为什么要记录 outer join 的 nullable side、join order 约束和可消除条件，哪些 predicate 能把 outer join 简化为 inner join，哪些不能跨越 null-extension 边界？
```

核心矛盾：outer join 保留未匹配行的语义限制了 join reorder 和 qual 下推，但某些严格谓词又能证明 null-extended 行必定被过滤；优化器要尽量简化 join，又不能把本应保留的 NULL 扩展行提前删除。

学完后应能判断一个 LEFT JOIN 为什么被 reduce 成 INNER/ANTI，为什么某个 qual 不能下推到 nullable side，为什么 FULL JOIN 通常形成更强顺序约束。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

本节是 preprocessing 组中最重要的正确性边界。下一节会看 qual 进入 optimizer 后为什么要被包装为 RestrictInfo，以便携带 required relids、nullable relids、pseudoconstant 和 joinability 等元数据。

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
`reduce_outer_joins()` 在表达式 preprocessing 后运行；第一遍收集 jointree 每侧 relids、outer join 和 nullable 信息，第二遍利用严格谓词和 forced-null 信息降低 join 强度，并在后续 `deconstruct_jointree()` 中通过 `SpecialJoinInfo` 保护合法 join order。
```

这个模型里有三个稳定判断。

- 语义状态和搜索状态必须分开。
- planner-local 状态可以很丰富，但不能泄漏给 executor。
- 每一次改写都要能回到后续可验证的 Path、Plan 或 EXPLAIN 现象。

如果读源码时只记函数名，很容易把当前阶段的临时状态误认为最终语义。正确的读法是沿着一个状态对象追踪：它在哪里创建，谁写入关键字段，什么时候被下游消费，什么时候被抛弃。

## 3. 核心文件分工与阅读顺序

| 项 | 说明 |
| --- | --- |
| 1 | `src/backend/optimizer/prep/prepjointree.c`：`reduce_outer_joins()`、`reduce_outer_joins_pass1()`、`reduce_outer_joins_pass2()`。 |
| 2 | `src/backend/optimizer/plan/initsplan.c`：`deconstruct_jointree()`、`make_outerjoininfo()`、`distribute_qual_to_rels()`。 |
| 3 | `src/backend/optimizer/path/joinrels.c`：`join_is_legal()` 使用 SpecialJoinInfo 检查 join order。 |
| 4 | `src/backend/optimizer/plan/analyzejoins.c`：join removal 和 RestrictInfo / SpecialJoinInfo 后续修正。 |
| 5 | `src/include/nodes/pathnodes.h`：`SpecialJoinInfo`、`JoinDomain`、`RestrictInfo` 字段定义。 |
| 6 | `src/backend/optimizer/util/clauses.c`：`find_nonnullable_rels()`、`find_forced_null_vars()` 等严格性分析。 |
| 7 | `src/backend/optimizer/util/predtest.c`：predicate implication 与约束判断辅助。 |
| 8 | `src/backend/optimizer/README`：outer join reorder identities 和 SpecialJoinInfo 设计说明。 |

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
| nullable side | outer join 中可能被 NULL 扩展的一侧，qual 不能随意提前穿过这个边界。 |
| outer_join_rels | root 中记录 outer join relid，参与 all_query_rels 和 join order 判断。 |
| SpecialJoinInfo | 描述非 inner join 的最小左右侧 relids、join type、ojrelid 和顺序限制。 |
| JoinDomain | 限制等价类和 join clause 在 outer join 语义域内传播。 |
| strict predicate | 对 nullable side 列严格的谓词可证明 NULL 扩展行不会通过。 |
| forced-null predicate | `IS NULL` 等谓词可与 strict join qual 结合，把 LEFT JOIN 识别为 anti join。 |
| nullingrels | Var 上记录可能使它变 NULL 的 outer join relids，后续简化需要清理。 |

判断这些状态时有一个通用规则：

```text
raw field 不是语义；
field + 生命周期 + 阶段边界 + 访问者，才是 planner 中可诊断的语义。
```

planner 的很多对象都在普通 backend-local memory context 中分配。它们不是 shared memory，不跨 backend 可见，也不应该被保存到执行期长期结构里。

### 状态切片 1：nullable side

outer join 中可能被 NULL 扩展的一侧，qual 不能随意提前穿过这个边界。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

### 状态切片 2：outer_join_rels

root 中记录 outer join relid，参与 all_query_rels 和 join order 判断。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

### 状态切片 3：SpecialJoinInfo

描述非 inner join 的最小左右侧 relids、join type、ojrelid 和顺序限制。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

### 状态切片 4：JoinDomain

限制等价类和 join clause 在 outer join 语义域内传播。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

### 状态切片 5：strict predicate

对 nullable side 列严格的谓词可证明 NULL 扩展行不会通过。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

### 状态切片 6：forced-null predicate

`IS NULL` 等谓词可与 strict join qual 结合，把 LEFT JOIN 识别为 anti join。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

### 状态切片 7：nullingrels

Var 上记录可能使它变 NULL 的 outer join relids，后续简化需要清理。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

## 5. 主流程源码 walkthrough

下面按时间顺序跟主链路。每个小段都只回答一个问题：当前状态发生了什么变化。

| 项 | 说明 |
| --- | --- |
| 表达式 preprocessing 完成 | JOIN alias 已展开，qual 已 canonicalize，严格性分析才能可靠工作。 |
| `reduce_outer_joins_pass1()` | 自底向上收集每个 jointree 节点下面的 relids、outer join 存在性和 nullable rels。 |
| `reduce_outer_joins_pass2()` | 结合上层 qual 的 nonnullable / forced-null 信息决定 join type 是否可降级。 |
| LEFT/RIGHT 处理 | RIGHT JOIN 在内部常被翻转成 LEFT JOIN，减少后续语义分支。 |
| LEFT 到 INNER | 如果 nullable side 被上层 strict qual 要求非 NULL，null-extended 行必定被过滤，可降为 inner join。 |
| LEFT 到 ANTI | `LEFT JOIN ... WHERE nullable_col IS NULL` 且匹配行不可能为 NULL 时，可识别为 anti join。 |
| 清理 nullingrels | 降级后要调用 `remove_nulling_relids()`，否则 Var 仍携带已经不存在的 NULL 扩展来源。 |
| `deconstruct_jointree()` | 为仍然存在的 outer/semi/anti join 构造 SpecialJoinInfo，供 join search 检查合法性。 |

### 5.1. 表达式 preprocessing 完成

JOIN alias 已展开，qual 已 canonicalize，严格性分析才能可靠工作。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

### 5.2. `reduce_outer_joins_pass1()`

自底向上收集每个 jointree 节点下面的 relids、outer join 存在性和 nullable rels。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

### 5.3. `reduce_outer_joins_pass2()`

结合上层 qual 的 nonnullable / forced-null 信息决定 join type 是否可降级。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

这一段常见的误读是把“现在能看到的字段”当成“最终执行器会看到的字段”。多数情况下，当前字段只是下一阶段搜索或整理的材料。

### 5.4. LEFT/RIGHT 处理

RIGHT JOIN 在内部常被翻转成 LEFT JOIN，减少后续语义分支。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

### 5.5. LEFT 到 INNER

如果 nullable side 被上层 strict qual 要求非 NULL，null-extended 行必定被过滤，可降为 inner join。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

### 5.6. LEFT 到 ANTI

`LEFT JOIN ... WHERE nullable_col IS NULL` 且匹配行不可能为 NULL 时，可识别为 anti join。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

这一段常见的误读是把“现在能看到的字段”当成“最终执行器会看到的字段”。多数情况下，当前字段只是下一阶段搜索或整理的材料。

### 5.7. 清理 nullingrels

降级后要调用 `remove_nulling_relids()`，否则 Var 仍携带已经不存在的 NULL 扩展来源。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

### 5.8. `deconstruct_jointree()`

为仍然存在的 outer/semi/anti join 构造 SpecialJoinInfo，供 join search 检查合法性。

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
| 1 | outer join 的 ON qual 和 WHERE qual 不同，前者参与匹配，后者过滤 join 结果。 |
| 2 | 只有能证明 NULL 扩展行必定被过滤的 predicate 才能降低 outer join 强度。 |
| 3 | FULL JOIN 两侧都可 NULL 扩展，通常不能像 LEFT JOIN 那样自由重排。 |
| 4 | SpecialJoinInfo 保护 join order，不是成本偏好；非法顺序不应进入 Path 竞争。 |
| 5 | outer join clause 进入 EquivalenceClass 时要小心，不能把可能为 NULL 的等价关系当作普通 inner join 等价。 |

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
| 1 | 如果 strictness 分析在 join alias 展开之前做，可能对别名表达式得出错误结论。 |
| 2 | 如果降低 outer join 后不清理 nullingrels，后续 Var 语义会继续认为它可能被旧 outer join 置 NULL。 |
| 3 | 如果把非退化 outer join qual 当作普通 pushed-down qual，会提前过滤非 nullable side，改变结果行数。 |
| 4 | 如果 join search 忽略 SpecialJoinInfo，可能生成语义错误的 join tree。 |

遇到 fallback 时要先问它保护的是语义、生命周期、资源成本还是扩展契约。不同原因对应的修复方向完全不同。

### 异常观察 1

如果 strictness 分析在 join alias 展开之前做，可能对别名表达式得出错误结论。

验证时不要只看最终节点名，还要回到触发保守路径的源码条件。

### 异常观察 2

如果降低 outer join 后不清理 nullingrels，后续 Var 语义会继续认为它可能被旧 outer join 置 NULL。

验证时不要只看最终节点名，还要回到触发保守路径的源码条件。

### 异常观察 3

如果把非退化 outer join qual 当作普通 pushed-down qual，会提前过滤非 nullable side，改变结果行数。

验证时不要只看最终节点名，还要回到触发保守路径的源码条件。

### 异常观察 4

如果 join search 忽略 SpecialJoinInfo，可能生成语义错误的 join tree。

验证时不要只看最终节点名，还要回到触发保守路径的源码条件。

## 9. 成本、资源与跨模块传播

planner 成本不只来自最终计划，也来自规划期搜索空间和状态传播。

| 项 | 说明 |
| --- | --- |
| 1 | outer join reduction 可以把受约束的 join 变成 inner join，扩大 join reorder 和等价类推导空间。 |
| 2 | 保留 outer join 会限制 join order，减少搜索空间但也可能错过更低成本路径。 |
| 3 | strictness 和 forced-null 分析是 planner CPU 成本，但通常远小于后续错误 join search 的代价。 |
| 4 | 错误估算 outer join 选择率会放大到 join order，但正确性边界优先于成本。 |

这些成本最后会在三个地方表现出来：planner CPU 时间、规划期间内存占用、以及最终 executor 计划的 runtime 成本。诊断时要把三者分开。

## 10. 观测与诊断入口

优化器内部状态多数不可直接从 SQL 看到，但可以通过最终 Plan、日志和断点间接定位。

| 项 | 说明 |
| --- | --- |
| 1 | 用 `EXPLAIN` 比较 `LEFT JOIN` 加 `WHERE right.col IS NOT NULL` 前后计划，可观察 join type 降级。 |
| 2 | 断在 `reduce_outer_joins_pass2()`，查看 `inner_reduced` 和 `partial_reduced`。 |
| 3 | 断在 `make_outerjoininfo()`，观察 SpecialJoinInfo 的 min_lefthand、min_righthand、syn_lefthand、syn_righthand。 |
| 4 | 对 FULL JOIN 查询观察计划，可看到 join order 受到更强限制。 |

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

### 实验 1：LEFT 到 INNER

构造 `a LEFT JOIN b ON a.id=b.id WHERE b.id IS NOT NULL`，确认计划是否变为 inner join。

实验记录至少包含三项：SQL、EXPLAIN 输出中的关键差异、对应源码断点或函数入口。

### 实验 2：ANTI 识别

构造 `a LEFT JOIN b ON a.id=b.id WHERE b.id IS NULL`，观察是否出现 anti join。

实验记录至少包含三项：SQL、EXPLAIN 输出中的关键差异、对应源码断点或函数入口。

### 实验 3：ON/WHERE 对比

把同一条件放在 ON 和 WHERE，比较结果行数和计划形态。

实验记录至少包含三项：SQL、EXPLAIN 输出中的关键差异、对应源码断点或函数入口。

### 实验 4：FULL JOIN 边界

用 FULL JOIN 加 strict predicate，观察哪些侧能被部分降级。

实验记录至少包含三项：SQL、EXPLAIN 输出中的关键差异、对应源码断点或函数入口。

实验不追求覆盖所有路径，只追求把一个可见现象解释回源码中的状态变化。

## 13. 讨论题

1. outer join reduction 是正确性优化还是性能优化？为什么首先是正确性证明？

2. 为什么 WHERE 里的 strict predicate 可以降低 LEFT JOIN，而 ON 里的相同 predicate 不一定可以？

3. SpecialJoinInfo 如果只保存 join type 而不保存 relid 集合，join search 会缺什么信息？

回答这些问题时，需要明确引用源码阶段和状态对象，避免只用抽象概念解释抽象概念。

## 14. 源码分段阅读

这一节把第 3 节的源码顺序再拆成可执行的阅读任务。每个任务都要能回到本节唯一主问题，而不是停留在“看过这个文件”。

### 14.1. `src/backend/optimizer/prep/prepjointree.c`

`reduce_outer_joins()`、`reduce_outer_joins_pass1()`、`reduce_outer_joins_pass2()`。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

### 14.2. `src/backend/optimizer/plan/initsplan.c`

`deconstruct_jointree()`、`make_outerjoininfo()`、`distribute_qual_to_rels()`。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

### 14.3. `src/backend/optimizer/path/joinrels.c`

`join_is_legal()` 使用 SpecialJoinInfo 检查 join order。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

### 14.4. `src/backend/optimizer/plan/analyzejoins.c`

join removal 和 RestrictInfo / SpecialJoinInfo 后续修正。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

### 14.5. `src/include/nodes/pathnodes.h`

`SpecialJoinInfo`、`JoinDomain`、`RestrictInfo` 字段定义。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

### 14.6. `src/backend/optimizer/util/clauses.c`

`find_nonnullable_rels()`、`find_forced_null_vars()` 等严格性分析。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

### 14.7. `src/backend/optimizer/util/predtest.c`

predicate implication 与约束判断辅助。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

### 14.8. `src/backend/optimizer/README`

outer join reorder identities 和 SpecialJoinInfo 设计说明。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

## 15. 状态迁移检查

下面把主流程再拆成状态迁移。它们不是新的知识点，而是防止把函数调用链读成 API 清单。

### 15.1. 表达式 preprocessing 完成

JOIN alias 已展开，qual 已 canonicalize，严格性分析才能可靠工作。

进入前先确认 `nullable side` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `outer_join_rels` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.2. `reduce_outer_joins_pass1()`

自底向上收集每个 jointree 节点下面的 relids、outer join 存在性和 nullable rels。

进入前先确认 `outer_join_rels` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `SpecialJoinInfo` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.3. `reduce_outer_joins_pass2()`

结合上层 qual 的 nonnullable / forced-null 信息决定 join type 是否可降级。

进入前先确认 `SpecialJoinInfo` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `JoinDomain` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.4. LEFT/RIGHT 处理

RIGHT JOIN 在内部常被翻转成 LEFT JOIN，减少后续语义分支。

进入前先确认 `JoinDomain` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `strict predicate` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.5. LEFT 到 INNER

如果 nullable side 被上层 strict qual 要求非 NULL，null-extended 行必定被过滤，可降为 inner join。

进入前先确认 `strict predicate` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `forced-null predicate` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.6. LEFT 到 ANTI

`LEFT JOIN ... WHERE nullable_col IS NULL` 且匹配行不可能为 NULL 时，可识别为 anti join。

进入前先确认 `forced-null predicate` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `nullingrels` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.7. 清理 nullingrels

降级后要调用 `remove_nulling_relids()`，否则 Var 仍携带已经不存在的 NULL 扩展来源。

进入前先确认 `nullingrels` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `nullable side` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.8. `deconstruct_jointree()`

为仍然存在的 outer/semi/anti join 构造 SpecialJoinInfo，供 join search 检查合法性。

进入前先确认 `nullable side` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `outer_join_rels` 是否被创建、改写、选择、缓存或转交给下游。

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

- 检查点 1：在 `src/backend/optimizer/prep/prepjointree.c` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 2：在 `src/backend/optimizer/plan/initsplan.c` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 3：在 `src/backend/optimizer/path/joinrels.c` 中验证：字段是否被后续阶段继续改写。
- 检查点 4：在 `src/backend/optimizer/plan/analyzejoins.c` 中验证：字段是否会越过 planner 生命周期。
- 检查点 5：在 `src/include/nodes/pathnodes.h` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 6：在 `src/backend/optimizer/util/clauses.c` 中验证：最终计划是否能从源码中的同一条状态线解释。
- 检查点 7：在 `src/backend/optimizer/util/predtest.c` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 8：在 `src/backend/optimizer/README` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 9：在 `src/backend/optimizer/prep/prepjointree.c` 中验证：字段是否被后续阶段继续改写。
- 检查点 10：在 `src/backend/optimizer/plan/initsplan.c` 中验证：字段是否会越过 planner 生命周期。
- 检查点 11：在 `src/backend/optimizer/path/joinrels.c` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 12：在 `src/backend/optimizer/plan/analyzejoins.c` 中验证：最终计划是否能从源码中的同一条状态线解释。
- 检查点 13：在 `src/include/nodes/pathnodes.h` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 14：在 `src/backend/optimizer/util/clauses.c` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 15：在 `src/backend/optimizer/util/predtest.c` 中验证：字段是否被后续阶段继续改写。
- 检查点 16：在 `src/backend/optimizer/README` 中验证：字段是否会越过 planner 生命周期。
- 检查点 17：在 `src/backend/optimizer/prep/prepjointree.c` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 18：在 `src/backend/optimizer/plan/initsplan.c` 中验证：最终计划是否能从源码中的同一条状态线解释。
- 检查点 19：在 `src/backend/optimizer/path/joinrels.c` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 20：在 `src/backend/optimizer/plan/analyzejoins.c` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 21：在 `src/include/nodes/pathnodes.h` 中验证：字段是否被后续阶段继续改写。
- 检查点 22：在 `src/backend/optimizer/util/clauses.c` 中验证：字段是否会越过 planner 生命周期。
- 检查点 23：在 `src/backend/optimizer/util/predtest.c` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 24：在 `src/backend/optimizer/README` 中验证：最终计划是否能从源码中的同一条状态线解释。
- 检查点 25：在 `src/backend/optimizer/prep/prepjointree.c` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 26：在 `src/backend/optimizer/plan/initsplan.c` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 27：在 `src/backend/optimizer/path/joinrels.c` 中验证：字段是否被后续阶段继续改写。
- 检查点 28：在 `src/backend/optimizer/plan/analyzejoins.c` 中验证：字段是否会越过 planner 生命周期。
- 检查点 29：在 `src/include/nodes/pathnodes.h` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 30：在 `src/backend/optimizer/util/clauses.c` 中验证：最终计划是否能从源码中的同一条状态线解释。
- 检查点 31：在 `src/backend/optimizer/util/predtest.c` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 32：在 `src/backend/optimizer/README` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 33：在 `src/backend/optimizer/prep/prepjointree.c` 中验证：字段是否被后续阶段继续改写。
- 检查点 34：在 `src/backend/optimizer/plan/initsplan.c` 中验证：字段是否会越过 planner 生命周期。
- 检查点 35：在 `src/backend/optimizer/path/joinrels.c` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 36：在 `src/backend/optimizer/plan/analyzejoins.c` 中验证：最终计划是否能从源码中的同一条状态线解释。
- 检查点 37：在 `src/include/nodes/pathnodes.h` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 38：在 `src/backend/optimizer/util/clauses.c` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 39：在 `src/backend/optimizer/util/predtest.c` 中验证：字段是否被后续阶段继续改写。
- 检查点 40：在 `src/backend/optimizer/README` 中验证：字段是否会越过 planner 生命周期。
- 检查点 41：在 `src/backend/optimizer/prep/prepjointree.c` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 42：在 `src/backend/optimizer/plan/initsplan.c` 中验证：最终计划是否能从源码中的同一条状态线解释。
- 检查点 43：在 `src/backend/optimizer/path/joinrels.c` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 44：在 `src/backend/optimizer/plan/analyzejoins.c` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 45：在 `src/include/nodes/pathnodes.h` 中验证：字段是否被后续阶段继续改写。
- 检查点 46：在 `src/backend/optimizer/util/clauses.c` 中验证：字段是否会越过 planner 生命周期。
- 检查点 47：在 `src/backend/optimizer/util/predtest.c` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 48：在 `src/backend/optimizer/README` 中验证：最终计划是否能从源码中的同一条状态线解释。

## 18. 误判与校正

下面这些条目用于把课堂讨论落到排查动作上。每一条都从一个容易误判的现象开始，再给出回到源码的校正路径。

### 18.1. 正确性边界

outer join 的 ON qual 和 WHERE qual 不同，前者参与匹配，后者过滤 join 结果。

先确认语义是否保持，再确认候选路径是否只是搜索空间变化。

源码回看点：`src/backend/optimizer/prep/prepjointree.c`。

### 18.2. 正确性边界

只有能证明 NULL 扩展行必定被过滤的 predicate 才能降低 outer join 强度。

先确认语义是否保持，再确认候选路径是否只是搜索空间变化。

源码回看点：`src/backend/optimizer/plan/initsplan.c`。

### 18.3. 正确性边界

FULL JOIN 两侧都可 NULL 扩展，通常不能像 LEFT JOIN 那样自由重排。

先确认语义是否保持，再确认候选路径是否只是搜索空间变化。

源码回看点：`src/backend/optimizer/path/joinrels.c`。

### 18.4. 正确性边界

SpecialJoinInfo 保护 join order，不是成本偏好；非法顺序不应进入 Path 竞争。

先确认语义是否保持，再确认候选路径是否只是搜索空间变化。

源码回看点：`src/backend/optimizer/plan/analyzejoins.c`。

### 18.5. 正确性边界

outer join clause 进入 EquivalenceClass 时要小心，不能把可能为 NULL 的等价关系当作普通 inner join 等价。

先确认语义是否保持，再确认候选路径是否只是搜索空间变化。

源码回看点：`src/include/nodes/pathnodes.h`。

### 18.6. 异常或 fallback

如果 strictness 分析在 join alias 展开之前做，可能对别名表达式得出错误结论。

先找到保守分支的源码条件，再判断它保护的是语义、资源还是生命周期。

源码回看点：`src/backend/optimizer/util/clauses.c`。

### 18.7. 异常或 fallback

如果降低 outer join 后不清理 nullingrels，后续 Var 语义会继续认为它可能被旧 outer join 置 NULL。

先找到保守分支的源码条件，再判断它保护的是语义、资源还是生命周期。

源码回看点：`src/backend/optimizer/util/predtest.c`。

### 18.8. 异常或 fallback

如果把非退化 outer join qual 当作普通 pushed-down qual，会提前过滤非 nullable side，改变结果行数。

先找到保守分支的源码条件，再判断它保护的是语义、资源还是生命周期。

源码回看点：`src/backend/optimizer/README`。

### 18.9. 异常或 fallback

如果 join search 忽略 SpecialJoinInfo，可能生成语义错误的 join tree。

先找到保守分支的源码条件，再判断它保护的是语义、资源还是生命周期。

源码回看点：`src/backend/optimizer/prep/prepjointree.c`。

### 18.10. 成本传播

outer join reduction 可以把受约束的 join 变成 inner join，扩大 join reorder 和等价类推导空间。

同时记录 planner CPU、候选数量和 executor runtime，避免把三种成本混在一起。

源码回看点：`src/backend/optimizer/plan/initsplan.c`。

### 18.11. 成本传播

保留 outer join 会限制 join order，减少搜索空间但也可能错过更低成本路径。

同时记录 planner CPU、候选数量和 executor runtime，避免把三种成本混在一起。

源码回看点：`src/backend/optimizer/path/joinrels.c`。

### 18.12. 成本传播

strictness 和 forced-null 分析是 planner CPU 成本，但通常远小于后续错误 join search 的代价。

同时记录 planner CPU、候选数量和 executor runtime，避免把三种成本混在一起。

源码回看点：`src/backend/optimizer/plan/analyzejoins.c`。

### 18.13. 成本传播

错误估算 outer join 选择率会放大到 join order，但正确性边界优先于成本。

同时记录 planner CPU、候选数量和 executor runtime，避免把三种成本混在一起。

源码回看点：`src/include/nodes/pathnodes.h`。

### 18.14. 观测入口

用 `EXPLAIN` 比较 `LEFT JOIN` 加 `WHERE right.col IS NOT NULL` 前后计划，可观察 join type 降级。

用 EXPLAIN 或 gdb 只确认一个假设，确认后再扩大到完整 SQL。

源码回看点：`src/backend/optimizer/util/clauses.c`。

### 18.15. 观测入口

断在 `reduce_outer_joins_pass2()`，查看 `inner_reduced` 和 `partial_reduced`。

用 EXPLAIN 或 gdb 只确认一个假设，确认后再扩大到完整 SQL。

源码回看点：`src/backend/optimizer/util/predtest.c`。

### 18.16. 观测入口

断在 `make_outerjoininfo()`，观察 SpecialJoinInfo 的 min_lefthand、min_righthand、syn_lefthand、syn_righthand。

用 EXPLAIN 或 gdb 只确认一个假设，确认后再扩大到完整 SQL。

源码回看点：`src/backend/optimizer/README`。

### 18.17. 观测入口

对 FULL JOIN 查询观察计划，可看到 join order 受到更强限制。

用 EXPLAIN 或 gdb 只确认一个假设，确认后再扩大到完整 SQL。

源码回看点：`src/backend/optimizer/prep/prepjointree.c`。

## 19. 本节小结

- outer join 的本质边界是 NULL 扩展行什么时候仍必须保留。
- `reduce_outer_joins()` 负责在有证明时降低 join 强度，`SpecialJoinInfo` 负责在没有证明时保护 join order。
- 慢 SQL 诊断中看到 outer join 限制 join order 时，不要先调 GUC，要先判断谓词能否安全表达出非 NULL 约束。

把本节内容压缩成一个可迁移规律：

```text
复杂优化器的关键不是把所有规则塞进一个入口，
而是把语义改写、候选搜索、成本比较和执行契约放在不同生命周期里。
```
