# PostgreSQL ExecutorRun 的方向、计数与 Portal 边界

## 课程定位

前置知识：已经读过 executor 生命周期的目录规划，并理解 PostgreSQL 的查询执行由 Portal、QueryDesc、EState、PlanState 和 TupleTableSlot 共同完成。上一节已经把 QueryDesc 启动成可运行的 EState 和 PlanState tree。

本节唯一主问题：

```text
ExecutorRun() 为什么要接受 direction 和 count，并通过 DestReceiver 与 Portal 协作，而不是只提供一个“把计划跑完”的接口？
```

核心矛盾：executor 节点希望按统一 tuple 流运行 vs SQL 游标、FETCH、客户端背压和目标接收器要求分批、可中止、可改变方向的外层协议。

学完后应能判断：能区分 executor 的 tuple 生产协议、Portal 的消费协议和 DestReceiver 的输出协议。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

上一节已经把 QueryDesc 启动成可运行的 EState 和 PlanState tree。

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

下一节会把 ExecutorRun 内部的 ExecutePlan 拉取循环拆开。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
Portal 决定本次要向前还是向后取多少行；ExecutorRun() 把这个请求写入 EState，启动 DestReceiver，然后让 ExecutePlan() 拉取不超过 count 个顶层 tuple。
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
| 1 | src/backend/executor/execMain.c | ExecutorRun() / standard_ExecutorRun() / ExecutePlan()：方向、计数、DestReceiver 主线。 |
| 2 | src/backend/tcop/pquery.c | PortalRun() / PortalRunSelect() / PortalRunFetch()：Portal 如何多次调用 ExecutorRun。 |
| 3 | src/include/tcop/pquery.h | PortalRun / PortalRunFetch 对外声明。 |
| 4 | src/include/executor/execdesc.h | QueryDesc 中 dest、tupDesc、estate、planstate 的调用者边界。 |
| 5 | src/include/tcop/dest.h | DestReceiver 的 rStartup / receiveSlot / rShutdown 协议。 |
| 6 | src/include/nodes/execnodes.h | EState 的 es_direction、es_processed、es_total_processed。 |
| 7 | src/backend/commands/prepare.c | EXECUTE count 如何向 PortalRun 传播。 |
| 8 | src/backend/commands/portalcmds.c | FETCH / MOVE 如何映射为 PortalRunFetch。 |

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
| Portal | 外层可暂停的执行对象，保存位置、状态、QueryDesc 和调用约束。 |
| QueryDesc.dest | 本次输出的接收器；可能是客户端、SPI、tuplestore、EXPLAIN 或 None_Receiver。 |
| ScanDirection | 本次拉取方向，写入 estate->es_direction，供节点和 access method 解释。 |
| count | 顶层返回 tuple 上限；0 表示本次 run to completion。 |
| es_processed | 单次 ExecutorRun 的顶层处理行数。 |
| es_total_processed | 同一 executor 多次 Run 的累计顶层行数。 |
| already_executed | QueryDesc 标记，用于并行模式和部分执行边界。 |
| DestReceiver | 输出协议对象，它可以接收 tuple，也可以提前返回 false 终止发送。 |

本节最容易混淆的边界有四个：

- planner-time 描述与 executor-time 状态不同。
- backend-local 指针与跨 backend 可见状态不同。
- memory context 释放内存，不等于释放所有外部资源。
- 顶层调用边界与节点内部状态机边界不同。

读源码时要持续追问：这个状态现在是谁的，下一步谁会改变它，如果中途 ERROR，谁会兜底。

## 5. 主流程源码 walkthrough

下面按时间顺序读主链路。每一步都回答“状态发生了什么变化”，而不是只记住函数名。

### 5.1. 步骤 1

PortalRun() 根据 portal strategy 判断是单个 SELECT、utility，还是多语句 portal。

阅读这一段时的重点不是记住第 1 个函数名，而是确认状态边界：读到这里先确认这一步是在创建状态、注册外部资源，还是只是在写入控制字段。

### 5.2. 步骤 2

PortalRunSelect() 把 forward/count 翻译成 ScanDirection 和 ExecutorRun() 参数。

阅读这一段时的重点不是记住第 2 个函数名，而是确认状态边界：断点停在这里时，优先打印 owner 指针，再打印本步刚写入的字段。

### 5.3. 步骤 3

standard_ExecutorRun() 断言 estate 已经存在，并确认不是 EXPLAIN-only 执行。

阅读这一段时的重点不是记住第 3 个函数名，而是确认状态边界：这一步如果提前失败，后续状态还没有 owner，不能按完整 executor cleanup 想象。

### 5.4. 步骤 4

run 期间切入 estate->es_query_cxt，避免输出路径和节点路径把 per-query 对象分配到错误 context。

阅读这一段时的重点不是记住第 4 个函数名，而是确认状态边界：这一步如果已经完成，后续代码就可以假定对应 runtime state 已经稳定。

### 5.5. 步骤 5

如果 query_instr 存在，ExecutorRun 整体运行时间在入口和出口被 InstrStart/InstrStop 包住。

阅读这一段时的重点不是记住第 5 个函数名，而是确认状态边界：这里要区分“指针存在”和“语义已经注册”两件事。

### 5.6. 步骤 6

