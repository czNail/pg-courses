# PostgreSQL ExecutorFinish 与副作用收尾

## 课程定位

前置知识：已经读过 executor 生命周期的目录规划，并理解 PostgreSQL 的查询执行由 Portal、QueryDesc、EState、PlanState 和 TupleTableSlot 共同完成。上一节看到 ExecutePlan 可以因为 count、EOF 或输出端停止而结束一次拉取。

本节唯一主问题：

```text
ExecutorFinish() 为什么必须独立存在于 ExecutorRun() 和 ExecutorEnd() 之间，专门 drain ModifyTable、AFTER trigger 和其它排队副作用？
```

核心矛盾：顶层 tuple 输出可以被 count 或客户端提前截断 vs SQL 语句的副作用必须在语句边界完整、可预测地收尾。

学完后应能判断：能判断一个动作应该在 run 的 tuple 流中发生，还是必须留到 finish 的语句级收尾阶段。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

上一节看到 ExecutePlan 可以因为 count、EOF 或输出端停止而结束一次拉取。

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

下一节会把最终 teardown 的顺序拆开。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
ExecutorFinish() 在 per-query context 中调用 ExecPostprocessPlan() 把辅助 ModifyTable 节点跑到完成，再调用 AfterTriggerEndQuery() 处理语句级触发器队列，然后标记 es_finished。
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
| 1 | src/backend/executor/execMain.c | ExecutorFinish() / standard_ExecutorFinish() / ExecPostprocessPlan()。 |
| 2 | src/backend/executor/nodeModifyTable.c | ExecModifyTable() 与辅助 ModifyTable 完成边界。 |
| 3 | src/backend/commands/trigger.c | AfterTriggerBeginQuery() / AfterTriggerEndQuery()。 |
| 4 | src/include/nodes/execnodes.h | EState 的 es_finished、es_auxmodifytables、es_insert_pending_result_relations。 |
| 5 | src/backend/executor/execReplication.c | DML 与 logical decoding / replica identity 相邻语义入口。 |
| 6 | src/backend/foreign/foreign.c | FDW routine 参与 DML 副作用的外部边界。 |
| 7 | src/backend/tcop/pquery.c | PortalRunMulti() 如何在语句结束时调用 ExecutorFinish。 |
| 8 | src/backend/commands/explain.c | EXPLAIN ANALYZE 为什么把 finish 成本纳入执行时间。 |

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
| es_finished | ExecutorFinish 完成标记，ExecutorEnd 通过断言检查调用者是否漏掉 finish。 |
| es_auxmodifytables | 辅助 ModifyTableState 列表，finish 阶段必须跑到 EOF。 |
| AFTER trigger queue | 语句级触发器队列，ExecutorStart begin，ExecutorFinish end。 |
| es_insert_pending_result_relations | 批量 insert / foreign table pending side effect 的运行期集合。 |
| query_instr | 整体 executor instrumentation，会把 finish 时间也计入。 |
| es_direction | ExecPostprocessPlan 强制设为 ForwardScanDirection。 |
| per-output-tuple context | finish drain 辅助节点时仍按 tuple 循环重置。 |
| ResultRelInfo | DML 副作用和 trigger/FDW/constraint 的归属对象。 |

本节最容易混淆的边界有四个：

- planner-time 描述与 executor-time 状态不同。
- backend-local 指针与跨 backend 可见状态不同。
- memory context 释放内存，不等于释放所有外部资源。
- 顶层调用边界与节点内部状态机边界不同。

读源码时要持续追问：这个状态现在是谁的，下一步谁会改变它，如果中途 ERROR，谁会兜底。

## 5. 主流程源码 walkthrough

下面按时间顺序读主链路。每一步都回答“状态发生了什么变化”，而不是只记住函数名。

### 5.1. 步骤 1

ExecutorFinish() 同样先经过 ExecutorFinish_hook，扩展可以观察或包裹 finish。

阅读这一段时的重点不是记住第 1 个函数名，而是确认状态边界：读到这里先确认这一步是在创建状态、注册外部资源，还是只是在写入控制字段。

### 5.2. 步骤 2

standard_ExecutorFinish() 断言 QueryDesc 和 EState 已经存在，并且不是 EXPLAIN-only。

阅读这一段时的重点不是记住第 2 个函数名，而是确认状态边界：断点停在这里时，优先打印 owner 指针，再打印本步刚写入的字段。

### 5.3. 步骤 3

它要求 es_finished 仍为 false，说明同一 executor instance 只能 finish 一次。

阅读这一段时的重点不是记住第 3 个函数名，而是确认状态边界：这一步如果提前失败，后续状态还没有 owner，不能按完整 executor cleanup 想象。

### 5.4. 步骤 4

进入 estate->es_query_cxt，保证 finish 中可能产生的 executor 对象归入同一生命周期。

阅读这一段时的重点不是记住第 4 个函数名，而是确认状态边界：这一步如果已经完成，后续代码就可以假定对应 runtime state 已经稳定。

### 5.5. 步骤 5

query_instr 存在时，finish 也被计入 executor overall runtime。

阅读这一段时的重点不是记住第 5 个函数名，而是确认状态边界：这里要区分“指针存在”和“语义已经注册”两件事。

