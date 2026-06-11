# PostgreSQL RestrictInfo 的包装意义

## 课程定位

前置知识：已经理解表达式 canonicalize、子查询 pullup 和 outer join reduction 会把 Query 改写到更适合优化的形态。

本节唯一主问题：

```text
为什么一个普通 qual 进入 optimizer 后要包装成 `RestrictInfo`，它如何携带 required relids、nullable relids、pseudoconstant、leakproof、security level、mergejoinable / hashjoinable 等元数据？
```

核心矛盾：一个 qual 的原始表达式只说明要计算什么布尔值，却不说明何时能计算、能下推到哪里、是否可做等价类、是否可做 join clause、是否安全提前执行；这些元数据如果每次重新推导，会让 planner 慢且容易不一致。

学完后应能判断一个 qual 在 `baserestrictinfo`、`joininfo`、EquivalenceClass、outer join clause list 或 pseudoconstant gating qual 之间为什么走不同路径。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

本节收束 preprocessing 组的前半段。下一节会继续看同一个 RestrictInfo 为什么有时能下推，有时必须 delay 到 outer join 之后执行。

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
`deconstruct_jointree()` 把 implicit-AND qual 交给 `distribute_qual_to_rels()`；后者计算 relids、outer join 边界、pseudoconstant、security 和 pushed-down 属性，调用 `make_restrictinfo()` 包装，再根据单 rel、多 rel、等价类或 outer join 特例分发到对应列表。
```

这个模型里有三个稳定判断。

- 语义状态和搜索状态必须分开。
- planner-local 状态可以很丰富，但不能泄漏给 executor。
- 每一次改写都要能回到后续可验证的 Path、Plan 或 EXPLAIN 现象。

如果读源码时只记函数名，很容易把当前阶段的临时状态误认为最终语义。正确的读法是沿着一个状态对象追踪：它在哪里创建，谁写入关键字段，什么时候被下游消费，什么时候被抛弃。

## 3. 核心文件分工与阅读顺序

| 项 | 说明 |
| --- | --- |
| 1 | `src/include/nodes/pathnodes.h`：`RestrictInfo` 字段定义，尤其 clause、required_relids、incompatible_relids、outer_relids、security_level、mergeopfamilies。 |
| 2 | `src/include/optimizer/restrictinfo.h`：`make_restrictinfo()`、`restriction_is_or_clause()`、`join_clause_is_movable_to()` 声明。 |
| 3 | `src/backend/optimizer/util/restrictinfo.c`：`make_restrictinfo()`、`make_plain_restrictinfo()`、`commute_restrictinfo()`。 |
| 4 | `src/backend/optimizer/plan/initsplan.c`：`distribute_qual_to_rels()`、`distribute_restrictinfo_to_rels()`、`check_mergejoinable()`、`check_hashjoinable()`。 |
| 5 | `src/backend/optimizer/path/equivclass.c`：`process_equivalence()` 如何吸收部分 RestrictInfo。 |
| 6 | `src/backend/optimizer/path/indxpath.c`：索引匹配如何消费 RestrictInfo。 |
| 7 | `src/backend/optimizer/plan/createplan.c`：最终 Plan qual 如何从 RestrictInfo 还原为 bare expression。 |
| 8 | `src/backend/optimizer/util/clauses.c`：volatility、leakproof 和 cost/selectivity 辅助判断。 |

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
| clause | 原始 Expr，表示布尔计算本身。 |
| is_pushed_down | qual 是否被视为可在语法 join level 以下执行，outer join 判断会依赖它。 |
| pseudoconstant | 当前 query level 无 Var 且无 volatile 函数，可变成 gating Result 的 one-time qual。 |
| security_level | security barrier 和 row security 下的执行顺序约束。 |
| clause_relids | 表达式实际提到的 relids，包含 varnullingrels 影响。 |
| required_relids | 计算该 qual 至少需要的 relids，决定 base restriction 或 join qual 归属。 |
| incompatible_relids | 某些 outer join clone qual 不能在其下方计算的 relids。 |
| outer_relids | outer join qual 的外侧 relids，用于判断 movable 和 pushed-down 语义。 |
| mergeopfamilies / hashjoinoperator | 能否作为 merge/hash join 条件的缓存判断。 |
| eval_cost / norm_selec / outer_selec | 成本和选择率缓存，避免重复估算。 |

判断这些状态时有一个通用规则：

```text
raw field 不是语义；
field + 生命周期 + 阶段边界 + 访问者，才是 planner 中可诊断的语义。
```

planner 的很多对象都在普通 backend-local memory context 中分配。它们不是 shared memory，不跨 backend 可见，也不应该被保存到执行期长期结构里。

### 状态切片 1：clause

原始 Expr，表示布尔计算本身。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

### 状态切片 2：is_pushed_down

qual 是否被视为可在语法 join level 以下执行，outer join 判断会依赖它。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

### 状态切片 3：pseudoconstant

当前 query level 无 Var 且无 volatile 函数，可变成 gating Result 的 one-time qual。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

### 状态切片 4：security_level

security barrier 和 row security 下的执行顺序约束。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

### 状态切片 5：clause_relids

表达式实际提到的 relids，包含 varnullingrels 影响。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

### 状态切片 6：required_relids

计算该 qual 至少需要的 relids，决定 base restriction 或 join qual 归属。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

### 状态切片 7：incompatible_relids

某些 outer join clone qual 不能在其下方计算的 relids。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

### 状态切片 8：outer_relids

outer join qual 的外侧 relids，用于判断 movable 和 pushed-down 语义。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

### 状态切片 9：mergeopfamilies / hashjoinoperator

能否作为 merge/hash join 条件的缓存判断。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

### 状态切片 10：eval_cost / norm_selec / outer_selec

成本和选择率缓存，避免重复估算。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

## 5. 主流程源码 walkthrough

下面按时间顺序跟主链路。每个小段都只回答一个问题：当前状态发生了什么变化。

| 项 | 说明 |
| --- | --- |
| `deconstruct_jointree()` 扫描 jointree | 收集 WHERE 和 JOIN/ON qual，并形成 joinlist。 |
| `distribute_qual_to_rels()` 计算 relids | `pull_varnos()` 得到表达式涉及的 relid 集合。 |
| 处理 lateral 和 outer join scope | 如果 qual 引用超出当前 syntactic scope，可能延迟到父 join level。 |
| 识别 pseudoconstant | 无 Var 且无 volatile 的 qual 标记为 one-time gating 候选。 |
| 判断 outer join delay | 非退化 outer join qual 不能下推，必须保留在 outer join 语义层。 |
| `make_restrictinfo()` | 创建 RestrictInfo，并对 OR clause 递归插入子 RestrictInfo。 |
| `check_mergejoinable()` | 尝试标记可进入 EquivalenceClass 或 merge join 的 clause。 |
| `process_equivalence()` | 真正的等值内连接条件可能被 EC 吸收，不直接进入 baserestrictinfo/joininfo。 |
| `distribute_restrictinfo_to_rels()` | 单 rel 进入 baserestrictinfo，多 rel 进入相关 base rel 的 joininfo。 |
| createplan 阶段剥离包装 | 最终 Plan qual 只需要表达式，createplan 会从 RestrictInfo list 中提取 bare clause。 |

### 5.1. `deconstruct_jointree()` 扫描 jointree

收集 WHERE 和 JOIN/ON qual，并形成 joinlist。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

### 5.2. `distribute_qual_to_rels()` 计算 relids

`pull_varnos()` 得到表达式涉及的 relid 集合。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

### 5.3. 处理 lateral 和 outer join scope

如果 qual 引用超出当前 syntactic scope，可能延迟到父 join level。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

这一段常见的误读是把“现在能看到的字段”当成“最终执行器会看到的字段”。多数情况下，当前字段只是下一阶段搜索或整理的材料。

### 5.4. 识别 pseudoconstant

无 Var 且无 volatile 的 qual 标记为 one-time gating 候选。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

### 5.5. 判断 outer join delay

非退化 outer join qual 不能下推，必须保留在 outer join 语义层。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

### 5.6. `make_restrictinfo()`

创建 RestrictInfo，并对 OR clause 递归插入子 RestrictInfo。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

这一段常见的误读是把“现在能看到的字段”当成“最终执行器会看到的字段”。多数情况下，当前字段只是下一阶段搜索或整理的材料。

### 5.7. `check_mergejoinable()`

尝试标记可进入 EquivalenceClass 或 merge join 的 clause。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

### 5.8. `process_equivalence()`

真正的等值内连接条件可能被 EC 吸收，不直接进入 baserestrictinfo/joininfo。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

### 5.9. `distribute_restrictinfo_to_rels()`

单 rel 进入 baserestrictinfo，多 rel 进入相关 base rel 的 joininfo。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

这一段常见的误读是把“现在能看到的字段”当成“最终执行器会看到的字段”。多数情况下，当前字段只是下一阶段搜索或整理的材料。

### 5.10. createplan 阶段剥离包装

最终 Plan qual 只需要表达式，createplan 会从 RestrictInfo list 中提取 bare clause。

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
| 1 | raw Expr 不是完整 optimizer 语义；Expr 加 relids、security、outer join context 才能决定执行位置。 |
| 2 | outer join 非退化 qual 的 `is_pushed_down=false` 是语义保护，不是成本偏好。 |
| 3 | pseudoconstant qual 仍要放入普通分发流程，直到 createplan 决定 gating Result 位置。 |
| 4 | EquivalenceClass 吸收 RestrictInfo 后，原始 qual 可能不会直接出现在 joininfo list 中。 |
| 5 | OR clause 的 RestrictInfo 有 `orclause`，内部子 clause 也可能被包装，不能把它当普通 Expr list。 |

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
| 1 | 如果 required_relids 算错，base restriction 可能被放到错误 relation，造成错误结果或丢失索引机会。 |
| 2 | 如果 leakproof / security_level 被忽略，security barrier 可能被绕过。 |
| 3 | 如果 hashjoinable 只按操作符名判断而不看左右 relids，会把单表条件误当 join 条件。 |
| 4 | 如果 createplan 没有正确剥离 pseudoconstant，one-time qual 可能被每行重复计算。 |

遇到 fallback 时要先问它保护的是语义、生命周期、资源成本还是扩展契约。不同原因对应的修复方向完全不同。

### 异常观察 1

如果 required_relids 算错，base restriction 可能被放到错误 relation，造成错误结果或丢失索引机会。

验证时不要只看最终节点名，还要回到触发保守路径的源码条件。

### 异常观察 2

如果 leakproof / security_level 被忽略，security barrier 可能被绕过。

验证时不要只看最终节点名，还要回到触发保守路径的源码条件。

### 异常观察 3

如果 hashjoinable 只按操作符名判断而不看左右 relids，会把单表条件误当 join 条件。

验证时不要只看最终节点名，还要回到触发保守路径的源码条件。

### 异常观察 4

如果 createplan 没有正确剥离 pseudoconstant，one-time qual 可能被每行重复计算。

验证时不要只看最终节点名，还要回到触发保守路径的源码条件。

## 9. 成本、资源与跨模块传播

planner 成本不只来自最终计划，也来自规划期搜索空间和状态传播。

| 项 | 说明 |
| --- | --- |
| 1 | RestrictInfo 缓存 eval cost 和 selectivity，减少同一个 clause 在多个 Path 比较中的重复计算。 |
| 2 | 包装会增加 planner 内存，但换来 clause 分类、移动和索引匹配的一致性。 |
| 3 | mergejoinable / hashjoinable 缓存让 join path 生成可以快速过滤不可用条件。 |
| 4 | 错误地保守设置 required_relids 会阻止下推，增大扫描和 join 的行数。 |

这些成本最后会在三个地方表现出来：planner CPU 时间、规划期间内存占用、以及最终 executor 计划的 runtime 成本。诊断时要把三者分开。

## 10. 观测与诊断入口

优化器内部状态多数不可直接从 SQL 看到，但可以通过最终 Plan、日志和断点间接定位。

| 项 | 说明 |
| --- | --- |
| 1 | gdb 断在 `make_restrictinfo()`，打印 `clause_relids`、`required_relids`、`is_pushed_down`。 |
| 2 | 断在 `distribute_restrictinfo_to_rels()`，观察 qual 是进入 `baserestrictinfo` 还是 `joininfo`。 |
| 3 | 用 `EXPLAIN` 观察 Index Cond、Filter、Join Filter，可以反推 RestrictInfo 最终落点。 |
| 4 | 对等值 join 断在 `process_equivalence()`，看哪些 RestrictInfo 被 EC 吸收。 |

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

### 实验 1：base restriction

执行 `SELECT * FROM t WHERE a=1`，观察 RestrictInfo 进入 base rel 的 baserestrictinfo。

实验记录至少包含三项：SQL、EXPLAIN 输出中的关键差异、对应源码断点或函数入口。

### 实验 2：join qual

执行 `SELECT * FROM a JOIN b ON a.id=b.id`，观察 RestrictInfo 进入双方 joininfo 或 EC。

实验记录至少包含三项：SQL、EXPLAIN 输出中的关键差异、对应源码断点或函数入口。

### 实验 3：pseudoconstant

执行 `WHERE current_setting('server_version') IS NOT NULL` 与 volatile 表达式对比，观察 gating 差异。

实验记录至少包含三项：SQL、EXPLAIN 输出中的关键差异、对应源码断点或函数入口。

### 实验 4：outer join qual

把同一条件放在 LEFT JOIN 的 ON 与 WHERE，比较 `is_pushed_down` 和计划位置。

实验记录至少包含三项：SQL、EXPLAIN 输出中的关键差异、对应源码断点或函数入口。

实验不追求覆盖所有路径，只追求把一个可见现象解释回源码中的状态变化。

## 13. 讨论题

1. 为什么不每次需要时重新计算 relids、selectivity、joinability？

2. RestrictInfo 哪些字段是稳定语义，哪些只是当前 planner 阶段的缓存？

3. 当 EXPLAIN 中 Filter 没有下推时，应如何从 RestrictInfo 字段追原因？

回答这些问题时，需要明确引用源码阶段和状态对象，避免只用抽象概念解释抽象概念。

## 14. 源码分段阅读

这一节把第 3 节的源码顺序再拆成可执行的阅读任务。每个任务都要能回到本节唯一主问题，而不是停留在“看过这个文件”。

### 14.1. `src/include/nodes/pathnodes.h`

`RestrictInfo` 字段定义，尤其 clause、required_relids、incompatible_relids、outer_relids、security_level、mergeopfamilies。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

### 14.2. `src/include/optimizer/restrictinfo.h`

`make_restrictinfo()`、`restriction_is_or_clause()`、`join_clause_is_movable_to()` 声明。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

### 14.3. `src/backend/optimizer/util/restrictinfo.c`

`make_restrictinfo()`、`make_plain_restrictinfo()`、`commute_restrictinfo()`。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

### 14.4. `src/backend/optimizer/plan/initsplan.c`

`distribute_qual_to_rels()`、`distribute_restrictinfo_to_rels()`、`check_mergejoinable()`、`check_hashjoinable()`。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

### 14.5. `src/backend/optimizer/path/equivclass.c`

`process_equivalence()` 如何吸收部分 RestrictInfo。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

### 14.6. `src/backend/optimizer/path/indxpath.c`

索引匹配如何消费 RestrictInfo。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

### 14.7. `src/backend/optimizer/plan/createplan.c`

最终 Plan qual 如何从 RestrictInfo 还原为 bare expression。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

### 14.8. `src/backend/optimizer/util/clauses.c`

volatility、leakproof 和 cost/selectivity 辅助判断。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

## 15. 状态迁移检查

下面把主流程再拆成状态迁移。它们不是新的知识点，而是防止把函数调用链读成 API 清单。

### 15.1. `deconstruct_jointree()` 扫描 jointree

收集 WHERE 和 JOIN/ON qual，并形成 joinlist。

进入前先确认 `clause` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `is_pushed_down` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.2. `distribute_qual_to_rels()` 计算 relids

`pull_varnos()` 得到表达式涉及的 relid 集合。

进入前先确认 `is_pushed_down` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `pseudoconstant` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.3. 处理 lateral 和 outer join scope

如果 qual 引用超出当前 syntactic scope，可能延迟到父 join level。

进入前先确认 `pseudoconstant` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `security_level` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.4. 识别 pseudoconstant

无 Var 且无 volatile 的 qual 标记为 one-time gating 候选。

进入前先确认 `security_level` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `clause_relids` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.5. 判断 outer join delay

非退化 outer join qual 不能下推，必须保留在 outer join 语义层。

进入前先确认 `clause_relids` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `required_relids` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.6. `make_restrictinfo()`

创建 RestrictInfo，并对 OR clause 递归插入子 RestrictInfo。

进入前先确认 `required_relids` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `incompatible_relids` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.7. `check_mergejoinable()`

尝试标记可进入 EquivalenceClass 或 merge join 的 clause。

进入前先确认 `incompatible_relids` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `outer_relids` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.8. `process_equivalence()`

真正的等值内连接条件可能被 EC 吸收，不直接进入 baserestrictinfo/joininfo。

进入前先确认 `outer_relids` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `mergeopfamilies / hashjoinoperator` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.9. `distribute_restrictinfo_to_rels()`

单 rel 进入 baserestrictinfo，多 rel 进入相关 base rel 的 joininfo。

进入前先确认 `mergeopfamilies / hashjoinoperator` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `eval_cost / norm_selec / outer_selec` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.10. createplan 阶段剥离包装

最终 Plan qual 只需要表达式，createplan 会从 RestrictInfo list 中提取 bare clause。

进入前先确认 `eval_cost / norm_selec / outer_selec` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `clause` 是否被创建、改写、选择、缓存或转交给下游。

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

- 检查点 1：在 `src/include/nodes/pathnodes.h` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 2：在 `src/include/optimizer/restrictinfo.h` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 3：在 `src/backend/optimizer/util/restrictinfo.c` 中验证：字段是否被后续阶段继续改写。
- 检查点 4：在 `src/backend/optimizer/plan/initsplan.c` 中验证：字段是否会越过 planner 生命周期。
- 检查点 5：在 `src/backend/optimizer/path/equivclass.c` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 6：在 `src/backend/optimizer/path/indxpath.c` 中验证：最终计划是否能从源码中的同一条状态线解释。
- 检查点 7：在 `src/backend/optimizer/plan/createplan.c` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 8：在 `src/backend/optimizer/util/clauses.c` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 9：在 `src/include/nodes/pathnodes.h` 中验证：字段是否被后续阶段继续改写。
- 检查点 10：在 `src/include/optimizer/restrictinfo.h` 中验证：字段是否会越过 planner 生命周期。
- 检查点 11：在 `src/backend/optimizer/util/restrictinfo.c` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 12：在 `src/backend/optimizer/plan/initsplan.c` 中验证：最终计划是否能从源码中的同一条状态线解释。
- 检查点 13：在 `src/backend/optimizer/path/equivclass.c` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 14：在 `src/backend/optimizer/path/indxpath.c` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 15：在 `src/backend/optimizer/plan/createplan.c` 中验证：字段是否被后续阶段继续改写。
- 检查点 16：在 `src/backend/optimizer/util/clauses.c` 中验证：字段是否会越过 planner 生命周期。
- 检查点 17：在 `src/include/nodes/pathnodes.h` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 18：在 `src/include/optimizer/restrictinfo.h` 中验证：最终计划是否能从源码中的同一条状态线解释。
- 检查点 19：在 `src/backend/optimizer/util/restrictinfo.c` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 20：在 `src/backend/optimizer/plan/initsplan.c` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 21：在 `src/backend/optimizer/path/equivclass.c` 中验证：字段是否被后续阶段继续改写。
- 检查点 22：在 `src/backend/optimizer/path/indxpath.c` 中验证：字段是否会越过 planner 生命周期。
- 检查点 23：在 `src/backend/optimizer/plan/createplan.c` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 24：在 `src/backend/optimizer/util/clauses.c` 中验证：最终计划是否能从源码中的同一条状态线解释。
- 检查点 25：在 `src/include/nodes/pathnodes.h` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 26：在 `src/include/optimizer/restrictinfo.h` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 27：在 `src/backend/optimizer/util/restrictinfo.c` 中验证：字段是否被后续阶段继续改写。
- 检查点 28：在 `src/backend/optimizer/plan/initsplan.c` 中验证：字段是否会越过 planner 生命周期。
- 检查点 29：在 `src/backend/optimizer/path/equivclass.c` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 30：在 `src/backend/optimizer/path/indxpath.c` 中验证：最终计划是否能从源码中的同一条状态线解释。
- 检查点 31：在 `src/backend/optimizer/plan/createplan.c` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 32：在 `src/backend/optimizer/util/clauses.c` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 33：在 `src/include/nodes/pathnodes.h` 中验证：字段是否被后续阶段继续改写。
- 检查点 34：在 `src/include/optimizer/restrictinfo.h` 中验证：字段是否会越过 planner 生命周期。
- 检查点 35：在 `src/backend/optimizer/util/restrictinfo.c` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 36：在 `src/backend/optimizer/plan/initsplan.c` 中验证：最终计划是否能从源码中的同一条状态线解释。
- 检查点 37：在 `src/backend/optimizer/path/equivclass.c` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 38：在 `src/backend/optimizer/path/indxpath.c` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 39：在 `src/backend/optimizer/plan/createplan.c` 中验证：字段是否被后续阶段继续改写。
- 检查点 40：在 `src/backend/optimizer/util/clauses.c` 中验证：字段是否会越过 planner 生命周期。
- 检查点 41：在 `src/include/nodes/pathnodes.h` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 42：在 `src/include/optimizer/restrictinfo.h` 中验证：最终计划是否能从源码中的同一条状态线解释。
- 检查点 43：在 `src/backend/optimizer/util/restrictinfo.c` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 44：在 `src/backend/optimizer/plan/initsplan.c` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 45：在 `src/backend/optimizer/path/equivclass.c` 中验证：字段是否被后续阶段继续改写。
- 检查点 46：在 `src/backend/optimizer/path/indxpath.c` 中验证：字段是否会越过 planner 生命周期。
- 检查点 47：在 `src/backend/optimizer/plan/createplan.c` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 48：在 `src/backend/optimizer/util/clauses.c` 中验证：最终计划是否能从源码中的同一条状态线解释。

## 18. 误判与校正

下面这些条目用于把课堂讨论落到排查动作上。每一条都从一个容易误判的现象开始，再给出回到源码的校正路径。

### 18.1. 正确性边界

raw Expr 不是完整 optimizer 语义；Expr 加 relids、security、outer join context 才能决定执行位置。

先确认语义是否保持，再确认候选路径是否只是搜索空间变化。

源码回看点：`src/include/nodes/pathnodes.h`。

### 18.2. 正确性边界

outer join 非退化 qual 的 `is_pushed_down=false` 是语义保护，不是成本偏好。

先确认语义是否保持，再确认候选路径是否只是搜索空间变化。

源码回看点：`src/include/optimizer/restrictinfo.h`。

### 18.3. 正确性边界

pseudoconstant qual 仍要放入普通分发流程，直到 createplan 决定 gating Result 位置。

先确认语义是否保持，再确认候选路径是否只是搜索空间变化。

源码回看点：`src/backend/optimizer/util/restrictinfo.c`。

### 18.4. 正确性边界

EquivalenceClass 吸收 RestrictInfo 后，原始 qual 可能不会直接出现在 joininfo list 中。

先确认语义是否保持，再确认候选路径是否只是搜索空间变化。

源码回看点：`src/backend/optimizer/plan/initsplan.c`。

### 18.5. 正确性边界

OR clause 的 RestrictInfo 有 `orclause`，内部子 clause 也可能被包装，不能把它当普通 Expr list。

先确认语义是否保持，再确认候选路径是否只是搜索空间变化。

源码回看点：`src/backend/optimizer/path/equivclass.c`。

### 18.6. 异常或 fallback

如果 required_relids 算错，base restriction 可能被放到错误 relation，造成错误结果或丢失索引机会。

先找到保守分支的源码条件，再判断它保护的是语义、资源还是生命周期。

源码回看点：`src/backend/optimizer/path/indxpath.c`。

### 18.7. 异常或 fallback

如果 leakproof / security_level 被忽略，security barrier 可能被绕过。

先找到保守分支的源码条件，再判断它保护的是语义、资源还是生命周期。

源码回看点：`src/backend/optimizer/plan/createplan.c`。

### 18.8. 异常或 fallback

如果 hashjoinable 只按操作符名判断而不看左右 relids，会把单表条件误当 join 条件。

先找到保守分支的源码条件，再判断它保护的是语义、资源还是生命周期。

源码回看点：`src/backend/optimizer/util/clauses.c`。

### 18.9. 异常或 fallback

如果 createplan 没有正确剥离 pseudoconstant，one-time qual 可能被每行重复计算。

先找到保守分支的源码条件，再判断它保护的是语义、资源还是生命周期。

源码回看点：`src/include/nodes/pathnodes.h`。

### 18.10. 成本传播

RestrictInfo 缓存 eval cost 和 selectivity，减少同一个 clause 在多个 Path 比较中的重复计算。

同时记录 planner CPU、候选数量和 executor runtime，避免把三种成本混在一起。

源码回看点：`src/include/optimizer/restrictinfo.h`。

### 18.11. 成本传播

包装会增加 planner 内存，但换来 clause 分类、移动和索引匹配的一致性。

同时记录 planner CPU、候选数量和 executor runtime，避免把三种成本混在一起。

源码回看点：`src/backend/optimizer/util/restrictinfo.c`。

### 18.12. 成本传播

mergejoinable / hashjoinable 缓存让 join path 生成可以快速过滤不可用条件。

同时记录 planner CPU、候选数量和 executor runtime，避免把三种成本混在一起。

源码回看点：`src/backend/optimizer/plan/initsplan.c`。

### 18.13. 成本传播

错误地保守设置 required_relids 会阻止下推，增大扫描和 join 的行数。

同时记录 planner CPU、候选数量和 executor runtime，避免把三种成本混在一起。

源码回看点：`src/backend/optimizer/path/equivclass.c`。

### 18.14. 观测入口

gdb 断在 `make_restrictinfo()`，打印 `clause_relids`、`required_relids`、`is_pushed_down`。

用 EXPLAIN 或 gdb 只确认一个假设，确认后再扩大到完整 SQL。

源码回看点：`src/backend/optimizer/path/indxpath.c`。

### 18.15. 观测入口

断在 `distribute_restrictinfo_to_rels()`，观察 qual 是进入 `baserestrictinfo` 还是 `joininfo`。

用 EXPLAIN 或 gdb 只确认一个假设，确认后再扩大到完整 SQL。

源码回看点：`src/backend/optimizer/plan/createplan.c`。

### 18.16. 观测入口

用 `EXPLAIN` 观察 Index Cond、Filter、Join Filter，可以反推 RestrictInfo 最终落点。

用 EXPLAIN 或 gdb 只确认一个假设，确认后再扩大到完整 SQL。

源码回看点：`src/backend/optimizer/util/clauses.c`。

### 18.17. 观测入口

对等值 join 断在 `process_equivalence()`，看哪些 RestrictInfo 被 EC 吸收。

用 EXPLAIN 或 gdb 只确认一个假设，确认后再扩大到完整 SQL。

源码回看点：`src/include/nodes/pathnodes.h`。

## 19. 本节小结

- `RestrictInfo` 是 qual 进入 optimizer 后的语义外壳。
- 它把表达式、可执行位置、安全级别、outer join 边界、joinability、成本和选择率缓存绑定在一起。
- 读优化器源码时，看到 qual list 要先判断它是 bare Expr 还是 RestrictInfo list，两者能回答的问题不同。

把本节内容压缩成一个可迁移规律：

```text
复杂优化器的关键不是把所有规则塞进一个入口，
而是把语义改写、候选搜索、成本比较和执行契约放在不同生命周期里。
```