根据 operation 和 RETURNING 判断 sendTuples；只有需要向外发 tuple 时才启动 DestReceiver。

阅读这一段时的重点不是记住第 6 个函数名，而是确认状态边界：这里的状态变化通常会影响后续 ExecProcNode、slot、snapshot 或 receiver 的解释。

### 5.7. 步骤 7

NoMovementScanDirection 只启动和关闭 receiver，不继续拉取计划树。

阅读这一段时的重点不是记住第 7 个函数名，而是确认状态边界：如果以后诊断慢查询，这一步更像 setup 成本，不应误归到单个 tuple。

### 5.8. 步骤 8

ExecutePlan() 写 estate->es_direction，并决定是否允许本次使用 parallel mode。

阅读这一段时的重点不是记住第 8 个函数名，而是确认状态边界：如果以后诊断异常退出，这一步要回看是否已经进入 ResourceOwner、MemoryContext 或 snapshot 注册边界。

### 5.9. 步骤 9

每次返回 tuple 后，顶层 SELECT 才增加 estate->es_processed；DML 行数由 ModifyTable 节点负责。

阅读这一段时的重点不是记住第 9 个函数名，而是确认状态边界：这一步的调用者协议比函数内部细节更重要：调用者是否允许重复进入，是否允许部分执行。

### 5.10. 步骤 10

dest->receiveSlot() 返回 false 时，ExecutorRun 提前停止，Portal 可以保留后续状态。

阅读这一段时的重点不是记住第 10 个函数名，而是确认状态边界：这一步之后要看下一个 owner 边界，而不是继续把所有状态当作局部变量。

### 5.11. 步骤 11

本次 run 结束后 es_total_processed 累加 es_processed。

阅读这一段时的重点不是记住第 11 个函数名，而是确认状态边界：如果这一步涉及 hook 或 wrapper，要继续追到标准实现或真实节点函数。

### 5.12. 步骤 12

DestReceiver shutdown 与 query_instr stop 都在 ExecutorRun 边界完成。

阅读这一段时的重点不是记住第 12 个函数名，而是确认状态边界：如果这一步涉及 slot 或 tuple，必须同时确认数据内容和底层资源是否同寿命。

主链路可以压缩成：

```text
01. PortalRun() 根据 portal strategy 判断是单个 SELECT、utility，还是多语句 portal。
02. PortalRunSelect() 把 forward/count 翻译成 ScanDirection 和 ExecutorRun() 参数。
03. standard_ExecutorRun() 断言 estate 已经存在，并确认不是 EXPLAIN-only 执行。
04. run 期间切入 estate->es_query_cxt，避免输出路径和节点路径把 per-query 对象分配到错误 context。
05. 如果 query_instr 存在，ExecutorRun 整体运行时间在入口和出口被 InstrStart/InstrStop 包住。
06. 根据 operation 和 RETURNING 判断 sendTuples；只有需要向外发 tuple 时才启动 DestReceiver。
07. NoMovementScanDirection 只启动和关闭 receiver，不继续拉取计划树。
08. ExecutePlan() 写 estate->es_direction，并决定是否允许本次使用 parallel mode。
09. 每次返回 tuple 后，顶层 SELECT 才增加 estate->es_processed；DML 行数由 ModifyTable 节点负责。
10. dest->receiveSlot() 返回 false 时，ExecutorRun 提前停止，Portal 可以保留后续状态。
11. 本次 run 结束后 es_total_processed 累加 es_processed。
12. DestReceiver shutdown 与 query_instr stop 都在 ExecutorRun 边界完成。
```

## 6. 状态随时间推进的故事

把源码主线换成一个对象的生命周期，会更容易看出本节 tension。

### 6.1. 创建前

在“创建前”阶段，关键观察点是：PortalRun() 根据 portal strategy 判断是单个 SELECT、utility，还是多语句 portal。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

### 6.2. 创建时

在“创建时”阶段，关键观察点是：PortalRunSelect() 把 forward/count 翻译成 ScanDirection 和 ExecutorRun() 参数。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

### 6.3. 第一次使用前

在“第一次使用前”阶段，关键观察点是：standard_ExecutorRun() 断言 estate 已经存在，并确认不是 EXPLAIN-only 执行。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

### 6.4. 正常推进中

在“正常推进中”阶段，关键观察点是：run 期间切入 estate->es_query_cxt，避免输出路径和节点路径把 per-query 对象分配到错误 context。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

### 6.5. 遇到边界时

在“遇到边界时”阶段，关键观察点是：如果 query_instr 存在，ExecutorRun 整体运行时间在入口和出口被 InstrStart/InstrStop 包住。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

### 6.6. 提前停止时

在“提前停止时”阶段，关键观察点是：根据 operation 和 RETURNING 判断 sendTuples；只有需要向外发 tuple 时才启动 DestReceiver。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

### 6.7. 收尾时

在“收尾时”阶段，关键观察点是：NoMovementScanDirection 只启动和关闭 receiver，不继续拉取计划树。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

### 6.8. ERROR 后

在“ERROR 后”阶段，关键观察点是：ExecutePlan() 写 estate->es_direction，并决定是否允许本次使用 parallel mode。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

## 7. 生命周期 / ownership / cleanup