### 5.6. 步骤 6

ExecPostprocessPlan() 把 es_direction 设成 ForwardScanDirection。

阅读这一段时的重点不是记住第 6 个函数名，而是确认状态边界：这里的状态变化通常会影响后续 ExecProcNode、slot、snapshot 或 receiver 的解释。

### 5.7. 步骤 7

它遍历 es_auxmodifytables，对每个 PlanState 反复 ResetPerTupleExprContext 和 ExecProcNode。

阅读这一段时的重点不是记住第 7 个函数名，而是确认状态边界：如果以后诊断慢查询，这一步更像 setup 成本，不应误归到单个 tuple。

### 5.8. 步骤 8

辅助 ModifyTable 返回 NULL slot 后才算 drain 完成。

阅读这一段时的重点不是记住第 8 个函数名，而是确认状态边界：如果以后诊断异常退出，这一步要回看是否已经进入 ResourceOwner、MemoryContext 或 snapshot 注册边界。

### 5.9. 步骤 9

如果没有 EXEC_FLAG_SKIP_TRIGGERS，AfterTriggerEndQuery(estate) 执行排队的 AFTER trigger。

阅读这一段时的重点不是记住第 9 个函数名，而是确认状态边界：这一步的调用者协议比函数内部细节更重要：调用者是否允许重复进入，是否允许部分执行。

### 5.10. 步骤 10

query_instr 停止，切回 oldcontext。

阅读这一段时的重点不是记住第 10 个函数名，而是确认状态边界：这一步之后要看下一个 owner 边界，而不是继续把所有状态当作局部变量。

### 5.11. 步骤 11

最后 estate->es_finished = true，给 ExecutorEnd 的顺序检查提供依据。

阅读这一段时的重点不是记住第 11 个函数名，而是确认状态边界：如果这一步涉及 hook 或 wrapper，要继续追到标准实现或真实节点函数。

### 5.12. 步骤 12

ExecutorEnd 之后不再允许补做 finish，因为 PlanState 与 slot 可能已经被清理。

阅读这一段时的重点不是记住第 12 个函数名，而是确认状态边界：如果这一步涉及 slot 或 tuple，必须同时确认数据内容和底层资源是否同寿命。

主链路可以压缩成：

```text
01. ExecutorFinish() 同样先经过 ExecutorFinish_hook，扩展可以观察或包裹 finish。
02. standard_ExecutorFinish() 断言 QueryDesc 和 EState 已经存在，并且不是 EXPLAIN-only。
03. 它要求 es_finished 仍为 false，说明同一 executor instance 只能 finish 一次。
04. 进入 estate->es_query_cxt，保证 finish 中可能产生的 executor 对象归入同一生命周期。
05. query_instr 存在时，finish 也被计入 executor overall runtime。
06. ExecPostprocessPlan() 把 es_direction 设成 ForwardScanDirection。
07. 它遍历 es_auxmodifytables，对每个 PlanState 反复 ResetPerTupleExprContext 和 ExecProcNode。
08. 辅助 ModifyTable 返回 NULL slot 后才算 drain 完成。
09. 如果没有 EXEC_FLAG_SKIP_TRIGGERS，AfterTriggerEndQuery(estate) 执行排队的 AFTER trigger。
10. query_instr 停止，切回 oldcontext。
11. 最后 estate->es_finished = true，给 ExecutorEnd 的顺序检查提供依据。
12. ExecutorEnd 之后不再允许补做 finish，因为 PlanState 与 slot 可能已经被清理。
```

## 6. 状态随时间推进的故事

把源码主线换成一个对象的生命周期，会更容易看出本节 tension。

### 6.1. 创建前

在“创建前”阶段，关键观察点是：ExecutorFinish() 同样先经过 ExecutorFinish_hook，扩展可以观察或包裹 finish。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

### 6.2. 创建时

在“创建时”阶段，关键观察点是：standard_ExecutorFinish() 断言 QueryDesc 和 EState 已经存在，并且不是 EXPLAIN-only。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

### 6.3. 第一次使用前

在“第一次使用前”阶段，关键观察点是：它要求 es_finished 仍为 false，说明同一 executor instance 只能 finish 一次。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

### 6.4. 正常推进中

在“正常推进中”阶段，关键观察点是：进入 estate->es_query_cxt，保证 finish 中可能产生的 executor 对象归入同一生命周期。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

### 6.5. 遇到边界时

在“遇到边界时”阶段，关键观察点是：query_instr 存在时，finish 也被计入 executor overall runtime。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

### 6.6. 提前停止时

在“提前停止时”阶段，关键观察点是：ExecPostprocessPlan() 把 es_direction 设成 ForwardScanDirection。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

### 6.7. 收尾时

在“收尾时”阶段，关键观察点是：它遍历 es_auxmodifytables，对每个 PlanState 反复 ResetPerTupleExprContext 和 ExecProcNode。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

### 6.8. ERROR 后

在“ERROR 后”阶段，关键观察点是：辅助 ModifyTable 返回 NULL slot 后才算 drain 完成。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

## 7. 生命周期 / ownership / cleanup

