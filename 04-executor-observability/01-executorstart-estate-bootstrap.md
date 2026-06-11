# PostgreSQL ExecutorStart 如何把 Plan 变成 EState

## 课程定位

前置知识：已经读过 executor 生命周期的目录规划，并理解 PostgreSQL 的查询执行由 Portal、QueryDesc、EState、PlanState 和 TupleTableSlot 共同完成。本节是 04 目录的入口：从 planner 产物进入 executor runtime。

本节唯一主问题：

```text
ExecutorStart() 为什么必须先创建 EState、注册 snapshot、初始化 range table / ResultRelInfo / trigger 状态，再递归构造 PlanState tree，而不是让 ExecutorRun() 边跑边补齐这些状态？
```

核心矛盾：启动成本和懒初始化的诱惑 vs 执行期需要一个已经稳定、可清理、可观测的 runtime 根状态。

学完后应能判断：能独立判断一个 executor 状态应该在 start 阶段创建、run 阶段推进，还是 end 阶段释放。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

本节是 04 目录的入口：从 planner 产物进入 executor runtime。

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

下一节会继续看已经启动好的 executor 如何被 Portal 分批驱动。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
ExecutorStart() 把 QueryDesc 中的 PlannedStmt 固化成 EState 和 PlanState tree；之后 ExecutorRun() 只按这个 runtime tree 拉取 tuple，不再重新解释 planner tree 的全局语义。
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
| 1 | src/backend/executor/execMain.c | ExecutorStart() / standard_ExecutorStart() / InitPlan()：启动主线和 QueryDesc 填充。 |
| 2 | src/backend/executor/execUtils.c | CreateExecutorState() / FreeExecutorState()：EState 与 executor memory context。 |
| 3 | src/backend/executor/execProcnode.c | ExecInitNode()：Plan node 到 PlanState node 的递归状态化。 |
| 4 | src/include/nodes/execnodes.h | EState / PlanState 字段边界。 |
| 5 | src/backend/executor/execTuples.c | ExecInitExtraTupleSlot() / ExecInitResultTupleSlotTL()：slot 放入 EState tuple table。 |
| 6 | src/backend/tcop/pquery.c | CreateQueryDesc() / PortalStart 路径：executor 外层调用者。 |
| 7 | src/backend/commands/trigger.c | AfterTriggerBeginQuery()：语句级副作用队列的启动边界。 |
| 8 | src/include/executor/executor.h | executor hook 与公开入口声明。 |

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
| QueryDesc | executor 与调用者之间的句柄，start 前只带 plannedstmt、snapshot、dest 等外部输入，start 后补上 estate、planstate、tupDesc。 |
| EState | 一次 executor invocation 的 backend-local 根状态，挂在 es_query_cxt 下。 |
| es_query_cxt | executor 工作内存生命周期边界，ExecutorEnd() 通过 FreeExecutorState() 删除。 |
| es_snapshot | 由 RegisterSnapshot() 注册，保证执行期间 snapshot 引用有效。 |
| es_output_cid | DML、SELECT FOR UPDATE 和 modifying CTE 标记 tuple 时使用的 command id。 |
| es_range_table | runtime 访问 range table 的入口，后续打开 Relation、row mark、ResultRelInfo 都依赖它。 |
| es_tupleTable | 所有 executor slot 的集中清理入口，EndPlan 时释放 pin 和 TupleDesc 引用。 |
| planstate | 由 ExecInitNode() 递归构造的运行时树，不再是 planner tree 的只读描述。 |

本节最容易混淆的边界有四个：

- planner-time 描述与 executor-time 状态不同。
- backend-local 指针与跨 backend 可见状态不同。
- memory context 释放内存，不等于释放所有外部资源。
- 顶层调用边界与节点内部状态机边界不同。

读源码时要持续追问：这个状态现在是谁的，下一步谁会改变它，如果中途 ERROR，谁会兜底。

## 5. 主流程源码 walkthrough

下面按时间顺序读主链路。每一步都回答“状态发生了什么变化”，而不是只记住函数名。

### 5.1. 步骤 1

ExecutorStart() 先上报 query id，再通过 ExecutorStart_hook 给扩展一个包裹启动的机会。

阅读这一段时的重点不是记住第 1 个函数名，而是确认状态边界：读到这里先确认这一步是在创建状态、注册外部资源，还是只是在写入控制字段。

### 5.2. 步骤 2

standard_ExecutorStart() 断言 QueryDesc 还没有 estate，并要求 active snapshot 正是 queryDesc->snapshot。

阅读这一段时的重点不是记住第 2 个函数名，而是确认状态边界：断点停在这里时，优先打印 owner 指针，再打印本步刚写入的字段。

### 5.3. 步骤 3

读写检查在启动期完成：read-only 事务和 parallel mode 不能执行不安全写路径。

阅读这一段时的重点不是记住第 3 个函数名，而是确认状态边界：这一步如果提前失败，后续状态还没有 owner，不能按完整 executor cleanup 想象。

### 5.4. 步骤 4

CreateExecutorState() 创建 ExecutorState memory context，并把 EState 节点放在这个 context 内。

阅读这一段时的重点不是记住第 4 个函数名，而是确认状态边界：这一步如果已经完成，后续代码就可以假定对应 runtime state 已经稳定。

### 5.5. 步骤 5

启动过程切入 estate->es_query_cxt，后续 ParamExecData、row mark、slot、PlanState 都归入 executor 生命周期。