- Portal 创建 QueryDesc 并调用 ExecutorStart()，后续可以多次调用 ExecutorRun()。
- 每次 ExecutorRun 只拥有本次 direction/count/sendTuples 边界，不拥有整个 Portal 生命周期。
- DestReceiver 的 rStartup/rShutdown 包住单次 ExecutorRun，而不是整个 cursor 生命周期。
- 如果 FETCH 分批读取，PlanState tree 和 slot 状态会跨多次 ExecutorRun 保留在 EState 内。
- 最终 Portal 或调用者负责调用 ExecutorFinish() 和 ExecutorEnd()。

用 ownership 语言重写本节：

| 问题 | 判断方式 |
| --- | --- |
| 谁创建 | 看入口函数和 memory context 切换点。 |
| 谁持有 | 看 QueryDesc、EState、PlanState 或 slot 列表中的指针。 |
| 谁释放 | 看 ExecutorFinish、ExecutorEnd、ExecEndNode、ExecResetTupleTable 或 ResourceOwner。 |
| ERROR 怎么办 | 看资源是否有 ResourceOwner、MemoryContext、snapshot register 或 callback 兜底。 |
| 能否跨边界保存 | 看对象是否 backend-local、是否依赖 slot 内容、buffer pin 或 per-tuple context。 |

## 8. 正确性机制层次

- direction 必须写入 EState，而不是作为隐式全局变量散落到节点。
- count 只限制顶层返回 tuple，不保证内部节点或 DML 副作用也只处理同样数量。
- NoMovement 保护已经 EOF 的 executor 不被错误再次驱动。
- DestReceiver 返回 false 被解释为输出端关闭，executor 必须停止向外发送。
- parallel mode 只支持完整执行；部分 count 或已经执行过的 QueryDesc 不能重新走 parallel。
- es_processed 与 es_total_processed 的分离保证 FETCH 多次调用仍能得到本次与累计两个口径。

这些机制互相补位。snapshot 解决可见性，不解决内存生命周期；MemoryContext 解决内存批量释放，不解决 buffer pin；函数指针解决分派成本，不解决节点协议正确性。

| 层次 | 保证 | 不能误解为 |
| --- | --- | --- |
| snapshot / command id | tuple 可见性和写入标记边界 | 并发互斥或资源释放 |
| PlanState / slot 协议 | 节点间传递和推进方式一致 | 所有节点都非阻塞 |
| MemoryContext | backend-local 内存归属清楚 | 外部资源自动释放 |
| ResourceOwner / pin / refcount | 外部资源不会被过早回收 | 对象语义仍然新鲜 |
| hook / instrumentation | 观测或扩展入口 | 可以任意改变核心语义 |

## 9. 错误路径 / 异常路径 / fallback

- EXPLAIN-only QueryDesc 不能进入 standard_ExecutorRun() 的真实执行路径。
- NoMovementScanDirection 不等于空操作，它仍然可能启动和关闭 DestReceiver。
- direction 反向并不意味着所有节点都支持倒扫；是否支持由 eflags 和节点初始化决定。
- 客户端中断或 receiver 关闭可能让计划树只部分推进。
- 部分执行会禁用本次 parallel mode，这是语义保护而不是性能 fallback。

PostgreSQL executor 的异常路径不能按 C++ 析构模型理解。`ereport(ERROR)` 可能 longjmp 到上层，普通 C 栈上的清理代码不会自然逐层执行。

因此本节所有状态都要问一句：如果这个点 ERROR，已经注册的资源由谁释放，尚未注册的资源是否会泄漏或变成悬挂引用。

## 10. 成本、资源与跨模块传播

- 每次 ExecutorRun 都有 receiver startup/shutdown、query_instr 和 per-query context switch 成本。
- count 很小会放大 Portal 与 executor 边界成本。
- count 为 0 能让 executor run to completion，也让并行计划更容易启用。
- DestReceiver 的 receiveSlot 可能成为客户端输出、COPY、SPI 或 tuplestore 的主要成本点。
- 倒向 FETCH 需要下层节点支持 backward 或 materialization，代价常被隐藏在 plan shape 中。

executor 的成本常常不是单个函数慢，而是边界次数被 workload 放大：每行、每节点、每次 FETCH、每个 trigger、每个 slot clear，都可能成为可见成本。

| 放大因素 | 典型表现 |
| --- | --- |
| tuple 数 | ExecProcNode、slot deform、per-tuple reset 和 receiver 调用被放大。 |
| plan node 数 | ExecInitNode、ExecEndNode、instrumentation 对象和 EXPLAIN 输出被放大。 |
| relation 数 | range table relation 打开、ResultRelInfo、锁和权限检查被放大。 |
| FETCH 次数 | ExecutorRun 边界、DestReceiver startup/shutdown 和 Portal 状态切换被放大。 |
| 副作用数量 | trigger、constraint、FDW flush、ModifyTable drain 被放大。 |

## 11. 观测与诊断入口

- gdb 断在 PortalRunSelect() 可以看到 count 如何传给 ExecutorRun()。
- 断在 standard_ExecutorRun() 可以看 direction、count、sendTuples 和 dest。
- EXPLAIN ANALYZE 的 overall query instrumentation 会包住 ExecutorRun 和 ExecutorFinish。
- 游标 FETCH 10 可以观察 es_processed 每次约为 10，而 es_total_processed 逐步累加。
- 客户端慢收数据时，receiveSlot 之后的输出路径可能成为 profiler 热点。

