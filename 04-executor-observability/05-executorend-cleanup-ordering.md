# PostgreSQL ExecutorEnd 与 ERROR-safe teardown 顺序

## 课程定位

前置知识：已经读过 executor 生命周期的目录规划，并理解 PostgreSQL 的查询执行由 Portal、QueryDesc、EState、PlanState 和 TupleTableSlot 共同完成。上一节说明了 ExecutorFinish 为什么先 drain 语句级副作用。

本节唯一主问题：

```text
ExecutorEnd() 为什么必须按节点结束、tuple table 清理、Relation 关闭、snapshot 注销、memory context 删除的顺序收尾，而不是直接删除 executor memory context？
```

核心矛盾：memory context 能批量释放内存 vs buffer pin、TupleDesc refcount、Relation、JIT context、trigger callback 等非内存资源必须按协议显式释放。

学完后应能判断：能判断 executor 中一个资源是靠 MemoryContext 释放，还是必须由 ExecEndPlan、ExecResetTupleTable、ResourceOwner 或 callback 显式释放。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

上一节说明了 ExecutorFinish 为什么先 drain 语句级副作用。

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

下一节进入 Plan 到 PlanState 的更细边界。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
ExecutorEnd() 在 es_query_cxt 中调用 ExecEndPlan()，让节点先释放外部资源，再清 tuple table 和 Relation，随后注销 snapshot，最后 FreeExecutorState() 删除 executor context。
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
| 1 | src/backend/executor/execMain.c | ExecutorEnd() / standard_ExecutorEnd() / ExecEndPlan()。 |
| 2 | src/backend/executor/execProcnode.c | ExecEndNode() / ExecShutdownNode()：节点级结束和提前停机。 |
| 3 | src/backend/executor/execTuples.c | ExecResetTupleTable()：slot、buffer pin、TupleDesc refcount 清理。 |
| 4 | src/backend/executor/execUtils.c | FreeExecutorState() / FreeExprContext()：EState 和 ExprContext cleanup。 |
| 5 | src/include/nodes/execnodes.h | EState 中 tuple table、exprcontexts、result relations、subplanstates。 |
| 6 | src/backend/storage/buffer/bufmgr.c | buffer pin 是 ResourceOwner 和 slot clear 共同关心的外部资源。 |
| 7 | src/backend/utils/time/snapmgr.c | UnregisterSnapshot()：snapshot 生命周期结束。 |
| 8 | src/backend/jit/jit.c | jit_release_context()：JIT context 非普通 palloc 语义。 |

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
| es_finished | End 前必须为 true，除非 EXPLAIN-only。 |
| planstate | 节点树的根，ExecEndNode 递归调用每类 ExecEnd*。 |
| es_subplanstates | 独立初始化的 subplan 状态，也必须逐个 ExecEndNode。 |
| es_tupleTable | slot 列表，EndPlan 用 ExecResetTupleTable 释放 pin 与 TupleDesc ref。 |
| es_relations | range table relation cache，EndPlan 关闭但不释放事务锁。 |
| es_exprcontexts | FreeExecutorState 中反向 shutdown callback。 |
| es_snapshot | 注册过的 snapshot，需要 UnregisterSnapshot。 |
| es_query_cxt | 最后删除的 per-query memory context。 |

本节最容易混淆的边界有四个：

- planner-time 描述与 executor-time 状态不同。
- backend-local 指针与跨 backend 可见状态不同。
- memory context 释放内存，不等于释放所有外部资源。
- 顶层调用边界与节点内部状态机边界不同。

读源码时要持续追问：这个状态现在是谁的，下一步谁会改变它，如果中途 ERROR，谁会兜底。

## 5. 主流程源码 walkthrough

下面按时间顺序读主链路。每一步都回答“状态发生了什么变化”，而不是只记住函数名。

### 5.1. 步骤 1

ExecutorEnd() 先经过 ExecutorEnd_hook。

阅读这一段时的重点不是记住第 1 个函数名，而是确认状态边界：读到这里先确认这一步是在创建状态、注册外部资源，还是只是在写入控制字段。

### 5.2. 步骤 2

standard_ExecutorEnd() 断言 QueryDesc 与 EState 存在。

阅读这一段时的重点不是记住第 2 个函数名，而是确认状态边界：断点停在这里时，优先打印 owner 指针，再打印本步刚写入的字段。

### 5.3. 步骤 3

如果启动过并行 worker，先上报 parallel worker stats。

阅读这一段时的重点不是记住第 3 个函数名，而是确认状态边界：这一步如果提前失败，后续状态还没有 owner，不能按完整 executor cleanup 想象。

### 5.4. 步骤 4

End 检查 es_finished，除非这是 EXPLAIN-only path。

阅读这一段时的重点不是记住第 4 个函数名，而是确认状态边界：这一步如果已经完成，后续代码就可以假定对应 runtime state 已经稳定。

### 5.5. 步骤 5

切入 estate->es_query_cxt，保证节点 end 中仍可访问 executor 对象。

阅读这一段时的重点不是记住第 5 个函数名，而是确认状态边界：这里要区分“指针存在”和“语义已经注册”两件事。

### 5.6. 步骤 6

