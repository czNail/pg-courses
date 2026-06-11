# PostgreSQL ExecutePlan 与 demand-driven tuple 拉取模型

## 课程定位

前置知识：已经读过 executor 生命周期的目录规划，并理解 PostgreSQL 的查询执行由 Portal、QueryDesc、EState、PlanState 和 TupleTableSlot 共同完成。上一节说明了 Portal 如何决定一次 ExecutorRun 的方向和数量。

本节唯一主问题：

```text
为什么大多数执行节点通过 ExecProcNode() 一次返回一个 TupleTableSlot，而不是由父节点一次性要求子树把所有结果 materialize 出来？
```

核心矛盾：tuple-at-a-time 的低延迟和低内存占用 vs 阻塞节点、DML 副作用、junk filter、并行模式和输出端提前关闭带来的状态复杂性。

学完后应能判断：能沿一个 tuple 从 scan node 到 root，再到 DestReceiver 的路径解释执行器为什么是 demand-driven。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

上一节说明了 Portal 如何决定一次 ExecutorRun 的方向和数量。

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

下一节会看为什么 run 完顶层 tuple 后还需要 ExecutorFinish 单独 drain 副作用。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
ExecutePlan() 是顶层拉取循环；它反复重置 per-output-tuple context，调用 root ExecProcNode()，处理 junk filter 和 DestReceiver，直到 EOF、count 达到或输出端停止。
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
| 1 | src/backend/executor/execMain.c | ExecutePlan()：顶层 tuple 拉取循环。 |
| 2 | src/include/executor/executor.h | ExecProcNode() inline：chgParam rescan 与函数指针调用。 |
| 3 | src/backend/executor/execProcnode.c | ExecProcNodeFirst() / ExecSetExecProcNode()：首调用 wrapper 和 instrumentation wrapper。 |
| 4 | src/backend/executor/execScan.c | ExecScan()：scan node 通用扫描、qual、projection 逻辑。 |
| 5 | src/backend/executor/nodeSeqscan.c | ExecSeqScan*()：最简单的 tuple source 示例。 |
| 6 | src/backend/executor/nodeSort.c | ExecSort()：阻塞节点如何在同一协议下工作。 |
| 7 | src/backend/executor/nodeModifyTable.c | ExecModifyTable()：DML 节点如何返回 RETURNING 或空 slot。 |
| 8 | src/backend/executor/execTuples.c | TupleTableSlot 在节点间传递，而不是复制 tuple。 |

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
| PlanState->ExecProcNode | 当前节点的 next-tuple 函数指针。 |
| PlanState->chgParam | 参数变化触发的 rescan 边界，ExecProcNode inline 会先检查。 |
| TupleTableSlot | 节点之间传递的 tuple 容器；可能是 virtual、heap、minimal 或 buffer-backed。 |
| es_direction | 本次 ExecutePlan 的扫描方向。 |
| es_per_tuple_exprcontext | 顶层每个输出 tuple 重置的短生命周期内存。 |
| es_junkFilter | 顶层 SELECT 去掉 resjunk 属性的投影边界。 |
| current_tuple_count | ExecutePlan 本地 count 计数，不跨 ExecutorRun。 |
| parallel mode flag | 本次是否允许进入并行模式。 |

本节最容易混淆的边界有四个：

- planner-time 描述与 executor-time 状态不同。
- backend-local 指针与跨 backend 可见状态不同。
- memory context 释放内存，不等于释放所有外部资源。
- 顶层调用边界与节点内部状态机边界不同。

读源码时要持续追问：这个状态现在是谁的，下一步谁会改变它，如果中途 ERROR，谁会兜底。

## 5. 主流程源码 walkthrough

下面按时间顺序读主链路。每一步都回答“状态发生了什么变化”，而不是只记住函数名。

### 5.1. 步骤 1

ExecutePlan() 先把 direction 写入 estate->es_direction。

阅读这一段时的重点不是记住第 1 个函数名，而是确认状态边界：读到这里先确认这一步是在创建状态、注册外部资源，还是只是在写入控制字段。

### 5.2. 步骤 2

它根据 already_executed 和 numberTuples 判断本次是否可以使用 parallel mode。

阅读这一段时的重点不是记住第 2 个函数名，而是确认状态边界：断点停在这里时，优先打印 owner 指针，再打印本步刚写入的字段。

### 5.3. 步骤 3

进入循环前 current_tuple_count 为 0。

阅读这一段时的重点不是记住第 3 个函数名，而是确认状态边界：这一步如果提前失败，后续状态还没有 owner，不能按完整 executor cleanup 想象。

### 5.4. 步骤 4

每轮先 ResetPerTupleExprContext(estate)，清理顶层输出 tuple 的临时表达式内存。

阅读这一段时的重点不是记住第 4 个函数名，而是确认状态边界：这一步如果已经完成，后续代码就可以假定对应 runtime state 已经稳定。

### 5.5. 步骤 5

root ExecProcNode(planstate) 被调用，父节点向子节点继续拉取，直到某个叶子节点生产 slot。