建议的断点顺序：

1. `PortalRun`
2. `PortalRunSelect`
3. `PortalRunFetch`
4. `ExecutorRun`
5. `standard_ExecutorRun`
6. `ExecutePlan`
7. `DestReceiver`
8. `es_total_processed`

断点不只是为了停住程序。每个断点都应该带一个问题：这个状态是否已经初始化，owner 是谁，下一步是否会被改写。

## 12. 课堂实验

### 实验 1

DECLARE c CURSOR FOR SELECT * FROM generate_series(1,100); FETCH 7 FROM c，观察 ExecutorRun count。

观察点：
- 断点前后同一个字段是否发生变化。
- 这个变化是否发生在本节主流程上。
- 如果把 SQL 改成 cursor、EXPLAIN、DML 或带 trigger，边界是否改变。

### 实验 2

FETCH 7 后再 FETCH 7，比较 es_processed 和 es_total_processed。

观察点：
- 断点前后同一个字段是否发生变化。
- 这个变化是否发生在本节主流程上。
- 如果把 SQL 改成 cursor、EXPLAIN、DML 或带 trigger，边界是否改变。

### 实验 3

MOVE FORWARD 10 FROM c，观察 None_Receiver 或不发送 tuple 的路径。

观察点：
- 断点前后同一个字段是否发生变化。
- 这个变化是否发生在本节主流程上。
- 如果把 SQL 改成 cursor、EXPLAIN、DML 或带 trigger，边界是否改变。

### 实验 4

用 EXPLAIN ANALYZE SELECT generate_series(1,100) 对比一次跑完和 cursor 分批。

观察点：
- 断点前后同一个字段是否发生变化。
- 这个变化是否发生在本节主流程上。
- 如果把 SQL 改成 cursor、EXPLAIN、DML 或带 trigger，边界是否改变。

### 实验 5

在 dest->receiveSlot 返回点打断点，确认 tuple 是在 executor 外部输出协议中被消费。

观察点：
- 断点前后同一个字段是否发生变化。
- 这个变化是否发生在本节主流程上。
- 如果把 SQL 改成 cursor、EXPLAIN、DML 或带 trigger，边界是否改变。

### 实验 6

设置 enable_parallel_append 等参数，比较 count=0 和小 count 对 parallel mode 的影响。

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

### 练习 1：`PortalRun`

在 `/home/highgo/postgres` 中定位 `PortalRun`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

### 练习 2：`PortalRunSelect`

在 `/home/highgo/postgres` 中定位 `PortalRunSelect`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

### 练习 3：`PortalRunFetch`

在 `/home/highgo/postgres` 中定位 `PortalRunFetch`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

### 练习 4：`ExecutorRun`

在 `/home/highgo/postgres` 中定位 `ExecutorRun`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

### 练习 5：`standard_ExecutorRun`

在 `/home/highgo/postgres` 中定位 `standard_ExecutorRun`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

### 练习 6：`ExecutePlan`

在 `/home/highgo/postgres` 中定位 `ExecutePlan`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

### 练习 7：`DestReceiver`

在 `/home/highgo/postgres` 中定位 `DestReceiver`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

### 练习 8：`es_total_processed`

在 `/home/highgo/postgres` 中定位 `es_total_processed`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

## 14. 常见误区

- 不要把 count 理解成扫描节点最多访问多少行；它只限制顶层返回。
- 不要把 Portal 看成 executor 内部对象；它是 tcop/pquery 层的执行协议边界。
- 不要认为 receiver shutdown 表示查询结束；它只是本次 ExecutorRun 的输出边界结束。
- 不要把 es_processed 用来解释所有 DML 行数，ModifyTable 有自己的计数责任。
- 不要忽略 NoMovement，它是避免 EOF 后重复驱动 executor 的保护。

这些误区的共同点是把 runtime 状态当成静态结构，或者把静态计划当成可以直接推进的状态机。

## 15. 讨论题

1. 为什么 FETCH 小批量会影响并行计划启用，而不是只影响输出端？
2. 如果 DestReceiver 在收到一半 tuple 后返回 false，哪些状态必须保留到下次 run？
3. Portal 层和 executor 层分别应该负责哪些错误信息？
4. 为什么 ExecutorRun 不直接返回一个 List<Tuple>？

讨论时要尽量回到一个具体 runtime 现象，比如一次 FETCH、一次 EXPLAIN ANALYZE、一个 trigger 或一个 slot materialize，而不是停留在抽象偏好。

## 16. 本节小结

本节唯一主问题是：ExecutorRun() 为什么要接受 direction 和 count，并通过 DestReceiver 与 Portal 协作，而不是只提供一个“把计划跑完”的接口？

可以带走的一句话模型是：Portal 决定本次要向前还是向后取多少行；ExecutorRun() 把这个请求写入 EState，启动 DestReceiver，然后让 ExecutePlan() 拉取不超过 count 个顶层 tuple。

- 源码阅读应先找状态创建点，再找推进点，最后找 cleanup。
- 任何字段都要和 owner、生命周期、异常路径一起理解。
- 观测数据只有能回到具体 runtime 边界时，才适合用于诊断。

