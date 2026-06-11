# PostgreSQL ExecProcNode 函数指针与节点调度

## 课程定位

前置知识：已经读过 executor 生命周期的目录规划，并理解 PostgreSQL 的查询执行由 Portal、QueryDesc、EState、PlanState 和 TupleTableSlot 共同完成。上一节建立了 Plan 到 PlanState 的状态化边界。

本节唯一主问题：

```text
PlanState->ExecProcNode 为什么用函数指针和首调用 wrapper 分派节点，而不是每取一行都在统一 dispatcher 中 switch nodeTag？
```

核心矛盾：executor 需要支持大量节点、instrumentation、rescan 和扩展回调 vs 每行 hot path 必须尽量少做分派和条件判断。

学完后应能判断：能解释一次 ExecProcNode 调用从 inline chgParam 检查到节点真实函数的完整路径。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

上一节建立了 Plan 到 PlanState 的状态化边界。

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

下一节会解释 ExecProcNode 返回的 TupleTableSlot 为什么有多种物理形态。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
ExecInit* 设置节点真实 next-tuple 函数；ExecSetExecProcNode() 先装 ExecProcNodeFirst()，首调用做 stack check 和 instrumentation wrapper 选择，然后后续直接调用节点函数或 ExecProcNodeInstr。
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
| 1 | src/include/executor/executor.h | ExecProcNode() inline：chgParam 检查和函数指针调用。 |
| 2 | src/backend/executor/execProcnode.c | ExecSetExecProcNode() / ExecProcNodeFirst() / MultiExecProcNode()。 |
| 3 | src/backend/executor/instrument.c | ExecProcNodeInstr()：instrumentation wrapper。 |
| 4 | src/include/nodes/execnodes.h | PlanState 的 ExecProcNode、ExecProcNodeReal、instrument。 |
| 5 | src/backend/executor/nodeSeqscan.c | ExecInitSeqScan() 如何选择 ExecSeqScan variants。 |
| 6 | src/backend/executor/nodeSort.c | ExecSort()：阻塞节点仍以同一函数指针协议返回 tuple。 |
| 7 | src/backend/executor/nodeAgg.c | ExecAgg()：复杂节点在同一 next-tuple 协议中推进多阶段状态。 |
| 8 | src/backend/executor/execAmi.c | ExecReScan() / mark/restore 等节点访问方法。 |

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
| ExecProcNode | PlanState 当前被调用的 next-tuple 函数，可能是首调用 wrapper、instrument wrapper 或真实节点函数。 |
| ExecProcNodeReal | 真实节点函数，instrumentation 需要 wrapper 时保存在这里。 |
| instrument | 节点级计时和行数统计对象，存在时首调用会切换到 ExecProcNodeInstr。 |
| chgParam | 参数变化集合，inline ExecProcNode 在调用函数指针前触发 ExecReScan。 |
| node-specific function | ExecSeqScan、ExecSort、ExecAgg、ExecModifyTable 等真实推进函数。 |
| MultiExecProcNode | 非 tuple 返回节点的并行协议。 |
| stack depth check | 首调用进行，避免每行重复付出成本。 |
| CHECK_FOR_INTERRUPTS | MultiExecProcNode 等路径显式处理长循环中断。 |

本节最容易混淆的边界有四个：

- planner-time 描述与 executor-time 状态不同。
- backend-local 指针与跨 backend 可见状态不同。
- memory context 释放内存，不等于释放所有外部资源。
- 顶层调用边界与节点内部状态机边界不同。

读源码时要持续追问：这个状态现在是谁的，下一步谁会改变它，如果中途 ERROR，谁会兜底。

## 5. 主流程源码 walkthrough

下面按时间顺序读主链路。每一步都回答“状态发生了什么变化”，而不是只记住函数名。

### 5.1. 步骤 1

ExecInitNode() 返回具体 PlanState 后，ExecSetExecProcNode(result, result->ExecProcNode) 包装节点函数。

阅读这一段时的重点不是记住第 1 个函数名，而是确认状态边界：读到这里先确认这一步是在创建状态、注册外部资源，还是只是在写入控制字段。

### 5.2. 步骤 2

ExecSetExecProcNode() 把真实函数保存到 ExecProcNodeReal。

阅读这一段时的重点不是记住第 2 个函数名，而是确认状态边界：断点停在这里时，优先打印 owner 指针，再打印本步刚写入的字段。

### 5.3. 步骤 3

它把 ExecProcNode 设置为 ExecProcNodeFirst。

阅读这一段时的重点不是记住第 3 个函数名，而是确认状态边界：这一步如果提前失败，后续状态还没有 owner，不能按完整 executor cleanup 想象。

### 5.4. 步骤 4

父节点或 ExecutePlan 调用 inline ExecProcNode(node)。

阅读这一段时的重点不是记住第 4 个函数名，而是确认状态边界：这一步如果已经完成，后续代码就可以假定对应 runtime state 已经稳定。

### 5.5. 步骤 5

inline 先检查 node->chgParam；非空时调用 ExecReScan(node)。