阅读这一段时的重点不是记住第 5 个函数名，而是确认状态边界：这里要区分“指针存在”和“语义已经注册”两件事。

### 5.6. 步骤 6

slot 为空表示 EOF，顶层循环结束。

阅读这一段时的重点不是记住第 6 个函数名，而是确认状态边界：这里的状态变化通常会影响后续 ExecProcNode、slot、snapshot 或 receiver 的解释。

### 5.7. 步骤 7

如果有 junk filter，ExecFilterJunk() 把内部 tuple 形状投影成客户端可见形状。

阅读这一段时的重点不是记住第 7 个函数名，而是确认状态边界：如果以后诊断慢查询，这一步更像 setup 成本，不应误归到单个 tuple。

### 5.8. 步骤 8

sendTuples 为真时，dest->receiveSlot(slot,dest) 消费这个 slot。

阅读这一段时的重点不是记住第 8 个函数名，而是确认状态边界：如果以后诊断异常退出，这一步要回看是否已经进入 ResourceOwner、MemoryContext 或 snapshot 注册边界。

### 5.9. 步骤 9

receiveSlot 返回 false 时，输出端停止，ExecutePlan 不再继续拉取。

阅读这一段时的重点不是记住第 9 个函数名，而是确认状态边界：这一步的调用者协议比函数内部细节更重要：调用者是否允许重复进入，是否允许部分执行。

### 5.10. 步骤 10

SELECT 顶层计数增加 es_processed，DML 则由 ModifyTable 自己计数。

阅读这一段时的重点不是记住第 10 个函数名，而是确认状态边界：这一步之后要看下一个 owner 边界，而不是继续把所有状态当作局部变量。

### 5.11. 步骤 11

numberTuples 非 0 且达到上限时停止。

阅读这一段时的重点不是记住第 11 个函数名，而是确认状态边界：如果这一步涉及 hook 或 wrapper，要继续追到标准实现或真实节点函数。

### 5.12. 步骤 12

不需要 backward 时，ExecShutdownNode(planstate) 尝试尽早停止异步资源消费。

阅读这一段时的重点不是记住第 12 个函数名，而是确认状态边界：如果这一步涉及 slot 或 tuple，必须同时确认数据内容和底层资源是否同寿命。

主链路可以压缩成：

```text
01. ExecutePlan() 先把 direction 写入 estate->es_direction。
02. 它根据 already_executed 和 numberTuples 判断本次是否可以使用 parallel mode。
03. 进入循环前 current_tuple_count 为 0。
04. 每轮先 ResetPerTupleExprContext(estate)，清理顶层输出 tuple 的临时表达式内存。
05. root ExecProcNode(planstate) 被调用，父节点向子节点继续拉取，直到某个叶子节点生产 slot。
06. slot 为空表示 EOF，顶层循环结束。
07. 如果有 junk filter，ExecFilterJunk() 把内部 tuple 形状投影成客户端可见形状。
08. sendTuples 为真时，dest->receiveSlot(slot,dest) 消费这个 slot。
09. receiveSlot 返回 false 时，输出端停止，ExecutePlan 不再继续拉取。
10. SELECT 顶层计数增加 es_processed，DML 则由 ModifyTable 自己计数。
11. numberTuples 非 0 且达到上限时停止。
12. 不需要 backward 时，ExecShutdownNode(planstate) 尝试尽早停止异步资源消费。
```

## 6. 状态随时间推进的故事

把源码主线换成一个对象的生命周期，会更容易看出本节 tension。

### 6.1. 创建前

在“创建前”阶段，关键观察点是：ExecutePlan() 先把 direction 写入 estate->es_direction。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

### 6.2. 创建时

在“创建时”阶段，关键观察点是：它根据 already_executed 和 numberTuples 判断本次是否可以使用 parallel mode。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

### 6.3. 第一次使用前

在“第一次使用前”阶段，关键观察点是：进入循环前 current_tuple_count 为 0。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

### 6.4. 正常推进中

在“正常推进中”阶段，关键观察点是：每轮先 ResetPerTupleExprContext(estate)，清理顶层输出 tuple 的临时表达式内存。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

### 6.5. 遇到边界时

在“遇到边界时”阶段，关键观察点是：root ExecProcNode(planstate) 被调用，父节点向子节点继续拉取，直到某个叶子节点生产 slot。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

### 6.6. 提前停止时

在“提前停止时”阶段，关键观察点是：slot 为空表示 EOF，顶层循环结束。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

### 6.7. 收尾时

在“收尾时”阶段，关键观察点是：如果有 junk filter，ExecFilterJunk() 把内部 tuple 形状投影成客户端可见形状。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

### 6.8. ERROR 后

在“ERROR 后”阶段，关键观察点是：sendTuples 为真时，dest->receiveSlot(slot,dest) 消费这个 slot。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

## 7. 生命周期 / ownership / cleanup