下一节会把 ExecutorRun 内部的 ExecutePlan 拉取循环拆开。

## 17. 从现象回源码的诊断案例

下面的案例都只服务本节主问题。它们的共同读法是先看现象，再找 runtime 边界，最后回到 owner 和 cleanup。

### 案例 1：第一次执行停在入口但状态还没有建立

现象：

- 你在 `ExecutorRun` 附近停住，看到 SQL 还没有返回，但某些 executor 状态已经改变。
- 这时不要急着沿调用栈往下追完整执行，而要先回到本节主问题：ExecutorRun() 为什么要接受 direction 和 count，并通过 DestReceiver 与 Portal 协作，而不是只提供一个“把计划跑完”的接口？
- 如果现象来自 EXPLAIN、cursor、trigger 或扩展 hook，要先确认它落在哪个 executor 生命周期边界。

源码回看：

- 入口函数：`ExecutorRun`。
- 本步主线：PortalRun() 根据 portal strategy 判断是单个 SELECT、utility，还是多语句 portal。
- 关键状态：`Portal`，语义是：外层可暂停的执行对象，保存位置、状态、QueryDesc 和调用约束。
- 相邻状态：`QueryDesc.dest`，用来判断这一步之后谁继续持有资源。

判断方式：

- 如果字段刚被写入但还没有注册到 owner，异常路径通常只能靠当前函数局部清理。
- 如果状态已经挂到 EState、PlanState、slot table 或 snapshot manager，后续 cleanup 应该能从 owner 找回。
- 如果这个状态只影响观测，不影响语义，诊断时不要把它解释成执行计划行为改变。
- 如果这个状态影响 tuple 生命周期，必须同时检查 slot、TupleDesc、buffer pin 或 per-tuple context。

### 案例 2：查询只返回少量行但内部状态已经推进

现象：

- 你在 `ExecutorRun` 附近停住，看到 SQL 还没有返回，但某些 executor 状态已经改变。
- 这时不要急着沿调用栈往下追完整执行，而要先回到本节主问题：ExecutorRun() 为什么要接受 direction 和 count，并通过 DestReceiver 与 Portal 协作，而不是只提供一个“把计划跑完”的接口？
- 如果现象来自 EXPLAIN、cursor、trigger 或扩展 hook，要先确认它落在哪个 executor 生命周期边界。

源码回看：

- 入口函数：`ExecutorRun`。
- 本步主线：PortalRunSelect() 把 forward/count 翻译成 ScanDirection 和 ExecutorRun() 参数。
- 关键状态：`QueryDesc.dest`，语义是：本次输出的接收器；可能是客户端、SPI、tuplestore、EXPLAIN 或 None_Receiver。
- 相邻状态：`ScanDirection`，用来判断这一步之后谁继续持有资源。

判断方式：

- 如果字段刚被写入但还没有注册到 owner，异常路径通常只能靠当前函数局部清理。
- 如果状态已经挂到 EState、PlanState、slot table 或 snapshot manager，后续 cleanup 应该能从 owner 找回。
- 如果这个状态只影响观测，不影响语义，诊断时不要把它解释成执行计划行为改变。
- 如果这个状态影响 tuple 生命周期，必须同时检查 slot、TupleDesc、buffer pin 或 per-tuple context。

### 案例 3：EXPLAIN ANALYZE 时间与节点时间不一致

现象：

- 你在 `ExecutorRun` 附近停住，看到 SQL 还没有返回，但某些 executor 状态已经改变。
- 这时不要急着沿调用栈往下追完整执行，而要先回到本节主问题：ExecutorRun() 为什么要接受 direction 和 count，并通过 DestReceiver 与 Portal 协作，而不是只提供一个“把计划跑完”的接口？
- 如果现象来自 EXPLAIN、cursor、trigger 或扩展 hook，要先确认它落在哪个 executor 生命周期边界。

源码回看：

- 入口函数：`ExecutorRun`。
- 本步主线：standard_ExecutorRun() 断言 estate 已经存在，并确认不是 EXPLAIN-only 执行。
- 关键状态：`ScanDirection`，语义是：本次拉取方向，写入 estate->es_direction，供节点和 access method 解释。
- 相邻状态：`count`，用来判断这一步之后谁继续持有资源。

判断方式：

- 如果字段刚被写入但还没有注册到 owner，异常路径通常只能靠当前函数局部清理。
- 如果状态已经挂到 EState、PlanState、slot table 或 snapshot manager，后续 cleanup 应该能从 owner 找回。
- 如果这个状态只影响观测，不影响语义，诊断时不要把它解释成执行计划行为改变。
- 如果这个状态影响 tuple 生命周期，必须同时检查 slot、TupleDesc、buffer pin 或 per-tuple context。

### 案例 4：cursor 或 receiver 让执行被分成多段

现象：

- 你在 `ExecutorRun` 附近停住，看到 SQL 还没有返回，但某些 executor 状态已经改变。
- 这时不要急着沿调用栈往下追完整执行，而要先回到本节主问题：ExecutorRun() 为什么要接受 direction 和 count，并通过 DestReceiver 与 Portal 协作，而不是只提供一个“把计划跑完”的接口？
- 如果现象来自 EXPLAIN、cursor、trigger 或扩展 hook，要先确认它落在哪个 executor 生命周期边界。