阅读这一段时的重点不是记住第 5 个函数名，而是确认状态边界：这里要区分“指针存在”和“语义已经注册”两件事。

### 5.6. 步骤 6

外部参数、sourceText、queryEnv、operation、output command id 被复制到 EState。

阅读这一段时的重点不是记住第 6 个函数名，而是确认状态边界：这里的状态变化通常会影响后续 ExecProcNode、slot、snapshot 或 receiver 的解释。

### 5.7. 步骤 7

query snapshot 和 crosscheck snapshot 通过 RegisterSnapshot() 注册，End 阶段必须 UnregisterSnapshot()。

阅读这一段时的重点不是记住第 7 个函数名，而是确认状态边界：如果以后诊断慢查询，这一步更像 setup 成本，不应误归到单个 tuple。

### 5.8. 步骤 8

instrumentation、JIT flag、AFTER trigger statement context 在启动阶段完成开关判定。

阅读这一段时的重点不是记住第 8 个函数名，而是确认状态边界：如果以后诊断异常退出，这一步要回看是否已经进入 ResourceOwner、MemoryContext 或 snapshot 注册边界。

### 5.9. 步骤 9

InitPlan() 做权限检查、range table 初始化、初始 partition pruning 和 row mark 状态创建。

阅读这一段时的重点不是记住第 9 个函数名，而是确认状态边界：这一步的调用者协议比函数内部细节更重要：调用者是否允许重复进入，是否允许部分执行。

### 5.10. 步骤 10

SubPlan state 必须先于主计划初始化，因为表达式里的 SubPlanState 需要能被父节点找到。

阅读这一段时的重点不是记住第 10 个函数名，而是确认状态边界：这一步之后要看下一个 owner 边界，而不是继续把所有状态当作局部变量。

### 5.11. 步骤 11

ExecInitNode() 递归把 root Plan 变成 root PlanState，并在各节点 init 函数中创建表达式状态、slot 和子节点。

阅读这一段时的重点不是记住第 11 个函数名，而是确认状态边界：如果这一步涉及 hook 或 wrapper，要继续追到标准实现或真实节点函数。

### 5.12. 步骤 12

InitPlan() 根据顶层 targetlist 决定 tupDesc 和 junk filter，最后写回 QueryDesc。

阅读这一段时的重点不是记住第 12 个函数名，而是确认状态边界：如果这一步涉及 slot 或 tuple，必须同时确认数据内容和底层资源是否同寿命。

主链路可以压缩成：

```text
01. ExecutorStart() 先上报 query id，再通过 ExecutorStart_hook 给扩展一个包裹启动的机会。
02. standard_ExecutorStart() 断言 QueryDesc 还没有 estate，并要求 active snapshot 正是 queryDesc->snapshot。
03. 读写检查在启动期完成：read-only 事务和 parallel mode 不能执行不安全写路径。
04. CreateExecutorState() 创建 ExecutorState memory context，并把 EState 节点放在这个 context 内。
05. 启动过程切入 estate->es_query_cxt，后续 ParamExecData、row mark、slot、PlanState 都归入 executor 生命周期。
06. 外部参数、sourceText、queryEnv、operation、output command id 被复制到 EState。
07. query snapshot 和 crosscheck snapshot 通过 RegisterSnapshot() 注册，End 阶段必须 UnregisterSnapshot()。
08. instrumentation、JIT flag、AFTER trigger statement context 在启动阶段完成开关判定。
09. InitPlan() 做权限检查、range table 初始化、初始 partition pruning 和 row mark 状态创建。
10. SubPlan state 必须先于主计划初始化，因为表达式里的 SubPlanState 需要能被父节点找到。
11. ExecInitNode() 递归把 root Plan 变成 root PlanState，并在各节点 init 函数中创建表达式状态、slot 和子节点。
12. InitPlan() 根据顶层 targetlist 决定 tupDesc 和 junk filter，最后写回 QueryDesc。
```

## 6. 状态随时间推进的故事

把源码主线换成一个对象的生命周期，会更容易看出本节 tension。

### 6.1. 创建前

在“创建前”阶段，关键观察点是：ExecutorStart() 先上报 query id，再通过 ExecutorStart_hook 给扩展一个包裹启动的机会。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

### 6.2. 创建时

在“创建时”阶段，关键观察点是：standard_ExecutorStart() 断言 QueryDesc 还没有 estate，并要求 active snapshot 正是 queryDesc->snapshot。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

### 6.3. 第一次使用前

在“第一次使用前”阶段，关键观察点是：读写检查在启动期完成：read-only 事务和 parallel mode 不能执行不安全写路径。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

### 6.4. 正常推进中

在“正常推进中”阶段，关键观察点是：CreateExecutorState() 创建 ExecutorState memory context，并把 EState 节点放在这个 context 内。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

### 6.5. 遇到边界时

在“遇到边界时”阶段，关键观察点是：启动过程切入 estate->es_query_cxt，后续 ParamExecData、row mark、slot、PlanState 都归入 executor 生命周期。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

### 6.6. 提前停止时

在“提前停止时”阶段，关键观察点是：外部参数、sourceText、queryEnv、operation、output command id 被复制到 EState。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

### 6.7. 收尾时

在“收尾时”阶段，关键观察点是：query snapshot 和 crosscheck snapshot 通过 RegisterSnapshot() 注册，End 阶段必须 UnregisterSnapshot()。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

### 6.8. ERROR 后

在“ERROR 后”阶段，关键观察点是：instrumentation、JIT flag、AFTER trigger statement context 在启动阶段完成开关判定。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