- 副作用队列在 ExecutorStart 阶段 begin，在 ExecutorRun 期间累积，在 ExecutorFinish 阶段 drain。
- 辅助 ModifyTableState 与主 planstate 同属于 EState，由 ExecutorEnd 统一释放。
- AfterTriggerEndQuery 执行的是语句级边界，不等价于事务结束触发器清理。
- 如果 ERROR 发生在 finish 中，事务 abort 路径负责回滚已经执行但未提交的变更。
- es_finished 是调用顺序标记，不是资源释放标记。

用 ownership 语言重写本节：

| 问题 | 判断方式 |
| --- | --- |
| 谁创建 | 看入口函数和 memory context 切换点。 |
| 谁持有 | 看 QueryDesc、EState、PlanState 或 slot 列表中的指针。 |
| 谁释放 | 看 ExecutorFinish、ExecutorEnd、ExecEndNode、ExecResetTupleTable 或 ResourceOwner。 |
| ERROR 怎么办 | 看资源是否有 ResourceOwner、MemoryContext、snapshot register 或 callback 兜底。 |
| 能否跨边界保存 | 看对象是否 backend-local、是否依赖 slot 内容、buffer pin 或 per-tuple context。 |

## 8. 正确性机制层次

- 语句副作用不能取决于客户端是否把所有 RETURNING tuple 都取走。
- 辅助 ModifyTable 必须跑完，避免 modifying CTE 只因顶层消费不足而产生不可预测结果。
- AFTER trigger 必须在 End 清理 PlanState 前执行，因为它还需要 EState、ResultRelInfo 和 tuple 信息。
- Finish 被计入 EXPLAIN ANALYZE，因为用户关心完整语句执行成本。
- EXEC_FLAG_SKIP_TRIGGERS 只在安全场景下设置，比如无 modifying CTE 的 SELECT。
- finish 一次性标记 es_finished，避免二次触发副作用。

这些机制互相补位。snapshot 解决可见性，不解决内存生命周期；MemoryContext 解决内存批量释放，不解决 buffer pin；函数指针解决分派成本，不解决节点协议正确性。

| 层次 | 保证 | 不能误解为 |
| --- | --- | --- |
| snapshot / command id | tuple 可见性和写入标记边界 | 并发互斥或资源释放 |
| PlanState / slot 协议 | 节点间传递和推进方式一致 | 所有节点都非阻塞 |
| MemoryContext | backend-local 内存归属清楚 | 外部资源自动释放 |
| ResourceOwner / pin / refcount | 外部资源不会被过早回收 | 对象语义仍然新鲜 |
| hook / instrumentation | 观测或扩展入口 | 可以任意改变核心语义 |

## 9. 错误路径 / 异常路径 / fallback

- EXPLAIN-only 不应调用 ExecutorFinish，因为它不执行真实副作用。
- 没有副作用时 Finish 仍然存在，主要完成顺序协议和 instrumentation 边界。
- AFTER trigger 中 ERROR 会中断 finish，事务机制负责整体回滚。
- skip triggers 是启动期根据计划特征决定的，不是 finish 阶段随意跳过。
- 辅助 ModifyTable 的 drain 只说明执行器层完成，不代表事务已经提交。

PostgreSQL executor 的异常路径不能按 C++ 析构模型理解。`ereport(ERROR)` 可能 longjmp 到上层，普通 C 栈上的清理代码不会自然逐层执行。

因此本节所有状态都要问一句：如果这个点 ERROR，已经注册的资源由谁释放，尚未注册的资源是否会泄漏或变成悬挂引用。

## 10. 成本、资源与跨模块传播

- 如果 modifying CTE 或触发器很多，用户看到的耗时可能集中在 ExecutorFinish。
- 触发器函数、外键检查、FDW batch flush 都可能让 finish 成为慢 SQL 尾部成本。
- EXPLAIN ANALYZE 会把 finish 纳入总时间，避免只统计 tuple 拉取造成低估。
- 每个辅助 ModifyTable 仍走 ExecProcNode 协议，成本随待 drain 行数扩张。
- per-tuple context reset 仍然保护 finish 中的表达式内存。

executor 的成本常常不是单个函数慢，而是边界次数被 workload 放大：每行、每节点、每次 FETCH、每个 trigger、每个 slot clear，都可能成为可见成本。

| 放大因素 | 典型表现 |
| --- | --- |
| tuple 数 | ExecProcNode、slot deform、per-tuple reset 和 receiver 调用被放大。 |
| plan node 数 | ExecInitNode、ExecEndNode、instrumentation 对象和 EXPLAIN 输出被放大。 |
| relation 数 | range table relation 打开、ResultRelInfo、锁和权限检查被放大。 |
| FETCH 次数 | ExecutorRun 边界、DestReceiver startup/shutdown 和 Portal 状态切换被放大。 |
| 副作用数量 | trigger、constraint、FDW flush、ModifyTable drain 被放大。 |

## 11. 观测与诊断入口

- gdb 断在 standard_ExecutorFinish 可以确认 es_finished 从 false 到 true。
- 断在 ExecPostprocessPlan 可以观察 es_auxmodifytables 是否为空。
- 带 AFTER trigger 的 UPDATE 可在 AfterTriggerEndQuery 看到触发器执行。
- EXPLAIN ANALYZE 中触发器时间会让实际总耗时超过节点 tuple 时间之和。
- auto_explain 的 executor finish hook 可以观测慢查询尾部。

