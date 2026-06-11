# PostgreSQL 子查询 pullup 与 jointree 改写

## 课程定位

前置知识：已经理解 qual 会先被 canonicalize 和 implicit-AND 拆分。本节看更大粒度的 preprocessing：把可拉平的子查询并入父查询 jointree。

本节唯一主问题：

```text
哪些 simple subquery 可以被拉平到上层 join tree，为什么 security barrier、聚合、limit、volatile 表达式和 lateral 引用会阻止这种改写？
```

核心矛盾：拉平子查询能扩大 join order 和 predicate pushdown 空间，但子查询本身可能携带安全边界、求值顺序、去重、聚合、limit 或 lateral 依赖；拉平过度会改变语义，保守过度又会丢失优化机会。

学完后应能从 SQL 形态和源码检查点判断一个 RTE_SUBQUERY 为什么被 pull up、保留为 SubqueryScan，或转成 append relation。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

本节承接表达式规范化，说明 Query 的 jointree 也会被改写。下一节会看 outer join 为什么给这种改写和后续 join order 施加边界。

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
`pull_up_subqueries()` 递归扫描 jointree；遇到 RTE_SUBQUERY 时先用 `is_simple_subquery()` 检查语义边界，再用 `pull_up_simple_subquery()` 替换父查询中的 RangeTblRef、Var 和 qual；不安全子查询保留为独立规划单元。
```

这个模型里有三个稳定判断。

- 语义状态和搜索状态必须分开。
- planner-local 状态可以很丰富，但不能泄漏给 executor。
- 每一次改写都要能回到后续可验证的 Path、Plan 或 EXPLAIN 现象。

如果读源码时只记函数名，很容易把当前阶段的临时状态误认为最终语义。正确的读法是沿着一个状态对象追踪：它在哪里创建，谁写入关键字段，什么时候被下游消费，什么时候被抛弃。

## 3. 核心文件分工与阅读顺序

| 项 | 说明 |
| --- | --- |
| 1 | `src/backend/optimizer/prep/prepjointree.c`：`pull_up_subqueries()`、`pull_up_subqueries_recurse()`、`is_simple_subquery()`、`pull_up_simple_subquery()`。 |
| 2 | `src/backend/optimizer/prep/prepunion.c`：`flatten_simple_union_all()` 和 append relation 相关路径。 |
| 3 | `src/backend/optimizer/util/appendinfo.c`：pullup 后 parent/child Var 替换和 AppendRelInfo 处理。 |
| 4 | `src/backend/optimizer/plan/planner.c`：`subquery_planner()` 调用 pullup 的阶段位置。 |
| 5 | `src/backend/optimizer/path/allpaths.c`：不能 pullup 的 subquery 后续如何走 SubqueryScan path。 |
| 6 | `src/include/nodes/parsenodes.h`：`RangeTblEntry`、`Query`、`FromExpr`、`JoinExpr` 的结构边界。 |
| 7 | `src/include/nodes/pathnodes.h`：`AppendRelInfo`、`PlaceHolderVar`、lateral 相关 planner 状态。 |
| 8 | `src/include/optimizer/prep.h`：preprocessing 入口声明。 |

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
| RTE_SUBQUERY | range table 中的子查询入口，可能被 pull up，也可能保留为 SubqueryScan。 |
| jointree | 父查询 FROM/JOIN 结构，是 pullup 改写的主要目标。 |
| lowest_outer_join | 递归过程中记录所处 outer join 边界，限制 lateral 和 null-extension 下的 pullup。 |
| AppendRelInfo | simple UNION ALL 或 append member pullup 后描述父子 relation 的替换关系。 |
| PlaceHolderVar | pullup 后某些表达式必须延迟到正确 join level 才能求值，用 PHV 表达这个边界。 |
| security_barrier | 安全屏障视图或 RTE 会限制 qual 穿透，保护 leakproof 语义。 |
| lateral reference | 子查询引用外层 rel 时会限制可拉平位置和 qual 移动。 |

判断这些状态时有一个通用规则：

```text
raw field 不是语义；
field + 生命周期 + 阶段边界 + 访问者，才是 planner 中可诊断的语义。
```

planner 的很多对象都在普通 backend-local memory context 中分配。它们不是 shared memory，不跨 backend 可见，也不应该被保存到执行期长期结构里。

### 状态切片 1：RTE_SUBQUERY

range table 中的子查询入口，可能被 pull up，也可能保留为 SubqueryScan。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

### 状态切片 2：jointree

父查询 FROM/JOIN 结构，是 pullup 改写的主要目标。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

### 状态切片 3：lowest_outer_join

递归过程中记录所处 outer join 边界，限制 lateral 和 null-extension 下的 pullup。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

### 状态切片 4：AppendRelInfo

simple UNION ALL 或 append member pullup 后描述父子 relation 的替换关系。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

### 状态切片 5：PlaceHolderVar

pullup 后某些表达式必须延迟到正确 join level 才能求值，用 PHV 表达这个边界。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

### 状态切片 6：security_barrier

安全屏障视图或 RTE 会限制 qual 穿透，保护 leakproof 语义。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

### 状态切片 7：lateral reference

子查询引用外层 rel 时会限制可拉平位置和 qual 移动。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

## 5. 主流程源码 walkthrough

下面按时间顺序跟主链路。每个小段都只回答一个问题：当前状态发生了什么变化。

| 项 | 说明 |
| --- | --- |
| `subquery_planner()` 调用 `preprocess_function_rtes()` | 函数 RTE 可能先被 inline 成 subquery，为后续 pullup 提供机会。 |
| `pull_up_subqueries()` 从顶层 FromExpr 开始 | 递归遍历 jointree，并保持 jointree 在递归期间始终可被全局替换。 |
| 遇到 `RangeTblRef` | 检查对应 RTE 是否是 `RTE_SUBQUERY`、simple UNION ALL、VALUES 或可 inline FUNCTION。 |
| `is_simple_subquery()` | 排除聚合、窗口、distinct、limit、setop、volatile target、安全屏障等会改变语义的场景。 |
| `pull_up_simple_subquery()` | 规划或改写子查询，把子查询 targetlist 映射到父查询 Var，并替换父查询引用。 |
| qual 和 target 替换 | 父查询中指向子查询输出列的 Var 被替换成子查询表达式或 PlaceHolderVar。 |
| append relation 处理 | simple UNION ALL 可能转成 appendrel，后续 base rel path 生成会统一比较。 |
| 不能 pullup 的路径 | 保留 RTE_SUBQUERY，后续由 `set_subquery_pathlist()` 独立规划子查询。 |

### 5.1. `subquery_planner()` 调用 `preprocess_function_rtes()`

函数 RTE 可能先被 inline 成 subquery，为后续 pullup 提供机会。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

### 5.2. `pull_up_subqueries()` 从顶层 FromExpr 开始

递归遍历 jointree，并保持 jointree 在递归期间始终可被全局替换。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

### 5.3. 遇到 `RangeTblRef`

检查对应 RTE 是否是 `RTE_SUBQUERY`、simple UNION ALL、VALUES 或可 inline FUNCTION。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

这一段常见的误读是把“现在能看到的字段”当成“最终执行器会看到的字段”。多数情况下，当前字段只是下一阶段搜索或整理的材料。

### 5.4. `is_simple_subquery()`

排除聚合、窗口、distinct、limit、setop、volatile target、安全屏障等会改变语义的场景。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

### 5.5. `pull_up_simple_subquery()`

规划或改写子查询，把子查询 targetlist 映射到父查询 Var，并替换父查询引用。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

### 5.6. qual 和 target 替换

父查询中指向子查询输出列的 Var 被替换成子查询表达式或 PlaceHolderVar。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

这一段常见的误读是把“现在能看到的字段”当成“最终执行器会看到的字段”。多数情况下，当前字段只是下一阶段搜索或整理的材料。

### 5.7. append relation 处理

simple UNION ALL 可能转成 appendrel，后续 base rel path 生成会统一比较。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

### 5.8. 不能 pullup 的路径

保留 RTE_SUBQUERY，后续由 `set_subquery_pathlist()` 独立规划子查询。

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
| 1 | 拉平必须保持子查询输出行数、重复性、排序不承诺、limit 语义和 volatile 求值次数。 |
| 2 | security barrier 防止非 leakproof qual 被推到屏障内泄漏信息。 |
| 3 | outer join 下的 pullup 要避免把本应 null-extended 后求值的表达式提前。 |
| 4 | lateral 引用决定子查询依赖哪些外层 rel，不能让 join order 把依赖关系反过来。 |
| 5 | pullup 替换 Var 时要同步处理 targetlist、quals、append_rel_list 和 placeholder。 |

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
| 1 | 带 LIMIT/OFFSET 的子查询通常不能简单拉平，因为父查询过滤可能改变保留的前 N 行。 |
| 2 | 带聚合或 GROUP BY 的子查询不能被当作普通 join 输入随意重排。 |
| 3 | 带 volatile target 的子查询如果被复制到多个父查询位置，求值次数会变化。 |
| 4 | security barrier 视图下推条件需要 leakproof 判断，否则可能通过错误信息或 timing 泄漏数据。 |

遇到 fallback 时要先问它保护的是语义、生命周期、资源成本还是扩展契约。不同原因对应的修复方向完全不同。

### 异常观察 1

带 LIMIT/OFFSET 的子查询通常不能简单拉平，因为父查询过滤可能改变保留的前 N 行。

验证时不要只看最终节点名，还要回到触发保守路径的源码条件。

### 异常观察 2

带聚合或 GROUP BY 的子查询不能被当作普通 join 输入随意重排。

验证时不要只看最终节点名，还要回到触发保守路径的源码条件。

### 异常观察 3

带 volatile target 的子查询如果被复制到多个父查询位置，求值次数会变化。

验证时不要只看最终节点名，还要回到触发保守路径的源码条件。

### 异常观察 4

security barrier 视图下推条件需要 leakproof 判断，否则可能通过错误信息或 timing 泄漏数据。

验证时不要只看最终节点名，还要回到触发保守路径的源码条件。

## 9. 成本、资源与跨模块传播

planner 成本不只来自最终计划，也来自规划期搜索空间和状态传播。

| 项 | 说明 |
| --- | --- |
| 1 | pullup 会扩大 join 搜索空间，可能带来更好计划，也可能增加 planner 时间。 |
| 2 | 保留 SubqueryScan 缩小搜索空间，但可能阻止 predicate pushdown 和 join reorder。 |
| 3 | simple UNION ALL flatten 成 appendrel 后，每个 child 的统计和 path 可以单独估算。 |
| 4 | PlaceHolderVar 会增加 join level 约束，换取拉平表达式后仍保持求值位置。 |

这些成本最后会在三个地方表现出来：planner CPU 时间、规划期间内存占用、以及最终 executor 计划的 runtime 成本。诊断时要把三者分开。

## 10. 观测与诊断入口

优化器内部状态多数不可直接从 SQL 看到，但可以通过最终 Plan、日志和断点间接定位。

| 项 | 说明 |
| --- | --- |
| 1 | `EXPLAIN (VERBOSE)` 中是否出现 Subquery Scan，是判断 pullup 是否失败的直接入口。 |
| 2 | gdb 断在 `is_simple_subquery()` 可以看到具体阻止条件。 |
| 3 | 比较带 LIMIT 与不带 LIMIT 的子查询计划，可以观察 pullup 边界。 |
| 4 | 对 security barrier view 与普通 view 比较谓词下推，可看到安全边界。 |

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

### 实验 1：simple pullup

执行 `SELECT * FROM (SELECT * FROM t) s WHERE s.a=1`，观察是否没有 Subquery Scan。

实验记录至少包含三项：SQL、EXPLAIN 输出中的关键差异、对应源码断点或函数入口。

### 实验 2：LIMIT 阻止

给子查询增加 `LIMIT 10`，比较计划是否保留 Subquery Scan。

实验记录至少包含三项：SQL、EXPLAIN 输出中的关键差异、对应源码断点或函数入口。

### 实验 3：聚合阻止

用 `SELECT * FROM (SELECT a,count(*) FROM t GROUP BY a) s WHERE a=1`，观察 group 边界。

实验记录至少包含三项：SQL、EXPLAIN 输出中的关键差异、对应源码断点或函数入口。

### 实验 4：security barrier

创建 security_barrier view，比较普通 view 与屏障 view 的 qual 下推。

实验记录至少包含三项：SQL、EXPLAIN 输出中的关键差异、对应源码断点或函数入口。

实验不追求覆盖所有路径，只追求把一个可见现象解释回源码中的状态变化。

## 13. 讨论题

1. 子查询 pullup 是语义改写还是搜索空间优化？为什么两者都对？

2. 如果一个 pullup 能显著改善性能但依赖 volatile 表达式，为什么仍不能做？

3. SubqueryScan 出现在计划里时，如何判断是必须的语义边界还是可以通过改 SQL 消除？

回答这些问题时，需要明确引用源码阶段和状态对象，避免只用抽象概念解释抽象概念。

## 14. 源码分段阅读

这一节把第 3 节的源码顺序再拆成可执行的阅读任务。每个任务都要能回到本节唯一主问题，而不是停留在“看过这个文件”。

### 14.1. `src/backend/optimizer/prep/prepjointree.c`

`pull_up_subqueries()`、`pull_up_subqueries_recurse()`、`is_simple_subquery()`、`pull_up_simple_subquery()`。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

### 14.2. `src/backend/optimizer/prep/prepunion.c`

`flatten_simple_union_all()` 和 append relation 相关路径。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

### 14.3. `src/backend/optimizer/util/appendinfo.c`

pullup 后 parent/child Var 替换和 AppendRelInfo 处理。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

### 14.4. `src/backend/optimizer/plan/planner.c`

`subquery_planner()` 调用 pullup 的阶段位置。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

### 14.5. `src/backend/optimizer/path/allpaths.c`

不能 pullup 的 subquery 后续如何走 SubqueryScan path。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

### 14.6. `src/include/nodes/parsenodes.h`

`RangeTblEntry`、`Query`、`FromExpr`、`JoinExpr` 的结构边界。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

### 14.7. `src/include/nodes/pathnodes.h`

`AppendRelInfo`、`PlaceHolderVar`、lateral 相关 planner 状态。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

### 14.8. `src/include/optimizer/prep.h`

preprocessing 入口声明。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

## 15. 状态迁移检查

下面把主流程再拆成状态迁移。它们不是新的知识点，而是防止把函数调用链读成 API 清单。

### 15.1. `subquery_planner()` 调用 `preprocess_function_rtes()`

函数 RTE 可能先被 inline 成 subquery，为后续 pullup 提供机会。

进入前先确认 `RTE_SUBQUERY` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `jointree` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.2. `pull_up_subqueries()` 从顶层 FromExpr 开始

递归遍历 jointree，并保持 jointree 在递归期间始终可被全局替换。

进入前先确认 `jointree` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `lowest_outer_join` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.3. 遇到 `RangeTblRef`

检查对应 RTE 是否是 `RTE_SUBQUERY`、simple UNION ALL、VALUES 或可 inline FUNCTION。

进入前先确认 `lowest_outer_join` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `AppendRelInfo` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.4. `is_simple_subquery()`

排除聚合、窗口、distinct、limit、setop、volatile target、安全屏障等会改变语义的场景。

进入前先确认 `AppendRelInfo` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `PlaceHolderVar` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.5. `pull_up_simple_subquery()`

规划或改写子查询，把子查询 targetlist 映射到父查询 Var，并替换父查询引用。

进入前先确认 `PlaceHolderVar` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `security_barrier` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.6. qual 和 target 替换

父查询中指向子查询输出列的 Var 被替换成子查询表达式或 PlaceHolderVar。

进入前先确认 `security_barrier` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `lateral reference` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.7. append relation 处理

simple UNION ALL 可能转成 appendrel，后续 base rel path 生成会统一比较。

进入前先确认 `lateral reference` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `RTE_SUBQUERY` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.8. 不能 pullup 的路径

保留 RTE_SUBQUERY，后续由 `set_subquery_pathlist()` 独立规划子查询。

进入前先确认 `RTE_SUBQUERY` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `jointree` 是否被创建、改写、选择、缓存或转交给下游。

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
- 检查点 2：在 `src/backend/optimizer/prep/prepunion.c` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 3：在 `src/backend/optimizer/util/appendinfo.c` 中验证：字段是否被后续阶段继续改写。
- 检查点 4：在 `src/backend/optimizer/plan/planner.c` 中验证：字段是否会越过 planner 生命周期。
- 检查点 5：在 `src/backend/optimizer/path/allpaths.c` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 6：在 `src/include/nodes/parsenodes.h` 中验证：最终计划是否能从源码中的同一条状态线解释。
- 检查点 7：在 `src/include/nodes/pathnodes.h` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 8：在 `src/include/optimizer/prep.h` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 9：在 `src/backend/optimizer/prep/prepjointree.c` 中验证：字段是否被后续阶段继续改写。
- 检查点 10：在 `src/backend/optimizer/prep/prepunion.c` 中验证：字段是否会越过 planner 生命周期。
- 检查点 11：在 `src/backend/optimizer/util/appendinfo.c` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 12：在 `src/backend/optimizer/plan/planner.c` 中验证：最终计划是否能从源码中的同一条状态线解释。
- 检查点 13：在 `src/backend/optimizer/path/allpaths.c` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 14：在 `src/include/nodes/parsenodes.h` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 15：在 `src/include/nodes/pathnodes.h` 中验证：字段是否被后续阶段继续改写。
- 检查点 16：在 `src/include/optimizer/prep.h` 中验证：字段是否会越过 planner 生命周期。
- 检查点 17：在 `src/backend/optimizer/prep/prepjointree.c` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 18：在 `src/backend/optimizer/prep/prepunion.c` 中验证：最终计划是否能从源码中的同一条状态线解释。
- 检查点 19：在 `src/backend/optimizer/util/appendinfo.c` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 20：在 `src/backend/optimizer/plan/planner.c` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 21：在 `src/backend/optimizer/path/allpaths.c` 中验证：字段是否被后续阶段继续改写。
- 检查点 22：在 `src/include/nodes/parsenodes.h` 中验证：字段是否会越过 planner 生命周期。
- 检查点 23：在 `src/include/nodes/pathnodes.h` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 24：在 `src/include/optimizer/prep.h` 中验证：最终计划是否能从源码中的同一条状态线解释。
- 检查点 25：在 `src/backend/optimizer/prep/prepjointree.c` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 26：在 `src/backend/optimizer/prep/prepunion.c` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 27：在 `src/backend/optimizer/util/appendinfo.c` 中验证：字段是否被后续阶段继续改写。
- 检查点 28：在 `src/backend/optimizer/plan/planner.c` 中验证：字段是否会越过 planner 生命周期。
- 检查点 29：在 `src/backend/optimizer/path/allpaths.c` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 30：在 `src/include/nodes/parsenodes.h` 中验证：最终计划是否能从源码中的同一条状态线解释。
- 检查点 31：在 `src/include/nodes/pathnodes.h` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 32：在 `src/include/optimizer/prep.h` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 33：在 `src/backend/optimizer/prep/prepjointree.c` 中验证：字段是否被后续阶段继续改写。
- 检查点 34：在 `src/backend/optimizer/prep/prepunion.c` 中验证：字段是否会越过 planner 生命周期。
- 检查点 35：在 `src/backend/optimizer/util/appendinfo.c` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 36：在 `src/backend/optimizer/plan/planner.c` 中验证：最终计划是否能从源码中的同一条状态线解释。
- 检查点 37：在 `src/backend/optimizer/path/allpaths.c` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 38：在 `src/include/nodes/parsenodes.h` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 39：在 `src/include/nodes/pathnodes.h` 中验证：字段是否被后续阶段继续改写。
- 检查点 40：在 `src/include/optimizer/prep.h` 中验证：字段是否会越过 planner 生命周期。
- 检查点 41：在 `src/backend/optimizer/prep/prepjointree.c` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 42：在 `src/backend/optimizer/prep/prepunion.c` 中验证：最终计划是否能从源码中的同一条状态线解释。
- 检查点 43：在 `src/backend/optimizer/util/appendinfo.c` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 44：在 `src/backend/optimizer/plan/planner.c` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 45：在 `src/backend/optimizer/path/allpaths.c` 中验证：字段是否被后续阶段继续改写。
- 检查点 46：在 `src/include/nodes/parsenodes.h` 中验证：字段是否会越过 planner 生命周期。
- 检查点 47：在 `src/include/nodes/pathnodes.h` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 48：在 `src/include/optimizer/prep.h` 中验证：最终计划是否能从源码中的同一条状态线解释。

## 18. 误判与校正

下面这些条目用于把课堂讨论落到排查动作上。每一条都从一个容易误判的现象开始，再给出回到源码的校正路径。

### 18.1. 正确性边界

拉平必须保持子查询输出行数、重复性、排序不承诺、limit 语义和 volatile 求值次数。

先确认语义是否保持，再确认候选路径是否只是搜索空间变化。

源码回看点：`src/backend/optimizer/prep/prepjointree.c`。

### 18.2. 正确性边界

security barrier 防止非 leakproof qual 被推到屏障内泄漏信息。

先确认语义是否保持，再确认候选路径是否只是搜索空间变化。

源码回看点：`src/backend/optimizer/prep/prepunion.c`。

### 18.3. 正确性边界

outer join 下的 pullup 要避免把本应 null-extended 后求值的表达式提前。

先确认语义是否保持，再确认候选路径是否只是搜索空间变化。

源码回看点：`src/backend/optimizer/util/appendinfo.c`。

### 18.4. 正确性边界

lateral 引用决定子查询依赖哪些外层 rel，不能让 join order 把依赖关系反过来。

先确认语义是否保持，再确认候选路径是否只是搜索空间变化。

源码回看点：`src/backend/optimizer/plan/planner.c`。

### 18.5. 正确性边界

pullup 替换 Var 时要同步处理 targetlist、quals、append_rel_list 和 placeholder。

先确认语义是否保持，再确认候选路径是否只是搜索空间变化。

源码回看点：`src/backend/optimizer/path/allpaths.c`。

### 18.6. 异常或 fallback

带 LIMIT/OFFSET 的子查询通常不能简单拉平，因为父查询过滤可能改变保留的前 N 行。

先找到保守分支的源码条件，再判断它保护的是语义、资源还是生命周期。

源码回看点：`src/include/nodes/parsenodes.h`。

### 18.7. 异常或 fallback

带聚合或 GROUP BY 的子查询不能被当作普通 join 输入随意重排。

先找到保守分支的源码条件，再判断它保护的是语义、资源还是生命周期。

源码回看点：`src/include/nodes/pathnodes.h`。

### 18.8. 异常或 fallback

带 volatile target 的子查询如果被复制到多个父查询位置，求值次数会变化。

先找到保守分支的源码条件，再判断它保护的是语义、资源还是生命周期。

源码回看点：`src/include/optimizer/prep.h`。

### 18.9. 异常或 fallback

security barrier 视图下推条件需要 leakproof 判断，否则可能通过错误信息或 timing 泄漏数据。

先找到保守分支的源码条件，再判断它保护的是语义、资源还是生命周期。

源码回看点：`src/backend/optimizer/prep/prepjointree.c`。

### 18.10. 成本传播

pullup 会扩大 join 搜索空间，可能带来更好计划，也可能增加 planner 时间。

同时记录 planner CPU、候选数量和 executor runtime，避免把三种成本混在一起。

源码回看点：`src/backend/optimizer/prep/prepunion.c`。

### 18.11. 成本传播

保留 SubqueryScan 缩小搜索空间，但可能阻止 predicate pushdown 和 join reorder。

同时记录 planner CPU、候选数量和 executor runtime，避免把三种成本混在一起。

源码回看点：`src/backend/optimizer/util/appendinfo.c`。

### 18.12. 成本传播

simple UNION ALL flatten 成 appendrel 后，每个 child 的统计和 path 可以单独估算。

同时记录 planner CPU、候选数量和 executor runtime，避免把三种成本混在一起。

源码回看点：`src/backend/optimizer/plan/planner.c`。

### 18.13. 成本传播

PlaceHolderVar 会增加 join level 约束，换取拉平表达式后仍保持求值位置。

同时记录 planner CPU、候选数量和 executor runtime，避免把三种成本混在一起。

源码回看点：`src/backend/optimizer/path/allpaths.c`。

### 18.14. 观测入口

`EXPLAIN (VERBOSE)` 中是否出现 Subquery Scan，是判断 pullup 是否失败的直接入口。

用 EXPLAIN 或 gdb 只确认一个假设，确认后再扩大到完整 SQL。

源码回看点：`src/include/nodes/parsenodes.h`。

### 18.15. 观测入口

gdb 断在 `is_simple_subquery()` 可以看到具体阻止条件。

用 EXPLAIN 或 gdb 只确认一个假设，确认后再扩大到完整 SQL。

源码回看点：`src/include/nodes/pathnodes.h`。

### 18.16. 观测入口

比较带 LIMIT 与不带 LIMIT 的子查询计划，可以观察 pullup 边界。

用 EXPLAIN 或 gdb 只确认一个假设，确认后再扩大到完整 SQL。

源码回看点：`src/include/optimizer/prep.h`。

### 18.17. 观测入口

对 security barrier view 与普通 view 比较谓词下推，可看到安全边界。

用 EXPLAIN 或 gdb 只确认一个假设，确认后再扩大到完整 SQL。

源码回看点：`src/backend/optimizer/prep/prepjointree.c`。

## 19. 本节小结

- 子查询 pullup 的核心是用更大的 join tree 换取优化机会。
- 安全屏障、聚合、limit、volatile 和 lateral 都是阻止拉平的语义边界。
- 读 pullup 源码时不要只看是否替换 RTE，要同时看 Var、qual、PlaceHolderVar 和 append relation 如何保持语义。

把本节内容压缩成一个可迁移规律：

```text
复杂优化器的关键不是把所有规则塞进一个入口，
而是把语义改写、候选搜索、成本比较和执行契约放在不同生命周期里。
```