## 7. 生命周期 / ownership / cleanup

- 创建者是 ExecutorStart()，实际分配由 CreateExecutorState()、InitPlan() 和各 ExecInit*() 完成。
- 持有者是 QueryDesc；外层 Portal、SPI、EXPLAIN 或 utility code 只通过 QueryDesc 驱动 executor。
- 释放者是 ExecutorEnd()；它先 ExecEndPlan()，再注销 snapshot，最后 FreeExecutorState() 删除 memory context。
- ERROR 路径通常由 Portal / transaction cleanup 兜底，但正常调用者仍必须遵守 Start、Run、Finish、End 顺序。
- EState 指针不能跨 backend，也不能在 ExecutorEnd() 后继续缓存。

用 ownership 语言重写本节：

| 问题 | 判断方式 |
| --- | --- |
| 谁创建 | 看入口函数和 memory context 切换点。 |
| 谁持有 | 看 QueryDesc、EState、PlanState 或 slot 列表中的指针。 |
| 谁释放 | 看 ExecutorFinish、ExecutorEnd、ExecEndNode、ExecResetTupleTable 或 ResourceOwner。 |
| ERROR 怎么办 | 看资源是否有 ResourceOwner、MemoryContext、snapshot register 或 callback 兜底。 |
| 能否跨边界保存 | 看对象是否 backend-local、是否依赖 slot 内容、buffer pin 或 per-tuple context。 |

## 8. 正确性机制层次

- active snapshot 断言把执行可见性绑定到调用者建立的 snapshot 栈。
- RegisterSnapshot() 避免执行期间 snapshot 被过早释放。
- row mark 初始化把 FOR UPDATE/SHARE 的锁语义提前挂到 runtime 状态。
- read-only 与 parallel mode 检查在执行前失败，避免部分执行后才发现语义非法。
- trigger begin 与 output cid 保证 DML 副作用在同一个 command 边界内被解释。
- PlanState tree 构造完成后，运行期函数只处理 tuple 流，不重算全局权限和状态布局。

这些机制互相补位。snapshot 解决可见性，不解决内存生命周期；MemoryContext 解决内存批量释放，不解决 buffer pin；函数指针解决分派成本，不解决节点协议正确性。

| 层次 | 保证 | 不能误解为 |
| --- | --- | --- |
| snapshot / command id | tuple 可见性和写入标记边界 | 并发互斥或资源释放 |
| PlanState / slot 协议 | 节点间传递和推进方式一致 | 所有节点都非阻塞 |
| MemoryContext | backend-local 内存归属清楚 | 外部资源自动释放 |
| ResourceOwner / pin / refcount | 外部资源不会被过早回收 | 对象语义仍然新鲜 |
| hook / instrumentation | 观测或扩展入口 | 可以任意改变核心语义 |

## 9. 错误路径 / 异常路径 / fallback

- queryDesc->estate 已存在说明重复启动，同一 QueryDesc 不能 start 两次。
- snapshot 不在 active stack 上会触发断言，这是调用者协议错误。
- EXPLAIN-only 通过 EXEC_FLAG_EXPLAIN_ONLY 影响触发器和 End 阶段断言。
- read-only 或 parallel mode 下的写计划在启动期报错。
- ExecInitNode() 遇到未知 nodeTag 会 elog(ERROR)，因为 runtime 无法安全解释未知计划节点。

PostgreSQL executor 的异常路径不能按 C++ 析构模型理解。`ereport(ERROR)` 可能 longjmp 到上层，普通 C 栈上的清理代码不会自然逐层执行。

因此本节所有状态都要问一句：如果这个点 ERROR，已经注册的资源由谁释放，尚未注册的资源是否会泄漏或变成悬挂引用。

## 10. 成本、资源与跨模块传播

- 启动成本与 range table 大小、subplan 数、计划节点数、ResultRelInfo 数和 partition pruning 状态有关。
- 把状态提前建好会增加 first tuple latency，但降低每个 tuple 的重复判断成本。
- snapshot register、Relation 打开、slot 分配和表达式初始化都属于 per-query 成本。
- hook 和 instrumentation 只在开关打开时增加对象分配与 wrapper 成本。
- JIT flag 只是记录，真正 JIT context 常常延迟到表达式需要时创建。

executor 的成本常常不是单个函数慢，而是边界次数被 workload 放大：每行、每节点、每次 FETCH、每个 trigger、每个 slot clear，都可能成为可见成本。

| 放大因素 | 典型表现 |
| --- | --- |
| tuple 数 | ExecProcNode、slot deform、per-tuple reset 和 receiver 调用被放大。 |
| plan node 数 | ExecInitNode、ExecEndNode、instrumentation 对象和 EXPLAIN 输出被放大。 |
| relation 数 | range table relation 打开、ResultRelInfo、锁和权限检查被放大。 |
| FETCH 次数 | ExecutorRun 边界、DestReceiver startup/shutdown 和 Portal 状态切换被放大。 |
| 副作用数量 | trigger、constraint、FDW flush、ModifyTable drain 被放大。 |

## 11. 观测与诊断入口

- EXPLAIN 不执行时仍会调用 ExecutorStart() 构造可展示的 PlanState 边界。
- gdb 断在 standard_ExecutorStart() 可以看 queryDesc->estate 从 NULL 变成 EState。
- 断在 InitPlan() 末尾可以比较 queryDesc->plannedstmt->planTree 与 queryDesc->planstate。
- pg_stat query id 在 ExecutorStart() 开头被报告，扩展 hook 也从这里进入。
- 内存可通过 pg_backend_memory_contexts 观察 ExecutorState context 的存在和释放。