阅读这一段时的重点不是记住第 5 个函数名，而是确认状态边界：这里要区分“指针存在”和“语义已经注册”两件事。

### 5.6. 步骤 6

然后 inline 调用 node->ExecProcNode(node)，第一次进入 ExecProcNodeFirst。

阅读这一段时的重点不是记住第 6 个函数名，而是确认状态边界：这里的状态变化通常会影响后续 ExecProcNode、slot、snapshot 或 receiver 的解释。

### 5.7. 步骤 7

ExecProcNodeFirst 做 check_stack_depth。

阅读这一段时的重点不是记住第 7 个函数名，而是确认状态边界：如果以后诊断慢查询，这一步更像 setup 成本，不应误归到单个 tuple。

### 5.8. 步骤 8

如果 node->instrument 存在，node->ExecProcNode 改为 ExecProcNodeInstr。

阅读这一段时的重点不是记住第 8 个函数名，而是确认状态边界：如果以后诊断异常退出，这一步要回看是否已经进入 ResourceOwner、MemoryContext 或 snapshot 注册边界。

### 5.9. 步骤 9

如果没有 instrumentation，node->ExecProcNode 直接改为 ExecProcNodeReal。

阅读这一段时的重点不是记住第 9 个函数名，而是确认状态边界：这一步的调用者协议比函数内部细节更重要：调用者是否允许重复进入，是否允许部分执行。

### 5.10. 步骤 10

首调用最后再次调用 node->ExecProcNode(node)，进入 wrapper 或真实函数。

阅读这一段时的重点不是记住第 10 个函数名，而是确认状态边界：这一步之后要看下一个 owner 边界，而不是继续把所有状态当作局部变量。

### 5.11. 步骤 11

后续每行调用跳过首调用检查，hot path 只剩 chgParam 判断和函数指针调用。

阅读这一段时的重点不是记住第 11 个函数名，而是确认状态边界：如果这一步涉及 hook 或 wrapper，要继续追到标准实现或真实节点函数。

### 5.12. 步骤 12

节点真实函数内部按自己的状态机拉取子节点或返回 slot。

阅读这一段时的重点不是记住第 12 个函数名，而是确认状态边界：如果这一步涉及 slot 或 tuple，必须同时确认数据内容和底层资源是否同寿命。

主链路可以压缩成：

```text
01. ExecInitNode() 返回具体 PlanState 后，ExecSetExecProcNode(result, result->ExecProcNode) 包装节点函数。
02. ExecSetExecProcNode() 把真实函数保存到 ExecProcNodeReal。
03. 它把 ExecProcNode 设置为 ExecProcNodeFirst。
04. 父节点或 ExecutePlan 调用 inline ExecProcNode(node)。
05. inline 先检查 node->chgParam；非空时调用 ExecReScan(node)。
06. 然后 inline 调用 node->ExecProcNode(node)，第一次进入 ExecProcNodeFirst。
07. ExecProcNodeFirst 做 check_stack_depth。
08. 如果 node->instrument 存在，node->ExecProcNode 改为 ExecProcNodeInstr。
09. 如果没有 instrumentation，node->ExecProcNode 直接改为 ExecProcNodeReal。
10. 首调用最后再次调用 node->ExecProcNode(node)，进入 wrapper 或真实函数。
11. 后续每行调用跳过首调用检查，hot path 只剩 chgParam 判断和函数指针调用。
12. 节点真实函数内部按自己的状态机拉取子节点或返回 slot。
```

## 6. 状态随时间推进的故事

把源码主线换成一个对象的生命周期，会更容易看出本节 tension。

### 6.1. 创建前

在“创建前”阶段，关键观察点是：ExecInitNode() 返回具体 PlanState 后，ExecSetExecProcNode(result, result->ExecProcNode) 包装节点函数。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

### 6.2. 创建时

在“创建时”阶段，关键观察点是：ExecSetExecProcNode() 把真实函数保存到 ExecProcNodeReal。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

### 6.3. 第一次使用前

在“第一次使用前”阶段，关键观察点是：它把 ExecProcNode 设置为 ExecProcNodeFirst。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

### 6.4. 正常推进中

在“正常推进中”阶段，关键观察点是：父节点或 ExecutePlan 调用 inline ExecProcNode(node)。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

### 6.5. 遇到边界时

在“遇到边界时”阶段，关键观察点是：inline 先检查 node->chgParam；非空时调用 ExecReScan(node)。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

### 6.6. 提前停止时

在“提前停止时”阶段，关键观察点是：然后 inline 调用 node->ExecProcNode(node)，第一次进入 ExecProcNodeFirst。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

### 6.7. 收尾时

在“收尾时”阶段，关键观察点是：ExecProcNodeFirst 做 check_stack_depth。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

### 6.8. ERROR 后

在“ERROR 后”阶段，关键观察点是：如果 node->instrument 存在，node->ExecProcNode 改为 ExecProcNodeInstr。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

## 7. 生命周期 / ownership / cleanup