建议的断点顺序：

1. `ExecutorFinish`
2. `standard_ExecutorFinish`
3. `ExecPostprocessPlan`
4. `AfterTriggerEndQuery`
5. `ExecModifyTable`
6. `es_auxmodifytables`
7. `es_finished`
8. `query_instr`

断点不只是为了停住程序。每个断点都应该带一个问题：这个状态是否已经初始化，owner 是谁，下一步是否会被改写。

## 12. 课堂实验

### 实验 1

创建 AFTER UPDATE trigger，在 UPDATE ... RETURNING 中只 FETCH 部分行，观察 finish 仍执行 trigger。

观察点：
- 断点前后同一个字段是否发生变化。
- 这个变化是否发生在本节主流程上。
- 如果把 SQL 改成 cursor、EXPLAIN、DML 或带 trigger，边界是否改变。

### 实验 2

写一个 modifying CTE，顶层 SELECT LIMIT 1，观察辅助 ModifyTable 是否被 drain。

观察点：
- 断点前后同一个字段是否发生变化。
- 这个变化是否发生在本节主流程上。
- 如果把 SQL 改成 cursor、EXPLAIN、DML 或带 trigger，边界是否改变。

### 实验 3

对 standard_ExecutorFinish 和 AfterTriggerEndQuery 打断点，比较无 trigger SELECT 与 DML。

观察点：
- 断点前后同一个字段是否发生变化。
- 这个变化是否发生在本节主流程上。
- 如果把 SQL 改成 cursor、EXPLAIN、DML 或带 trigger，边界是否改变。

### 实验 4

用 EXPLAIN ANALYZE 执行带慢触发器的语句，观察总时间与节点时间差。

观察点：
- 断点前后同一个字段是否发生变化。
- 这个变化是否发生在本节主流程上。
- 如果把 SQL 改成 cursor、EXPLAIN、DML 或带 trigger，边界是否改变。

### 实验 5

设置 session_replication_role 或相关触发器条件，观察 skip trigger 场景的边界。

观察点：
- 断点前后同一个字段是否发生变化。
- 这个变化是否发生在本节主流程上。
- 如果把 SQL 改成 cursor、EXPLAIN、DML 或带 trigger，边界是否改变。

### 实验 6

在 ExecPostprocessPlan 中打印 estate->es_direction，确认 finish 强制 forward。

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

### 练习 1：`ExecutorFinish`

在 `/home/highgo/postgres` 中定位 `ExecutorFinish`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

### 练习 2：`standard_ExecutorFinish`

在 `/home/highgo/postgres` 中定位 `standard_ExecutorFinish`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

### 练习 3：`ExecPostprocessPlan`

在 `/home/highgo/postgres` 中定位 `ExecPostprocessPlan`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

### 练习 4：`AfterTriggerEndQuery`

在 `/home/highgo/postgres` 中定位 `AfterTriggerEndQuery`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

### 练习 5：`ExecModifyTable`

在 `/home/highgo/postgres` 中定位 `ExecModifyTable`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

### 练习 6：`es_auxmodifytables`

在 `/home/highgo/postgres` 中定位 `es_auxmodifytables`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

### 练习 7：`es_finished`

在 `/home/highgo/postgres` 中定位 `es_finished`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

### 练习 8：`query_instr`

在 `/home/highgo/postgres` 中定位 `query_instr`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

## 14. 常见误区

- 不要把 ExecutorRun 结束理解成语句副作用已经全部完成。
- 不要把 ExecutorFinish 当成释放资源的函数；释放在 ExecutorEnd。
- 不要认为 LIMIT 可以截断所有 DML 副作用，modifying CTE 有独立收尾规则。
- 不要把 trigger 时间误归因到单个 scan 节点。
- 不要在 hook 中跳过 previous ExecutorFinish hook，否则可能破坏扩展链和副作用边界。

这些误区的共同点是把 runtime 状态当成静态结构，或者把静态计划当成可以直接推进的状态机。

## 15. 讨论题

1. 为什么 finish 必须在 End 之前，而不能作为 ExecEndPlan 的第一步？
2. 如果触发器执行被计入节点 instrumentation，会带来哪些解释误差？
3. 哪些副作用可以随 tuple 流发生，哪些必须等语句边界？
4. 扩展在 ExecutorFinish_hook 中应该如何控制开销和异常安全？

讨论时要尽量回到一个具体 runtime 现象，比如一次 FETCH、一次 EXPLAIN ANALYZE、一个 trigger 或一个 slot materialize，而不是停留在抽象偏好。

## 16. 本节小结

本节唯一主问题是：ExecutorFinish() 为什么必须独立存在于 ExecutorRun() 和 ExecutorEnd() 之间，专门 drain ModifyTable、AFTER trigger 和其它排队副作用？

可以带走的一句话模型是：ExecutorFinish() 在 per-query context 中调用 ExecPostprocessPlan() 把辅助 ModifyTable 节点跑到完成，再调用 AfterTriggerEndQuery() 处理语句级触发器队列，然后标记 es_finished。