建议的断点顺序：

1. `ExecutorStart`
2. `standard_ExecutorStart`
3. `CreateExecutorState`
4. `InitPlan`
5. `ExecInitNode`
6. `RegisterSnapshot`
7. `AfterTriggerBeginQuery`
8. `ExecInitExtraTupleSlot`

断点不只是为了停住程序。每个断点都应该带一个问题：这个状态是否已经初始化，owner 是谁，下一步是否会被改写。

## 12. 课堂实验

### 实验 1

在 gdb 对 standard_ExecutorStart 设置断点，执行 SELECT 1，观察 queryDesc->estate 初始为 NULL。

观察点：
- 断点前后同一个字段是否发生变化。
- 这个变化是否发生在本节主流程上。
- 如果把 SQL 改成 cursor、EXPLAIN、DML 或带 trigger，边界是否改变。

### 实验 2

单步到 CreateExecutorState() 后打印 estate->es_query_cxt->name。

观察点：
- 断点前后同一个字段是否发生变化。
- 这个变化是否发生在本节主流程上。
- 如果把 SQL 改成 cursor、EXPLAIN、DML 或带 trigger，边界是否改变。

### 实验 3

单步到 InitPlan() 末尾打印 queryDesc->planstate->type。

观察点：
- 断点前后同一个字段是否发生变化。
- 这个变化是否发生在本节主流程上。
- 如果把 SQL 改成 cursor、EXPLAIN、DML 或带 trigger，边界是否改变。

### 实验 4

执行 SELECT * FROM t WHERE id = 1 FOR UPDATE，观察 plannedstmt->rowMarks 与 estate->es_rowmarks。

观察点：
- 断点前后同一个字段是否发生变化。
- 这个变化是否发生在本节主流程上。
- 如果把 SQL 改成 cursor、EXPLAIN、DML 或带 trigger，边界是否改变。

### 实验 5

执行 EXPLAIN SELECT * FROM t，比较 EXEC_FLAG_EXPLAIN_ONLY 对 ExecutorFinish 调用的影响。

观察点：
- 断点前后同一个字段是否发生变化。
- 这个变化是否发生在本节主流程上。
- 如果把 SQL 改成 cursor、EXPLAIN、DML 或带 trigger，边界是否改变。

### 实验 6

查询 pg_backend_memory_contexts，确认 ExecutorState context 在 cursor 保持期间存在。

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

### 练习 1：`ExecutorStart`

在 `/home/highgo/postgres` 中定位 `ExecutorStart`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

### 练习 2：`standard_ExecutorStart`

在 `/home/highgo/postgres` 中定位 `standard_ExecutorStart`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

### 练习 3：`CreateExecutorState`

在 `/home/highgo/postgres` 中定位 `CreateExecutorState`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

### 练习 4：`InitPlan`

在 `/home/highgo/postgres` 中定位 `InitPlan`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

### 练习 5：`ExecInitNode`

在 `/home/highgo/postgres` 中定位 `ExecInitNode`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

### 练习 6：`RegisterSnapshot`

在 `/home/highgo/postgres` 中定位 `RegisterSnapshot`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

### 练习 7：`AfterTriggerBeginQuery`

在 `/home/highgo/postgres` 中定位 `AfterTriggerBeginQuery`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

### 练习 8：`ExecInitExtraTupleSlot`

在 `/home/highgo/postgres` 中定位 `ExecInitExtraTupleSlot`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

## 14. 常见误区

- 不要把 PlannedStmt 当成可直接执行的对象；它缺少执行期 ownership 和函数指针。
- 不要把 EState 理解成全局 executor 单例；它只属于一次 QueryDesc。
- 不要把 snapshot 字段当作普通指针；注册与注销才构成完整生命周期。
- 不要以为 InitPlan 只是递归调用 ExecInitNode；权限、row mark、junk filter 都在这里定边界。
- 不要把 ExecutorStart hook 当成替代 standard_ExecutorStart 的理由；多数扩展必须调用 previous hook 或标准实现。

这些误区的共同点是把 runtime 状态当成静态结构，或者把静态计划当成可以直接推进的状态机。

## 15. 讨论题

1. 如果把 row mark 延迟到第一次访问对应 relation 时初始化，会让哪些错误变成部分执行后的错误？
2. 为什么 EState 应该有一个统一 es_query_cxt，而不是让每个节点自己挂在 PortalContext 下？
3. ExecutorStart hook 适合观测什么，不适合改写什么？
4. 哪些初始化属于 correctness 边界，哪些只是性能预热？

讨论时要尽量回到一个具体 runtime 现象，比如一次 FETCH、一次 EXPLAIN ANALYZE、一个 trigger 或一个 slot materialize，而不是停留在抽象偏好。

## 16. 本节小结

本节唯一主问题是：ExecutorStart() 为什么必须先创建 EState、注册 snapshot、初始化 range table / ResultRelInfo / trigger 状态，再递归构造 PlanState tree，而不是让 ExecutorRun() 边跑边补齐这些状态？

可以带走的一句话模型是：ExecutorStart() 把 QueryDesc 中的 PlannedStmt 固化成 EState 和 PlanState tree；之后 ExecutorRun() 只按这个 runtime tree 拉取 tuple，不再重新解释 planner tree 的全局语义。