- 函数指针在 ExecInit* 阶段设置，在首调用时可能被改写。
- instrumentation 对象由 ExecutorStart/Explain path 分配，PlanState 只持有指针。
- chgParam 由参数变化路径设置，由 ExecReScan 消费并重建节点状态。
- ExecProcNodeReal 在 instrumentation wrapper 中保留真实入口。
- ExecutorEnd 后 PlanState 消失，函数指针不能被扩展缓存使用。

用 ownership 语言重写本节：

| 问题 | 判断方式 |
| --- | --- |
| 谁创建 | 看入口函数和 memory context 切换点。 |
| 谁持有 | 看 QueryDesc、EState、PlanState 或 slot 列表中的指针。 |
| 谁释放 | 看 ExecutorFinish、ExecutorEnd、ExecEndNode、ExecResetTupleTable 或 ResourceOwner。 |
| ERROR 怎么办 | 看资源是否有 ResourceOwner、MemoryContext、snapshot register 或 callback 兜底。 |
| 能否跨边界保存 | 看对象是否 backend-local、是否依赖 slot 内容、buffer pin 或 per-tuple context。 |

## 8. 正确性机制层次

- chgParam 检查在统一 inline 层保证父节点不需要记住每个子节点的 rescan 细节。
- 首调用 stack check 保护深计划树，但避免每行重复昂贵检查。
- instrumentation wrapper 在节点边界包裹真实函数，保证 rows/timing 归属到正确节点。
- 真实节点函数返回空 slot 表示 EOF，不能用其它 sentinel。
- 节点可以在初始化时选择优化 variants，例如 SeqScan 根据 qual/projection 选择不同函数。
- MultiExecProcNode 明确区分非 tuple 节点，避免破坏 ExecProcNode 的 tuple 协议。

这些机制互相补位。snapshot 解决可见性，不解决内存生命周期；MemoryContext 解决内存批量释放，不解决 buffer pin；函数指针解决分派成本，不解决节点协议正确性。

| 层次 | 保证 | 不能误解为 |
| --- | --- | --- |
| snapshot / command id | tuple 可见性和写入标记边界 | 并发互斥或资源释放 |
| PlanState / slot 协议 | 节点间传递和推进方式一致 | 所有节点都非阻塞 |
| MemoryContext | backend-local 内存归属清楚 | 外部资源自动释放 |
| ResourceOwner / pin / refcount | 外部资源不会被过早回收 | 对象语义仍然新鲜 |
| hook / instrumentation | 观测或扩展入口 | 可以任意改变核心语义 |

## 9. 错误路径 / 异常路径 / fallback

- 某些节点会在执行中改写 ExecProcNode，比如并行 hash join 状态机切换。
- ExecSetExecProcNode 注释说明执行开始后改函数会再次经过首调用 wrapper，这是可接受的。
- instrumentation 不存在时 actual rows 不会被普通执行无条件统计。
- CustomScan 节点可以把真实函数交给扩展提供，但仍必须遵守 slot 协议。
- 如果节点函数长时间内部循环不返回 tuple，需要自己承担 interrupt 和 instrumentation 细节。

PostgreSQL executor 的异常路径不能按 C++ 析构模型理解。`ereport(ERROR)` 可能 longjmp 到上层，普通 C 栈上的清理代码不会自然逐层执行。

因此本节所有状态都要问一句：如果这个点 ERROR，已经注册的资源由谁释放，尚未注册的资源是否会泄漏或变成悬挂引用。

## 10. 成本、资源与跨模块传播

- 函数指针分派避免每行 switch nodeTag，适合 executor hot path。
- inline chgParam 判断仍是每行成本，但换来统一 rescan 正确性。
- instrumentation wrapper 增加每行计时和 row count 成本，所以普通执行不默认开启。
- 首调用 wrapper 把 stack depth check 移出稳定 hot path。
- 节点 variants 让 SeqScan 这类高频节点少做 qual/projection 分支。

executor 的成本常常不是单个函数慢，而是边界次数被 workload 放大：每行、每节点、每次 FETCH、每个 trigger、每个 slot clear，都可能成为可见成本。

| 放大因素 | 典型表现 |
| --- | --- |
| tuple 数 | ExecProcNode、slot deform、per-tuple reset 和 receiver 调用被放大。 |
| plan node 数 | ExecInitNode、ExecEndNode、instrumentation 对象和 EXPLAIN 输出被放大。 |
| relation 数 | range table relation 打开、ResultRelInfo、锁和权限检查被放大。 |
| FETCH 次数 | ExecutorRun 边界、DestReceiver startup/shutdown 和 Portal 状态切换被放大。 |
| 副作用数量 | trigger、constraint、FDW flush、ModifyTable drain 被放大。 |

## 11. 观测与诊断入口

- gdb 断在 ExecProcNodeFirst，第一次命中后打印 node->ExecProcNode 的变化。
- 开启 EXPLAIN ANALYZE，观察 node->ExecProcNode 变成 ExecProcNodeInstr。
- 不开 instrumentation 时，首调用后 node->ExecProcNode 指向 ExecSeqScan 等真实函数。
- 对 ExecReScan 设置断点，修改 PARAM_EXEC 或执行嵌套循环参数化内侧扫描时观察 chgParam。
- perf 中函数指针会让调用栈显示真实节点函数，而不是大型 dispatcher。