- 源码阅读应先找状态创建点，再找推进点，最后找 cleanup。
- 任何字段都要和 owner、生命周期、异常路径一起理解。
- 观测数据只有能回到具体 runtime 边界时，才适合用于诊断。

下一节会把最终 teardown 的顺序拆开。

## 17. 从现象回源码的诊断案例

下面的案例都只服务本节主问题。它们的共同读法是先看现象，再找 runtime 边界，最后回到 owner 和 cleanup。

### 案例 1：第一次执行停在入口但状态还没有建立

现象：

- 你在 `ExecutorRun` 附近停住，看到 SQL 还没有返回，但某些 executor 状态已经改变。
- 这时不要急着沿调用栈往下追完整执行，而要先回到本节主问题：ExecutorFinish() 为什么必须独立存在于 ExecutorRun() 和 ExecutorEnd() 之间，专门 drain ModifyTable、AFTER trigger 和其它排队副作用？
- 如果现象来自 EXPLAIN、cursor、trigger 或扩展 hook，要先确认它落在哪个 executor 生命周期边界。

源码回看：

- 入口函数：`ExecutorRun`。
- 本步主线：ExecutorFinish() 同样先经过 ExecutorFinish_hook，扩展可以观察或包裹 finish。
- 关键状态：`es_finished`，语义是：ExecutorFinish 完成标记，ExecutorEnd 通过断言检查调用者是否漏掉 finish。
- 相邻状态：`es_auxmodifytables`，用来判断这一步之后谁继续持有资源。

判断方式：

- 如果字段刚被写入但还没有注册到 owner，异常路径通常只能靠当前函数局部清理。
- 如果状态已经挂到 EState、PlanState、slot table 或 snapshot manager，后续 cleanup 应该能从 owner 找回。
- 如果这个状态只影响观测，不影响语义，诊断时不要把它解释成执行计划行为改变。
- 如果这个状态影响 tuple 生命周期，必须同时检查 slot、TupleDesc、buffer pin 或 per-tuple context。

### 案例 2：查询只返回少量行但内部状态已经推进

现象：

- 你在 `ExecutorRun` 附近停住，看到 SQL 还没有返回，但某些 executor 状态已经改变。
- 这时不要急着沿调用栈往下追完整执行，而要先回到本节主问题：ExecutorFinish() 为什么必须独立存在于 ExecutorRun() 和 ExecutorEnd() 之间，专门 drain ModifyTable、AFTER trigger 和其它排队副作用？
- 如果现象来自 EXPLAIN、cursor、trigger 或扩展 hook，要先确认它落在哪个 executor 生命周期边界。

源码回看：

- 入口函数：`ExecutorRun`。
- 本步主线：standard_ExecutorFinish() 断言 QueryDesc 和 EState 已经存在，并且不是 EXPLAIN-only。
- 关键状态：`es_auxmodifytables`，语义是：辅助 ModifyTableState 列表，finish 阶段必须跑到 EOF。
- 相邻状态：`AFTER trigger queue`，用来判断这一步之后谁继续持有资源。

判断方式：

- 如果字段刚被写入但还没有注册到 owner，异常路径通常只能靠当前函数局部清理。
- 如果状态已经挂到 EState、PlanState、slot table 或 snapshot manager，后续 cleanup 应该能从 owner 找回。
- 如果这个状态只影响观测，不影响语义，诊断时不要把它解释成执行计划行为改变。
- 如果这个状态影响 tuple 生命周期，必须同时检查 slot、TupleDesc、buffer pin 或 per-tuple context。

### 案例 3：EXPLAIN ANALYZE 时间与节点时间不一致

现象：

- 你在 `ExecutorRun` 附近停住，看到 SQL 还没有返回，但某些 executor 状态已经改变。
- 这时不要急着沿调用栈往下追完整执行，而要先回到本节主问题：ExecutorFinish() 为什么必须独立存在于 ExecutorRun() 和 ExecutorEnd() 之间，专门 drain ModifyTable、AFTER trigger 和其它排队副作用？
- 如果现象来自 EXPLAIN、cursor、trigger 或扩展 hook，要先确认它落在哪个 executor 生命周期边界。

源码回看：

- 入口函数：`ExecutorRun`。
- 本步主线：它要求 es_finished 仍为 false，说明同一 executor instance 只能 finish 一次。
- 关键状态：`AFTER trigger queue`，语义是：语句级触发器队列，ExecutorStart begin，ExecutorFinish end。
- 相邻状态：`es_insert_pending_result_relations`，用来判断这一步之后谁继续持有资源。

判断方式：

- 如果字段刚被写入但还没有注册到 owner，异常路径通常只能靠当前函数局部清理。
- 如果状态已经挂到 EState、PlanState、slot table 或 snapshot manager，后续 cleanup 应该能从 owner 找回。
- 如果这个状态只影响观测，不影响语义，诊断时不要把它解释成执行计划行为改变。
- 如果这个状态影响 tuple 生命周期，必须同时检查 slot、TupleDesc、buffer pin 或 per-tuple context。

### 案例 4：cursor 或 receiver 让执行被分成多段

现象：