- 源码阅读应先找状态创建点，再找推进点，最后找 cleanup。
- 任何字段都要和 owner、生命周期、异常路径一起理解。
- 观测数据只有能回到具体 runtime 边界时，才适合用于诊断。

下一节会继续看已经启动好的 executor 如何被 Portal 分批驱动。

## 17. 从现象回源码的诊断案例

下面的案例都只服务本节主问题。它们的共同读法是先看现象，再找 runtime 边界，最后回到 owner 和 cleanup。

### 案例 1：第一次执行停在入口但状态还没有建立

现象：

- 你在 `ExecutorRun` 附近停住，看到 SQL 还没有返回，但某些 executor 状态已经改变。
- 这时不要急着沿调用栈往下追完整执行，而要先回到本节主问题：ExecutorStart() 为什么必须先创建 EState、注册 snapshot、初始化 range table / ResultRelInfo / trigger 状态，再递归构造 PlanState tree，而不是让 ExecutorRun() 边跑边补齐这些状态？
- 如果现象来自 EXPLAIN、cursor、trigger 或扩展 hook，要先确认它落在哪个 executor 生命周期边界。

源码回看：

- 入口函数：`ExecutorRun`。
- 本步主线：ExecutorStart() 先上报 query id，再通过 ExecutorStart_hook 给扩展一个包裹启动的机会。
- 关键状态：`QueryDesc`，语义是：executor 与调用者之间的句柄，start 前只带 plannedstmt、snapshot、dest 等外部输入，start 后补上 estate、planstate、tupDesc。
- 相邻状态：`EState`，用来判断这一步之后谁继续持有资源。

判断方式：

- 如果字段刚被写入但还没有注册到 owner，异常路径通常只能靠当前函数局部清理。
- 如果状态已经挂到 EState、PlanState、slot table 或 snapshot manager，后续 cleanup 应该能从 owner 找回。
- 如果这个状态只影响观测，不影响语义，诊断时不要把它解释成执行计划行为改变。
- 如果这个状态影响 tuple 生命周期，必须同时检查 slot、TupleDesc、buffer pin 或 per-tuple context。

### 案例 2：查询只返回少量行但内部状态已经推进

现象：

- 你在 `ExecutorRun` 附近停住，看到 SQL 还没有返回，但某些 executor 状态已经改变。
- 这时不要急着沿调用栈往下追完整执行，而要先回到本节主问题：ExecutorStart() 为什么必须先创建 EState、注册 snapshot、初始化 range table / ResultRelInfo / trigger 状态，再递归构造 PlanState tree，而不是让 ExecutorRun() 边跑边补齐这些状态？
- 如果现象来自 EXPLAIN、cursor、trigger 或扩展 hook，要先确认它落在哪个 executor 生命周期边界。

源码回看：

- 入口函数：`ExecutorRun`。
- 本步主线：standard_ExecutorStart() 断言 QueryDesc 还没有 estate，并要求 active snapshot 正是 queryDesc->snapshot。
- 关键状态：`EState`，语义是：一次 executor invocation 的 backend-local 根状态，挂在 es_query_cxt 下。
- 相邻状态：`es_query_cxt`，用来判断这一步之后谁继续持有资源。

判断方式：

- 如果字段刚被写入但还没有注册到 owner，异常路径通常只能靠当前函数局部清理。
- 如果状态已经挂到 EState、PlanState、slot table 或 snapshot manager，后续 cleanup 应该能从 owner 找回。
- 如果这个状态只影响观测，不影响语义，诊断时不要把它解释成执行计划行为改变。
- 如果这个状态影响 tuple 生命周期，必须同时检查 slot、TupleDesc、buffer pin 或 per-tuple context。

### 案例 3：EXPLAIN ANALYZE 时间与节点时间不一致

现象：

- 你在 `ExecutorRun` 附近停住，看到 SQL 还没有返回，但某些 executor 状态已经改变。
- 这时不要急着沿调用栈往下追完整执行，而要先回到本节主问题：ExecutorStart() 为什么必须先创建 EState、注册 snapshot、初始化 range table / ResultRelInfo / trigger 状态，再递归构造 PlanState tree，而不是让 ExecutorRun() 边跑边补齐这些状态？
- 如果现象来自 EXPLAIN、cursor、trigger 或扩展 hook，要先确认它落在哪个 executor 生命周期边界。

源码回看：

- 入口函数：`ExecutorRun`。
- 本步主线：读写检查在启动期完成：read-only 事务和 parallel mode 不能执行不安全写路径。
- 关键状态：`es_query_cxt`，语义是：executor 工作内存生命周期边界，ExecutorEnd() 通过 FreeExecutorState() 删除。
- 相邻状态：`es_snapshot`，用来判断这一步之后谁继续持有资源。

判断方式：

- 如果字段刚被写入但还没有注册到 owner，异常路径通常只能靠当前函数局部清理。
- 如果状态已经挂到 EState、PlanState、slot table 或 snapshot manager，后续 cleanup 应该能从 owner 找回。
- 如果这个状态只影响观测，不影响语义，诊断时不要把它解释成执行计划行为改变。
- 如果这个状态影响 tuple 生命周期，必须同时检查 slot、TupleDesc、buffer pin 或 per-tuple context。

### 案例 4：cursor 或 receiver 让执行被分成多段

现象：

