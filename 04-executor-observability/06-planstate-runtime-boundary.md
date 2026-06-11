# PostgreSQL Plan 到 PlanState 的状态化边界

## 课程定位

前置知识：已经读过 executor 生命周期的目录规划，并理解 PostgreSQL 的查询执行由 Portal、QueryDesc、EState、PlanState 和 TupleTableSlot 共同完成。前五节完成了 executor 生命周期的 start/run/finish/end 主线。

本节唯一主问题：

```text
planner 输出的 Plan 为什么不能直接执行，ExecInitNode() 为什么要为每类节点构造 PlanState、表达式状态、slot、子节点和运行时私有字段？
```

核心矛盾：planner tree 需要稳定、可复制、偏声明式 vs executor 需要可变、带 ownership、带函数指针、带资源句柄的 runtime state。

学完后应能判断：能判断一个字段应该存在 Plan 中、PlanState 公共区中，还是某个具体节点 State 的私有区中。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

前五节完成了 executor 生命周期的 start/run/finish/end 主线。

04 目录的主线不是“执行器有哪些函数”，而是跟踪一个计划从可执行状态、tuple 流、节点调度、slot 传递，到可观测指标的生命周期。

```text
planner output
  -> ExecutorStart 建立 runtime state
  -> ExecutorRun / ExecutePlan 按需拉取 tuple
  -> ExecutorFinish drain 语句级副作用
  -> ExecutorEnd 按 ownership 顺序清理
  -> EXPLAIN / pg_stat / wait event 从这些边界采样
```

本节只解决自己的唯一主问题。相邻主题会被点到为止，只在解释本节状态推进时出现。