建议的断点顺序：

1. `ExecProcNode`
2. `ExecSetExecProcNode`
3. `ExecProcNodeFirst`
4. `ExecProcNodeInstr`
5. `ExecReScan`
6. `ExecSeqScanWithQualProject`
7. `ExecSort`
8. `MultiExecProcNode`

断点不只是为了停住程序。每个断点都应该带一个问题：这个状态是否已经初始化，owner 是谁，下一步是否会被改写。

## 12. 课堂实验

### 实验 1

执行 SELECT * FROM t，断在 ExecProcNodeFirst，打印 nodeTag(node) 和 ExecProcNodeReal。

观察点：
- 断点前后同一个字段是否发生变化。
- 这个变化是否发生在本节主流程上。
- 如果把 SQL 改成 cursor、EXPLAIN、DML 或带 trigger，边界是否改变。

### 实验 2

执行 EXPLAIN ANALYZE SELECT * FROM t，确认 instrument 非空时 wrapper 选择不同。

观察点：
- 断点前后同一个字段是否发生变化。
- 这个变化是否发生在本节主流程上。
- 如果把 SQL 改成 cursor、EXPLAIN、DML 或带 trigger，边界是否改变。

### 实验 3

构造 WHERE 子句和投影，观察 ExecInitSeqScan 选择 ExecSeqScanWithQualProject。

观察点：
- 断点前后同一个字段是否发生变化。
- 这个变化是否发生在本节主流程上。
- 如果把 SQL 改成 cursor、EXPLAIN、DML 或带 trigger，边界是否改变。

### 实验 4

执行 ORDER BY，观察 ExecSort 第一次调用内部先完成 tuplesort。

观察点：
- 断点前后同一个字段是否发生变化。
- 这个变化是否发生在本节主流程上。
- 如果把 SQL 改成 cursor、EXPLAIN、DML 或带 trigger，边界是否改变。

### 实验 5

执行参数化 Nested Loop，断在 ExecReScan，观察 chgParam 清理。

观察点：
- 断点前后同一个字段是否发生变化。
- 这个变化是否发生在本节主流程上。
- 如果把 SQL 改成 cursor、EXPLAIN、DML 或带 trigger，边界是否改变。

### 实验 6

对 MultiExecProcNode 设置断点，用 bitmap scan 查询触发非 tuple 协议。

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

### 练习 1：`ExecProcNode`

在 `/home/highgo/postgres` 中定位 `ExecProcNode`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

### 练习 2：`ExecSetExecProcNode`

在 `/home/highgo/postgres` 中定位 `ExecSetExecProcNode`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

### 练习 3：`ExecProcNodeFirst`

在 `/home/highgo/postgres` 中定位 `ExecProcNodeFirst`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

### 练习 4：`ExecProcNodeInstr`

在 `/home/highgo/postgres` 中定位 `ExecProcNodeInstr`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

### 练习 5：`ExecReScan`

在 `/home/highgo/postgres` 中定位 `ExecReScan`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

### 练习 6：`ExecSeqScanWithQualProject`

在 `/home/highgo/postgres` 中定位 `ExecSeqScanWithQualProject`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

### 练习 7：`ExecSort`

在 `/home/highgo/postgres` 中定位 `ExecSort`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

### 练习 8：`MultiExecProcNode`

在 `/home/highgo/postgres` 中定位 `MultiExecProcNode`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

## 14. 常见误区

- 不要以为 ExecProcNode 是一个大 switch；大 switch 在初始化和 end，而不是每行 hot path。
- 不要跳过 chgParam 检查直接调用 node->ExecProcNodeReal。
- 不要把 ExecProcNodeFirst 看成优化细节，它同时承担 stack check 和 instrumentation 选择。
- 不要把 instrumentation 的 rows 等同于 ExecutePlan 的顶层 es_processed。
- 不要让扩展的 CustomScan 返回不符合 slot 生命周期的对象。

这些误区的共同点是把 runtime 状态当成静态结构，或者把静态计划当成可以直接推进的状态机。

## 15. 讨论题

1. 函数指针分派比 switch 分派省下的是什么成本，又增加了什么调试难度？
2. 为什么 instrumentation wrapper 必须在节点边界，而不是只包顶层 ExecutePlan？
3. chgParam 放在 inline ExecProcNode 层有什么好处？
4. 节点 variants 和 JIT 分别解决 hot path 的哪一类成本？

讨论时要尽量回到一个具体 runtime 现象，比如一次 FETCH、一次 EXPLAIN ANALYZE、一个 trigger 或一个 slot materialize，而不是停留在抽象偏好。

## 16. 本节小结

本节唯一主问题是：PlanState->ExecProcNode 为什么用函数指针和首调用 wrapper 分派节点，而不是每取一行都在统一 dispatcher 中 switch nodeTag？

可以带走的一句话模型是：ExecInit* 设置节点真实 next-tuple 函数；ExecSetExecProcNode() 先装 ExecProcNodeFirst()，首调用做 stack check 和 instrumentation wrapper 选择，然后后续直接调用节点函数或 ExecProcNodeInstr。