ExecEndPlan(planstate, estate) 首先 ExecEndNode(root)。

阅读这一段时的重点不是记住第 6 个函数名，而是确认状态边界：这里的状态变化通常会影响后续 ExecProcNode、slot、snapshot 或 receiver 的解释。

### 5.7. 步骤 7

ExecEndPlan 继续对 estate->es_subplanstates 调用 ExecEndNode。

阅读这一段时的重点不是记住第 7 个函数名，而是确认状态边界：如果以后诊断慢查询，这一步更像 setup 成本，不应误归到单个 tuple。

### 5.8. 步骤 8

ExecResetTupleTable(estate->es_tupleTable,false) 清空所有 slot 内容、释放 slot ops 资源和 TupleDesc ref。

阅读这一段时的重点不是记住第 8 个函数名，而是确认状态边界：如果以后诊断异常退出，这一步要回看是否已经进入 ResourceOwner、MemoryContext 或 snapshot 注册边界。

### 5.9. 步骤 9

ExecCloseResultRelations(estate) 关闭 result relation 的 index 和额外 relation。

阅读这一段时的重点不是记住第 9 个函数名，而是确认状态边界：这一步的调用者协议比函数内部细节更重要：调用者是否允许重复进入，是否允许部分执行。

### 5.10. 步骤 10

ExecCloseRangeTableRelations(estate) 关闭 range table 中打开的 Relation。

阅读这一段时的重点不是记住第 10 个函数名，而是确认状态边界：这一步之后要看下一个 owner 边界，而不是继续把所有状态当作局部变量。

### 5.11. 步骤 11

回到 oldcontext 后，UnregisterSnapshot() 注销 es_snapshot 和 es_crosscheck_snapshot。

阅读这一段时的重点不是记住第 11 个函数名，而是确认状态边界：如果这一步涉及 hook 或 wrapper，要继续追到标准实现或真实节点函数。

### 5.12. 步骤 12

FreeExecutorState() 释放剩余 ExprContext callback、JIT context、partition directory，最后删除 es_query_cxt。

阅读这一段时的重点不是记住第 12 个函数名，而是确认状态边界：如果这一步涉及 slot 或 tuple，必须同时确认数据内容和底层资源是否同寿命。

### 5.13. 步骤 13

QueryDesc 中 tupDesc、estate、planstate、query_instr 被置空，避免悬挂引用。

阅读这一段时的重点不是记住第 13 个函数名，而是确认状态边界：读到这里先确认这一步是在创建状态、注册外部资源，还是只是在写入控制字段。

主链路可以压缩成：

```text
01. ExecutorEnd() 先经过 ExecutorEnd_hook。
02. standard_ExecutorEnd() 断言 QueryDesc 与 EState 存在。
03. 如果启动过并行 worker，先上报 parallel worker stats。
04. End 检查 es_finished，除非这是 EXPLAIN-only path。
05. 切入 estate->es_query_cxt，保证节点 end 中仍可访问 executor 对象。
06. ExecEndPlan(planstate, estate) 首先 ExecEndNode(root)。
07. ExecEndPlan 继续对 estate->es_subplanstates 调用 ExecEndNode。
08. ExecResetTupleTable(estate->es_tupleTable,false) 清空所有 slot 内容、释放 slot ops 资源和 TupleDesc ref。
09. ExecCloseResultRelations(estate) 关闭 result relation 的 index 和额外 relation。
10. ExecCloseRangeTableRelations(estate) 关闭 range table 中打开的 Relation。
11. 回到 oldcontext 后，UnregisterSnapshot() 注销 es_snapshot 和 es_crosscheck_snapshot。
12. FreeExecutorState() 释放剩余 ExprContext callback、JIT context、partition directory，最后删除 es_query_cxt。
13. QueryDesc 中 tupDesc、estate、planstate、query_instr 被置空，避免悬挂引用。
```

## 6. 状态随时间推进的故事

把源码主线换成一个对象的生命周期，会更容易看出本节 tension。

### 6.1. 创建前

在“创建前”阶段，关键观察点是：ExecutorEnd() 先经过 ExecutorEnd_hook。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

### 6.2. 创建时

在“创建时”阶段，关键观察点是：standard_ExecutorEnd() 断言 QueryDesc 与 EState 存在。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

### 6.3. 第一次使用前

在“第一次使用前”阶段，关键观察点是：如果启动过并行 worker，先上报 parallel worker stats。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

### 6.4. 正常推进中

在“正常推进中”阶段，关键观察点是：End 检查 es_finished，除非这是 EXPLAIN-only path。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

### 6.5. 遇到边界时

在“遇到边界时”阶段，关键观察点是：切入 estate->es_query_cxt，保证节点 end 中仍可访问 executor 对象。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

### 6.6. 提前停止时

在“提前停止时”阶段，关键观察点是：ExecEndPlan(planstate, estate) 首先 ExecEndNode(root)。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

### 6.7. 收尾时

在“收尾时”阶段，关键观察点是：ExecEndPlan 继续对 estate->es_subplanstates 调用 ExecEndNode。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

### 6.8. ERROR 后