- 你在 `ExecutorRun` 附近停住，看到 SQL 还没有返回，但某些 executor 状态已经改变。
- 这时不要急着沿调用栈往下追完整执行，而要先回到本节主问题：ExecutorFinish() 为什么必须独立存在于 ExecutorRun() 和 ExecutorEnd() 之间，专门 drain ModifyTable、AFTER trigger 和其它排队副作用？
- 如果现象来自 EXPLAIN、cursor、trigger 或扩展 hook，要先确认它落在哪个 executor 生命周期边界。

源码回看：

- 入口函数：`ExecutorRun`。
- 本步主线：进入 estate->es_query_cxt，保证 finish 中可能产生的 executor 对象归入同一生命周期。
- 关键状态：`es_insert_pending_result_relations`，语义是：批量 insert / foreign table pending side effect 的运行期集合。
- 相邻状态：`query_instr`，用来判断这一步之后谁继续持有资源。

判断方式：

- 如果字段刚被写入但还没有注册到 owner，异常路径通常只能靠当前函数局部清理。
- 如果状态已经挂到 EState、PlanState、slot table 或 snapshot manager，后续 cleanup 应该能从 owner 找回。
- 如果这个状态只影响观测，不影响语义，诊断时不要把它解释成执行计划行为改变。
- 如果这个状态影响 tuple 生命周期，必须同时检查 slot、TupleDesc、buffer pin 或 per-tuple context。

### 案例 5：ERROR 发生后没有按普通调用栈返回

现象：

- 你在 `ExecutorRun` 附近停住，看到 SQL 还没有返回，但某些 executor 状态已经改变。
- 这时不要急着沿调用栈往下追完整执行，而要先回到本节主问题：ExecutorFinish() 为什么必须独立存在于 ExecutorRun() 和 ExecutorEnd() 之间，专门 drain ModifyTable、AFTER trigger 和其它排队副作用？
- 如果现象来自 EXPLAIN、cursor、trigger 或扩展 hook，要先确认它落在哪个 executor 生命周期边界。

源码回看：

- 入口函数：`ExecutorRun`。
- 本步主线：query_instr 存在时，finish 也被计入 executor overall runtime。
- 关键状态：`query_instr`，语义是：整体 executor instrumentation，会把 finish 时间也计入。
- 相邻状态：`es_direction`，用来判断这一步之后谁继续持有资源。

判断方式：

- 如果字段刚被写入但还没有注册到 owner，异常路径通常只能靠当前函数局部清理。
- 如果状态已经挂到 EState、PlanState、slot table 或 snapshot manager，后续 cleanup 应该能从 owner 找回。
- 如果这个状态只影响观测，不影响语义，诊断时不要把它解释成执行计划行为改变。
- 如果这个状态影响 tuple 生命周期，必须同时检查 slot、TupleDesc、buffer pin 或 per-tuple context。

### 案例 6：扩展 hook 看到的边界与核心 executor 边界不同

现象：

- 你在 `ExecutorRun` 附近停住，看到 SQL 还没有返回，但某些 executor 状态已经改变。
- 这时不要急着沿调用栈往下追完整执行，而要先回到本节主问题：ExecutorFinish() 为什么必须独立存在于 ExecutorRun() 和 ExecutorEnd() 之间，专门 drain ModifyTable、AFTER trigger 和其它排队副作用？
- 如果现象来自 EXPLAIN、cursor、trigger 或扩展 hook，要先确认它落在哪个 executor 生命周期边界。

源码回看：

- 入口函数：`ExecutorRun`。
- 本步主线：ExecPostprocessPlan() 把 es_direction 设成 ForwardScanDirection。
- 关键状态：`es_direction`，语义是：ExecPostprocessPlan 强制设为 ForwardScanDirection。
- 相邻状态：`per-output-tuple context`，用来判断这一步之后谁继续持有资源。

判断方式：

- 如果字段刚被写入但还没有注册到 owner，异常路径通常只能靠当前函数局部清理。
- 如果状态已经挂到 EState、PlanState、slot table 或 snapshot manager，后续 cleanup 应该能从 owner 找回。
- 如果这个状态只影响观测，不影响语义，诊断时不要把它解释成执行计划行为改变。
- 如果这个状态影响 tuple 生命周期，必须同时检查 slot、TupleDesc、buffer pin 或 per-tuple context。

## 18. 源码断点与字段记录表

下面这张表用于课堂现场跟读。每一行都应该实际在 gdb 中停一次，记录进入前和返回后的字段差异。

