# PostgreSQL Hook、GUC 与 planner 可插拔边界

## 课程定位

前置知识：已经理解 `PlannerInfo` 集中了优化过程状态，并知道标准 planner 会按阶段生成 Path、选择 Path、再生成 Plan。

本节唯一主问题：

```text
`planner_hook`、`set_rel_pathlist_hook`、`set_join_pathlist_hook`、`join_search_hook`、`enable_*` GUC 和 cost 参数能改变哪些决策，为什么它们只能影响搜索空间和代价，不能破坏 Query / Plan 的语义边界？
```

核心矛盾：PostgreSQL 允许扩展和运维参数影响计划选择，但必须保护 SQL 语义、权限、可见性和 executor 契约；可插拔性越强，越需要清晰限制扩展能碰到的状态层。

学完后应能判断一个计划变化是 hook 扩展、enable GUC、cost 参数、统计信息，还是语义改写造成的，并能说清哪些修改越过了安全边界。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

本节结束 planner 主流程的第一组课程。下一组转向 preprocessing：为什么在真正估算和生成 Path 之前，优化器必须先把表达式和 jointree 改写成可推导、可下推、可分类的形态。

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
planner hook 可以包裹或替换标准规划入口；pathlist hook 可以给已有 RelOptInfo 增加候选 Path；join_search_hook 可以替换 join order 搜索；enable_* 和 cost GUC 影响候选生成与成本比较；最终输出仍必须是语义等价的 PlannedStmt。
```

这个模型里有三个稳定判断。

- 语义状态和搜索状态必须分开。
- planner-local 状态可以很丰富，但不能泄漏给 executor。
- 每一次改写都要能回到后续可验证的 Path、Plan 或 EXPLAIN 现象。

如果读源码时只记函数名，很容易把当前阶段的临时状态误认为最终语义。正确的读法是沿着一个状态对象追踪：它在哪里创建，谁写入关键字段，什么时候被下游消费，什么时候被抛弃。

## 3. 核心文件分工与阅读顺序

| 项 | 说明 |
| --- | --- |
| 1 | `src/include/optimizer/planner.h`：`planner_hook_type`、`planner_setup_hook_type`、`planner_shutdown_hook_type`。 |
| 2 | `src/backend/optimizer/plan/planner.c`：`planner()` 调用 hook，`standard_planner()` 使用 enable_* 初始化 strategy mask。 |
| 3 | `src/include/optimizer/paths.h`：`set_rel_pathlist_hook_type`、`set_join_pathlist_hook_type`、`join_search_hook_type`。 |
| 4 | `src/backend/optimizer/path/allpaths.c`：`set_rel_pathlist_hook`、`join_search_hook` 的调用点。 |
| 5 | `src/backend/optimizer/path/joinpath.c`：`set_join_pathlist_hook` 的调用点。 |
| 6 | `src/backend/optimizer/path/costsize.c`：cost GUC 和 enable_* 全局变量定义与成本模型入口。 |
| 7 | `src/include/optimizer/cost.h`：cost 参数和 enable_* 声明。 |
| 8 | `src/backend/utils/misc/guc_tables.c`：planner 相关 GUC 的注册位置。 |

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
| planner_hook | 最外层 planner 入口，可以替换或包裹 `standard_planner()`。 |
| planner_setup_hook / planner_shutdown_hook | 标准规划前后对 `PlannerGlobal`、`Query` 和结果做扩展处理。 |
| set_rel_pathlist_hook | base relation pathlist 生成后可加入扩展 Path。 |
| set_join_pathlist_hook | joinrel pathlist 生成过程中可加入扩展 join Path。 |
| join_search_hook | 可替换 `standard_join_search()` / GEQO 的 join order 搜索入口。 |
| enable_* GUC | 关闭或弱化某类 Path 策略，通常通过 disabled_nodes 或 pgs mask 影响比较。 |
| cost GUC | 改变相对代价单位，影响 Path 排名，不等价于真实耗时预测。 |

判断这些状态时有一个通用规则：

```text
raw field 不是语义；
field + 生命周期 + 阶段边界 + 访问者，才是 planner 中可诊断的语义。
```

planner 的很多对象都在普通 backend-local memory context 中分配。它们不是 shared memory，不跨 backend 可见，也不应该被保存到执行期长期结构里。

### 状态切片 1：planner_hook

最外层 planner 入口，可以替换或包裹 `standard_planner()`。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

### 状态切片 2：planner_setup_hook / planner_shutdown_hook

标准规划前后对 `PlannerGlobal`、`Query` 和结果做扩展处理。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

### 状态切片 3：set_rel_pathlist_hook

base relation pathlist 生成后可加入扩展 Path。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

### 状态切片 4：set_join_pathlist_hook

joinrel pathlist 生成过程中可加入扩展 join Path。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

### 状态切片 5：join_search_hook

可替换 `standard_join_search()` / GEQO 的 join order 搜索入口。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

### 状态切片 6：enable_* GUC

关闭或弱化某类 Path 策略，通常通过 disabled_nodes 或 pgs mask 影响比较。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

### 状态切片 7：cost GUC

改变相对代价单位，影响 Path 排名，不等价于真实耗时预测。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

## 5. 主流程源码 walkthrough

下面按时间顺序跟主链路。每个小段都只回答一个问题：当前状态发生了什么变化。

| 项 | 说明 |
| --- | --- |
| `planner()` 检查 `planner_hook` | 扩展最早能接管规划，但必须返回合法 `PlannedStmt`。 |
| `standard_planner()` 初始化 `PlannerGlobal` | enable_* 被折叠进 `default_pgs_mask`，parallel hazard 和 tuple_fraction 也在此确定。 |
| `planner_setup_hook` | 标准 planner 已有 glob，但还没递归进入 subquery_planner，适合准备扩展私有状态。 |
| `set_base_rel_pathlists()` | 标准扫描路径生成后触发 `set_rel_pathlist_hook`，扩展可以给 base rel 增加候选。 |
| `add_paths_to_joinrel()` | 标准 join path 生成过程中触发 `set_join_pathlist_hook`，扩展可以补 join path。 |
| `make_one_rel()` | 需要 join search 时可由 `join_search_hook` 替换 join order 搜索。 |
| `create_plan()` | 无论 Path 如何来，最终必须能被 createplan 转成合法 Plan，或由 CustomPath 提供对应 Plan。 |
| `planner_shutdown_hook` | PlannedStmt 已经形成，扩展可以记录信息，但不能留下悬挂 root 指针。 |

### 5.1. `planner()` 检查 `planner_hook`

扩展最早能接管规划，但必须返回合法 `PlannedStmt`。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

### 5.2. `standard_planner()` 初始化 `PlannerGlobal`

enable_* 被折叠进 `default_pgs_mask`，parallel hazard 和 tuple_fraction 也在此确定。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

### 5.3. `planner_setup_hook`

标准 planner 已有 glob，但还没递归进入 subquery_planner，适合准备扩展私有状态。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

这一段常见的误读是把“现在能看到的字段”当成“最终执行器会看到的字段”。多数情况下，当前字段只是下一阶段搜索或整理的材料。

### 5.4. `set_base_rel_pathlists()`

标准扫描路径生成后触发 `set_rel_pathlist_hook`，扩展可以给 base rel 增加候选。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

### 5.5. `add_paths_to_joinrel()`

标准 join path 生成过程中触发 `set_join_pathlist_hook`，扩展可以补 join path。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

### 5.6. `make_one_rel()`

需要 join search 时可由 `join_search_hook` 替换 join order 搜索。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

这一段常见的误读是把“现在能看到的字段”当成“最终执行器会看到的字段”。多数情况下，当前字段只是下一阶段搜索或整理的材料。

### 5.7. `create_plan()`

无论 Path 如何来，最终必须能被 createplan 转成合法 Plan，或由 CustomPath 提供对应 Plan。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

### 5.8. `planner_shutdown_hook`

PlannedStmt 已经形成，扩展可以记录信息，但不能留下悬挂 root 指针。

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
| 1 | hook 不能改变 SQL 权限、可见性、outer join 语义或 row security 语义。 |
| 2 | pathlist hook 加入的 Path 必须标注正确 rows、cost、pathkeys、parallel safety、parameterization。 |
| 3 | enable_* GUC 通常不是硬性语义禁用；某些情况下 planner 仍可能为了合法计划保留受惩罚路径。 |
| 4 | cost 参数只用于比较 Path，不能被解释成毫秒或 I/O 次数的准确预测。 |
| 5 | CustomPath 或扩展 Path 必须遵守 executor contract，不能让 Plan 依赖 planner-local 指针。 |

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
| 1 | 如果 hook 不调用标准 planner 又没有完整实现所有语义检查，容易生成非法 PlannedStmt。 |
| 2 | 如果 pathlist hook 生成的 Path 漏标 required_outer，nested loop 参数化路径会被错误重排。 |
| 3 | 如果 extension 保存 PlannerInfo 到规划后使用，会越过 memory context 生命周期。 |
| 4 | 如果误把 enable_seqscan=off 当成绝对禁止 seqscan，诊断计划时会得出错误结论。 |

遇到 fallback 时要先问它保护的是语义、生命周期、资源成本还是扩展契约。不同原因对应的修复方向完全不同。

### 异常观察 1

如果 hook 不调用标准 planner 又没有完整实现所有语义检查，容易生成非法 PlannedStmt。

验证时不要只看最终节点名，还要回到触发保守路径的源码条件。

### 异常观察 2

如果 pathlist hook 生成的 Path 漏标 required_outer，nested loop 参数化路径会被错误重排。

验证时不要只看最终节点名，还要回到触发保守路径的源码条件。

### 异常观察 3

如果 extension 保存 PlannerInfo 到规划后使用，会越过 memory context 生命周期。

验证时不要只看最终节点名，还要回到触发保守路径的源码条件。

### 异常观察 4

如果误把 enable_seqscan=off 当成绝对禁止 seqscan，诊断计划时会得出错误结论。

验证时不要只看最终节点名，还要回到触发保守路径的源码条件。

## 9. 成本、资源与跨模块传播

planner 成本不只来自最终计划，也来自规划期搜索空间和状态传播。

| 项 | 说明 |
| --- | --- |
| 1 | enable_* 会改变候选数量或 disabled_nodes，进而影响 add_path 的剪枝结果。 |
| 2 | cost GUC 改变相对单位，通常需要结合实际硬件、缓存命中和 workload 校准。 |
| 3 | hook 增加 Path 会扩大搜索空间，过多候选会增加 planner CPU 和内存。 |
| 4 | join_search_hook 可以绕过标准动态规划，可能降低规划时间，也可能牺牲可解释性。 |

这些成本最后会在三个地方表现出来：planner CPU 时间、规划期间内存占用、以及最终 executor 计划的 runtime 成本。诊断时要把三者分开。

## 10. 观测与诊断入口

优化器内部状态多数不可直接从 SQL 看到，但可以通过最终 Plan、日志和断点间接定位。

| 项 | 说明 |
| --- | --- |
| 1 | `EXPLAIN` 能看到最终计划是否使用某类节点，但不能直接说明 hook 曾经加入过哪些失败候选。 |
| 2 | 改变 `enable_hashjoin`、`enable_nestloop`、`enable_mergejoin` 可以快速判断候选空间是否受限。 |
| 3 | 改变 `random_page_cost`、`cpu_tuple_cost` 后比较计划，有助于区分行数估算和成本参数问题。 |
| 4 | gdb 断在 hook 调用点，可以确认扩展是在 base rel、joinrel 还是 whole planner 层介入。 |

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

### 实验 1：enable GUC

对同一 join 查询分别关闭 hashjoin、mergejoin、nestloop，观察最终计划和 fallback。

实验记录至少包含三项：SQL、EXPLAIN 输出中的关键差异、对应源码断点或函数入口。

### 实验 2：cost 参数

调高 random_page_cost，观察 index scan 与 seq scan 的边界变化。

实验记录至少包含三项：SQL、EXPLAIN 输出中的关键差异、对应源码断点或函数入口。

### 实验 3：hook 断点

即使没有自定义扩展，也可在 `planner()`、`set_base_rel_pathlists()`、`add_paths_to_joinrel()` 附近跟踪 hook 检查点。

实验记录至少包含三项：SQL、EXPLAIN 输出中的关键差异、对应源码断点或函数入口。

### 实验 4：CustomPath 阅读

阅读 `CustomPath` 相关结构，理解扩展 Path 如何最终转成 Plan。

实验记录至少包含三项：SQL、EXPLAIN 输出中的关键差异、对应源码断点或函数入口。

实验不追求覆盖所有路径，只追求把一个可见现象解释回源码中的状态变化。

## 13. 讨论题

1. 为什么 pathlist hook 可以添加候选 Path，却不应该直接修改 Query 语义？

2. 什么时候应该调 cost 参数，什么时候应该修统计信息或 SQL 结构？

3. 扩展替换 join search 后，如何证明没有破坏 outer join 或 lateral 的顺序约束？

回答这些问题时，需要明确引用源码阶段和状态对象，避免只用抽象概念解释抽象概念。

## 14. 源码分段阅读

这一节把第 3 节的源码顺序再拆成可执行的阅读任务。每个任务都要能回到本节唯一主问题，而不是停留在“看过这个文件”。

### 14.1. `src/include/optimizer/planner.h`

`planner_hook_type`、`planner_setup_hook_type`、`planner_shutdown_hook_type`。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

### 14.2. `src/backend/optimizer/plan/planner.c`

`planner()` 调用 hook，`standard_planner()` 使用 enable_* 初始化 strategy mask。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

### 14.3. `src/include/optimizer/paths.h`

`set_rel_pathlist_hook_type`、`set_join_pathlist_hook_type`、`join_search_hook_type`。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

### 14.4. `src/backend/optimizer/path/allpaths.c`

`set_rel_pathlist_hook`、`join_search_hook` 的调用点。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

### 14.5. `src/backend/optimizer/path/joinpath.c`

`set_join_pathlist_hook` 的调用点。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

### 14.6. `src/backend/optimizer/path/costsize.c`

cost GUC 和 enable_* 全局变量定义与成本模型入口。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

### 14.7. `src/include/optimizer/cost.h`

cost 参数和 enable_* 声明。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

### 14.8. `src/backend/utils/misc/guc_tables.c`

planner 相关 GUC 的注册位置。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

## 15. 状态迁移检查

下面把主流程再拆成状态迁移。它们不是新的知识点，而是防止把函数调用链读成 API 清单。

### 15.1. `planner()` 检查 `planner_hook`

扩展最早能接管规划，但必须返回合法 `PlannedStmt`。

进入前先确认 `planner_hook` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `planner_setup_hook / planner_shutdown_hook` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.2. `standard_planner()` 初始化 `PlannerGlobal`

enable_* 被折叠进 `default_pgs_mask`，parallel hazard 和 tuple_fraction 也在此确定。

进入前先确认 `planner_setup_hook / planner_shutdown_hook` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `set_rel_pathlist_hook` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.3. `planner_setup_hook`

标准 planner 已有 glob，但还没递归进入 subquery_planner，适合准备扩展私有状态。

进入前先确认 `set_rel_pathlist_hook` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `set_join_pathlist_hook` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.4. `set_base_rel_pathlists()`

标准扫描路径生成后触发 `set_rel_pathlist_hook`，扩展可以给 base rel 增加候选。

进入前先确认 `set_join_pathlist_hook` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `join_search_hook` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.5. `add_paths_to_joinrel()`

标准 join path 生成过程中触发 `set_join_pathlist_hook`，扩展可以补 join path。

进入前先确认 `join_search_hook` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `enable_* GUC` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.6. `make_one_rel()`

需要 join search 时可由 `join_search_hook` 替换 join order 搜索。

进入前先确认 `enable_* GUC` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `cost GUC` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.7. `create_plan()`

无论 Path 如何来，最终必须能被 createplan 转成合法 Plan，或由 CustomPath 提供对应 Plan。

进入前先确认 `cost GUC` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `planner_hook` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.8. `planner_shutdown_hook`

PlannedStmt 已经形成，扩展可以记录信息，但不能留下悬挂 root 指针。

进入前先确认 `planner_hook` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `planner_setup_hook / planner_shutdown_hook` 是否被创建、改写、选择、缓存或转交给下游。

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

- 检查点 1：在 `src/include/optimizer/planner.h` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 2：在 `src/backend/optimizer/plan/planner.c` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 3：在 `src/include/optimizer/paths.h` 中验证：字段是否被后续阶段继续改写。
- 检查点 4：在 `src/backend/optimizer/path/allpaths.c` 中验证：字段是否会越过 planner 生命周期。
- 检查点 5：在 `src/backend/optimizer/path/joinpath.c` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 6：在 `src/backend/optimizer/path/costsize.c` 中验证：最终计划是否能从源码中的同一条状态线解释。
- 检查点 7：在 `src/include/optimizer/cost.h` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 8：在 `src/backend/utils/misc/guc_tables.c` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 9：在 `src/include/optimizer/planner.h` 中验证：字段是否被后续阶段继续改写。
- 检查点 10：在 `src/backend/optimizer/plan/planner.c` 中验证：字段是否会越过 planner 生命周期。
- 检查点 11：在 `src/include/optimizer/paths.h` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 12：在 `src/backend/optimizer/path/allpaths.c` 中验证：最终计划是否能从源码中的同一条状态线解释。
- 检查点 13：在 `src/backend/optimizer/path/joinpath.c` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 14：在 `src/backend/optimizer/path/costsize.c` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 15：在 `src/include/optimizer/cost.h` 中验证：字段是否被后续阶段继续改写。
- 检查点 16：在 `src/backend/utils/misc/guc_tables.c` 中验证：字段是否会越过 planner 生命周期。
- 检查点 17：在 `src/include/optimizer/planner.h` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 18：在 `src/backend/optimizer/plan/planner.c` 中验证：最终计划是否能从源码中的同一条状态线解释。
- 检查点 19：在 `src/include/optimizer/paths.h` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 20：在 `src/backend/optimizer/path/allpaths.c` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 21：在 `src/backend/optimizer/path/joinpath.c` 中验证：字段是否被后续阶段继续改写。
- 检查点 22：在 `src/backend/optimizer/path/costsize.c` 中验证：字段是否会越过 planner 生命周期。
- 检查点 23：在 `src/include/optimizer/cost.h` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 24：在 `src/backend/utils/misc/guc_tables.c` 中验证：最终计划是否能从源码中的同一条状态线解释。
- 检查点 25：在 `src/include/optimizer/planner.h` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 26：在 `src/backend/optimizer/plan/planner.c` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 27：在 `src/include/optimizer/paths.h` 中验证：字段是否被后续阶段继续改写。
- 检查点 28：在 `src/backend/optimizer/path/allpaths.c` 中验证：字段是否会越过 planner 生命周期。
- 检查点 29：在 `src/backend/optimizer/path/joinpath.c` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 30：在 `src/backend/optimizer/path/costsize.c` 中验证：最终计划是否能从源码中的同一条状态线解释。
- 检查点 31：在 `src/include/optimizer/cost.h` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 32：在 `src/backend/utils/misc/guc_tables.c` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 33：在 `src/include/optimizer/planner.h` 中验证：字段是否被后续阶段继续改写。
- 检查点 34：在 `src/backend/optimizer/plan/planner.c` 中验证：字段是否会越过 planner 生命周期。
- 检查点 35：在 `src/include/optimizer/paths.h` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 36：在 `src/backend/optimizer/path/allpaths.c` 中验证：最终计划是否能从源码中的同一条状态线解释。
- 检查点 37：在 `src/backend/optimizer/path/joinpath.c` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 38：在 `src/backend/optimizer/path/costsize.c` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 39：在 `src/include/optimizer/cost.h` 中验证：字段是否被后续阶段继续改写。
- 检查点 40：在 `src/backend/utils/misc/guc_tables.c` 中验证：字段是否会越过 planner 生命周期。
- 检查点 41：在 `src/include/optimizer/planner.h` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 42：在 `src/backend/optimizer/plan/planner.c` 中验证：最终计划是否能从源码中的同一条状态线解释。
- 检查点 43：在 `src/include/optimizer/paths.h` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 44：在 `src/backend/optimizer/path/allpaths.c` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 45：在 `src/backend/optimizer/path/joinpath.c` 中验证：字段是否被后续阶段继续改写。
- 检查点 46：在 `src/backend/optimizer/path/costsize.c` 中验证：字段是否会越过 planner 生命周期。
- 检查点 47：在 `src/include/optimizer/cost.h` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 48：在 `src/backend/utils/misc/guc_tables.c` 中验证：最终计划是否能从源码中的同一条状态线解释。

## 18. 误判与校正

下面这些条目用于把课堂讨论落到排查动作上。每一条都从一个容易误判的现象开始，再给出回到源码的校正路径。

### 18.1. 正确性边界

hook 不能改变 SQL 权限、可见性、outer join 语义或 row security 语义。

先确认语义是否保持，再确认候选路径是否只是搜索空间变化。

源码回看点：`src/include/optimizer/planner.h`。

### 18.2. 正确性边界

pathlist hook 加入的 Path 必须标注正确 rows、cost、pathkeys、parallel safety、parameterization。

先确认语义是否保持，再确认候选路径是否只是搜索空间变化。

源码回看点：`src/backend/optimizer/plan/planner.c`。

### 18.3. 正确性边界

enable_* GUC 通常不是硬性语义禁用；某些情况下 planner 仍可能为了合法计划保留受惩罚路径。

先确认语义是否保持，再确认候选路径是否只是搜索空间变化。

源码回看点：`src/include/optimizer/paths.h`。

### 18.4. 正确性边界

cost 参数只用于比较 Path，不能被解释成毫秒或 I/O 次数的准确预测。

先确认语义是否保持，再确认候选路径是否只是搜索空间变化。

源码回看点：`src/backend/optimizer/path/allpaths.c`。

### 18.5. 正确性边界

CustomPath 或扩展 Path 必须遵守 executor contract，不能让 Plan 依赖 planner-local 指针。

先确认语义是否保持，再确认候选路径是否只是搜索空间变化。

源码回看点：`src/backend/optimizer/path/joinpath.c`。

### 18.6. 异常或 fallback

如果 hook 不调用标准 planner 又没有完整实现所有语义检查，容易生成非法 PlannedStmt。

先找到保守分支的源码条件，再判断它保护的是语义、资源还是生命周期。

源码回看点：`src/backend/optimizer/path/costsize.c`。

### 18.7. 异常或 fallback

如果 pathlist hook 生成的 Path 漏标 required_outer，nested loop 参数化路径会被错误重排。

先找到保守分支的源码条件，再判断它保护的是语义、资源还是生命周期。

源码回看点：`src/include/optimizer/cost.h`。

### 18.8. 异常或 fallback

如果 extension 保存 PlannerInfo 到规划后使用，会越过 memory context 生命周期。

先找到保守分支的源码条件，再判断它保护的是语义、资源还是生命周期。

源码回看点：`src/backend/utils/misc/guc_tables.c`。

### 18.9. 异常或 fallback

如果误把 enable_seqscan=off 当成绝对禁止 seqscan，诊断计划时会得出错误结论。

先找到保守分支的源码条件，再判断它保护的是语义、资源还是生命周期。

源码回看点：`src/include/optimizer/planner.h`。

### 18.10. 成本传播

enable_* 会改变候选数量或 disabled_nodes，进而影响 add_path 的剪枝结果。

同时记录 planner CPU、候选数量和 executor runtime，避免把三种成本混在一起。

源码回看点：`src/backend/optimizer/plan/planner.c`。

### 18.11. 成本传播

cost GUC 改变相对单位，通常需要结合实际硬件、缓存命中和 workload 校准。

同时记录 planner CPU、候选数量和 executor runtime，避免把三种成本混在一起。

源码回看点：`src/include/optimizer/paths.h`。

### 18.12. 成本传播

hook 增加 Path 会扩大搜索空间，过多候选会增加 planner CPU 和内存。

同时记录 planner CPU、候选数量和 executor runtime，避免把三种成本混在一起。

源码回看点：`src/backend/optimizer/path/allpaths.c`。

### 18.13. 成本传播

join_search_hook 可以绕过标准动态规划，可能降低规划时间，也可能牺牲可解释性。

同时记录 planner CPU、候选数量和 executor runtime，避免把三种成本混在一起。

源码回看点：`src/backend/optimizer/path/joinpath.c`。

### 18.14. 观测入口

`EXPLAIN` 能看到最终计划是否使用某类节点，但不能直接说明 hook 曾经加入过哪些失败候选。

用 EXPLAIN 或 gdb 只确认一个假设，确认后再扩大到完整 SQL。

源码回看点：`src/backend/optimizer/path/costsize.c`。

### 18.15. 观测入口

改变 `enable_hashjoin`、`enable_nestloop`、`enable_mergejoin` 可以快速判断候选空间是否受限。

用 EXPLAIN 或 gdb 只确认一个假设，确认后再扩大到完整 SQL。

源码回看点：`src/include/optimizer/cost.h`。

### 18.16. 观测入口

改变 `random_page_cost`、`cpu_tuple_cost` 后比较计划，有助于区分行数估算和成本参数问题。

用 EXPLAIN 或 gdb 只确认一个假设，确认后再扩大到完整 SQL。

源码回看点：`src/backend/utils/misc/guc_tables.c`。

### 18.17. 观测入口

gdb 断在 hook 调用点，可以确认扩展是在 base rel、joinrel 还是 whole planner 层介入。

用 EXPLAIN 或 gdb 只确认一个假设，确认后再扩大到完整 SQL。

源码回看点：`src/include/optimizer/planner.h`。

## 19. 本节小结

- hook 和 GUC 是影响 planner 选择的入口，不是绕开语义边界的入口。
- 扩展能插入搜索空间和成本比较，但最终必须交出合法 PlannedStmt。
- 诊断计划变化时，要把 hook、enable_*、cost、统计信息和 Query 改写分开判断。

把本节内容压缩成一个可迁移规律：

```text
复杂优化器的关键不是把所有规则塞进一个入口，
而是把语义改写、候选搜索、成本比较和执行契约放在不同生命周期里。
```