- 你在 `ExecutorRun` 附近停住，看到 SQL 还没有返回，但某些 executor 状态已经改变。
- 这时不要急着沿调用栈往下追完整执行，而要先回到本节主问题：ExecutorStart() 为什么必须先创建 EState、注册 snapshot、初始化 range table / ResultRelInfo / trigger 状态，再递归构造 PlanState tree，而不是让 ExecutorRun() 边跑边补齐这些状态？
- 如果现象来自 EXPLAIN、cursor、trigger 或扩展 hook，要先确认它落在哪个 executor 生命周期边界。

源码回看：

- 入口函数：`ExecutorRun`。
- 本步主线：CreateExecutorState() 创建 ExecutorState memory context，并把 EState 节点放在这个 context 内。
- 关键状态：`es_snapshot`，语义是：由 RegisterSnapshot() 注册，保证执行期间 snapshot 引用有效。
- 相邻状态：`es_output_cid`，用来判断这一步之后谁继续持有资源。

判断方式：

- 如果字段刚被写入但还没有注册到 owner，异常路径通常只能靠当前函数局部清理。
- 如果状态已经挂到 EState、PlanState、slot table 或 snapshot manager，后续 cleanup 应该能从 owner 找回。
- 如果这个状态只影响观测，不影响语义，诊断时不要把它解释成执行计划行为改变。
- 如果这个状态影响 tuple 生命周期，必须同时检查 slot、TupleDesc、buffer pin 或 per-tuple context。

### 案例 5：ERROR 发生后没有按普通调用栈返回

现象：

- 你在 `ExecutorRun` 附近停住，看到 SQL 还没有返回，但某些 executor 状态已经改变。
- 这时不要急着沿调用栈往下追完整执行，而要先回到本节主问题：ExecutorStart() 为什么必须先创建 EState、注册 snapshot、初始化 range table / ResultRelInfo / trigger 状态，再递归构造 PlanState tree，而不是让 ExecutorRun() 边跑边补齐这些状态？
- 如果现象来自 EXPLAIN、cursor、trigger 或扩展 hook，要先确认它落在哪个 executor 生命周期边界。

源码回看：

- 入口函数：`ExecutorRun`。
- 本步主线：启动过程切入 estate->es_query_cxt，后续 ParamExecData、row mark、slot、PlanState 都归入 executor 生命周期。
- 关键状态：`es_output_cid`，语义是：DML、SELECT FOR UPDATE 和 modifying CTE 标记 tuple 时使用的 command id。
- 相邻状态：`es_range_table`，用来判断这一步之后谁继续持有资源。

判断方式：

- 如果字段刚被写入但还没有注册到 owner，异常路径通常只能靠当前函数局部清理。
- 如果状态已经挂到 EState、PlanState、slot table 或 snapshot manager，后续 cleanup 应该能从 owner 找回。
- 如果这个状态只影响观测，不影响语义，诊断时不要把它解释成执行计划行为改变。
- 如果这个状态影响 tuple 生命周期，必须同时检查 slot、TupleDesc、buffer pin 或 per-tuple context。

### 案例 6：扩展 hook 看到的边界与核心 executor 边界不同

现象：

- 你在 `ExecutorRun` 附近停住，看到 SQL 还没有返回，但某些 executor 状态已经改变。
- 这时不要急着沿调用栈往下追完整执行，而要先回到本节主问题：ExecutorStart() 为什么必须先创建 EState、注册 snapshot、初始化 range table / ResultRelInfo / trigger 状态，再递归构造 PlanState tree，而不是让 ExecutorRun() 边跑边补齐这些状态？
- 如果现象来自 EXPLAIN、cursor、trigger 或扩展 hook，要先确认它落在哪个 executor 生命周期边界。

源码回看：

- 入口函数：`ExecutorRun`。
- 本步主线：外部参数、sourceText、queryEnv、operation、output command id 被复制到 EState。
- 关键状态：`es_range_table`，语义是：runtime 访问 range table 的入口，后续打开 Relation、row mark、ResultRelInfo 都依赖它。
- 相邻状态：`es_tupleTable`，用来判断这一步之后谁继续持有资源。

判断方式：

- 如果字段刚被写入但还没有注册到 owner，异常路径通常只能靠当前函数局部清理。
- 如果状态已经挂到 EState、PlanState、slot table 或 snapshot manager，后续 cleanup 应该能从 owner 找回。
- 如果这个状态只影响观测，不影响语义，诊断时不要把它解释成执行计划行为改变。
- 如果这个状态影响 tuple 生命周期，必须同时检查 slot、TupleDesc、buffer pin 或 per-tuple context。

## 18. 源码断点与字段记录表

下面这张表用于课堂现场跟读。每一行都应该实际在 gdb 中停一次，记录进入前和返回后的字段差异。