| 序号 | 断点 | 进入前看什么 | 返回后看什么 | 常见误判 |
| --- | --- | --- | --- | --- |
| 1 | `ExecutorRun` | `es_finished` 是否已经有效 | ExecutorFinish() 同样先经过 ExecutorFinish_hook，扩展可以观察或包裹 finish。 | 把字段存在误认为语义已经完成 |
| 2 | `ExecutorRun` | `es_auxmodifytables` 是否已经有效 | standard_ExecutorFinish() 断言 QueryDesc 和 EState 已经存在，并且不是 EXPLAIN-only。 | 把字段存在误认为语义已经完成 |
| 3 | `ExecutorRun` | `AFTER trigger queue` 是否已经有效 | 它要求 es_finished 仍为 false，说明同一 executor instance 只能 finish 一次。 | 把字段存在误认为语义已经完成 |
| 4 | `ExecutorRun` | `es_insert_pending_result_relations` 是否已经有效 | 进入 estate->es_query_cxt，保证 finish 中可能产生的 executor 对象归入同一生命周期。 | 把字段存在误认为语义已经完成 |
| 5 | `ExecutorRun` | `query_instr` 是否已经有效 | query_instr 存在时，finish 也被计入 executor overall runtime。 | 把字段存在误认为语义已经完成 |
| 6 | `ExecutorRun` | `es_direction` 是否已经有效 | ExecPostprocessPlan() 把 es_direction 设成 ForwardScanDirection。 | 把字段存在误认为语义已经完成 |
| 7 | `ExecutorRun` | `per-output-tuple context` 是否已经有效 | 它遍历 es_auxmodifytables，对每个 PlanState 反复 ResetPerTupleExprContext 和 ExecProcNode。 | 把字段存在误认为语义已经完成 |
| 8 | `ExecutorRun` | `ResultRelInfo` 是否已经有效 | 辅助 ModifyTable 返回 NULL slot 后才算 drain 完成。 | 把字段存在误认为语义已经完成 |
| 9 | `ExecutorRun` | `es_finished` 是否已经有效 | 如果没有 EXEC_FLAG_SKIP_TRIGGERS，AfterTriggerEndQuery(estate) 执行排队的 AFTER trigger。 | 把字段存在误认为语义已经完成 |
| 10 | `ExecutorRun` | `es_auxmodifytables` 是否已经有效 | query_instr 停止，切回 oldcontext。 | 把字段存在误认为语义已经完成 |
| 11 | `ExecutorRun` | `AFTER trigger queue` 是否已经有效 | 最后 estate->es_finished = true，给 ExecutorEnd 的顺序检查提供依据。 | 把字段存在误认为语义已经完成 |
| 12 | `ExecutorRun` | `es_insert_pending_result_relations` 是否已经有效 | ExecutorEnd 之后不再允许补做 finish，因为 PlanState 与 slot 可能已经被清理。 | 把字段存在误认为语义已经完成 |

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
- 只观察一个状态变化：ExecutorFinish() 同样先经过 ExecutorFinish_hook，扩展可以观察或包裹 finish。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 2：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：standard_ExecutorFinish() 断言 QueryDesc 和 EState 已经存在，并且不是 EXPLAIN-only。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 3：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：它要求 es_finished 仍为 false，说明同一 executor instance 只能 finish 一次。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 4：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：进入 estate->es_query_cxt，保证 finish 中可能产生的 executor 对象归入同一生命周期。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 5：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：query_instr 存在时，finish 也被计入 executor overall runtime。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 6：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：ExecPostprocessPlan() 把 es_direction 设成 ForwardScanDirection。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 7：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：它遍历 es_auxmodifytables，对每个 PlanState 反复 ResetPerTupleExprContext 和 ExecProcNode。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 8：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：辅助 ModifyTable 返回 NULL slot 后才算 drain 完成。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 9：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：如果没有 EXEC_FLAG_SKIP_TRIGGERS，AfterTriggerEndQuery(estate) 执行排队的 AFTER trigger。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 10：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：query_instr 停止，切回 oldcontext。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 11：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：最后 estate->es_finished = true，给 ExecutorEnd 的顺序检查提供依据。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 12：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：ExecutorEnd 之后不再允许补做 finish，因为 PlanState 与 slot 可能已经被清理。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 13：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：ExecutorFinish() 同样先经过 ExecutorFinish_hook，扩展可以观察或包裹 finish。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 14：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：standard_ExecutorFinish() 断言 QueryDesc 和 EState 已经存在，并且不是 EXPLAIN-only。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 15：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：它要求 es_finished 仍为 false，说明同一 executor instance 只能 finish 一次。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 16：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：进入 estate->es_query_cxt，保证 finish 中可能产生的 executor 对象归入同一生命周期。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 17：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：query_instr 存在时，finish 也被计入 executor overall runtime。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 18：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：ExecPostprocessPlan() 把 es_direction 设成 ForwardScanDirection。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 19：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：它遍历 es_auxmodifytables，对每个 PlanState 反复 ResetPerTupleExprContext 和 ExecProcNode。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 20：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：辅助 ModifyTable 返回 NULL slot 后才算 drain 完成。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 21：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：如果没有 EXEC_FLAG_SKIP_TRIGGERS，AfterTriggerEndQuery(estate) 执行排队的 AFTER trigger。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 22：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：query_instr 停止，切回 oldcontext。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 23：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：最后 estate->es_finished = true，给 ExecutorEnd 的顺序检查提供依据。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 24：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：ExecutorEnd 之后不再允许补做 finish，因为 PlanState 与 slot 可能已经被清理。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 25：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：ExecutorFinish() 同样先经过 ExecutorFinish_hook，扩展可以观察或包裹 finish。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 26：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：standard_ExecutorFinish() 断言 QueryDesc 和 EState 已经存在，并且不是 EXPLAIN-only。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 27：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：它要求 es_finished 仍为 false，说明同一 executor instance 只能 finish 一次。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。
## 20. 一页纸定位模板