下一节会聚焦 PlanState->ExecProcNode 函数指针如何调度节点。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
Plan 是执行说明书，PlanState 是正在运行的机器；ExecInitNode() 按 nodeTag 分派到 ExecInit*，把只读计划转换为可推进、可 rescan、可 end 的状态树。
```

这个模型的重点不是函数名，而是状态边界：谁创建运行态，谁推进运行态，谁在异常或收尾时释放运行态。

理解时可以把问题压缩成三句话：

- 执行器不会在每一层重新解释完整 SQL 语义。
- 执行器把全局语义提前固化成可推进的 runtime state。
- 每个 runtime state 都必须有明确的 owner、清理点和观测入口。

后面所有源码阅读都围绕这三句话展开。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | src/backend/executor/execProcnode.c | ExecInitNode()：Plan 到 PlanState 的总分派。 |
| 2 | src/include/nodes/execnodes.h | PlanState 公共字段和 EState 指针。 |
| 3 | src/include/nodes/plannodes.h | Plan / Scan / Join / Agg 等 planner-side 节点。 |
| 4 | src/backend/executor/nodeSeqscan.c | ExecInitSeqScan()：scan node 状态化样例。 |
| 5 | src/backend/executor/nodeSort.c | ExecInitSort()：阻塞节点状态化样例。 |
| 6 | src/backend/executor/nodeAgg.c | ExecInitAgg()：复杂运行时私有状态样例。 |
| 7 | src/backend/executor/nodeModifyTable.c | ExecInitModifyTable()：DML runtime 边界样例。 |
| 8 | src/backend/executor/execExpr.c | ExecInitExpr() / ExecInitQual()：表达式从 Expr 到 ExprState。 |

阅读顺序按 mental model 排列，不按文件名排序。先看入口，再看状态结构，再看 ownership 和 cleanup，最后看观测或扩展边界。

本节源码核对基线：

```text
cd /home/highgo/postgres
git rev-parse --abbrev-ref HEAD
git rev-parse HEAD
```

期望输出分别是 `master` 和本课开头写明的提交号。

## 4. 关键状态与边界

状态不是字段清单。一个字段只有放回创建时机、访问者、生命周期和异常路径中才有语义。

| 状态 | 语义边界 |
| --- | --- |
| Plan | planner 产物，描述成本、目标列、子计划和节点类型。 |
| PlanState | executor 公共运行态，保存 plan 指针、EState 指针、函数指针、表达式状态和 slot。 |
| node-specific State | SeqScanState、AggState、SortState、ModifyTableState 等具体节点私有状态。 |
| ExprState | 表达式执行期状态，避免每行重新解释原始 Expr tree。 |
| lefttree/righttree | 运行时子节点链接，对应 Plan 的子树。 |
| ps_ResultTupleSlot | 节点输出 slot，形状由 targetlist 或子节点决定。 |
| ps_ExprContext | 节点表达式求值 context，承载 scan/inner/outer tuple。 |
| chgParam | 参数变化导致的 rescan 标记。 |

本节最容易混淆的边界有四个：

- planner-time 描述与 executor-time 状态不同。
- backend-local 指针与跨 backend 可见状态不同。
- memory context 释放内存，不等于释放所有外部资源。
- 顶层调用边界与节点内部状态机边界不同。

读源码时要持续追问：这个状态现在是谁的，下一步谁会改变它，如果中途 ERROR，谁会兜底。

## 5. 主流程源码 walkthrough

下面按时间顺序读主链路。每一步都回答“状态发生了什么变化”，而不是只记住函数名。

### 5.1. 步骤 1

InitPlan() 调用 ExecInitNode(root Plan, estate, eflags)。

阅读这一段时的重点不是记住第 1 个函数名，而是确认状态边界：读到这里先确认这一步是在创建状态、注册外部资源，还是只是在写入控制字段。

### 5.2. 步骤 2

ExecInitNode() 对 NULL node 直接返回 NULL，叶子结束。

阅读这一段时的重点不是记住第 2 个函数名，而是确认状态边界：断点停在这里时，优先打印 owner 指针，再打印本步刚写入的字段。

### 5.3. 步骤 3

它先 check_stack_depth，避免深计划树初始化时栈溢出。

阅读这一段时的重点不是记住第 3 个函数名，而是确认状态边界：这一步如果提前失败，后续状态还没有 owner，不能按完整 executor cleanup 想象。

### 5.4. 步骤 4

switch(nodeTag(node)) 把 T_SeqScan 分派到 ExecInitSeqScan，把 T_Agg 分派到 ExecInitAgg。

阅读这一段时的重点不是记住第 4 个函数名，而是确认状态边界：这一步如果已经完成，后续代码就可以假定对应 runtime state 已经稳定。

### 5.5. 步骤 5

每个 ExecInit* 创建具体 State 节点，并设置 ps.plan 和 ps.state。

阅读这一段时的重点不是记住第 5 个函数名，而是确认状态边界：这里要区分“指针存在”和“语义已经注册”两件事。

### 5.6. 步骤 6

节点初始化自己的 ExprContext、scan slot、result slot、projection、qual 和子节点。

阅读这一段时的重点不是记住第 6 个函数名，而是确认状态边界：这里的状态变化通常会影响后续 ExecProcNode、slot、snapshot 或 receiver 的解释。

### 5.7. 步骤 7

表达式通过 ExecInitExpr / ExecInitQual 变成 ExprState。

阅读这一段时的重点不是记住第 7 个函数名，而是确认状态边界：如果以后诊断慢查询，这一步更像 setup 成本，不应误归到单个 tuple。

### 5.8. 步骤 8

子计划递归调用 ExecInitNode，形成与 plan tree 形状相似但不完全相同的 PlanState tree。

阅读这一段时的重点不是记住第 8 个函数名，而是确认状态边界：如果以后诊断异常退出，这一步要回看是否已经进入 ResourceOwner、MemoryContext 或 snapshot 注册边界。

### 5.9. 步骤 9

节点设置 ExecProcNode 函数指针，或者先设置为具体函数再被 ExecSetExecProcNode 包装。

阅读这一段时的重点不是记住第 9 个函数名，而是确认状态边界：这一步的调用者协议比函数内部细节更重要：调用者是否允许重复进入，是否允许部分执行。

### 5.10. 步骤 10

初始化结束后 PlanState 已经包含运行需要的 Relation、slot、私有内存和函数入口。

阅读这一段时的重点不是记住第 10 个函数名，而是确认状态边界：这一步之后要看下一个 owner 边界，而不是继续把所有状态当作局部变量。

### 5.11. 步骤 11

ExecutorRun 不再 switch plan nodeTag 初始化资源，只通过 PlanState 推进。

阅读这一段时的重点不是记住第 11 个函数名，而是确认状态边界：如果这一步涉及 hook 或 wrapper，要继续追到标准实现或真实节点函数。

### 5.12. 步骤 12

ExecutorEnd 反向用 ExecEndNode 根据 PlanState nodeTag 调用对应 cleanup。

阅读这一段时的重点不是记住第 12 个函数名，而是确认状态边界：如果这一步涉及 slot 或 tuple，必须同时确认数据内容和底层资源是否同寿命。

主链路可以压缩成：

```text
01. InitPlan() 调用 ExecInitNode(root Plan, estate, eflags)。
02. ExecInitNode() 对 NULL node 直接返回 NULL，叶子结束。
03. 它先 check_stack_depth，避免深计划树初始化时栈溢出。
04. switch(nodeTag(node)) 把 T_SeqScan 分派到 ExecInitSeqScan，把 T_Agg 分派到 ExecInitAgg。
05. 每个 ExecInit* 创建具体 State 节点，并设置 ps.plan 和 ps.state。
06. 节点初始化自己的 ExprContext、scan slot、result slot、projection、qual 和子节点。
07. 表达式通过 ExecInitExpr / ExecInitQual 变成 ExprState。
08. 子计划递归调用 ExecInitNode，形成与 plan tree 形状相似但不完全相同的 PlanState tree。
09. 节点设置 ExecProcNode 函数指针，或者先设置为具体函数再被 ExecSetExecProcNode 包装。
10. 初始化结束后 PlanState 已经包含运行需要的 Relation、slot、私有内存和函数入口。
11. ExecutorRun 不再 switch plan nodeTag 初始化资源，只通过 PlanState 推进。
12. ExecutorEnd 反向用 ExecEndNode 根据 PlanState nodeTag 调用对应 cleanup。
```

## 6. 状态随时间推进的故事

把源码主线换成一个对象的生命周期，会更容易看出本节 tension。

### 6.1. 创建前

在“创建前”阶段，关键观察点是：InitPlan() 调用 ExecInitNode(root Plan, estate, eflags)。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

### 6.2. 创建时

在“创建时”阶段，关键观察点是：ExecInitNode() 对 NULL node 直接返回 NULL，叶子结束。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

### 6.3. 第一次使用前

在“第一次使用前”阶段，关键观察点是：它先 check_stack_depth，避免深计划树初始化时栈溢出。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

### 6.4. 正常推进中

在“正常推进中”阶段，关键观察点是：switch(nodeTag(node)) 把 T_SeqScan 分派到 ExecInitSeqScan，把 T_Agg 分派到 ExecInitAgg。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

### 6.5. 遇到边界时

在“遇到边界时”阶段，关键观察点是：每个 ExecInit* 创建具体 State 节点，并设置 ps.plan 和 ps.state。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

### 6.6. 提前停止时

在“提前停止时”阶段，关键观察点是：节点初始化自己的 ExprContext、scan slot、result slot、projection、qual 和子节点。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

### 6.7. 收尾时

在“收尾时”阶段，关键观察点是：表达式通过 ExecInitExpr / ExecInitQual 变成 ExprState。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

### 6.8. ERROR 后

在“ERROR 后”阶段，关键观察点是：子计划递归调用 ExecInitNode，形成与 plan tree 形状相似但不完全相同的 PlanState tree。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

## 7. 生命周期 / ownership / cleanup

- Plan 通常属于规划器或 query context；PlanState 属于 executor es_query_cxt。
- ExecInit* 创建 State，ExecProcNode 推进 State，ExecEnd* 释放 State 持有的外部资源。
- PlanState 持有 Plan 指针，但不能把 Plan 的只读字段误当成运行时可写状态。
- 节点私有状态生命周期和整个 executor invocation 绑定，除非节点自己有更短 context。
- rescan 改变运行态，不改变 planner tree。

用 ownership 语言重写本节：

| 问题 | 判断方式 |
| --- | --- |
| 谁创建 | 看入口函数和 memory context 切换点。 |
| 谁持有 | 看 QueryDesc、EState、PlanState 或 slot 列表中的指针。 |
| 谁释放 | 看 ExecutorFinish、ExecutorEnd、ExecEndNode、ExecResetTupleTable 或 ResourceOwner。 |
| ERROR 怎么办 | 看资源是否有 ResourceOwner、MemoryContext、snapshot register 或 callback 兜底。 |
| 能否跨边界保存 | 看对象是否 backend-local、是否依赖 slot 内容、buffer pin 或 per-tuple context。 |

## 8. 正确性机制层次

- PlanState->state 指向同一个 EState，保证 snapshot、params、tuple table 和 output cid 统一。
- ExprState 预初始化保证每行表达式求值在确定的 ExprContext 中运行。
- slot 类型在初始化期选择，后续节点按 slot ops 协议访问。
- chgParam 把参数变化延迟到 ExecProcNode 入口统一处理。
- 不同节点的 State 类型隔离私有状态，避免公共 PlanState 变成巨型结构。
- ExecEndNode 用 PlanState nodeTag 分派 cleanup，和 ExecInitNode 形成对称协议。

这些机制互相补位。snapshot 解决可见性，不解决内存生命周期；MemoryContext 解决内存批量释放，不解决 buffer pin；函数指针解决分派成本，不解决节点协议正确性。

| 层次 | 保证 | 不能误解为 |
| --- | --- | --- |
| snapshot / command id | tuple 可见性和写入标记边界 | 并发互斥或资源释放 |
| PlanState / slot 协议 | 节点间传递和推进方式一致 | 所有节点都非阻塞 |
| MemoryContext | backend-local 内存归属清楚 | 外部资源自动释放 |
| ResourceOwner / pin / refcount | 外部资源不会被过早回收 | 对象语义仍然新鲜 |
| hook / instrumentation | 观测或扩展入口 | 可以任意改变核心语义 |

## 9. 错误路径 / 异常路径 / fallback

- 有些节点不返回 tuple，需要 MultiExecProcNode 参与，例如 bitmap 或 hash 构建路径。
- ForeignScan 和 CustomScan 把一部分状态交给 FDW 或扩展 callbacks。
- SubPlanState 的初始化有独立顺序，不能简单等同于主 plan tree 递归。
- 并行 worker 中 PlanState 还会 attach DSM 或 worker instrumentation。
- EPQ 会创建 cut-down executor state，不是完整顶层 ExecutorStart 的重复。

PostgreSQL executor 的异常路径不能按 C++ 析构模型理解。`ereport(ERROR)` 可能 longjmp 到上层，普通 C 栈上的清理代码不会自然逐层执行。

因此本节所有状态都要问一句：如果这个点 ERROR，已经注册的资源由谁释放，尚未注册的资源是否会泄漏或变成悬挂引用。

## 10. 成本、资源与跨模块传播

- PlanState 初始化成本与计划节点数和表达式复杂度成正比。
- 表达式预编译和 slot 初始化提高每行执行效率。
- 节点私有结构让 hot path 少做类型判断，但增加 init/end 复杂性。
- 复杂节点如 Agg/Sort/ModifyTable 的 init 可能打开 relation、准备 projection 或分配大量数组。
- CustomScan/ForeignScan 增加 callback 间接层，但保持 executor core 不理解外部实现细节。

executor 的成本常常不是单个函数慢，而是边界次数被 workload 放大：每行、每节点、每次 FETCH、每个 trigger、每个 slot clear，都可能成为可见成本。

| 放大因素 | 典型表现 |
| --- | --- |
| tuple 数 | ExecProcNode、slot deform、per-tuple reset 和 receiver 调用被放大。 |
| plan node 数 | ExecInitNode、ExecEndNode、instrumentation 对象和 EXPLAIN 输出被放大。 |
| relation 数 | range table relation 打开、ResultRelInfo、锁和权限检查被放大。 |
| FETCH 次数 | ExecutorRun 边界、DestReceiver startup/shutdown 和 Portal 状态切换被放大。 |
| 副作用数量 | trigger、constraint、FDW flush、ModifyTable drain 被放大。 |

## 11. 观测与诊断入口

- gdb 断在 ExecInitNode 可以看 Plan nodeTag 到 PlanState nodeTag 的转换。
- 断在 ExecInitSeqScan 可观察 scanstate->ss.ps.plan 和 scanstate->ss.ps.state。
- EXPLAIN 展示的是 Plan 信息，但 EXPLAIN ANALYZE 的 actual 信息来自 PlanState instrumentation。
- 打印 planstate->ps_ResultTupleDesc 可以看到节点输出形状已在 init 阶段定好。
- perf 中的 ExecInit* 成本通常只出现在查询开头，不在每行 hot path。

建议的断点顺序：

1. `ExecInitNode`
2. `PlanState`
3. `ExecInitSeqScan`
4. `ExecInitSort`
5. `ExecInitAgg`
6. `ExecInitModifyTable`
7. `ExecInitExpr`
8. `ExecEndNode`

断点不只是为了停住程序。每个断点都应该带一个问题：这个状态是否已经初始化，owner 是谁，下一步是否会被改写。

## 12. 课堂实验

### 实验 1

对 ExecInitNode 设置条件断点 nodeTag(node)==T_SeqScan，执行简单 SELECT。

观察点：
- 断点前后同一个字段是否发生变化。
- 这个变化是否发生在本节主流程上。
- 如果把 SQL 改成 cursor、EXPLAIN、DML 或带 trigger，边界是否改变。

### 实验 2

打印 result->ExecProcNodeReal 或 result->ExecProcNode，观察初始化后的函数指针。

观察点：
- 断点前后同一个字段是否发生变化。
- 这个变化是否发生在本节主流程上。
- 如果把 SQL 改成 cursor、EXPLAIN、DML 或带 trigger，边界是否改变。

### 实验 3

比较 AggState 和 SeqScanState 的大小与字段，理解公共 PlanState 与私有 State 的分工。

观察点：
- 断点前后同一个字段是否发生变化。
- 这个变化是否发生在本节主流程上。
- 如果把 SQL 改成 cursor、EXPLAIN、DML 或带 trigger，边界是否改变。

### 实验 4

执行带 SubPlan 的查询，观察 estate->es_subplanstates 先于主树初始化。

观察点：
- 断点前后同一个字段是否发生变化。
- 这个变化是否发生在本节主流程上。
- 如果把 SQL 改成 cursor、EXPLAIN、DML 或带 trigger，边界是否改变。

### 实验 5

执行 SELECT * FROM t ORDER BY a，观察 SortState 的 result slot 与 tuplesort 状态。

观察点：
- 断点前后同一个字段是否发生变化。
- 这个变化是否发生在本节主流程上。
- 如果把 SQL 改成 cursor、EXPLAIN、DML 或带 trigger，边界是否改变。

### 实验 6

在 ExecEndNode 中打印 nodeTag(node)，确认 cleanup 使用运行态 nodeTag。

观察点：
- 断点前后同一个字段是否发生变化。
- 这个变化是否发生在本节主流程上。
- 如果把 SQL 改成 cursor、EXPLAIN、DML 或带 trigger，边界是否改变。

实验环境建议：

```text
CREATE TABLE exec_obs_t(id int primary key, v text);
INSERT INTO exec_obs_t SELECT g, 'v-' || g FROM generate_series(1, 20) g;
ANALYZE exec_obs_t;
```

## 13. 源码阅读练习

下面不是 API 清单，而是一条可复现的阅读路线。每个点都要回答它如何服务本节唯一主问题。

### 练习 1：`ExecInitNode`

在 `/home/highgo/postgres` 中定位 `ExecInitNode`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

### 练习 2：`PlanState`

在 `/home/highgo/postgres` 中定位 `PlanState`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

### 练习 3：`ExecInitSeqScan`

在 `/home/highgo/postgres` 中定位 `ExecInitSeqScan`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

### 练习 4：`ExecInitSort`

在 `/home/highgo/postgres` 中定位 `ExecInitSort`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

### 练习 5：`ExecInitAgg`

在 `/home/highgo/postgres` 中定位 `ExecInitAgg`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

### 练习 6：`ExecInitModifyTable`

在 `/home/highgo/postgres` 中定位 `ExecInitModifyTable`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

### 练习 7：`ExecInitExpr`

在 `/home/highgo/postgres` 中定位 `ExecInitExpr`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

### 练习 8：`ExecEndNode`

在 `/home/highgo/postgres` 中定位 `ExecEndNode`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

## 14. 常见误区

- 不要把 PlanState 看成 Plan 的浅拷贝；它是带资源和函数指针的运行态。
- 不要在运行期修改 Plan 字段来保存进度。
- 不要以为所有运行态都在 PlanState 公共字段中，复杂节点大量状态在私有 State。
- 不要把 ExecInitNode 的 switch 当成 API 清单，它表达的是类型到 lifecycle protocol 的绑定。
- 不要忽略 ExprState；表达式执行也是从声明式 tree 到 runtime state 的转换。

这些误区的共同点是把 runtime 状态当成静态结构，或者把静态计划当成可以直接推进的状态机。

## 15. 讨论题

1. 为什么不把 ExecProcNode 函数指针直接放在 Plan 里？
2. PlanState 公共字段增加一个新字段的成本是什么？
3. 哪些状态适合在 ExecInit* 中预计算，哪些必须延迟到第一次 ExecProcNode？
4. CustomScan 如何在不修改 executor core 的情况下接入 PlanState 协议？

讨论时要尽量回到一个具体 runtime 现象，比如一次 FETCH、一次 EXPLAIN ANALYZE、一个 trigger 或一个 slot materialize，而不是停留在抽象偏好。

## 16. 本节小结

本节唯一主问题是：planner 输出的 Plan 为什么不能直接执行，ExecInitNode() 为什么要为每类节点构造 PlanState、表达式状态、slot、子节点和运行时私有字段？

可以带走的一句话模型是：Plan 是执行说明书，PlanState 是正在运行的机器；ExecInitNode() 按 nodeTag 分派到 ExecInit*，把只读计划转换为可推进、可 rescan、可 end 的状态树。

- 源码阅读应先找状态创建点，再找推进点，最后找 cleanup。
- 任何字段都要和 owner、生命周期、异常路径一起理解。
- 观测数据只有能回到具体 runtime 边界时，才适合用于诊断。

下一节会聚焦 PlanState->ExecProcNode 函数指针如何调度节点。

## 17. 从现象回源码的诊断案例

下面的案例都只服务本节主问题。它们的共同读法是先看现象，再找 runtime 边界，最后回到 owner 和 cleanup。

### 案例 1：第一次执行停在入口但状态还没有建立

现象：

- 你在 `ExecutorRun` 附近停住，看到 SQL 还没有返回，但某些 executor 状态已经改变。
- 这时不要急着沿调用栈往下追完整执行，而要先回到本节主问题：planner 输出的 Plan 为什么不能直接执行，ExecInitNode() 为什么要为每类节点构造 PlanState、表达式状态、slot、子节点和运行时私有字段？
- 如果现象来自 EXPLAIN、cursor、trigger 或扩展 hook，要先确认它落在哪个 executor 生命周期边界。

源码回看：

- 入口函数：`ExecutorRun`。
- 本步主线：InitPlan() 调用 ExecInitNode(root Plan, estate, eflags)。
- 关键状态：`Plan`，语义是：planner 产物，描述成本、目标列、子计划和节点类型。
- 相邻状态：`PlanState`，用来判断这一步之后谁继续持有资源。

判断方式：

- 如果字段刚被写入但还没有注册到 owner，异常路径通常只能靠当前函数局部清理。
- 如果状态已经挂到 EState、PlanState、slot table 或 snapshot manager，后续 cleanup 应该能从 owner 找回。
- 如果这个状态只影响观测，不影响语义，诊断时不要把它解释成执行计划行为改变。
- 如果这个状态影响 tuple 生命周期，必须同时检查 slot、TupleDesc、buffer pin 或 per-tuple context。

### 案例 2：查询只返回少量行但内部状态已经推进

现象：

- 你在 `ExecutorRun` 附近停住，看到 SQL 还没有返回，但某些 executor 状态已经改变。
- 这时不要急着沿调用栈往下追完整执行，而要先回到本节主问题：planner 输出的 Plan 为什么不能直接执行，ExecInitNode() 为什么要为每类节点构造 PlanState、表达式状态、slot、子节点和运行时私有字段？
- 如果现象来自 EXPLAIN、cursor、trigger 或扩展 hook，要先确认它落在哪个 executor 生命周期边界。

源码回看：

- 入口函数：`ExecutorRun`。
- 本步主线：ExecInitNode() 对 NULL node 直接返回 NULL，叶子结束。
- 关键状态：`PlanState`，语义是：executor 公共运行态，保存 plan 指针、EState 指针、函数指针、表达式状态和 slot。
- 相邻状态：`node-specific State`，用来判断这一步之后谁继续持有资源。

判断方式：

- 如果字段刚被写入但还没有注册到 owner，异常路径通常只能靠当前函数局部清理。
- 如果状态已经挂到 EState、PlanState、slot table 或 snapshot manager，后续 cleanup 应该能从 owner 找回。
- 如果这个状态只影响观测，不影响语义，诊断时不要把它解释成执行计划行为改变。
- 如果这个状态影响 tuple 生命周期，必须同时检查 slot、TupleDesc、buffer pin 或 per-tuple context。

### 案例 3：EXPLAIN ANALYZE 时间与节点时间不一致

现象：

- 你在 `ExecutorRun` 附近停住，看到 SQL 还没有返回，但某些 executor 状态已经改变。
- 这时不要急着沿调用栈往下追完整执行，而要先回到本节主问题：planner 输出的 Plan 为什么不能直接执行，ExecInitNode() 为什么要为每类节点构造 PlanState、表达式状态、slot、子节点和运行时私有字段？
- 如果现象来自 EXPLAIN、cursor、trigger 或扩展 hook，要先确认它落在哪个 executor 生命周期边界。

源码回看：

- 入口函数：`ExecutorRun`。
- 本步主线：它先 check_stack_depth，避免深计划树初始化时栈溢出。
- 关键状态：`node-specific State`，语义是：SeqScanState、AggState、SortState、ModifyTableState 等具体节点私有状态。
- 相邻状态：`ExprState`，用来判断这一步之后谁继续持有资源。

判断方式：

- 如果字段刚被写入但还没有注册到 owner，异常路径通常只能靠当前函数局部清理。
- 如果状态已经挂到 EState、PlanState、slot table 或 snapshot manager，后续 cleanup 应该能从 owner 找回。
- 如果这个状态只影响观测，不影响语义，诊断时不要把它解释成执行计划行为改变。
- 如果这个状态影响 tuple 生命周期，必须同时检查 slot、TupleDesc、buffer pin 或 per-tuple context。

### 案例 4：cursor 或 receiver 让执行被分成多段

现象：

- 你在 `ExecutorRun` 附近停住，看到 SQL 还没有返回，但某些 executor 状态已经改变。
- 这时不要急着沿调用栈往下追完整执行，而要先回到本节主问题：planner 输出的 Plan 为什么不能直接执行，ExecInitNode() 为什么要为每类节点构造 PlanState、表达式状态、slot、子节点和运行时私有字段？
- 如果现象来自 EXPLAIN、cursor、trigger 或扩展 hook，要先确认它落在哪个 executor 生命周期边界。

源码回看：

- 入口函数：`ExecutorRun`。
- 本步主线：switch(nodeTag(node)) 把 T_SeqScan 分派到 ExecInitSeqScan，把 T_Agg 分派到 ExecInitAgg。
- 关键状态：`ExprState`，语义是：表达式执行期状态，避免每行重新解释原始 Expr tree。
- 相邻状态：`lefttree/righttree`，用来判断这一步之后谁继续持有资源。

判断方式：

- 如果字段刚被写入但还没有注册到 owner，异常路径通常只能靠当前函数局部清理。
- 如果状态已经挂到 EState、PlanState、slot table 或 snapshot manager，后续 cleanup 应该能从 owner 找回。
- 如果这个状态只影响观测，不影响语义，诊断时不要把它解释成执行计划行为改变。
- 如果这个状态影响 tuple 生命周期，必须同时检查 slot、TupleDesc、buffer pin 或 per-tuple context。

### 案例 5：ERROR 发生后没有按普通调用栈返回

现象：

- 你在 `ExecutorRun` 附近停住，看到 SQL 还没有返回，但某些 executor 状态已经改变。
- 这时不要急着沿调用栈往下追完整执行，而要先回到本节主问题：planner 输出的 Plan 为什么不能直接执行，ExecInitNode() 为什么要为每类节点构造 PlanState、表达式状态、slot、子节点和运行时私有字段？
- 如果现象来自 EXPLAIN、cursor、trigger 或扩展 hook，要先确认它落在哪个 executor 生命周期边界。

源码回看：

- 入口函数：`ExecutorRun`。
- 本步主线：每个 ExecInit* 创建具体 State 节点，并设置 ps.plan 和 ps.state。
- 关键状态：`lefttree/righttree`，语义是：运行时子节点链接，对应 Plan 的子树。
- 相邻状态：`ps_ResultTupleSlot`，用来判断这一步之后谁继续持有资源。

判断方式：

- 如果字段刚被写入但还没有注册到 owner，异常路径通常只能靠当前函数局部清理。
- 如果状态已经挂到 EState、PlanState、slot table 或 snapshot manager，后续 cleanup 应该能从 owner 找回。
- 如果这个状态只影响观测，不影响语义，诊断时不要把它解释成执行计划行为改变。
- 如果这个状态影响 tuple 生命周期，必须同时检查 slot、TupleDesc、buffer pin 或 per-tuple context。

### 案例 6：扩展 hook 看到的边界与核心 executor 边界不同

现象：

- 你在 `ExecutorRun` 附近停住，看到 SQL 还没有返回，但某些 executor 状态已经改变。
- 这时不要急着沿调用栈往下追完整执行，而要先回到本节主问题：planner 输出的 Plan 为什么不能直接执行，ExecInitNode() 为什么要为每类节点构造 PlanState、表达式状态、slot、子节点和运行时私有字段？
- 如果现象来自 EXPLAIN、cursor、trigger 或扩展 hook，要先确认它落在哪个 executor 生命周期边界。

源码回看：

- 入口函数：`ExecutorRun`。
- 本步主线：节点初始化自己的 ExprContext、scan slot、result slot、projection、qual 和子节点。
- 关键状态：`ps_ResultTupleSlot`，语义是：节点输出 slot，形状由 targetlist 或子节点决定。
- 相邻状态：`ps_ExprContext`，用来判断这一步之后谁继续持有资源。

判断方式：

- 如果字段刚被写入但还没有注册到 owner，异常路径通常只能靠当前函数局部清理。
- 如果状态已经挂到 EState、PlanState、slot table 或 snapshot manager，后续 cleanup 应该能从 owner 找回。
- 如果这个状态只影响观测，不影响语义，诊断时不要把它解释成执行计划行为改变。
- 如果这个状态影响 tuple 生命周期，必须同时检查 slot、TupleDesc、buffer pin 或 per-tuple context。

## 18. 源码断点与字段记录表

下面这张表用于课堂现场跟读。每一行都应该实际在 gdb 中停一次，记录进入前和返回后的字段差异。

| 序号 | 断点 | 进入前看什么 | 返回后看什么 | 常见误判 |
| --- | --- | --- | --- | --- |
| 1 | `ExecutorRun` | `Plan` 是否已经有效 | InitPlan() 调用 ExecInitNode(root Plan, estate, eflags)。 | 把字段存在误认为语义已经完成 |
| 2 | `ExecutorRun` | `PlanState` 是否已经有效 | ExecInitNode() 对 NULL node 直接返回 NULL，叶子结束。 | 把字段存在误认为语义已经完成 |
| 3 | `ExecutorRun` | `node-specific State` 是否已经有效 | 它先 check_stack_depth，避免深计划树初始化时栈溢出。 | 把字段存在误认为语义已经完成 |
| 4 | `ExecutorRun` | `ExprState` 是否已经有效 | switch(nodeTag(node)) 把 T_SeqScan 分派到 ExecInitSeqScan，把 T_Agg 分派到 ExecInitAgg。 | 把字段存在误认为语义已经完成 |
| 5 | `ExecutorRun` | `lefttree/righttree` 是否已经有效 | 每个 ExecInit* 创建具体 State 节点，并设置 ps.plan 和 ps.state。 | 把字段存在误认为语义已经完成 |
| 6 | `ExecutorRun` | `ps_ResultTupleSlot` 是否已经有效 | 节点初始化自己的 ExprContext、scan slot、result slot、projection、qual 和子节点。 | 把字段存在误认为语义已经完成 |
| 7 | `ExecutorRun` | `ps_ExprContext` 是否已经有效 | 表达式通过 ExecInitExpr / ExecInitQual 变成 ExprState。 | 把字段存在误认为语义已经完成 |
| 8 | `ExecutorRun` | `chgParam` 是否已经有效 | 子计划递归调用 ExecInitNode，形成与 plan tree 形状相似但不完全相同的 PlanState tree。 | 把字段存在误认为语义已经完成 |
| 9 | `ExecutorRun` | `Plan` 是否已经有效 | 节点设置 ExecProcNode 函数指针，或者先设置为具体函数再被 ExecSetExecProcNode 包装。 | 把字段存在误认为语义已经完成 |
| 10 | `ExecutorRun` | `PlanState` 是否已经有效 | 初始化结束后 PlanState 已经包含运行需要的 Relation、slot、私有内存和函数入口。 | 把字段存在误认为语义已经完成 |
| 11 | `ExecutorRun` | `node-specific State` 是否已经有效 | ExecutorRun 不再 switch plan nodeTag 初始化资源，只通过 PlanState 推进。 | 把字段存在误认为语义已经完成 |
| 12 | `ExecutorRun` | `ExprState` 是否已经有效 | ExecutorEnd 反向用 ExecEndNode 根据 PlanState nodeTag 调用对应 cleanup。 | 把字段存在误认为语义已经完成 |

记录时建议额外写三列：

- owner：这个状态归属于 QueryDesc、EState、PlanState、slot、Portal，还是其它模块。
- cleanup：正常路径由哪个函数释放，ERROR 路径由哪个机制兜底。
- observation：能在 EXPLAIN、pg_stat、wait event、gdb、perf 或日志中看到什么。

## 19. 复盘问题

1. 本节唯一主问题是否能用一句 runtime 模型回答。
2. 本节最关键的状态是否有明确 owner。
3. 主流程是否能从入口跟到 cleanup，而不是只停在中间函数。
4. 异常路径是否能说清楚哪些资源已经注册，哪些还没有。
5. 观测入口是否能回到源码中的具体状态变化。
6. 成本解释是否说明了被什么维度放大。
7. 相邻模块是否只是服务本节主问题，没有扩展成第二个主问题。
8. 最后得到的规律是否能迁移到其它 executor 主题。

如果这些问题都能回答，本节就不是函数目录，而是一条可以用于诊断真实执行器问题的状态推进链。

### 复走 1：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：InitPlan() 调用 ExecInitNode(root Plan, estate, eflags)。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 2：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：ExecInitNode() 对 NULL node 直接返回 NULL，叶子结束。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 3：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：它先 check_stack_depth，避免深计划树初始化时栈溢出。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 4：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：switch(nodeTag(node)) 把 T_SeqScan 分派到 ExecInitSeqScan，把 T_Agg 分派到 ExecInitAgg。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 5：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：每个 ExecInit* 创建具体 State 节点，并设置 ps.plan 和 ps.state。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 6：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：节点初始化自己的 ExprContext、scan slot、result slot、projection、qual 和子节点。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 7：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：表达式通过 ExecInitExpr / ExecInitQual 变成 ExprState。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 8：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：子计划递归调用 ExecInitNode，形成与 plan tree 形状相似但不完全相同的 PlanState tree。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 9：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：节点设置 ExecProcNode 函数指针，或者先设置为具体函数再被 ExecSetExecProcNode 包装。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 10：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：初始化结束后 PlanState 已经包含运行需要的 Relation、slot、私有内存和函数入口。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 11：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：ExecutorRun 不再 switch plan nodeTag 初始化资源，只通过 PlanState 推进。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 12：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：ExecutorEnd 反向用 ExecEndNode 根据 PlanState nodeTag 调用对应 cleanup。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 13：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：InitPlan() 调用 ExecInitNode(root Plan, estate, eflags)。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 14：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：ExecInitNode() 对 NULL node 直接返回 NULL，叶子结束。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 15：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：它先 check_stack_depth，避免深计划树初始化时栈溢出。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 16：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：switch(nodeTag(node)) 把 T_SeqScan 分派到 ExecInitSeqScan，把 T_Agg 分派到 ExecInitAgg。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 17：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：每个 ExecInit* 创建具体 State 节点，并设置 ps.plan 和 ps.state。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 18：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：节点初始化自己的 ExprContext、scan slot、result slot、projection、qual 和子节点。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 19：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：表达式通过 ExecInitExpr / ExecInitQual 变成 ExprState。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 20：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：子计划递归调用 ExecInitNode，形成与 plan tree 形状相似但不完全相同的 PlanState tree。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 21：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：节点设置 ExecProcNode 函数指针，或者先设置为具体函数再被 ExecSetExecProcNode 包装。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 22：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：初始化结束后 PlanState 已经包含运行需要的 Relation、slot、私有内存和函数入口。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 23：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：ExecutorRun 不再 switch plan nodeTag 初始化资源，只通过 PlanState 推进。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 24：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：ExecutorEnd 反向用 ExecEndNode 根据 PlanState nodeTag 调用对应 cleanup。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 25：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：InitPlan() 调用 ExecInitNode(root Plan, estate, eflags)。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 26：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：ExecInitNode() 对 NULL node 直接返回 NULL，叶子结束。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 27：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：它先 check_stack_depth，避免深计划树初始化时栈溢出。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。
## 20. 一页纸定位模板

这部分用于把课程内容压缩成排查动作。不要从这里开始背函数名；先带着现象进入，再沿状态边界回源码。

### 定位 1：执行还没返回，但 gdb 已经停在 executor 内部

- 先停在 `ExecInitNode`，确认当前调用属于 `PostgreSQL Plan 到 PlanState 的状态化边界` 的哪一个生命周期边界。
- 打印 `Plan`，不要只看字段值，还要解释它的语义：planner 产物，描述成本、目标列、子计划和节点类型。
- 向上追一层调用者，确认是谁要求 executor 进入这个边界。
- 向下追一层被调用者，确认这一步是否注册了资源、改写了状态或返回了 slot。
- 如果现象来自观测指标，判断指标采样是在进入本步之前、执行过程中，还是返回之后。
- 如果现象来自异常路径，判断资源是否已经挂到 owner；没有 owner 的对象必须由当前路径处理。

### 定位 2：EXPLAIN ANALYZE 的时间和直觉不一致

- 先停在 `PlanState`，确认当前调用属于 `PostgreSQL Plan 到 PlanState 的状态化边界` 的哪一个生命周期边界。
- 打印 `PlanState`，不要只看字段值，还要解释它的语义：executor 公共运行态，保存 plan 指针、EState 指针、函数指针、表达式状态和 slot。
- 向上追一层调用者，确认是谁要求 executor 进入这个边界。
- 向下追一层被调用者，确认这一步是否注册了资源、改写了状态或返回了 slot。
- 如果现象来自观测指标，判断指标采样是在进入本步之前、执行过程中，还是返回之后。
- 如果现象来自异常路径，判断资源是否已经挂到 owner；没有 owner 的对象必须由当前路径处理。

### 定位 3：cursor、FETCH 或客户端输出让执行分段

- 先停在 `ExecInitSeqScan`，确认当前调用属于 `PostgreSQL Plan 到 PlanState 的状态化边界` 的哪一个生命周期边界。
- 打印 `node-specific State`，不要只看字段值，还要解释它的语义：SeqScanState、AggState、SortState、ModifyTableState 等具体节点私有状态。
- 向上追一层调用者，确认是谁要求 executor 进入这个边界。
- 向下追一层被调用者，确认这一步是否注册了资源、改写了状态或返回了 slot。
- 如果现象来自观测指标，判断指标采样是在进入本步之前、执行过程中，还是返回之后。
- 如果现象来自异常路径，判断资源是否已经挂到 owner；没有 owner 的对象必须由当前路径处理。

### 定位 4：ERROR 后怀疑 executor 资源没有按预期释放

- 先停在 `ExecInitSort`，确认当前调用属于 `PostgreSQL Plan 到 PlanState 的状态化边界` 的哪一个生命周期边界。
- 打印 `ExprState`，不要只看字段值，还要解释它的语义：表达式执行期状态，避免每行重新解释原始 Expr tree。
- 向上追一层调用者，确认是谁要求 executor 进入这个边界。
- 向下追一层被调用者，确认这一步是否注册了资源、改写了状态或返回了 slot。
- 如果现象来自观测指标，判断指标采样是在进入本步之前、执行过程中，还是返回之后。
- 如果现象来自异常路径，判断资源是否已经挂到 owner；没有 owner 的对象必须由当前路径处理。

### 定位 5：扩展 hook 或 profiler 捕获到一个边界事件

- 先停在 `ExecInitAgg`，确认当前调用属于 `PostgreSQL Plan 到 PlanState 的状态化边界` 的哪一个生命周期边界。
- 打印 `lefttree/righttree`，不要只看字段值，还要解释它的语义：运行时子节点链接，对应 Plan 的子树。
- 向上追一层调用者，确认是谁要求 executor 进入这个边界。
- 向下追一层被调用者，确认这一步是否注册了资源、改写了状态或返回了 slot。
- 如果现象来自观测指标，判断指标采样是在进入本步之前、执行过程中，还是返回之后。
- 如果现象来自异常路径，判断资源是否已经挂到 owner；没有 owner 的对象必须由当前路径处理。

### 定位 6：源码阅读时分不清字段属于 Plan 还是 runtime state

- 先停在 `ExecInitModifyTable`，确认当前调用属于 `PostgreSQL Plan 到 PlanState 的状态化边界` 的哪一个生命周期边界。
- 打印 `ps_ResultTupleSlot`，不要只看字段值，还要解释它的语义：节点输出 slot，形状由 targetlist 或子节点决定。
- 向上追一层调用者，确认是谁要求 executor 进入这个边界。
- 向下追一层被调用者，确认这一步是否注册了资源、改写了状态或返回了 slot。
- 如果现象来自观测指标，判断指标采样是在进入本步之前、执行过程中，还是返回之后。
- 如果现象来自异常路径，判断资源是否已经挂到 owner；没有 owner 的对象必须由当前路径处理。

定位结束时只保留三句话：

1. 这个现象落在哪个 executor 生命周期边界。
2. 这个边界改变了哪个核心状态，以及 owner 是谁。
3. 正常路径和 ERROR 路径分别由哪个函数或机制收尾。