在“ERROR 后”阶段，关键观察点是：ExecResetTupleTable(estate->es_tupleTable,false) 清空所有 slot 内容、释放 slot ops 资源和 TupleDesc ref。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

## 7. 生命周期 / ownership / cleanup

- ExecutorStart 创建 executor 状态；ExecutorRun 推进；ExecutorFinish drain；ExecutorEnd teardown。
- 节点私有资源由对应 ExecEnd* 释放，不应只依赖 context delete。
- tuple table slot 属于 EState，但 slot 内容可能持有 buffer pin 或 tuple desc ref。
- snapshot 注册属于 snapmgr 语义，不能靠 pfree 释放。
- ERROR 路径下 ResourceOwner 会兜底释放很多外部资源，但正常路径仍要精确 End。

用 ownership 语言重写本节：

| 问题 | 判断方式 |
| --- | --- |
| 谁创建 | 看入口函数和 memory context 切换点。 |
| 谁持有 | 看 QueryDesc、EState、PlanState 或 slot 列表中的指针。 |
| 谁释放 | 看 ExecutorFinish、ExecutorEnd、ExecEndNode、ExecResetTupleTable 或 ResourceOwner。 |
| ERROR 怎么办 | 看资源是否有 ResourceOwner、MemoryContext、snapshot register 或 callback 兜底。 |
| 能否跨边界保存 | 看对象是否 backend-local、是否依赖 slot 内容、buffer pin 或 per-tuple context。 |

## 8. 正确性机制层次

- 先 End 节点再清 tuple table，避免节点 end 还需要访问自己的 slot。
- 先清 slot 再删除 memory context，才能释放 buffer pin 和 TupleDesc 引用。
- Relation close 不释放事务锁，锁生命周期仍由事务管理。
- snapshot unregister 必须发生在删除 EState 前，因为 snapshot 指针保存在 EState。
- ExprContext callback 在 FreeExecutorState 中显式执行，处理非内存资源。
- QueryDesc 字段置空让重复 End 或后续误用更容易暴露。

这些机制互相补位。snapshot 解决可见性，不解决内存生命周期；MemoryContext 解决内存批量释放，不解决 buffer pin；函数指针解决分派成本，不解决节点协议正确性。

| 层次 | 保证 | 不能误解为 |
| --- | --- | --- |
| snapshot / command id | tuple 可见性和写入标记边界 | 并发互斥或资源释放 |
| PlanState / slot 协议 | 节点间传递和推进方式一致 | 所有节点都非阻塞 |
| MemoryContext | backend-local 内存归属清楚 | 外部资源自动释放 |
| ResourceOwner / pin / refcount | 外部资源不会被过早回收 | 对象语义仍然新鲜 |
| hook / instrumentation | 观测或扩展入口 | 可以任意改变核心语义 |

## 9. 错误路径 / 异常路径 / fallback

- EXPLAIN-only 可以没有 ExecutorFinish，但仍需要 ExecutorEnd 释放启动阶段创建的 PlanState。
- 如果 ExecutorRun 提前停止，ExecutorEnd 仍必须释放已经打开的 relation 和 slot pin。
- ExecShutdownNode 可能在 ExecutePlan 结束时提前停止并行/异步资源，但不能替代 ExecEndNode。
- ERROR 期间 isCommit=false 的 ExprContext cleanup 不会执行全部正常 callback 语义。
- FreeExecutorState 说明自己不负责释放所有非内存资源，所以不能提前调用。

PostgreSQL executor 的异常路径不能按 C++ 析构模型理解。`ereport(ERROR)` 可能 longjmp 到上层，普通 C 栈上的清理代码不会自然逐层执行。

因此本节所有状态都要问一句：如果这个点 ERROR，已经注册的资源由谁释放，尚未注册的资源是否会泄漏或变成悬挂引用。

## 10. 成本、资源与跨模块传播

- teardown 成本随 plan node 数、slot 数、打开 relation 数、subplan 数扩张。
- TupleTableSlot 清理可能释放 buffer pin，在并发场景中影响等待链解除时间。
- JIT context release 和 partition directory destroy 可能出现在查询尾部。
- 大 executor memory context delete 是批量释放，通常比逐个 pfree 更便宜。
- 扩展 hook 如果在 End 中做重型工作，会污染用户对执行尾部时间的判断。

executor 的成本常常不是单个函数慢，而是边界次数被 workload 放大：每行、每节点、每次 FETCH、每个 trigger、每个 slot clear，都可能成为可见成本。

| 放大因素 | 典型表现 |
| --- | --- |
| tuple 数 | ExecProcNode、slot deform、per-tuple reset 和 receiver 调用被放大。 |
| plan node 数 | ExecInitNode、ExecEndNode、instrumentation 对象和 EXPLAIN 输出被放大。 |
| relation 数 | range table relation 打开、ResultRelInfo、锁和权限检查被放大。 |
| FETCH 次数 | ExecutorRun 边界、DestReceiver startup/shutdown 和 Portal 状态切换被放大。 |
| 副作用数量 | trigger、constraint、FDW flush、ModifyTable drain 被放大。 |

## 11. 观测与诊断入口