| 序号 | 断点 | 进入前看什么 | 返回后看什么 | 常见误判 |
| --- | --- | --- | --- | --- |
| 1 | `ExecutorRun` | `QueryDesc` 是否已经有效 | ExecutorStart() 先上报 query id，再通过 ExecutorStart_hook 给扩展一个包裹启动的机会。 | 把字段存在误认为语义已经完成 |
| 2 | `ExecutorRun` | `EState` 是否已经有效 | standard_ExecutorStart() 断言 QueryDesc 还没有 estate，并要求 active snapshot 正是 queryDesc->snapshot。 | 把字段存在误认为语义已经完成 |
| 3 | `ExecutorRun` | `es_query_cxt` 是否已经有效 | 读写检查在启动期完成：read-only 事务和 parallel mode 不能执行不安全写路径。 | 把字段存在误认为语义已经完成 |
| 4 | `ExecutorRun` | `es_snapshot` 是否已经有效 | CreateExecutorState() 创建 ExecutorState memory context，并把 EState 节点放在这个 context 内。 | 把字段存在误认为语义已经完成 |
| 5 | `ExecutorRun` | `es_output_cid` 是否已经有效 | 启动过程切入 estate->es_query_cxt，后续 ParamExecData、row mark、slot、PlanState 都归入 executor 生命周期。 | 把字段存在误认为语义已经完成 |
| 6 | `ExecutorRun` | `es_range_table` 是否已经有效 | 外部参数、sourceText、queryEnv、operation、output command id 被复制到 EState。 | 把字段存在误认为语义已经完成 |
| 7 | `ExecutorRun` | `es_tupleTable` 是否已经有效 | query snapshot 和 crosscheck snapshot 通过 RegisterSnapshot() 注册，End 阶段必须 UnregisterSnapshot()。 | 把字段存在误认为语义已经完成 |
| 8 | `ExecutorRun` | `planstate` 是否已经有效 | instrumentation、JIT flag、AFTER trigger statement context 在启动阶段完成开关判定。 | 把字段存在误认为语义已经完成 |
| 9 | `ExecutorRun` | `QueryDesc` 是否已经有效 | InitPlan() 做权限检查、range table 初始化、初始 partition pruning 和 row mark 状态创建。 | 把字段存在误认为语义已经完成 |
| 10 | `ExecutorRun` | `EState` 是否已经有效 | SubPlan state 必须先于主计划初始化，因为表达式里的 SubPlanState 需要能被父节点找到。 | 把字段存在误认为语义已经完成 |
| 11 | `ExecutorRun` | `es_query_cxt` 是否已经有效 | ExecInitNode() 递归把 root Plan 变成 root PlanState，并在各节点 init 函数中创建表达式状态、slot 和子节点。 | 把字段存在误认为语义已经完成 |
| 12 | `ExecutorRun` | `es_snapshot` 是否已经有效 | InitPlan() 根据顶层 targetlist 决定 tupDesc 和 junk filter，最后写回 QueryDesc。 | 把字段存在误认为语义已经完成 |

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
- 只观察一个状态变化：ExecutorStart() 先上报 query id，再通过 ExecutorStart_hook 给扩展一个包裹启动的机会。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 2：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：standard_ExecutorStart() 断言 QueryDesc 还没有 estate，并要求 active snapshot 正是 queryDesc->snapshot。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 3：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：读写检查在启动期完成：read-only 事务和 parallel mode 不能执行不安全写路径。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 4：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：CreateExecutorState() 创建 ExecutorState memory context，并把 EState 节点放在这个 context 内。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 5：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：启动过程切入 estate->es_query_cxt，后续 ParamExecData、row mark、slot、PlanState 都归入 executor 生命周期。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 6：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：外部参数、sourceText、queryEnv、operation、output command id 被复制到 EState。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 7：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：query snapshot 和 crosscheck snapshot 通过 RegisterSnapshot() 注册，End 阶段必须 UnregisterSnapshot()。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 8：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：instrumentation、JIT flag、AFTER trigger statement context 在启动阶段完成开关判定。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 9：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：InitPlan() 做权限检查、range table 初始化、初始 partition pruning 和 row mark 状态创建。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 10：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：SubPlan state 必须先于主计划初始化，因为表达式里的 SubPlanState 需要能被父节点找到。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 11：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：ExecInitNode() 递归把 root Plan 变成 root PlanState，并在各节点 init 函数中创建表达式状态、slot 和子节点。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 12：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：InitPlan() 根据顶层 targetlist 决定 tupDesc 和 junk filter，最后写回 QueryDesc。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 13：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：ExecutorStart() 先上报 query id，再通过 ExecutorStart_hook 给扩展一个包裹启动的机会。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 14：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：standard_ExecutorStart() 断言 QueryDesc 还没有 estate，并要求 active snapshot 正是 queryDesc->snapshot。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 15：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：读写检查在启动期完成：read-only 事务和 parallel mode 不能执行不安全写路径。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 16：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：CreateExecutorState() 创建 ExecutorState memory context，并把 EState 节点放在这个 context 内。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 17：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：启动过程切入 estate->es_query_cxt，后续 ParamExecData、row mark、slot、PlanState 都归入 executor 生命周期。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 18：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：外部参数、sourceText、queryEnv、operation、output command id 被复制到 EState。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 19：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：query snapshot 和 crosscheck snapshot 通过 RegisterSnapshot() 注册，End 阶段必须 UnregisterSnapshot()。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 20：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：instrumentation、JIT flag、AFTER trigger statement context 在启动阶段完成开关判定。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 21：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：InitPlan() 做权限检查、range table 初始化、初始 partition pruning 和 row mark 状态创建。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 22：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：SubPlan state 必须先于主计划初始化，因为表达式里的 SubPlanState 需要能被父节点找到。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 23：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：ExecInitNode() 递归把 root Plan 变成 root PlanState，并在各节点 init 函数中创建表达式状态、slot 和子节点。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 24：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：InitPlan() 根据顶层 targetlist 决定 tupDesc 和 junk filter，最后写回 QueryDesc。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 25：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：ExecutorStart() 先上报 query id，再通过 ExecutorStart_hook 给扩展一个包裹启动的机会。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 26：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：standard_ExecutorStart() 断言 QueryDesc 还没有 estate，并要求 active snapshot 正是 queryDesc->snapshot。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 27：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：读写检查在启动期完成：read-only 事务和 parallel mode 不能执行不安全写路径。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。
## 20. 一页纸定位模板