- 源码阅读应先找状态创建点，再找推进点，最后找 cleanup。
- 任何字段都要和 owner、生命周期、异常路径一起理解。
- 观测数据只有能回到具体 runtime 边界时，才适合用于诊断。

下一节会解释 ExecProcNode 返回的 TupleTableSlot 为什么有多种物理形态。

## 17. 从现象回源码的诊断案例

下面的案例都只服务本节主问题。它们的共同读法是先看现象，再找 runtime 边界，最后回到 owner 和 cleanup。

### 案例 1：第一次执行停在入口但状态还没有建立

现象：

- 你在 `ExecutorRun` 附近停住，看到 SQL 还没有返回，但某些 executor 状态已经改变。
- 这时不要急着沿调用栈往下追完整执行，而要先回到本节主问题：PlanState->ExecProcNode 为什么用函数指针和首调用 wrapper 分派节点，而不是每取一行都在统一 dispatcher 中 switch nodeTag？
- 如果现象来自 EXPLAIN、cursor、trigger 或扩展 hook，要先确认它落在哪个 executor 生命周期边界。

源码回看：

- 入口函数：`ExecutorRun`。
- 本步主线：ExecInitNode() 返回具体 PlanState 后，ExecSetExecProcNode(result, result->ExecProcNode) 包装节点函数。
- 关键状态：`ExecProcNode`，语义是：PlanState 当前被调用的 next-tuple 函数，可能是首调用 wrapper、instrument wrapper 或真实节点函数。
- 相邻状态：`ExecProcNodeReal`，用来判断这一步之后谁继续持有资源。

判断方式：

- 如果字段刚被写入但还没有注册到 owner，异常路径通常只能靠当前函数局部清理。
- 如果状态已经挂到 EState、PlanState、slot table 或 snapshot manager，后续 cleanup 应该能从 owner 找回。
- 如果这个状态只影响观测，不影响语义，诊断时不要把它解释成执行计划行为改变。
- 如果这个状态影响 tuple 生命周期，必须同时检查 slot、TupleDesc、buffer pin 或 per-tuple context。

### 案例 2：查询只返回少量行但内部状态已经推进

现象：

- 你在 `ExecutorRun` 附近停住，看到 SQL 还没有返回，但某些 executor 状态已经改变。
- 这时不要急着沿调用栈往下追完整执行，而要先回到本节主问题：PlanState->ExecProcNode 为什么用函数指针和首调用 wrapper 分派节点，而不是每取一行都在统一 dispatcher 中 switch nodeTag？
- 如果现象来自 EXPLAIN、cursor、trigger 或扩展 hook，要先确认它落在哪个 executor 生命周期边界。

源码回看：

- 入口函数：`ExecutorRun`。
- 本步主线：ExecSetExecProcNode() 把真实函数保存到 ExecProcNodeReal。
- 关键状态：`ExecProcNodeReal`，语义是：真实节点函数，instrumentation 需要 wrapper 时保存在这里。
- 相邻状态：`instrument`，用来判断这一步之后谁继续持有资源。

判断方式：

- 如果字段刚被写入但还没有注册到 owner，异常路径通常只能靠当前函数局部清理。
- 如果状态已经挂到 EState、PlanState、slot table 或 snapshot manager，后续 cleanup 应该能从 owner 找回。
- 如果这个状态只影响观测，不影响语义，诊断时不要把它解释成执行计划行为改变。
- 如果这个状态影响 tuple 生命周期，必须同时检查 slot、TupleDesc、buffer pin 或 per-tuple context。

### 案例 3：EXPLAIN ANALYZE 时间与节点时间不一致

现象：

- 你在 `ExecutorRun` 附近停住，看到 SQL 还没有返回，但某些 executor 状态已经改变。
- 这时不要急着沿调用栈往下追完整执行，而要先回到本节主问题：PlanState->ExecProcNode 为什么用函数指针和首调用 wrapper 分派节点，而不是每取一行都在统一 dispatcher 中 switch nodeTag？
- 如果现象来自 EXPLAIN、cursor、trigger 或扩展 hook，要先确认它落在哪个 executor 生命周期边界。

源码回看：

- 入口函数：`ExecutorRun`。
- 本步主线：它把 ExecProcNode 设置为 ExecProcNodeFirst。
- 关键状态：`instrument`，语义是：节点级计时和行数统计对象，存在时首调用会切换到 ExecProcNodeInstr。
- 相邻状态：`chgParam`，用来判断这一步之后谁继续持有资源。

判断方式：

- 如果字段刚被写入但还没有注册到 owner，异常路径通常只能靠当前函数局部清理。
- 如果状态已经挂到 EState、PlanState、slot table 或 snapshot manager，后续 cleanup 应该能从 owner 找回。
- 如果这个状态只影响观测，不影响语义，诊断时不要把它解释成执行计划行为改变。
- 如果这个状态影响 tuple 生命周期，必须同时检查 slot、TupleDesc、buffer pin 或 per-tuple context。

### 案例 4：cursor 或 receiver 让执行被分成多段

现象：