- ExecProcNode 协议创建于各节点 ExecInit*()，存活到 ExecEndNode()。
- slot 通常由节点 init 阶段放入 estate->es_tupleTable，由 ExecEndPlan() 统一清理。
- 每个返回 slot 的内容只保证在上层按协议使用期间有效，不能随意缓存底层 tuple 指针。
- per-output-tuple context 每个顶层 tuple 重置一次，表达式结果需要跨轮保存时必须 materialize。
- ExecShutdownNode() 不是最终释放，它只是让异步或并行资源尽早停止。

用 ownership 语言重写本节：

| 问题 | 判断方式 |
| --- | --- |
| 谁创建 | 看入口函数和 memory context 切换点。 |
| 谁持有 | 看 QueryDesc、EState、PlanState 或 slot 列表中的指针。 |
| 谁释放 | 看 ExecutorFinish、ExecutorEnd、ExecEndNode、ExecResetTupleTable 或 ResourceOwner。 |
| ERROR 怎么办 | 看资源是否有 ResourceOwner、MemoryContext、snapshot register 或 callback 兜底。 |
| 能否跨边界保存 | 看对象是否 backend-local、是否依赖 slot 内容、buffer pin 或 per-tuple context。 |

## 8. 正确性机制层次

- TupIsNull(slot) 是 next-tuple 协议的 EOF 标记。
- chgParam 触发 ExecReScan，保证参数化子计划不会继续读旧参数下的状态。
- junk filter 只在顶层 SELECT 输出前执行，不改变内部节点使用的 row identity。
- DestReceiver 返回 false 是合法停止条件，不能继续向关闭的输出端发送。
- parallel mode 的进入和退出包住完整 ExecutePlan，而不是单个节点随意开关。
- ExecShutdownNode 只在不需要 backward 时提前执行，避免破坏可倒扫或可重读需求。

这些机制互相补位。snapshot 解决可见性，不解决内存生命周期；MemoryContext 解决内存批量释放，不解决 buffer pin；函数指针解决分派成本，不解决节点协议正确性。

| 层次 | 保证 | 不能误解为 |
| --- | --- | --- |
| snapshot / command id | tuple 可见性和写入标记边界 | 并发互斥或资源释放 |
| PlanState / slot 协议 | 节点间传递和推进方式一致 | 所有节点都非阻塞 |
| MemoryContext | backend-local 内存归属清楚 | 外部资源自动释放 |
| ResourceOwner / pin / refcount | 外部资源不会被过早回收 | 对象语义仍然新鲜 |
| hook / instrumentation | 观测或扩展入口 | 可以任意改变核心语义 |

## 9. 错误路径 / 异常路径 / fallback

- 阻塞节点如 Sort 仍然遵守 ExecProcNode 协议，但第一次拉取可能先消费全部输入。
- ModifyTable 可能为 RETURNING 返回 tuple，也可能执行副作用后返回空 slot。
- MultiExecProcNode 用于 bitmap/hash 等非 tuple-by-tuple 节点，不走同样的 tuple 计数。
- 输出端停止不等于事务取消，它只是本次 run 的消费边界停止。
- ERROR 路径不会逐层返回，资源清理要依赖 executor End、ResourceOwner 和 memory context。

PostgreSQL executor 的异常路径不能按 C++ 析构模型理解。`ereport(ERROR)` 可能 longjmp 到上层，普通 C 栈上的清理代码不会自然逐层执行。

因此本节所有状态都要问一句：如果这个点 ERROR，已经注册的资源由谁释放，尚未注册的资源是否会泄漏或变成悬挂引用。

## 10. 成本、资源与跨模块传播

- tuple-at-a-time 降低峰值内存，但增加函数指针调用和节点间 slot 协议成本。
- per-tuple reset 把表达式泄漏控制在小 context 中，但每行都有 reset 成本。
- 虚拟 slot 避免构造物理 tuple，直到 materialize 或输出需要时才复制。
- Sort/Agg/Hash 等节点把成本从每次拉取平摊到 build 阶段。
- instrumentation wrapper 会给每次 ExecProcNode 调用增加计时与 row count 成本。

executor 的成本常常不是单个函数慢，而是边界次数被 workload 放大：每行、每节点、每次 FETCH、每个 trigger、每个 slot clear，都可能成为可见成本。

| 放大因素 | 典型表现 |
| --- | --- |
| tuple 数 | ExecProcNode、slot deform、per-tuple reset 和 receiver 调用被放大。 |
| plan node 数 | ExecInitNode、ExecEndNode、instrumentation 对象和 EXPLAIN 输出被放大。 |
| relation 数 | range table relation 打开、ResultRelInfo、锁和权限检查被放大。 |
| FETCH 次数 | ExecutorRun 边界、DestReceiver startup/shutdown 和 Portal 状态切换被放大。 |
| 副作用数量 | trigger、constraint、FDW flush、ModifyTable drain 被放大。 |

## 11. 观测与诊断入口

- gdb 断在 ExecutePlan 循环内可以看到 slot 从 root 节点返回。
- 断在 ExecProcNodeFirst() 可以观察首调用后函数指针被替换。
- perf 火焰图中 ExecProcNode、ExecScan、slot_deform_heap_tuple 常出现在 CPU 热点。
- EXPLAIN ANALYZE 的 actual rows 来自节点 instrumentation，而不是 ExecutePlan 的 es_processed。
- DestReceiver 慢时，执行器栈和输出栈会交替出现在 profile 中。