- gdb 断在 ExecEndPlan 可以按顺序观察 ExecEndNode、ExecResetTupleTable、ExecClose*。
- 断在 ExecResetTupleTable 可观察 buffer-backed slot clear 是否释放 pin。
- pg_backend_memory_contexts 可看到 ExecutorState context 在 End 后消失。
- pg_locks 中事务锁不会因 ExecutorEnd 消失，这是 Relation close 与 lock 生命周期分离的证据。
- EXPLAIN-only 路径可观察 es_finished 断言例外。

建议的断点顺序：

1. `ExecutorEnd`
2. `standard_ExecutorEnd`
3. `ExecEndPlan`
4. `ExecEndNode`
5. `ExecResetTupleTable`
6. `ExecCloseResultRelations`
7. `UnregisterSnapshot`
8. `FreeExecutorState`

断点不只是为了停住程序。每个断点都应该带一个问题：这个状态是否已经初始化，owner 是谁，下一步是否会被改写。

## 12. 课堂实验

### 实验 1

执行 DECLARE CURSOR 后查询 pg_backend_memory_contexts，关闭 cursor 后再查 ExecutorState。

观察点：
- 断点前后同一个字段是否发生变化。
- 这个变化是否发生在本节主流程上。
- 如果把 SQL 改成 cursor、EXPLAIN、DML 或带 trigger，边界是否改变。

### 实验 2

对 ExecutorEnd、ExecEndPlan、ExecResetTupleTable 设置断点，记录调用顺序。

观察点：
- 断点前后同一个字段是否发生变化。
- 这个变化是否发生在本节主流程上。
- 如果把 SQL 改成 cursor、EXPLAIN、DML 或带 trigger，边界是否改变。

### 实验 3

执行 SELECT * FROM t，断在 ExecEndSeqScan，观察 scan descriptor 释放。

观察点：
- 断点前后同一个字段是否发生变化。
- 这个变化是否发生在本节主流程上。
- 如果把 SQL 改成 cursor、EXPLAIN、DML 或带 trigger，边界是否改变。

### 实验 4

执行 SELECT * FROM t ORDER BY a，断在 ExecEndSort，观察 tuplesort 状态释放。

观察点：
- 断点前后同一个字段是否发生变化。
- 这个变化是否发生在本节主流程上。
- 如果把 SQL 改成 cursor、EXPLAIN、DML 或带 trigger，边界是否改变。

### 实验 5

在 ExecResetTupleTable 打印 slot->tts_ops，比较 virtual 与 buffer-backed slot。

观察点：
- 断点前后同一个字段是否发生变化。
- 这个变化是否发生在本节主流程上。
- 如果把 SQL 改成 cursor、EXPLAIN、DML 或带 trigger，边界是否改变。

### 实验 6

用 EXPLAIN SELECT 不实际执行，确认仍会走 End 释放 PlanState。

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

### 练习 1：`ExecutorEnd`

在 `/home/highgo/postgres` 中定位 `ExecutorEnd`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

### 练习 2：`standard_ExecutorEnd`

在 `/home/highgo/postgres` 中定位 `standard_ExecutorEnd`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

### 练习 3：`ExecEndPlan`

在 `/home/highgo/postgres` 中定位 `ExecEndPlan`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

### 练习 4：`ExecEndNode`

在 `/home/highgo/postgres` 中定位 `ExecEndNode`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

### 练习 5：`ExecResetTupleTable`

在 `/home/highgo/postgres` 中定位 `ExecResetTupleTable`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

### 练习 6：`ExecCloseResultRelations`

在 `/home/highgo/postgres` 中定位 `ExecCloseResultRelations`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

### 练习 7：`UnregisterSnapshot`

在 `/home/highgo/postgres` 中定位 `UnregisterSnapshot`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

### 练习 8：`FreeExecutorState`

在 `/home/highgo/postgres` 中定位 `FreeExecutorState`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

## 14. 常见误区

- 不要以为删除 es_query_cxt 就能安全释放一切；非内存资源有自己的协议。
- 不要把 Relation close 和 lock release 混为一谈。
- 不要把 ExecShutdownNode 与 ExecEndNode 混为一谈。
- 不要在 ExecutorEnd 后继续使用 QueryDesc->planstate 或 QueryDesc->estate。
- 不要忽略 subplanstates，它们不是 root planstate 递归树的一部分。

这些误区的共同点是把 runtime 状态当成静态结构，或者把静态计划当成可以直接推进的状态机。

## 15. 讨论题

1. 如果 ExecResetTupleTable 放在 ExecEndNode 之前，会破坏哪些节点假设？
2. 哪些资源应该由 ResourceOwner 兜底，哪些仍应该在正常 End 中显式释放？
3. QueryDesc 字段置空是防御式编程还是语义必需？
4. 为什么 executor 不在每个节点 end 中逐个 pfree 所有对象？

讨论时要尽量回到一个具体 runtime 现象，比如一次 FETCH、一次 EXPLAIN ANALYZE、一个 trigger 或一个 slot materialize，而不是停留在抽象偏好。

## 16. 本节小结

本节唯一主问题是：ExecutorEnd() 为什么必须按节点结束、tuple table 清理、Relation 关闭、snapshot 注销、memory context 删除的顺序收尾，而不是直接删除 executor memory context？