- 你在 `ExecutorRun` 附近停住，看到 SQL 还没有返回，但某些 executor 状态已经改变。
- 这时不要急着沿调用栈往下追完整执行，而要先回到本节主问题：PlanState->ExecProcNode 为什么用函数指针和首调用 wrapper 分派节点，而不是每取一行都在统一 dispatcher 中 switch nodeTag？
- 如果现象来自 EXPLAIN、cursor、trigger 或扩展 hook，要先确认它落在哪个 executor 生命周期边界。

源码回看：

- 入口函数：`ExecutorRun`。
- 本步主线：父节点或 ExecutePlan 调用 inline ExecProcNode(node)。
- 关键状态：`chgParam`，语义是：参数变化集合，inline ExecProcNode 在调用函数指针前触发 ExecReScan。
- 相邻状态：`node-specific function`，用来判断这一步之后谁继续持有资源。

判断方式：

- 如果字段刚被写入但还没有注册到 owner，异常路径通常只能靠当前函数局部清理。
- 如果状态已经挂到 EState、PlanState、slot table 或 snapshot manager，后续 cleanup 应该能从 owner 找回。
- 如果这个状态只影响观测，不影响语义，诊断时不要把它解释成执行计划行为改变。
- 如果这个状态影响 tuple 生命周期，必须同时检查 slot、TupleDesc、buffer pin 或 per-tuple context。

### 案例 5：ERROR 发生后没有按普通调用栈返回

现象：

- 你在 `ExecutorRun` 附近停住，看到 SQL 还没有返回，但某些 executor 状态已经改变。
- 这时不要急着沿调用栈往下追完整执行，而要先回到本节主问题：PlanState->ExecProcNode 为什么用函数指针和首调用 wrapper 分派节点，而不是每取一行都在统一 dispatcher 中 switch nodeTag？
- 如果现象来自 EXPLAIN、cursor、trigger 或扩展 hook，要先确认它落在哪个 executor 生命周期边界。

源码回看：

- 入口函数：`ExecutorRun`。
- 本步主线：inline 先检查 node->chgParam；非空时调用 ExecReScan(node)。
- 关键状态：`node-specific function`，语义是：ExecSeqScan、ExecSort、ExecAgg、ExecModifyTable 等真实推进函数。
- 相邻状态：`MultiExecProcNode`，用来判断这一步之后谁继续持有资源。

判断方式：

- 如果字段刚被写入但还没有注册到 owner，异常路径通常只能靠当前函数局部清理。
- 如果状态已经挂到 EState、PlanState、slot table 或 snapshot manager，后续 cleanup 应该能从 owner 找回。
- 如果这个状态只影响观测，不影响语义，诊断时不要把它解释成执行计划行为改变。
- 如果这个状态影响 tuple 生命周期，必须同时检查 slot、TupleDesc、buffer pin 或 per-tuple context。

### 案例 6：扩展 hook 看到的边界与核心 executor 边界不同

现象：

- 你在 `ExecutorRun` 附近停住，看到 SQL 还没有返回，但某些 executor 状态已经改变。
- 这时不要急着沿调用栈往下追完整执行，而要先回到本节主问题：PlanState->ExecProcNode 为什么用函数指针和首调用 wrapper 分派节点，而不是每取一行都在统一 dispatcher 中 switch nodeTag？
- 如果现象来自 EXPLAIN、cursor、trigger 或扩展 hook，要先确认它落在哪个 executor 生命周期边界。

源码回看：

- 入口函数：`ExecutorRun`。
- 本步主线：然后 inline 调用 node->ExecProcNode(node)，第一次进入 ExecProcNodeFirst。
- 关键状态：`MultiExecProcNode`，语义是：非 tuple 返回节点的并行协议。
- 相邻状态：`stack depth check`，用来判断这一步之后谁继续持有资源。

判断方式：

- 如果字段刚被写入但还没有注册到 owner，异常路径通常只能靠当前函数局部清理。
- 如果状态已经挂到 EState、PlanState、slot table 或 snapshot manager，后续 cleanup 应该能从 owner 找回。
- 如果这个状态只影响观测，不影响语义，诊断时不要把它解释成执行计划行为改变。
- 如果这个状态影响 tuple 生命周期，必须同时检查 slot、TupleDesc、buffer pin 或 per-tuple context。

## 18. 源码断点与字段记录表

下面这张表用于课堂现场跟读。每一行都应该实际在 gdb 中停一次，记录进入前和返回后的字段差异。