建议的断点顺序：

1. `ExecutePlan`
2. `ExecProcNode`
3. `ExecProcNodeFirst`
4. `ExecSetExecProcNode`
5. `ExecScan`
6. `ExecSeqScan`
7. `ExecSort`
8. `ExecModifyTable`

断点不只是为了停住程序。每个断点都应该带一个问题：这个状态是否已经初始化，owner 是谁，下一步是否会被改写。

## 12. 课堂实验

### 实验 1

执行 SELECT * FROM generate_series(1,5)，在 ExecutePlan 中打印 current_tuple_count。

观察点：
- 断点前后同一个字段是否发生变化。
- 这个变化是否发生在本节主流程上。
- 如果把 SQL 改成 cursor、EXPLAIN、DML 或带 trigger，边界是否改变。

### 实验 2

执行 SELECT * FROM t ORDER BY a，观察 ExecSort 首次返回 tuple 前会先读完输入。

观察点：
- 断点前后同一个字段是否发生变化。
- 这个变化是否发生在本节主流程上。
- 如果把 SQL 改成 cursor、EXPLAIN、DML 或带 trigger，边界是否改变。

### 实验 3

执行 SELECT ctid,* FROM t WHERE a=1，观察 junk filter 是否存在取决于顶层 resjunk。

观察点：
- 断点前后同一个字段是否发生变化。
- 这个变化是否发生在本节主流程上。
- 如果把 SQL 改成 cursor、EXPLAIN、DML 或带 trigger，边界是否改变。

### 实验 4

对带 RETURNING 的 UPDATE 打断点到 ExecModifyTable，观察它如何返回 slot。

观察点：
- 断点前后同一个字段是否发生变化。
- 这个变化是否发生在本节主流程上。
- 如果把 SQL 改成 cursor、EXPLAIN、DML 或带 trigger，边界是否改变。

### 实验 5

开启 EXPLAIN ANALYZE，观察 ExecProcNodeFirst 选择 ExecProcNodeInstr。

观察点：
- 断点前后同一个字段是否发生变化。
- 这个变化是否发生在本节主流程上。
- 如果把 SQL 改成 cursor、EXPLAIN、DML 或带 trigger，边界是否改变。

### 实验 6

用 cursor FETCH 1 多次执行，观察 PlanState 状态跨 ExecutePlan 调用保留。

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

### 练习 1：`ExecutePlan`

在 `/home/highgo/postgres` 中定位 `ExecutePlan`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

### 练习 2：`ExecProcNode`

在 `/home/highgo/postgres` 中定位 `ExecProcNode`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

### 练习 3：`ExecProcNodeFirst`

在 `/home/highgo/postgres` 中定位 `ExecProcNodeFirst`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

### 练习 4：`ExecSetExecProcNode`

在 `/home/highgo/postgres` 中定位 `ExecSetExecProcNode`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

### 练习 5：`ExecScan`

在 `/home/highgo/postgres` 中定位 `ExecScan`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

### 练习 6：`ExecSeqScan`

在 `/home/highgo/postgres` 中定位 `ExecSeqScan`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

### 练习 7：`ExecSort`

在 `/home/highgo/postgres` 中定位 `ExecSort`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

### 练习 8：`ExecModifyTable`

在 `/home/highgo/postgres` 中定位 `ExecModifyTable`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

## 14. 常见误区

- 不要把 demand-driven 理解成所有节点都非阻塞；它只是统一外部协议。
- 不要缓存从 slot 中拿到的 pass-by-reference Datum，除非确认生命周期或已经复制。
- 不要把 ExecShutdownNode 当作 ExecEndNode；前者停止活动，后者释放节点资源。
- 不要认为 ExecutePlan 知道每类节点细节；节点细节藏在 ExecProcNode 函数指针后。
- 不要把 current_tuple_count 和节点 actual rows 混为一谈。

这些误区的共同点是把 runtime 状态当成静态结构，或者把静态计划当成可以直接推进的状态机。

## 15. 讨论题

1. tuple-at-a-time 协议为什么适合嵌套循环和过滤器，却会让表达式调用成本更显眼？
2. 阻塞节点如何在不改变父节点协议的情况下表达“先吃完输入再吐输出”？
3. 如果没有 TupleTableSlot，节点间传递 HeapTuple 会引入哪些 ownership 问题？
4. 哪些执行器指标来自顶层循环，哪些必须由节点 instrumentation 记录？

讨论时要尽量回到一个具体 runtime 现象，比如一次 FETCH、一次 EXPLAIN ANALYZE、一个 trigger 或一个 slot materialize，而不是停留在抽象偏好。

## 16. 本节小结

本节唯一主问题是：为什么大多数执行节点通过 ExecProcNode() 一次返回一个 TupleTableSlot，而不是由父节点一次性要求子树把所有结果 materialize 出来？