可以带走的一句话模型是：ExecutorEnd() 在 es_query_cxt 中调用 ExecEndPlan()，让节点先释放外部资源，再清 tuple table 和 Relation，随后注销 snapshot，最后 FreeExecutorState() 删除 executor context。

- 源码阅读应先找状态创建点，再找推进点，最后找 cleanup。
- 任何字段都要和 owner、生命周期、异常路径一起理解。
- 观测数据只有能回到具体 runtime 边界时，才适合用于诊断。

下一节进入 Plan 到 PlanState 的更细边界。

## 17. 从现象回源码的诊断案例

下面的案例都只服务本节主问题。它们的共同读法是先看现象，再找 runtime 边界，最后回到 owner 和 cleanup。

### 案例 1：第一次执行停在入口但状态还没有建立

现象：

- 你在 `ExecutorRun` 附近停住，看到 SQL 还没有返回，但某些 executor 状态已经改变。
- 这时不要急着沿调用栈往下追完整执行，而要先回到本节主问题：ExecutorEnd() 为什么必须按节点结束、tuple table 清理、Relation 关闭、snapshot 注销、memory context 删除的顺序收尾，而不是直接删除 executor memory context？
- 如果现象来自 EXPLAIN、cursor、trigger 或扩展 hook，要先确认它落在哪个 executor 生命周期边界。

源码回看：

- 入口函数：`ExecutorRun`。
- 本步主线：ExecutorEnd() 先经过 ExecutorEnd_hook。
- 关键状态：`es_finished`，语义是：End 前必须为 true，除非 EXPLAIN-only。
- 相邻状态：`planstate`，用来判断这一步之后谁继续持有资源。

判断方式：

- 如果字段刚被写入但还没有注册到 owner，异常路径通常只能靠当前函数局部清理。
- 如果状态已经挂到 EState、PlanState、slot table 或 snapshot manager，后续 cleanup 应该能从 owner 找回。
- 如果这个状态只影响观测，不影响语义，诊断时不要把它解释成执行计划行为改变。
- 如果这个状态影响 tuple 生命周期，必须同时检查 slot、TupleDesc、buffer pin 或 per-tuple context。

### 案例 2：查询只返回少量行但内部状态已经推进

现象：

- 你在 `ExecutorRun` 附近停住，看到 SQL 还没有返回，但某些 executor 状态已经改变。
- 这时不要急着沿调用栈往下追完整执行，而要先回到本节主问题：ExecutorEnd() 为什么必须按节点结束、tuple table 清理、Relation 关闭、snapshot 注销、memory context 删除的顺序收尾，而不是直接删除 executor memory context？
- 如果现象来自 EXPLAIN、cursor、trigger 或扩展 hook，要先确认它落在哪个 executor 生命周期边界。

源码回看：

- 入口函数：`ExecutorRun`。
- 本步主线：standard_ExecutorEnd() 断言 QueryDesc 与 EState 存在。
- 关键状态：`planstate`，语义是：节点树的根，ExecEndNode 递归调用每类 ExecEnd*。
- 相邻状态：`es_subplanstates`，用来判断这一步之后谁继续持有资源。

判断方式：

- 如果字段刚被写入但还没有注册到 owner，异常路径通常只能靠当前函数局部清理。
- 如果状态已经挂到 EState、PlanState、slot table 或 snapshot manager，后续 cleanup 应该能从 owner 找回。
- 如果这个状态只影响观测，不影响语义，诊断时不要把它解释成执行计划行为改变。
- 如果这个状态影响 tuple 生命周期，必须同时检查 slot、TupleDesc、buffer pin 或 per-tuple context。

### 案例 3：EXPLAIN ANALYZE 时间与节点时间不一致

现象：

- 你在 `ExecutorRun` 附近停住，看到 SQL 还没有返回，但某些 executor 状态已经改变。
- 这时不要急着沿调用栈往下追完整执行，而要先回到本节主问题：ExecutorEnd() 为什么必须按节点结束、tuple table 清理、Relation 关闭、snapshot 注销、memory context 删除的顺序收尾，而不是直接删除 executor memory context？
- 如果现象来自 EXPLAIN、cursor、trigger 或扩展 hook，要先确认它落在哪个 executor 生命周期边界。

源码回看：

- 入口函数：`ExecutorRun`。
- 本步主线：如果启动过并行 worker，先上报 parallel worker stats。
- 关键状态：`es_subplanstates`，语义是：独立初始化的 subplan 状态，也必须逐个 ExecEndNode。
- 相邻状态：`es_tupleTable`，用来判断这一步之后谁继续持有资源。

判断方式：

- 如果字段刚被写入但还没有注册到 owner，异常路径通常只能靠当前函数局部清理。
- 如果状态已经挂到 EState、PlanState、slot table 或 snapshot manager，后续 cleanup 应该能从 owner 找回。
- 如果这个状态只影响观测，不影响语义，诊断时不要把它解释成执行计划行为改变。
- 如果这个状态影响 tuple 生命周期，必须同时检查 slot、TupleDesc、buffer pin 或 per-tuple context。

### 案例 4：cursor 或 receiver 让执行被分成多段

现象：