这部分用于把课程内容压缩成排查动作。不要从这里开始背函数名；先带着现象进入，再沿状态边界回源码。

### 定位 1：执行还没返回，但 gdb 已经停在 executor 内部

- 先停在 `ExecutorStart`，确认当前调用属于 `PostgreSQL ExecutorStart 如何把 Plan 变成 EState` 的哪一个生命周期边界。
- 打印 `QueryDesc`，不要只看字段值，还要解释它的语义：executor 与调用者之间的句柄，start 前只带 plannedstmt、snapshot、dest 等外部输入，start 后补上 estate、planstate、tupDesc。
- 向上追一层调用者，确认是谁要求 executor 进入这个边界。
- 向下追一层被调用者，确认这一步是否注册了资源、改写了状态或返回了 slot。
- 如果现象来自观测指标，判断指标采样是在进入本步之前、执行过程中，还是返回之后。
- 如果现象来自异常路径，判断资源是否已经挂到 owner；没有 owner 的对象必须由当前路径处理。

### 定位 2：EXPLAIN ANALYZE 的时间和直觉不一致

- 先停在 `standard_ExecutorStart`，确认当前调用属于 `PostgreSQL ExecutorStart 如何把 Plan 变成 EState` 的哪一个生命周期边界。
- 打印 `EState`，不要只看字段值，还要解释它的语义：一次 executor invocation 的 backend-local 根状态，挂在 es_query_cxt 下。
- 向上追一层调用者，确认是谁要求 executor 进入这个边界。
- 向下追一层被调用者，确认这一步是否注册了资源、改写了状态或返回了 slot。
- 如果现象来自观测指标，判断指标采样是在进入本步之前、执行过程中，还是返回之后。
- 如果现象来自异常路径，判断资源是否已经挂到 owner；没有 owner 的对象必须由当前路径处理。

### 定位 3：cursor、FETCH 或客户端输出让执行分段

- 先停在 `CreateExecutorState`，确认当前调用属于 `PostgreSQL ExecutorStart 如何把 Plan 变成 EState` 的哪一个生命周期边界。
- 打印 `es_query_cxt`，不要只看字段值，还要解释它的语义：executor 工作内存生命周期边界，ExecutorEnd() 通过 FreeExecutorState() 删除。
- 向上追一层调用者，确认是谁要求 executor 进入这个边界。
- 向下追一层被调用者，确认这一步是否注册了资源、改写了状态或返回了 slot。
- 如果现象来自观测指标，判断指标采样是在进入本步之前、执行过程中，还是返回之后。
- 如果现象来自异常路径，判断资源是否已经挂到 owner；没有 owner 的对象必须由当前路径处理。

### 定位 4：ERROR 后怀疑 executor 资源没有按预期释放

- 先停在 `InitPlan`，确认当前调用属于 `PostgreSQL ExecutorStart 如何把 Plan 变成 EState` 的哪一个生命周期边界。
- 打印 `es_snapshot`，不要只看字段值，还要解释它的语义：由 RegisterSnapshot() 注册，保证执行期间 snapshot 引用有效。
- 向上追一层调用者，确认是谁要求 executor 进入这个边界。
- 向下追一层被调用者，确认这一步是否注册了资源、改写了状态或返回了 slot。
- 如果现象来自观测指标，判断指标采样是在进入本步之前、执行过程中，还是返回之后。
- 如果现象来自异常路径，判断资源是否已经挂到 owner；没有 owner 的对象必须由当前路径处理。

### 定位 5：扩展 hook 或 profiler 捕获到一个边界事件

- 先停在 `ExecInitNode`，确认当前调用属于 `PostgreSQL ExecutorStart 如何把 Plan 变成 EState` 的哪一个生命周期边界。
- 打印 `es_output_cid`，不要只看字段值，还要解释它的语义：DML、SELECT FOR UPDATE 和 modifying CTE 标记 tuple 时使用的 command id。
- 向上追一层调用者，确认是谁要求 executor 进入这个边界。
- 向下追一层被调用者，确认这一步是否注册了资源、改写了状态或返回了 slot。
- 如果现象来自观测指标，判断指标采样是在进入本步之前、执行过程中，还是返回之后。
- 如果现象来自异常路径，判断资源是否已经挂到 owner；没有 owner 的对象必须由当前路径处理。

### 定位 6：源码阅读时分不清字段属于 Plan 还是 runtime state

- 先停在 `RegisterSnapshot`，确认当前调用属于 `PostgreSQL ExecutorStart 如何把 Plan 变成 EState` 的哪一个生命周期边界。
- 打印 `es_range_table`，不要只看字段值，还要解释它的语义：runtime 访问 range table 的入口，后续打开 Relation、row mark、ResultRelInfo 都依赖它。
- 向上追一层调用者，确认是谁要求 executor 进入这个边界。
- 向下追一层被调用者，确认这一步是否注册了资源、改写了状态或返回了 slot。
- 如果现象来自观测指标，判断指标采样是在进入本步之前、执行过程中，还是返回之后。
- 如果现象来自异常路径，判断资源是否已经挂到 owner；没有 owner 的对象必须由当前路径处理。

定位结束时只保留三句话：

1. 这个现象落在哪个 executor 生命周期边界。
2. 这个边界改变了哪个核心状态，以及 owner 是谁。
3. 正常路径和 ERROR 路径分别由哪个函数或机制收尾。