可以带走的一句话模型是：ExecutePlan() 是顶层拉取循环；它反复重置 per-output-tuple context，调用 root ExecProcNode()，处理 junk filter 和 DestReceiver，直到 EOF、count 达到或输出端停止。

- 源码阅读应先找状态创建点，再找推进点，最后找 cleanup。
- 任何字段都要和 owner、生命周期、异常路径一起理解。
- 观测数据只有能回到具体 runtime 边界时，才适合用于诊断。

下一节会看为什么 run 完顶层 tuple 后还需要 ExecutorFinish 单独 drain 副作用。

## 17. 从现象回源码的诊断案例

下面的案例都只服务本节主问题。它们的共同读法是先看现象，再找 runtime 边界，最后回到 owner 和 cleanup。

### 案例 1：第一次执行停在入口但状态还没有建立

现象：

- 你在 `ExecutorRun` 附近停住，看到 SQL 还没有返回，但某些 executor 状态已经改变。
- 这时不要急着沿调用栈往下追完整执行，而要先回到本节主问题：为什么大多数执行节点通过 ExecProcNode() 一次返回一个 TupleTableSlot，而不是由父节点一次性要求子树把所有结果 materialize 出来？
- 如果现象来自 EXPLAIN、cursor、trigger 或扩展 hook，要先确认它落在哪个 executor 生命周期边界。

源码回看：

- 入口函数：`ExecutorRun`。
- 本步主线：ExecutePlan() 先把 direction 写入 estate->es_direction。
- 关键状态：`PlanState->ExecProcNode`，语义是：当前节点的 next-tuple 函数指针。
- 相邻状态：`PlanState->chgParam`，用来判断这一步之后谁继续持有资源。

判断方式：

- 如果字段刚被写入但还没有注册到 owner，异常路径通常只能靠当前函数局部清理。
- 如果状态已经挂到 EState、PlanState、slot table 或 snapshot manager，后续 cleanup 应该能从 owner 找回。
- 如果这个状态只影响观测，不影响语义，诊断时不要把它解释成执行计划行为改变。
- 如果这个状态影响 tuple 生命周期，必须同时检查 slot、TupleDesc、buffer pin 或 per-tuple context。

### 案例 2：查询只返回少量行但内部状态已经推进

现象：

- 你在 `ExecutorRun` 附近停住，看到 SQL 还没有返回，但某些 executor 状态已经改变。
- 这时不要急着沿调用栈往下追完整执行，而要先回到本节主问题：为什么大多数执行节点通过 ExecProcNode() 一次返回一个 TupleTableSlot，而不是由父节点一次性要求子树把所有结果 materialize 出来？
- 如果现象来自 EXPLAIN、cursor、trigger 或扩展 hook，要先确认它落在哪个 executor 生命周期边界。

源码回看：

- 入口函数：`ExecutorRun`。
- 本步主线：它根据 already_executed 和 numberTuples 判断本次是否可以使用 parallel mode。
- 关键状态：`PlanState->chgParam`，语义是：参数变化触发的 rescan 边界，ExecProcNode inline 会先检查。
- 相邻状态：`TupleTableSlot`，用来判断这一步之后谁继续持有资源。

判断方式：

- 如果字段刚被写入但还没有注册到 owner，异常路径通常只能靠当前函数局部清理。
- 如果状态已经挂到 EState、PlanState、slot table 或 snapshot manager，后续 cleanup 应该能从 owner 找回。
- 如果这个状态只影响观测，不影响语义，诊断时不要把它解释成执行计划行为改变。
- 如果这个状态影响 tuple 生命周期，必须同时检查 slot、TupleDesc、buffer pin 或 per-tuple context。

### 案例 3：EXPLAIN ANALYZE 时间与节点时间不一致

现象：

- 你在 `ExecutorRun` 附近停住，看到 SQL 还没有返回，但某些 executor 状态已经改变。
- 这时不要急着沿调用栈往下追完整执行，而要先回到本节主问题：为什么大多数执行节点通过 ExecProcNode() 一次返回一个 TupleTableSlot，而不是由父节点一次性要求子树把所有结果 materialize 出来？
- 如果现象来自 EXPLAIN、cursor、trigger 或扩展 hook，要先确认它落在哪个 executor 生命周期边界。

源码回看：

- 入口函数：`ExecutorRun`。
- 本步主线：进入循环前 current_tuple_count 为 0。
- 关键状态：`TupleTableSlot`，语义是：节点之间传递的 tuple 容器；可能是 virtual、heap、minimal 或 buffer-backed。
- 相邻状态：`es_direction`，用来判断这一步之后谁继续持有资源。

判断方式：

- 如果字段刚被写入但还没有注册到 owner，异常路径通常只能靠当前函数局部清理。
- 如果状态已经挂到 EState、PlanState、slot table 或 snapshot manager，后续 cleanup 应该能从 owner 找回。
- 如果这个状态只影响观测，不影响语义，诊断时不要把它解释成执行计划行为改变。
- 如果这个状态影响 tuple 生命周期，必须同时检查 slot、TupleDesc、buffer pin 或 per-tuple context。