- 你在 `ExecutorRun` 附近停住，看到 SQL 还没有返回，但某些 executor 状态已经改变。
- 这时不要急着沿调用栈往下追完整执行，而要先回到本节主问题：ExecutorEnd() 为什么必须按节点结束、tuple table 清理、Relation 关闭、snapshot 注销、memory context 删除的顺序收尾，而不是直接删除 executor memory context？
- 如果现象来自 EXPLAIN、cursor、trigger 或扩展 hook，要先确认它落在哪个 executor 生命周期边界。

源码回看：

- 入口函数：`ExecutorRun`。
- 本步主线：End 检查 es_finished，除非这是 EXPLAIN-only path。
- 关键状态：`es_tupleTable`，语义是：slot 列表，EndPlan 用 ExecResetTupleTable 释放 pin 与 TupleDesc ref。
- 相邻状态：`es_relations`，用来判断这一步之后谁继续持有资源。

判断方式：

- 如果字段刚被写入但还没有注册到 owner，异常路径通常只能靠当前函数局部清理。
- 如果状态已经挂到 EState、PlanState、slot table 或 snapshot manager，后续 cleanup 应该能从 owner 找回。
- 如果这个状态只影响观测，不影响语义，诊断时不要把它解释成执行计划行为改变。
- 如果这个状态影响 tuple 生命周期，必须同时检查 slot、TupleDesc、buffer pin 或 per-tuple context。

### 案例 5：ERROR 发生后没有按普通调用栈返回

现象：

- 你在 `ExecutorRun` 附近停住，看到 SQL 还没有返回，但某些 executor 状态已经改变。
- 这时不要急着沿调用栈往下追完整执行，而要先回到本节主问题：ExecutorEnd() 为什么必须按节点结束、tuple table 清理、Relation 关闭、snapshot 注销、memory context 删除的顺序收尾，而不是直接删除 executor memory context？
- 如果现象来自 EXPLAIN、cursor、trigger 或扩展 hook，要先确认它落在哪个 executor 生命周期边界。

源码回看：

- 入口函数：`ExecutorRun`。
- 本步主线：切入 estate->es_query_cxt，保证节点 end 中仍可访问 executor 对象。
- 关键状态：`es_relations`，语义是：range table relation cache，EndPlan 关闭但不释放事务锁。
- 相邻状态：`es_exprcontexts`，用来判断这一步之后谁继续持有资源。

判断方式：

- 如果字段刚被写入但还没有注册到 owner，异常路径通常只能靠当前函数局部清理。
- 如果状态已经挂到 EState、PlanState、slot table 或 snapshot manager，后续 cleanup 应该能从 owner 找回。
- 如果这个状态只影响观测，不影响语义，诊断时不要把它解释成执行计划行为改变。
- 如果这个状态影响 tuple 生命周期，必须同时检查 slot、TupleDesc、buffer pin 或 per-tuple context。

### 案例 6：扩展 hook 看到的边界与核心 executor 边界不同

现象：

- 你在 `ExecutorRun` 附近停住，看到 SQL 还没有返回，但某些 executor 状态已经改变。
- 这时不要急着沿调用栈往下追完整执行，而要先回到本节主问题：ExecutorEnd() 为什么必须按节点结束、tuple table 清理、Relation 关闭、snapshot 注销、memory context 删除的顺序收尾，而不是直接删除 executor memory context？
- 如果现象来自 EXPLAIN、cursor、trigger 或扩展 hook，要先确认它落在哪个 executor 生命周期边界。

源码回看：

- 入口函数：`ExecutorRun`。
- 本步主线：ExecEndPlan(planstate, estate) 首先 ExecEndNode(root)。
- 关键状态：`es_exprcontexts`，语义是：FreeExecutorState 中反向 shutdown callback。
- 相邻状态：`es_snapshot`，用来判断这一步之后谁继续持有资源。

判断方式：

- 如果字段刚被写入但还没有注册到 owner，异常路径通常只能靠当前函数局部清理。
- 如果状态已经挂到 EState、PlanState、slot table 或 snapshot manager，后续 cleanup 应该能从 owner 找回。
- 如果这个状态只影响观测，不影响语义，诊断时不要把它解释成执行计划行为改变。
- 如果这个状态影响 tuple 生命周期，必须同时检查 slot、TupleDesc、buffer pin 或 per-tuple context。

## 18. 源码断点与字段记录表

下面这张表用于课堂现场跟读。每一行都应该实际在 gdb 中停一次，记录进入前和返回后的字段差异。