| 序号 | 断点 | 进入前看什么 | 返回后看什么 | 常见误判 |
| --- | --- | --- | --- | --- |
| 1 | `ExecutorRun` | `ExecProcNode` 是否已经有效 | ExecInitNode() 返回具体 PlanState 后，ExecSetExecProcNode(result, result->ExecProcNode) 包装节点函数。 | 把字段存在误认为语义已经完成 |
| 2 | `ExecutorRun` | `ExecProcNodeReal` 是否已经有效 | ExecSetExecProcNode() 把真实函数保存到 ExecProcNodeReal。 | 把字段存在误认为语义已经完成 |
| 3 | `ExecutorRun` | `instrument` 是否已经有效 | 它把 ExecProcNode 设置为 ExecProcNodeFirst。 | 把字段存在误认为语义已经完成 |
| 4 | `ExecutorRun` | `chgParam` 是否已经有效 | 父节点或 ExecutePlan 调用 inline ExecProcNode(node)。 | 把字段存在误认为语义已经完成 |
| 5 | `ExecutorRun` | `node-specific function` 是否已经有效 | inline 先检查 node->chgParam；非空时调用 ExecReScan(node)。 | 把字段存在误认为语义已经完成 |
| 6 | `ExecutorRun` | `MultiExecProcNode` 是否已经有效 | 然后 inline 调用 node->ExecProcNode(node)，第一次进入 ExecProcNodeFirst。 | 把字段存在误认为语义已经完成 |
| 7 | `ExecutorRun` | `stack depth check` 是否已经有效 | ExecProcNodeFirst 做 check_stack_depth。 | 把字段存在误认为语义已经完成 |
| 8 | `ExecutorRun` | `CHECK_FOR_INTERRUPTS` 是否已经有效 | 如果 node->instrument 存在，node->ExecProcNode 改为 ExecProcNodeInstr。 | 把字段存在误认为语义已经完成 |
| 9 | `ExecutorRun` | `ExecProcNode` 是否已经有效 | 如果没有 instrumentation，node->ExecProcNode 直接改为 ExecProcNodeReal。 | 把字段存在误认为语义已经完成 |
| 10 | `ExecutorRun` | `ExecProcNodeReal` 是否已经有效 | 首调用最后再次调用 node->ExecProcNode(node)，进入 wrapper 或真实函数。 | 把字段存在误认为语义已经完成 |
| 11 | `ExecutorRun` | `instrument` 是否已经有效 | 后续每行调用跳过首调用检查，hot path 只剩 chgParam 判断和函数指针调用。 | 把字段存在误认为语义已经完成 |
| 12 | `ExecutorRun` | `chgParam` 是否已经有效 | 节点真实函数内部按自己的状态机拉取子节点或返回 slot。 | 把字段存在误认为语义已经完成 |

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
- 只观察一个状态变化：ExecInitNode() 返回具体 PlanState 后，ExecSetExecProcNode(result, result->ExecProcNode) 包装节点函数。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 2：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：ExecSetExecProcNode() 把真实函数保存到 ExecProcNodeReal。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 3：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：它把 ExecProcNode 设置为 ExecProcNodeFirst。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 4：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：父节点或 ExecutePlan 调用 inline ExecProcNode(node)。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 5：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：inline 先检查 node->chgParam；非空时调用 ExecReScan(node)。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 6：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：然后 inline 调用 node->ExecProcNode(node)，第一次进入 ExecProcNodeFirst。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 7：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：ExecProcNodeFirst 做 check_stack_depth。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 8：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：如果 node->instrument 存在，node->ExecProcNode 改为 ExecProcNodeInstr。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 9：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：如果没有 instrumentation，node->ExecProcNode 直接改为 ExecProcNodeReal。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 10：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：首调用最后再次调用 node->ExecProcNode(node)，进入 wrapper 或真实函数。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 11：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：后续每行调用跳过首调用检查，hot path 只剩 chgParam 判断和函数指针调用。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 12：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：节点真实函数内部按自己的状态机拉取子节点或返回 slot。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 13：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：ExecInitNode() 返回具体 PlanState 后，ExecSetExecProcNode(result, result->ExecProcNode) 包装节点函数。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 14：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：ExecSetExecProcNode() 把真实函数保存到 ExecProcNodeReal。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 15：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：它把 ExecProcNode 设置为 ExecProcNodeFirst。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 16：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：父节点或 ExecutePlan 调用 inline ExecProcNode(node)。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 17：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：inline 先检查 node->chgParam；非空时调用 ExecReScan(node)。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 18：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：然后 inline 调用 node->ExecProcNode(node)，第一次进入 ExecProcNodeFirst。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 19：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：ExecProcNodeFirst 做 check_stack_depth。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 20：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：如果 node->instrument 存在，node->ExecProcNode 改为 ExecProcNodeInstr。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 21：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：如果没有 instrumentation，node->ExecProcNode 直接改为 ExecProcNodeReal。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 22：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：首调用最后再次调用 node->ExecProcNode(node)，进入 wrapper 或真实函数。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 23：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：后续每行调用跳过首调用检查，hot path 只剩 chgParam 判断和函数指针调用。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 24：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：节点真实函数内部按自己的状态机拉取子节点或返回 slot。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 25：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：ExecInitNode() 返回具体 PlanState 后，ExecSetExecProcNode(result, result->ExecProcNode) 包装节点函数。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 26：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：ExecSetExecProcNode() 把真实函数保存到 ExecProcNodeReal。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 27：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：它把 ExecProcNode 设置为 ExecProcNodeFirst。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。
## 20. 一页纸定位模板

这部分用于把课程内容压缩成排查动作。不要从这里开始背函数名；先带着现象进入，再沿状态边界回源码。