### 案例 4：cursor 或 receiver 让执行被分成多段

现象：

- 你在 `ExecutorRun` 附近停住，看到 SQL 还没有返回，但某些 executor 状态已经改变。
- 这时不要急着沿调用栈往下追完整执行，而要先回到本节主问题：为什么大多数执行节点通过 ExecProcNode() 一次返回一个 TupleTableSlot，而不是由父节点一次性要求子树把所有结果 materialize 出来？
- 如果现象来自 EXPLAIN、cursor、trigger 或扩展 hook，要先确认它落在哪个 executor 生命周期边界。

源码回看：

- 入口函数：`ExecutorRun`。
- 本步主线：每轮先 ResetPerTupleExprContext(estate)，清理顶层输出 tuple 的临时表达式内存。
- 关键状态：`es_direction`，语义是：本次 ExecutePlan 的扫描方向。
- 相邻状态：`es_per_tuple_exprcontext`，用来判断这一步之后谁继续持有资源。

判断方式：

- 如果字段刚被写入但还没有注册到 owner，异常路径通常只能靠当前函数局部清理。
- 如果状态已经挂到 EState、PlanState、slot table 或 snapshot manager，后续 cleanup 应该能从 owner 找回。
- 如果这个状态只影响观测，不影响语义，诊断时不要把它解释成执行计划行为改变。
- 如果这个状态影响 tuple 生命周期，必须同时检查 slot、TupleDesc、buffer pin 或 per-tuple context。

### 案例 5：ERROR 发生后没有按普通调用栈返回

现象：

- 你在 `ExecutorRun` 附近停住，看到 SQL 还没有返回，但某些 executor 状态已经改变。
- 这时不要急着沿调用栈往下追完整执行，而要先回到本节主问题：为什么大多数执行节点通过 ExecProcNode() 一次返回一个 TupleTableSlot，而不是由父节点一次性要求子树把所有结果 materialize 出来？
- 如果现象来自 EXPLAIN、cursor、trigger 或扩展 hook，要先确认它落在哪个 executor 生命周期边界。

源码回看：

- 入口函数：`ExecutorRun`。
- 本步主线：root ExecProcNode(planstate) 被调用，父节点向子节点继续拉取，直到某个叶子节点生产 slot。
- 关键状态：`es_per_tuple_exprcontext`，语义是：顶层每个输出 tuple 重置的短生命周期内存。
- 相邻状态：`es_junkFilter`，用来判断这一步之后谁继续持有资源。

判断方式：

- 如果字段刚被写入但还没有注册到 owner，异常路径通常只能靠当前函数局部清理。
- 如果状态已经挂到 EState、PlanState、slot table 或 snapshot manager，后续 cleanup 应该能从 owner 找回。
- 如果这个状态只影响观测，不影响语义，诊断时不要把它解释成执行计划行为改变。
- 如果这个状态影响 tuple 生命周期，必须同时检查 slot、TupleDesc、buffer pin 或 per-tuple context。

### 案例 6：扩展 hook 看到的边界与核心 executor 边界不同

现象：

- 你在 `ExecutorRun` 附近停住，看到 SQL 还没有返回，但某些 executor 状态已经改变。
- 这时不要急着沿调用栈往下追完整执行，而要先回到本节主问题：为什么大多数执行节点通过 ExecProcNode() 一次返回一个 TupleTableSlot，而不是由父节点一次性要求子树把所有结果 materialize 出来？
- 如果现象来自 EXPLAIN、cursor、trigger 或扩展 hook，要先确认它落在哪个 executor 生命周期边界。

源码回看：

- 入口函数：`ExecutorRun`。
- 本步主线：slot 为空表示 EOF，顶层循环结束。
- 关键状态：`es_junkFilter`，语义是：顶层 SELECT 去掉 resjunk 属性的投影边界。
- 相邻状态：`current_tuple_count`，用来判断这一步之后谁继续持有资源。

判断方式：

- 如果字段刚被写入但还没有注册到 owner，异常路径通常只能靠当前函数局部清理。
- 如果状态已经挂到 EState、PlanState、slot table 或 snapshot manager，后续 cleanup 应该能从 owner 找回。
- 如果这个状态只影响观测，不影响语义，诊断时不要把它解释成执行计划行为改变。
- 如果这个状态影响 tuple 生命周期，必须同时检查 slot、TupleDesc、buffer pin 或 per-tuple context。

## 18. 源码断点与字段记录表

下面这张表用于课堂现场跟读。每一行都应该实际在 gdb 中停一次，记录进入前和返回后的字段差异。