| 序号 | 断点 | 进入前看什么 | 返回后看什么 | 常见误判 |
| --- | --- | --- | --- | --- |
| 1 | `ExecutorRun` | `es_finished` 是否已经有效 | ExecutorEnd() 先经过 ExecutorEnd_hook。 | 把字段存在误认为语义已经完成 |
| 2 | `ExecutorRun` | `planstate` 是否已经有效 | standard_ExecutorEnd() 断言 QueryDesc 与 EState 存在。 | 把字段存在误认为语义已经完成 |
| 3 | `ExecutorRun` | `es_subplanstates` 是否已经有效 | 如果启动过并行 worker，先上报 parallel worker stats。 | 把字段存在误认为语义已经完成 |
| 4 | `ExecutorRun` | `es_tupleTable` 是否已经有效 | End 检查 es_finished，除非这是 EXPLAIN-only path。 | 把字段存在误认为语义已经完成 |
| 5 | `ExecutorRun` | `es_relations` 是否已经有效 | 切入 estate->es_query_cxt，保证节点 end 中仍可访问 executor 对象。 | 把字段存在误认为语义已经完成 |
| 6 | `ExecutorRun` | `es_exprcontexts` 是否已经有效 | ExecEndPlan(planstate, estate) 首先 ExecEndNode(root)。 | 把字段存在误认为语义已经完成 |
| 7 | `ExecutorRun` | `es_snapshot` 是否已经有效 | ExecEndPlan 继续对 estate->es_subplanstates 调用 ExecEndNode。 | 把字段存在误认为语义已经完成 |
| 8 | `ExecutorRun` | `es_query_cxt` 是否已经有效 | ExecResetTupleTable(estate->es_tupleTable,false) 清空所有 slot 内容、释放 slot ops 资源和 TupleDesc ref。 | 把字段存在误认为语义已经完成 |
| 9 | `ExecutorRun` | `es_finished` 是否已经有效 | ExecCloseResultRelations(estate) 关闭 result relation 的 index 和额外 relation。 | 把字段存在误认为语义已经完成 |
| 10 | `ExecutorRun` | `planstate` 是否已经有效 | ExecCloseRangeTableRelations(estate) 关闭 range table 中打开的 Relation。 | 把字段存在误认为语义已经完成 |
| 11 | `ExecutorRun` | `es_subplanstates` 是否已经有效 | 回到 oldcontext 后，UnregisterSnapshot() 注销 es_snapshot 和 es_crosscheck_snapshot。 | 把字段存在误认为语义已经完成 |
| 12 | `ExecutorRun` | `es_tupleTable` 是否已经有效 | FreeExecutorState() 释放剩余 ExprContext callback、JIT context、partition directory，最后删除 es_query_cxt。 | 把字段存在误认为语义已经完成 |
| 13 | `ExecutorRun` | `es_relations` 是否已经有效 | QueryDesc 中 tupDesc、estate、planstate、query_instr 被置空，避免悬挂引用。 | 把字段存在误认为语义已经完成 |

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
- 只观察一个状态变化：ExecutorEnd() 先经过 ExecutorEnd_hook。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 2：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：standard_ExecutorEnd() 断言 QueryDesc 与 EState 存在。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 3：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：如果启动过并行 worker，先上报 parallel worker stats。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 4：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：End 检查 es_finished，除非这是 EXPLAIN-only path。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 5：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：切入 estate->es_query_cxt，保证节点 end 中仍可访问 executor 对象。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 6：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：ExecEndPlan(planstate, estate) 首先 ExecEndNode(root)。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 7：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：ExecEndPlan 继续对 estate->es_subplanstates 调用 ExecEndNode。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 8：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：ExecResetTupleTable(estate->es_tupleTable,false) 清空所有 slot 内容、释放 slot ops 资源和 TupleDesc ref。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 9：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：ExecCloseResultRelations(estate) 关闭 result relation 的 index 和额外 relation。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 10：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：ExecCloseRangeTableRelations(estate) 关闭 range table 中打开的 Relation。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 11：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：回到 oldcontext 后，UnregisterSnapshot() 注销 es_snapshot 和 es_crosscheck_snapshot。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 12：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：FreeExecutorState() 释放剩余 ExprContext callback、JIT context、partition directory，最后删除 es_query_cxt。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 13：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：QueryDesc 中 tupDesc、estate、planstate、query_instr 被置空，避免悬挂引用。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 14：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：ExecutorEnd() 先经过 ExecutorEnd_hook。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 15：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：standard_ExecutorEnd() 断言 QueryDesc 与 EState 存在。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 16：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：如果启动过并行 worker，先上报 parallel worker stats。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 17：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：End 检查 es_finished，除非这是 EXPLAIN-only path。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 18：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：切入 estate->es_query_cxt，保证节点 end 中仍可访问 executor 对象。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 19：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：ExecEndPlan(planstate, estate) 首先 ExecEndNode(root)。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 20：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：ExecEndPlan 继续对 estate->es_subplanstates 调用 ExecEndNode。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 21：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：ExecResetTupleTable(estate->es_tupleTable,false) 清空所有 slot 内容、释放 slot ops 资源和 TupleDesc ref。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 22：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：ExecCloseResultRelations(estate) 关闭 result relation 的 index 和额外 relation。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 23：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：ExecCloseRangeTableRelations(estate) 关闭 range table 中打开的 Relation。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 24：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：回到 oldcontext 后，UnregisterSnapshot() 注销 es_snapshot 和 es_crosscheck_snapshot。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 25：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：FreeExecutorState() 释放剩余 ExprContext callback、JIT context、partition directory，最后删除 es_query_cxt。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 26：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：QueryDesc 中 tupDesc、estate、planstate、query_instr 被置空，避免悬挂引用。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。
## 20. 一页纸定位模板

这部分用于把课程内容压缩成排查动作。不要从这里开始背函数名；先带着现象进入，再沿状态边界回源码。