源码回看：

- 入口函数：`ExecutorRun`。
- 本步主线：run 期间切入 estate->es_query_cxt，避免输出路径和节点路径把 per-query 对象分配到错误 context。
- 关键状态：`count`，语义是：顶层返回 tuple 上限；0 表示本次 run to completion。
- 相邻状态：`es_processed`，用来判断这一步之后谁继续持有资源。

判断方式：

- 如果字段刚被写入但还没有注册到 owner，异常路径通常只能靠当前函数局部清理。
- 如果状态已经挂到 EState、PlanState、slot table 或 snapshot manager，后续 cleanup 应该能从 owner 找回。
- 如果这个状态只影响观测，不影响语义，诊断时不要把它解释成执行计划行为改变。
- 如果这个状态影响 tuple 生命周期，必须同时检查 slot、TupleDesc、buffer pin 或 per-tuple context。

### 案例 5：ERROR 发生后没有按普通调用栈返回

现象：

- 你在 `ExecutorRun` 附近停住，看到 SQL 还没有返回，但某些 executor 状态已经改变。
- 这时不要急着沿调用栈往下追完整执行，而要先回到本节主问题：ExecutorRun() 为什么要接受 direction 和 count，并通过 DestReceiver 与 Portal 协作，而不是只提供一个“把计划跑完”的接口？
- 如果现象来自 EXPLAIN、cursor、trigger 或扩展 hook，要先确认它落在哪个 executor 生命周期边界。

源码回看：

- 入口函数：`ExecutorRun`。
- 本步主线：如果 query_instr 存在，ExecutorRun 整体运行时间在入口和出口被 InstrStart/InstrStop 包住。
- 关键状态：`es_processed`，语义是：单次 ExecutorRun 的顶层处理行数。
- 相邻状态：`es_total_processed`，用来判断这一步之后谁继续持有资源。

判断方式：

- 如果字段刚被写入但还没有注册到 owner，异常路径通常只能靠当前函数局部清理。
- 如果状态已经挂到 EState、PlanState、slot table 或 snapshot manager，后续 cleanup 应该能从 owner 找回。
- 如果这个状态只影响观测，不影响语义，诊断时不要把它解释成执行计划行为改变。
- 如果这个状态影响 tuple 生命周期，必须同时检查 slot、TupleDesc、buffer pin 或 per-tuple context。

### 案例 6：扩展 hook 看到的边界与核心 executor 边界不同

现象：

- 你在 `ExecutorRun` 附近停住，看到 SQL 还没有返回，但某些 executor 状态已经改变。
- 这时不要急着沿调用栈往下追完整执行，而要先回到本节主问题：ExecutorRun() 为什么要接受 direction 和 count，并通过 DestReceiver 与 Portal 协作，而不是只提供一个“把计划跑完”的接口？
- 如果现象来自 EXPLAIN、cursor、trigger 或扩展 hook，要先确认它落在哪个 executor 生命周期边界。

源码回看：

- 入口函数：`ExecutorRun`。
- 本步主线：根据 operation 和 RETURNING 判断 sendTuples；只有需要向外发 tuple 时才启动 DestReceiver。
- 关键状态：`es_total_processed`，语义是：同一 executor 多次 Run 的累计顶层行数。
- 相邻状态：`already_executed`，用来判断这一步之后谁继续持有资源。

判断方式：

- 如果字段刚被写入但还没有注册到 owner，异常路径通常只能靠当前函数局部清理。
- 如果状态已经挂到 EState、PlanState、slot table 或 snapshot manager，后续 cleanup 应该能从 owner 找回。
- 如果这个状态只影响观测，不影响语义，诊断时不要把它解释成执行计划行为改变。
- 如果这个状态影响 tuple 生命周期，必须同时检查 slot、TupleDesc、buffer pin 或 per-tuple context。

## 18. 源码断点与字段记录表

下面这张表用于课堂现场跟读。每一行都应该实际在 gdb 中停一次，记录进入前和返回后的字段差异。