这部分用于把课程内容压缩成排查动作。不要从这里开始背函数名；先带着现象进入，再沿状态边界回源码。

### 定位 1：执行还没返回，但 gdb 已经停在 executor 内部

- 先停在 `ExecutorFinish`，确认当前调用属于 `PostgreSQL ExecutorFinish 与副作用收尾` 的哪一个生命周期边界。
- 打印 `es_finished`，不要只看字段值，还要解释它的语义：ExecutorFinish 完成标记，ExecutorEnd 通过断言检查调用者是否漏掉 finish。
- 向上追一层调用者，确认是谁要求 executor 进入这个边界。
- 向下追一层被调用者，确认这一步是否注册了资源、改写了状态或返回了 slot。
- 如果现象来自观测指标，判断指标采样是在进入本步之前、执行过程中，还是返回之后。
- 如果现象来自异常路径，判断资源是否已经挂到 owner；没有 owner 的对象必须由当前路径处理。

### 定位 2：EXPLAIN ANALYZE 的时间和直觉不一致

- 先停在 `standard_ExecutorFinish`，确认当前调用属于 `PostgreSQL ExecutorFinish 与副作用收尾` 的哪一个生命周期边界。
- 打印 `es_auxmodifytables`，不要只看字段值，还要解释它的语义：辅助 ModifyTableState 列表，finish 阶段必须跑到 EOF。
- 向上追一层调用者，确认是谁要求 executor 进入这个边界。
- 向下追一层被调用者，确认这一步是否注册了资源、改写了状态或返回了 slot。
- 如果现象来自观测指标，判断指标采样是在进入本步之前、执行过程中，还是返回之后。
- 如果现象来自异常路径，判断资源是否已经挂到 owner；没有 owner 的对象必须由当前路径处理。

### 定位 3：cursor、FETCH 或客户端输出让执行分段

- 先停在 `ExecPostprocessPlan`，确认当前调用属于 `PostgreSQL ExecutorFinish 与副作用收尾` 的哪一个生命周期边界。
- 打印 `AFTER trigger queue`，不要只看字段值，还要解释它的语义：语句级触发器队列，ExecutorStart begin，ExecutorFinish end。
- 向上追一层调用者，确认是谁要求 executor 进入这个边界。
- 向下追一层被调用者，确认这一步是否注册了资源、改写了状态或返回了 slot。
- 如果现象来自观测指标，判断指标采样是在进入本步之前、执行过程中，还是返回之后。
- 如果现象来自异常路径，判断资源是否已经挂到 owner；没有 owner 的对象必须由当前路径处理。

### 定位 4：ERROR 后怀疑 executor 资源没有按预期释放

- 先停在 `AfterTriggerEndQuery`，确认当前调用属于 `PostgreSQL ExecutorFinish 与副作用收尾` 的哪一个生命周期边界。
- 打印 `es_insert_pending_result_relations`，不要只看字段值，还要解释它的语义：批量 insert / foreign table pending side effect 的运行期集合。
- 向上追一层调用者，确认是谁要求 executor 进入这个边界。
- 向下追一层被调用者，确认这一步是否注册了资源、改写了状态或返回了 slot。
- 如果现象来自观测指标，判断指标采样是在进入本步之前、执行过程中，还是返回之后。
- 如果现象来自异常路径，判断资源是否已经挂到 owner；没有 owner 的对象必须由当前路径处理。

### 定位 5：扩展 hook 或 profiler 捕获到一个边界事件

- 先停在 `ExecModifyTable`，确认当前调用属于 `PostgreSQL ExecutorFinish 与副作用收尾` 的哪一个生命周期边界。
- 打印 `query_instr`，不要只看字段值，还要解释它的语义：整体 executor instrumentation，会把 finish 时间也计入。
- 向上追一层调用者，确认是谁要求 executor 进入这个边界。
- 向下追一层被调用者，确认这一步是否注册了资源、改写了状态或返回了 slot。
- 如果现象来自观测指标，判断指标采样是在进入本步之前、执行过程中，还是返回之后。
- 如果现象来自异常路径，判断资源是否已经挂到 owner；没有 owner 的对象必须由当前路径处理。

### 定位 6：源码阅读时分不清字段属于 Plan 还是 runtime state

- 先停在 `es_auxmodifytables`，确认当前调用属于 `PostgreSQL ExecutorFinish 与副作用收尾` 的哪一个生命周期边界。
- 打印 `es_direction`，不要只看字段值，还要解释它的语义：ExecPostprocessPlan 强制设为 ForwardScanDirection。
- 向上追一层调用者，确认是谁要求 executor 进入这个边界。
- 向下追一层被调用者，确认这一步是否注册了资源、改写了状态或返回了 slot。
- 如果现象来自观测指标，判断指标采样是在进入本步之前、执行过程中，还是返回之后。
- 如果现象来自异常路径，判断资源是否已经挂到 owner；没有 owner 的对象必须由当前路径处理。

定位结束时只保留三句话：

1. 这个现象落在哪个 executor 生命周期边界。
2. 这个边界改变了哪个核心状态，以及 owner 是谁。
3. 正常路径和 ERROR 路径分别由哪个函数或机制收尾。