| 序号 | 断点 | 进入前看什么 | 返回后看什么 | 常见误判 |
| --- | --- | --- | --- | --- |
| 1 | `ExecutorRun` | `PlanState->ExecProcNode` 是否已经有效 | ExecutePlan() 先把 direction 写入 estate->es_direction。 | 把字段存在误认为语义已经完成 |
| 2 | `ExecutorRun` | `PlanState->chgParam` 是否已经有效 | 它根据 already_executed 和 numberTuples 判断本次是否可以使用 parallel mode。 | 把字段存在误认为语义已经完成 |
| 3 | `ExecutorRun` | `TupleTableSlot` 是否已经有效 | 进入循环前 current_tuple_count 为 0。 | 把字段存在误认为语义已经完成 |
| 4 | `ExecutorRun` | `es_direction` 是否已经有效 | 每轮先 ResetPerTupleExprContext(estate)，清理顶层输出 tuple 的临时表达式内存。 | 把字段存在误认为语义已经完成 |
| 5 | `ExecutorRun` | `es_per_tuple_exprcontext` 是否已经有效 | root ExecProcNode(planstate) 被调用，父节点向子节点继续拉取，直到某个叶子节点生产 slot。 | 把字段存在误认为语义已经完成 |
| 6 | `ExecutorRun` | `es_junkFilter` 是否已经有效 | slot 为空表示 EOF，顶层循环结束。 | 把字段存在误认为语义已经完成 |
| 7 | `ExecutorRun` | `current_tuple_count` 是否已经有效 | 如果有 junk filter，ExecFilterJunk() 把内部 tuple 形状投影成客户端可见形状。 | 把字段存在误认为语义已经完成 |
| 8 | `ExecutorRun` | `parallel mode flag` 是否已经有效 | sendTuples 为真时，dest->receiveSlot(slot,dest) 消费这个 slot。 | 把字段存在误认为语义已经完成 |
| 9 | `ExecutorRun` | `PlanState->ExecProcNode` 是否已经有效 | receiveSlot 返回 false 时，输出端停止，ExecutePlan 不再继续拉取。 | 把字段存在误认为语义已经完成 |
| 10 | `ExecutorRun` | `PlanState->chgParam` 是否已经有效 | SELECT 顶层计数增加 es_processed，DML 则由 ModifyTable 自己计数。 | 把字段存在误认为语义已经完成 |
| 11 | `ExecutorRun` | `TupleTableSlot` 是否已经有效 | numberTuples 非 0 且达到上限时停止。 | 把字段存在误认为语义已经完成 |
| 12 | `ExecutorRun` | `es_direction` 是否已经有效 | 不需要 backward 时，ExecShutdownNode(planstate) 尝试尽早停止异步资源消费。 | 把字段存在误认为语义已经完成 |

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
- 只观察一个状态变化：ExecutePlan() 先把 direction 写入 estate->es_direction。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 2：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：它根据 already_executed 和 numberTuples 判断本次是否可以使用 parallel mode。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 3：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：进入循环前 current_tuple_count 为 0。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 4：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：每轮先 ResetPerTupleExprContext(estate)，清理顶层输出 tuple 的临时表达式内存。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 5：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：root ExecProcNode(planstate) 被调用，父节点向子节点继续拉取，直到某个叶子节点生产 slot。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 6：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：slot 为空表示 EOF，顶层循环结束。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 7：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：如果有 junk filter，ExecFilterJunk() 把内部 tuple 形状投影成客户端可见形状。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 8：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：sendTuples 为真时，dest->receiveSlot(slot,dest) 消费这个 slot。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 9：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：receiveSlot 返回 false 时，输出端停止，ExecutePlan 不再继续拉取。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 10：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：SELECT 顶层计数增加 es_processed，DML 则由 ModifyTable 自己计数。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 11：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：numberTuples 非 0 且达到上限时停止。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 12：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：不需要 backward 时，ExecShutdownNode(planstate) 尝试尽早停止异步资源消费。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 13：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：ExecutePlan() 先把 direction 写入 estate->es_direction。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 14：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：它根据 already_executed 和 numberTuples 判断本次是否可以使用 parallel mode。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 15：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：进入循环前 current_tuple_count 为 0。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 16：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：每轮先 ResetPerTupleExprContext(estate)，清理顶层输出 tuple 的临时表达式内存。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 17：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：root ExecProcNode(planstate) 被调用，父节点向子节点继续拉取，直到某个叶子节点生产 slot。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 18：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：slot 为空表示 EOF，顶层循环结束。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 19：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：如果有 junk filter，ExecFilterJunk() 把内部 tuple 形状投影成客户端可见形状。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 20：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：sendTuples 为真时，dest->receiveSlot(slot,dest) 消费这个 slot。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 21：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：receiveSlot 返回 false 时，输出端停止，ExecutePlan 不再继续拉取。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 22：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：SELECT 顶层计数增加 es_processed，DML 则由 ModifyTable 自己计数。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 23：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：numberTuples 非 0 且达到上限时停止。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 24：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：不需要 backward 时，ExecShutdownNode(planstate) 尝试尽早停止异步资源消费。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 25：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：ExecutePlan() 先把 direction 写入 estate->es_direction。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 26：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：它根据 already_executed 和 numberTuples 判断本次是否可以使用 parallel mode。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 27：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：进入循环前 current_tuple_count 为 0。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。
## 20. 一页纸定位模板

这部分用于把课程内容压缩成排查动作。不要从这里开始背函数名；先带着现象进入，再沿状态边界回源码。

### 定位 1：执行还没返回，但 gdb 已经停在 executor 内部