| 序号 | 断点 | 进入前看什么 | 返回后看什么 | 常见误判 |
| --- | --- | --- | --- | --- |
| 1 | `ExecutorRun` | `Portal` 是否已经有效 | PortalRun() 根据 portal strategy 判断是单个 SELECT、utility，还是多语句 portal。 | 把字段存在误认为语义已经完成 |
| 2 | `ExecutorRun` | `QueryDesc.dest` 是否已经有效 | PortalRunSelect() 把 forward/count 翻译成 ScanDirection 和 ExecutorRun() 参数。 | 把字段存在误认为语义已经完成 |
| 3 | `ExecutorRun` | `ScanDirection` 是否已经有效 | standard_ExecutorRun() 断言 estate 已经存在，并确认不是 EXPLAIN-only 执行。 | 把字段存在误认为语义已经完成 |
| 4 | `ExecutorRun` | `count` 是否已经有效 | run 期间切入 estate->es_query_cxt，避免输出路径和节点路径把 per-query 对象分配到错误 context。 | 把字段存在误认为语义已经完成 |
| 5 | `ExecutorRun` | `es_processed` 是否已经有效 | 如果 query_instr 存在，ExecutorRun 整体运行时间在入口和出口被 InstrStart/InstrStop 包住。 | 把字段存在误认为语义已经完成 |
| 6 | `ExecutorRun` | `es_total_processed` 是否已经有效 | 根据 operation 和 RETURNING 判断 sendTuples；只有需要向外发 tuple 时才启动 DestReceiver。 | 把字段存在误认为语义已经完成 |
| 7 | `ExecutorRun` | `already_executed` 是否已经有效 | NoMovementScanDirection 只启动和关闭 receiver，不继续拉取计划树。 | 把字段存在误认为语义已经完成 |
| 8 | `ExecutorRun` | `DestReceiver` 是否已经有效 | ExecutePlan() 写 estate->es_direction，并决定是否允许本次使用 parallel mode。 | 把字段存在误认为语义已经完成 |
| 9 | `ExecutorRun` | `Portal` 是否已经有效 | 每次返回 tuple 后，顶层 SELECT 才增加 estate->es_processed；DML 行数由 ModifyTable 节点负责。 | 把字段存在误认为语义已经完成 |
| 10 | `ExecutorRun` | `QueryDesc.dest` 是否已经有效 | dest->receiveSlot() 返回 false 时，ExecutorRun 提前停止，Portal 可以保留后续状态。 | 把字段存在误认为语义已经完成 |
| 11 | `ExecutorRun` | `ScanDirection` 是否已经有效 | 本次 run 结束后 es_total_processed 累加 es_processed。 | 把字段存在误认为语义已经完成 |
| 12 | `ExecutorRun` | `count` 是否已经有效 | DestReceiver shutdown 与 query_instr stop 都在 ExecutorRun 边界完成。 | 把字段存在误认为语义已经完成 |

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
- 只观察一个状态变化：PortalRun() 根据 portal strategy 判断是单个 SELECT、utility，还是多语句 portal。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 2：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：PortalRunSelect() 把 forward/count 翻译成 ScanDirection 和 ExecutorRun() 参数。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 3：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：standard_ExecutorRun() 断言 estate 已经存在，并确认不是 EXPLAIN-only 执行。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 4：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：run 期间切入 estate->es_query_cxt，避免输出路径和节点路径把 per-query 对象分配到错误 context。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 5：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：如果 query_instr 存在，ExecutorRun 整体运行时间在入口和出口被 InstrStart/InstrStop 包住。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 6：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：根据 operation 和 RETURNING 判断 sendTuples；只有需要向外发 tuple 时才启动 DestReceiver。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 7：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：NoMovementScanDirection 只启动和关闭 receiver，不继续拉取计划树。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 8：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：ExecutePlan() 写 estate->es_direction，并决定是否允许本次使用 parallel mode。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 9：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：每次返回 tuple 后，顶层 SELECT 才增加 estate->es_processed；DML 行数由 ModifyTable 节点负责。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 10：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：dest->receiveSlot() 返回 false 时，ExecutorRun 提前停止，Portal 可以保留后续状态。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 11：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：本次 run 结束后 es_total_processed 累加 es_processed。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 12：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：DestReceiver shutdown 与 query_instr stop 都在 ExecutorRun 边界完成。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 13：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：PortalRun() 根据 portal strategy 判断是单个 SELECT、utility，还是多语句 portal。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 14：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：PortalRunSelect() 把 forward/count 翻译成 ScanDirection 和 ExecutorRun() 参数。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 15：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：standard_ExecutorRun() 断言 estate 已经存在，并确认不是 EXPLAIN-only 执行。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 16：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：run 期间切入 estate->es_query_cxt，避免输出路径和节点路径把 per-query 对象分配到错误 context。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 17：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：如果 query_instr 存在，ExecutorRun 整体运行时间在入口和出口被 InstrStart/InstrStop 包住。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 18：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：根据 operation 和 RETURNING 判断 sendTuples；只有需要向外发 tuple 时才启动 DestReceiver。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 19：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：NoMovementScanDirection 只启动和关闭 receiver，不继续拉取计划树。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 20：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：ExecutePlan() 写 estate->es_direction，并决定是否允许本次使用 parallel mode。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 21：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：每次返回 tuple 后，顶层 SELECT 才增加 estate->es_processed；DML 行数由 ModifyTable 节点负责。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 22：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：dest->receiveSlot() 返回 false 时，ExecutorRun 提前停止，Portal 可以保留后续状态。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 23：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：本次 run 结束后 es_total_processed 累加 es_processed。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 24：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：DestReceiver shutdown 与 query_instr stop 都在 ExecutorRun 边界完成。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 25：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：PortalRun() 根据 portal strategy 判断是单个 SELECT、utility，还是多语句 portal。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 26：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：PortalRunSelect() 把 forward/count 翻译成 ScanDirection 和 ExecutorRun() 参数。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 27：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：standard_ExecutorRun() 断言 estate 已经存在，并确认不是 EXPLAIN-only 执行。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。
## 20. 一页纸定位模板

这部分用于把课程内容压缩成排查动作。不要从这里开始背函数名；先带着现象进入，再沿状态边界回源码。

### 定位 1：执行还没返回，但 gdb 已经停在 executor 内部