### 定位 1：执行还没返回，但 gdb 已经停在 executor 内部

- 先停在 `ExecutorEnd`，确认当前调用属于 `PostgreSQL ExecutorEnd 与 ERROR-safe teardown 顺序` 的哪一个生命周期边界。
- 打印 `es_finished`，不要只看字段值，还要解释它的语义：End 前必须为 true，除非 EXPLAIN-only。
- 向上追一层调用者，确认是谁要求 executor 进入这个边界。
- 向下追一层被调用者，确认这一步是否注册了资源、改写了状态或返回了 slot。
- 如果现象来自观测指标，判断指标采样是在进入本步之前、执行过程中，还是返回之后。
- 如果现象来自异常路径，判断资源是否已经挂到 owner；没有 owner 的对象必须由当前路径处理。

### 定位 2：EXPLAIN ANALYZE 的时间和直觉不一致

- 先停在 `standard_ExecutorEnd`，确认当前调用属于 `PostgreSQL ExecutorEnd 与 ERROR-safe teardown 顺序` 的哪一个生命周期边界。
- 打印 `planstate`，不要只看字段值，还要解释它的语义：节点树的根，ExecEndNode 递归调用每类 ExecEnd*。
- 向上追一层调用者，确认是谁要求 executor 进入这个边界。
- 向下追一层被调用者，确认这一步是否注册了资源、改写了状态或返回了 slot。
- 如果现象来自观测指标，判断指标采样是在进入本步之前、执行过程中，还是返回之后。
- 如果现象来自异常路径，判断资源是否已经挂到 owner；没有 owner 的对象必须由当前路径处理。

### 定位 3：cursor、FETCH 或客户端输出让执行分段

- 先停在 `ExecEndPlan`，确认当前调用属于 `PostgreSQL ExecutorEnd 与 ERROR-safe teardown 顺序` 的哪一个生命周期边界。
- 打印 `es_subplanstates`，不要只看字段值，还要解释它的语义：独立初始化的 subplan 状态，也必须逐个 ExecEndNode。
- 向上追一层调用者，确认是谁要求 executor 进入这个边界。
- 向下追一层被调用者，确认这一步是否注册了资源、改写了状态或返回了 slot。
- 如果现象来自观测指标，判断指标采样是在进入本步之前、执行过程中，还是返回之后。
- 如果现象来自异常路径，判断资源是否已经挂到 owner；没有 owner 的对象必须由当前路径处理。

### 定位 4：ERROR 后怀疑 executor 资源没有按预期释放

- 先停在 `ExecEndNode`，确认当前调用属于 `PostgreSQL ExecutorEnd 与 ERROR-safe teardown 顺序` 的哪一个生命周期边界。
- 打印 `es_tupleTable`，不要只看字段值，还要解释它的语义：slot 列表，EndPlan 用 ExecResetTupleTable 释放 pin 与 TupleDesc ref。
- 向上追一层调用者，确认是谁要求 executor 进入这个边界。
- 向下追一层被调用者，确认这一步是否注册了资源、改写了状态或返回了 slot。
- 如果现象来自观测指标，判断指标采样是在进入本步之前、执行过程中，还是返回之后。
- 如果现象来自异常路径，判断资源是否已经挂到 owner；没有 owner 的对象必须由当前路径处理。

### 定位 5：扩展 hook 或 profiler 捕获到一个边界事件

- 先停在 `ExecResetTupleTable`，确认当前调用属于 `PostgreSQL ExecutorEnd 与 ERROR-safe teardown 顺序` 的哪一个生命周期边界。
- 打印 `es_relations`，不要只看字段值，还要解释它的语义：range table relation cache，EndPlan 关闭但不释放事务锁。
- 向上追一层调用者，确认是谁要求 executor 进入这个边界。
- 向下追一层被调用者，确认这一步是否注册了资源、改写了状态或返回了 slot。
- 如果现象来自观测指标，判断指标采样是在进入本步之前、执行过程中，还是返回之后。
- 如果现象来自异常路径，判断资源是否已经挂到 owner；没有 owner 的对象必须由当前路径处理。

### 定位 6：源码阅读时分不清字段属于 Plan 还是 runtime state

- 先停在 `ExecCloseResultRelations`，确认当前调用属于 `PostgreSQL ExecutorEnd 与 ERROR-safe teardown 顺序` 的哪一个生命周期边界。
- 打印 `es_exprcontexts`，不要只看字段值，还要解释它的语义：FreeExecutorState 中反向 shutdown callback。
- 向上追一层调用者，确认是谁要求 executor 进入这个边界。
- 向下追一层被调用者，确认这一步是否注册了资源、改写了状态或返回了 slot。
- 如果现象来自观测指标，判断指标采样是在进入本步之前、执行过程中，还是返回之后。
- 如果现象来自异常路径，判断资源是否已经挂到 owner；没有 owner 的对象必须由当前路径处理。

定位结束时只保留三句话：

1. 这个现象落在哪个 executor 生命周期边界。
2. 这个边界改变了哪个核心状态，以及 owner 是谁。
3. 正常路径和 ERROR 路径分别由哪个函数或机制收尾。