### 定位 1：执行还没返回，但 gdb 已经停在 executor 内部

- 先停在 `ExecProcNode`，确认当前调用属于 `PostgreSQL ExecProcNode 函数指针与节点调度` 的哪一个生命周期边界。
- 打印 `ExecProcNode`，不要只看字段值，还要解释它的语义：PlanState 当前被调用的 next-tuple 函数，可能是首调用 wrapper、instrument wrapper 或真实节点函数。
- 向上追一层调用者，确认是谁要求 executor 进入这个边界。
- 向下追一层被调用者，确认这一步是否注册了资源、改写了状态或返回了 slot。
- 如果现象来自观测指标，判断指标采样是在进入本步之前、执行过程中，还是返回之后。
- 如果现象来自异常路径，判断资源是否已经挂到 owner；没有 owner 的对象必须由当前路径处理。

### 定位 2：EXPLAIN ANALYZE 的时间和直觉不一致

- 先停在 `ExecSetExecProcNode`，确认当前调用属于 `PostgreSQL ExecProcNode 函数指针与节点调度` 的哪一个生命周期边界。
- 打印 `ExecProcNodeReal`，不要只看字段值，还要解释它的语义：真实节点函数，instrumentation 需要 wrapper 时保存在这里。
- 向上追一层调用者，确认是谁要求 executor 进入这个边界。
- 向下追一层被调用者，确认这一步是否注册了资源、改写了状态或返回了 slot。
- 如果现象来自观测指标，判断指标采样是在进入本步之前、执行过程中，还是返回之后。
- 如果现象来自异常路径，判断资源是否已经挂到 owner；没有 owner 的对象必须由当前路径处理。

### 定位 3：cursor、FETCH 或客户端输出让执行分段

- 先停在 `ExecProcNodeFirst`，确认当前调用属于 `PostgreSQL ExecProcNode 函数指针与节点调度` 的哪一个生命周期边界。
- 打印 `instrument`，不要只看字段值，还要解释它的语义：节点级计时和行数统计对象，存在时首调用会切换到 ExecProcNodeInstr。
- 向上追一层调用者，确认是谁要求 executor 进入这个边界。
- 向下追一层被调用者，确认这一步是否注册了资源、改写了状态或返回了 slot。
- 如果现象来自观测指标，判断指标采样是在进入本步之前、执行过程中，还是返回之后。
- 如果现象来自异常路径，判断资源是否已经挂到 owner；没有 owner 的对象必须由当前路径处理。

### 定位 4：ERROR 后怀疑 executor 资源没有按预期释放

- 先停在 `ExecProcNodeInstr`，确认当前调用属于 `PostgreSQL ExecProcNode 函数指针与节点调度` 的哪一个生命周期边界。
- 打印 `chgParam`，不要只看字段值，还要解释它的语义：参数变化集合，inline ExecProcNode 在调用函数指针前触发 ExecReScan。
- 向上追一层调用者，确认是谁要求 executor 进入这个边界。
- 向下追一层被调用者，确认这一步是否注册了资源、改写了状态或返回了 slot。
- 如果现象来自观测指标，判断指标采样是在进入本步之前、执行过程中，还是返回之后。
- 如果现象来自异常路径，判断资源是否已经挂到 owner；没有 owner 的对象必须由当前路径处理。

### 定位 5：扩展 hook 或 profiler 捕获到一个边界事件

- 先停在 `ExecReScan`，确认当前调用属于 `PostgreSQL ExecProcNode 函数指针与节点调度` 的哪一个生命周期边界。
- 打印 `node-specific function`，不要只看字段值，还要解释它的语义：ExecSeqScan、ExecSort、ExecAgg、ExecModifyTable 等真实推进函数。
- 向上追一层调用者，确认是谁要求 executor 进入这个边界。
- 向下追一层被调用者，确认这一步是否注册了资源、改写了状态或返回了 slot。
- 如果现象来自观测指标，判断指标采样是在进入本步之前、执行过程中，还是返回之后。
- 如果现象来自异常路径，判断资源是否已经挂到 owner；没有 owner 的对象必须由当前路径处理。

### 定位 6：源码阅读时分不清字段属于 Plan 还是 runtime state

- 先停在 `ExecSeqScanWithQualProject`，确认当前调用属于 `PostgreSQL ExecProcNode 函数指针与节点调度` 的哪一个生命周期边界。
- 打印 `MultiExecProcNode`，不要只看字段值，还要解释它的语义：非 tuple 返回节点的并行协议。
- 向上追一层调用者，确认是谁要求 executor 进入这个边界。
- 向下追一层被调用者，确认这一步是否注册了资源、改写了状态或返回了 slot。
- 如果现象来自观测指标，判断指标采样是在进入本步之前、执行过程中，还是返回之后。
- 如果现象来自异常路径，判断资源是否已经挂到 owner；没有 owner 的对象必须由当前路径处理。

定位结束时只保留三句话：

1. 这个现象落在哪个 executor 生命周期边界。
2. 这个边界改变了哪个核心状态，以及 owner 是谁。
3. 正常路径和 ERROR 路径分别由哪个函数或机制收尾。