- 先停在 `ExecutePlan`，确认当前调用属于 `PostgreSQL ExecutePlan 与 demand-driven tuple 拉取模型` 的哪一个生命周期边界。
- 打印 `PlanState->ExecProcNode`，不要只看字段值，还要解释它的语义：当前节点的 next-tuple 函数指针。
- 向上追一层调用者，确认是谁要求 executor 进入这个边界。
- 向下追一层被调用者，确认这一步是否注册了资源、改写了状态或返回了 slot。
- 如果现象来自观测指标，判断指标采样是在进入本步之前、执行过程中，还是返回之后。
- 如果现象来自异常路径，判断资源是否已经挂到 owner；没有 owner 的对象必须由当前路径处理。

### 定位 2：EXPLAIN ANALYZE 的时间和直觉不一致

- 先停在 `ExecProcNode`，确认当前调用属于 `PostgreSQL ExecutePlan 与 demand-driven tuple 拉取模型` 的哪一个生命周期边界。
- 打印 `PlanState->chgParam`，不要只看字段值，还要解释它的语义：参数变化触发的 rescan 边界，ExecProcNode inline 会先检查。
- 向上追一层调用者，确认是谁要求 executor 进入这个边界。
- 向下追一层被调用者，确认这一步是否注册了资源、改写了状态或返回了 slot。
- 如果现象来自观测指标，判断指标采样是在进入本步之前、执行过程中，还是返回之后。
- 如果现象来自异常路径，判断资源是否已经挂到 owner；没有 owner 的对象必须由当前路径处理。

### 定位 3：cursor、FETCH 或客户端输出让执行分段

- 先停在 `ExecProcNodeFirst`，确认当前调用属于 `PostgreSQL ExecutePlan 与 demand-driven tuple 拉取模型` 的哪一个生命周期边界。
- 打印 `TupleTableSlot`，不要只看字段值，还要解释它的语义：节点之间传递的 tuple 容器；可能是 virtual、heap、minimal 或 buffer-backed。
- 向上追一层调用者，确认是谁要求 executor 进入这个边界。
- 向下追一层被调用者，确认这一步是否注册了资源、改写了状态或返回了 slot。
- 如果现象来自观测指标，判断指标采样是在进入本步之前、执行过程中，还是返回之后。
- 如果现象来自异常路径，判断资源是否已经挂到 owner；没有 owner 的对象必须由当前路径处理。

### 定位 4：ERROR 后怀疑 executor 资源没有按预期释放

- 先停在 `ExecSetExecProcNode`，确认当前调用属于 `PostgreSQL ExecutePlan 与 demand-driven tuple 拉取模型` 的哪一个生命周期边界。
- 打印 `es_direction`，不要只看字段值，还要解释它的语义：本次 ExecutePlan 的扫描方向。
- 向上追一层调用者，确认是谁要求 executor 进入这个边界。
- 向下追一层被调用者，确认这一步是否注册了资源、改写了状态或返回了 slot。
- 如果现象来自观测指标，判断指标采样是在进入本步之前、执行过程中，还是返回之后。
- 如果现象来自异常路径，判断资源是否已经挂到 owner；没有 owner 的对象必须由当前路径处理。

### 定位 5：扩展 hook 或 profiler 捕获到一个边界事件

- 先停在 `ExecScan`，确认当前调用属于 `PostgreSQL ExecutePlan 与 demand-driven tuple 拉取模型` 的哪一个生命周期边界。
- 打印 `es_per_tuple_exprcontext`，不要只看字段值，还要解释它的语义：顶层每个输出 tuple 重置的短生命周期内存。
- 向上追一层调用者，确认是谁要求 executor 进入这个边界。
- 向下追一层被调用者，确认这一步是否注册了资源、改写了状态或返回了 slot。
- 如果现象来自观测指标，判断指标采样是在进入本步之前、执行过程中，还是返回之后。
- 如果现象来自异常路径，判断资源是否已经挂到 owner；没有 owner 的对象必须由当前路径处理。

### 定位 6：源码阅读时分不清字段属于 Plan 还是 runtime state

- 先停在 `ExecSeqScan`，确认当前调用属于 `PostgreSQL ExecutePlan 与 demand-driven tuple 拉取模型` 的哪一个生命周期边界。
- 打印 `es_junkFilter`，不要只看字段值，还要解释它的语义：顶层 SELECT 去掉 resjunk 属性的投影边界。
- 向上追一层调用者，确认是谁要求 executor 进入这个边界。
- 向下追一层被调用者，确认这一步是否注册了资源、改写了状态或返回了 slot。
- 如果现象来自观测指标，判断指标采样是在进入本步之前、执行过程中，还是返回之后。
- 如果现象来自异常路径，判断资源是否已经挂到 owner；没有 owner 的对象必须由当前路径处理。

定位结束时只保留三句话：

1. 这个现象落在哪个 executor 生命周期边界。
2. 这个边界改变了哪个核心状态，以及 owner 是谁。
3. 正常路径和 ERROR 路径分别由哪个函数或机制收尾。