- 先停在 `PortalRun`，确认当前调用属于 `PostgreSQL ExecutorRun 的方向、计数与 Portal 边界` 的哪一个生命周期边界。
- 打印 `Portal`，不要只看字段值，还要解释它的语义：外层可暂停的执行对象，保存位置、状态、QueryDesc 和调用约束。
- 向上追一层调用者，确认是谁要求 executor 进入这个边界。
- 向下追一层被调用者，确认这一步是否注册了资源、改写了状态或返回了 slot。
- 如果现象来自观测指标，判断指标采样是在进入本步之前、执行过程中，还是返回之后。
- 如果现象来自异常路径，判断资源是否已经挂到 owner；没有 owner 的对象必须由当前路径处理。

### 定位 2：EXPLAIN ANALYZE 的时间和直觉不一致

- 先停在 `PortalRunSelect`，确认当前调用属于 `PostgreSQL ExecutorRun 的方向、计数与 Portal 边界` 的哪一个生命周期边界。
- 打印 `QueryDesc.dest`，不要只看字段值，还要解释它的语义：本次输出的接收器；可能是客户端、SPI、tuplestore、EXPLAIN 或 None_Receiver。
- 向上追一层调用者，确认是谁要求 executor 进入这个边界。
- 向下追一层被调用者，确认这一步是否注册了资源、改写了状态或返回了 slot。
- 如果现象来自观测指标，判断指标采样是在进入本步之前、执行过程中，还是返回之后。
- 如果现象来自异常路径，判断资源是否已经挂到 owner；没有 owner 的对象必须由当前路径处理。

### 定位 3：cursor、FETCH 或客户端输出让执行分段

- 先停在 `PortalRunFetch`，确认当前调用属于 `PostgreSQL ExecutorRun 的方向、计数与 Portal 边界` 的哪一个生命周期边界。
- 打印 `ScanDirection`，不要只看字段值，还要解释它的语义：本次拉取方向，写入 estate->es_direction，供节点和 access method 解释。
- 向上追一层调用者，确认是谁要求 executor 进入这个边界。
- 向下追一层被调用者，确认这一步是否注册了资源、改写了状态或返回了 slot。
- 如果现象来自观测指标，判断指标采样是在进入本步之前、执行过程中，还是返回之后。
- 如果现象来自异常路径，判断资源是否已经挂到 owner；没有 owner 的对象必须由当前路径处理。

### 定位 4：ERROR 后怀疑 executor 资源没有按预期释放

- 先停在 `ExecutorRun`，确认当前调用属于 `PostgreSQL ExecutorRun 的方向、计数与 Portal 边界` 的哪一个生命周期边界。
- 打印 `count`，不要只看字段值，还要解释它的语义：顶层返回 tuple 上限；0 表示本次 run to completion。
- 向上追一层调用者，确认是谁要求 executor 进入这个边界。
- 向下追一层被调用者，确认这一步是否注册了资源、改写了状态或返回了 slot。
- 如果现象来自观测指标，判断指标采样是在进入本步之前、执行过程中，还是返回之后。
- 如果现象来自异常路径，判断资源是否已经挂到 owner；没有 owner 的对象必须由当前路径处理。

### 定位 5：扩展 hook 或 profiler 捕获到一个边界事件

- 先停在 `standard_ExecutorRun`，确认当前调用属于 `PostgreSQL ExecutorRun 的方向、计数与 Portal 边界` 的哪一个生命周期边界。
- 打印 `es_processed`，不要只看字段值，还要解释它的语义：单次 ExecutorRun 的顶层处理行数。
- 向上追一层调用者，确认是谁要求 executor 进入这个边界。
- 向下追一层被调用者，确认这一步是否注册了资源、改写了状态或返回了 slot。
- 如果现象来自观测指标，判断指标采样是在进入本步之前、执行过程中，还是返回之后。
- 如果现象来自异常路径，判断资源是否已经挂到 owner；没有 owner 的对象必须由当前路径处理。

### 定位 6：源码阅读时分不清字段属于 Plan 还是 runtime state

- 先停在 `ExecutePlan`，确认当前调用属于 `PostgreSQL ExecutorRun 的方向、计数与 Portal 边界` 的哪一个生命周期边界。
- 打印 `es_total_processed`，不要只看字段值，还要解释它的语义：同一 executor 多次 Run 的累计顶层行数。
- 向上追一层调用者，确认是谁要求 executor 进入这个边界。
- 向下追一层被调用者，确认这一步是否注册了资源、改写了状态或返回了 slot。
- 如果现象来自观测指标，判断指标采样是在进入本步之前、执行过程中，还是返回之后。
- 如果现象来自异常路径，判断资源是否已经挂到 owner；没有 owner 的对象必须由当前路径处理。

定位结束时只保留三句话：

1. 这个现象落在哪个 executor 生命周期边界。
2. 这个边界改变了哪个核心状态，以及 owner 是谁。
3. 正常路径和 ERROR 路径分别由哪个函数或机制收尾。

## 下一步建议

如果你想把 Portal 边界继续向外追到客户端协议，建议阅读待补课程 `57-frontend-backend-protocol-message-loop.md` 到 `64-plan-cache-executor-boundary.md`。那组课程会解释 simple query、extended query、prepared statement、cached plan、Portal 和 DestReceiver 如何把客户端消息连接到本节的 `ExecutorRun()`。
