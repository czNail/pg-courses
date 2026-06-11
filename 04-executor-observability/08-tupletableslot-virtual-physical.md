# PostgreSQL TupleTableSlot 的虚拟 tuple 与物理 tuple 边界

## 课程定位

前置知识：已经读过 executor 生命周期的目录规划，并理解 PostgreSQL 的查询执行由 Portal、QueryDesc、EState、PlanState 和 TupleTableSlot 共同完成。上一节讲清楚 ExecProcNode 如何以统一协议返回下一个 tuple。

本节唯一主问题：

```text
TupleTableSlot 为什么同时支持 virtual、heap、minimal、buffer-backed 多种形态，并且要求节点传 slot 而不是直接传 HeapTuple？
```

核心矛盾：节点间需要低拷贝传递 tuple vs tuple 可能来自 buffer、palloc 内存、minimal tuple 或表达式 Datum 数组，ownership、pin、TupleDesc 和 materialize 边界完全不同。

学完后应能判断：能判断一个 slot 当前是否依赖外部 buffer 或下层内存，什么时候必须 ExecMaterializeSlot 或 copy。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

上一节讲清楚 ExecProcNode 如何以统一协议返回下一个 tuple。

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

后续课程会继续进入 TupleDesc、投影、ExprContext 和具体执行节点。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
TupleTableSlot 是 executor 的 tuple 容器协议；slot ops 定义 clear、deform、materialize、copy 等行为，让上层节点按统一接口消费 tuple，同时把物理表示和资源释放留给具体 slot 类型。
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
| 1 | src/include/executor/tuptable.h | TupleTableSlot / TupleTableSlotOps / TTSOpsVirtual 等公开边界。 |
| 2 | src/backend/executor/execTuples.c | MakeTupleTableSlot() / ExecAllocTableSlot() / ExecResetTupleTable()。 |
| 3 | src/backend/executor/execTuples.c | ExecStoreVirtualTuple() / ExecStoreHeapTuple() / ExecStoreBufferHeapTuple() / ExecMaterializeSlot()。 |
| 4 | src/backend/executor/nodeSeqscan.c | SeqNext() / ExecSeqScan* 如何把 table AM tuple 放入 scan slot。 |
| 5 | src/backend/executor/execScan.c | ExecScan() / ExecProject() 如何消费 scan slot 和 result slot。 |
| 6 | src/backend/access/heap/heapam_handler.c | heap table AM 与 slot callbacks 的相邻边界。 |
| 7 | src/include/access/tupdesc.h | TupleDesc refcount / pin 语义。 |
| 8 | src/backend/storage/buffer/bufmgr.c | buffer pin 与 buffer-backed slot 的资源语义。 |

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
| tts_ops | slot 实现类型，决定 clear、getsomeattrs、materialize、copy 行为。 |
| tts_flags | EMPTY、SHOULDFREE、FIXED 等状态位，必须结合 slot ops 解释。 |
| tts_nvalid | values/isnull 中已经 deform 出来的属性数量。 |
| tts_tupleDescriptor | slot 的 tuple 形状，可能被 fixed，也可能运行时设置。 |
| tts_values / tts_isnull | virtual tuple 或 deform 后属性数组。 |
| tts_mcxt | slot 自身和可变数组分配所在 memory context。 |
| buffer-backed tuple | slot 内容依赖 buffer pin，clear 时必须释放。 |
| materialized tuple | slot 内容已经成为 slot 自己拥有的拷贝。 |

本节最容易混淆的边界有四个：

- planner-time 描述与 executor-time 状态不同。
- backend-local 指针与跨 backend 可见状态不同。
- memory context 释放内存，不等于释放所有外部资源。
- 顶层调用边界与节点内部状态机边界不同。

读源码时要持续追问：这个状态现在是谁的，下一步谁会改变它，如果中途 ERROR，谁会兜底。

## 5. 主流程源码 walkthrough

下面按时间顺序读主链路。每一步都回答“状态发生了什么变化”，而不是只记住函数名。

### 5.1. 步骤 1

ExecInitScanTupleSlot() 或 ExecInitResultTupleSlotTL() 在节点初始化时创建 slot。

阅读这一段时的重点不是记住第 1 个函数名，而是确认状态边界：读到这里先确认这一步是在创建状态、注册外部资源，还是只是在写入控制字段。

### 5.2. 步骤 2

MakeTupleTableSlot() 根据 tupleDesc 和 tts_ops 分配 slot 结构和 Datum/isnull 数组。

阅读这一段时的重点不是记住第 2 个函数名，而是确认状态边界：断点停在这里时，优先打印 owner 指针，再打印本步刚写入的字段。

### 5.3. 步骤 3

tupleDesc 存在时 slot 标记为 fixed，并 PinTupleDesc。

阅读这一段时的重点不是记住第 3 个函数名，而是确认状态边界：这一步如果提前失败，后续状态还没有 owner，不能按完整 executor cleanup 想象。

### 5.4. 步骤 4

slot->tts_ops->init(slot) 执行类型特定初始化。

阅读这一段时的重点不是记住第 4 个函数名，而是确认状态边界：这一步如果已经完成，后续代码就可以假定对应 runtime state 已经稳定。

### 5.5. 步骤 5

SeqScan 从 table AM 拿到物理 tuple 后，可能调用 ExecStoreBufferHeapTuple() 放入 buffer-backed slot。

阅读这一段时的重点不是记住第 5 个函数名，而是确认状态边界：这里要区分“指针存在”和“语义已经注册”两件事。

### 5.6. 步骤 6

Projection 可能把表达式结果写入 Datum/isnull 数组，再调用 ExecStoreVirtualTuple()。

阅读这一段时的重点不是记住第 6 个函数名，而是确认状态边界：这里的状态变化通常会影响后续 ExecProcNode、slot、snapshot 或 receiver 的解释。

### 5.7. 步骤 7

上层节点通过 slot_getattr 或 ExecCopySlot* 访问数据，slot ops 按需 deform 或 copy。

阅读这一段时的重点不是记住第 7 个函数名，而是确认状态边界：如果以后诊断慢查询，这一步更像 setup 成本，不应误归到单个 tuple。

### 5.8. 步骤 8

如果 tuple 要跨越下层 slot 生命周期或 buffer pin 生命周期，调用 ExecMaterializeSlot()。

阅读这一段时的重点不是记住第 8 个函数名，而是确认状态边界：如果以后诊断异常退出，这一步要回看是否已经进入 ResourceOwner、MemoryContext 或 snapshot 注册边界。

### 5.9. 步骤 9

ExecClearTuple(slot) 调用 slot ops clear，只清内容，不清 tuple descriptor。

阅读这一段时的重点不是记住第 9 个函数名，而是确认状态边界：这一步的调用者协议比函数内部细节更重要：调用者是否允许重复进入，是否允许部分执行。

### 5.10. 步骤 10

ExecResetTupleTable() 遍历 EState slot 列表，clear、release、ReleaseTupleDesc。

阅读这一段时的重点不是记住第 10 个函数名，而是确认状态边界：这一步之后要看下一个 owner 边界，而不是继续把所有状态当作局部变量。

### 5.11. 步骤 11

shouldFree、transfer_pin、materialize 决定 clear 时释放内存还是释放 buffer pin。

阅读这一段时的重点不是记住第 11 个函数名，而是确认状态边界：如果这一步涉及 hook 或 wrapper，要继续追到标准实现或真实节点函数。

### 5.12. 步骤 12

ExecutorEnd 删除 es_query_cxt 前必须先 reset tuple table，避免 pin/refcount 遗留。

阅读这一段时的重点不是记住第 12 个函数名，而是确认状态边界：如果这一步涉及 slot 或 tuple，必须同时确认数据内容和底层资源是否同寿命。

主链路可以压缩成：

```text
01. ExecInitScanTupleSlot() 或 ExecInitResultTupleSlotTL() 在节点初始化时创建 slot。
02. MakeTupleTableSlot() 根据 tupleDesc 和 tts_ops 分配 slot 结构和 Datum/isnull 数组。
03. tupleDesc 存在时 slot 标记为 fixed，并 PinTupleDesc。
04. slot->tts_ops->init(slot) 执行类型特定初始化。
05. SeqScan 从 table AM 拿到物理 tuple 后，可能调用 ExecStoreBufferHeapTuple() 放入 buffer-backed slot。
06. Projection 可能把表达式结果写入 Datum/isnull 数组，再调用 ExecStoreVirtualTuple()。
07. 上层节点通过 slot_getattr 或 ExecCopySlot* 访问数据，slot ops 按需 deform 或 copy。
08. 如果 tuple 要跨越下层 slot 生命周期或 buffer pin 生命周期，调用 ExecMaterializeSlot()。
09. ExecClearTuple(slot) 调用 slot ops clear，只清内容，不清 tuple descriptor。
10. ExecResetTupleTable() 遍历 EState slot 列表，clear、release、ReleaseTupleDesc。
11. shouldFree、transfer_pin、materialize 决定 clear 时释放内存还是释放 buffer pin。
12. ExecutorEnd 删除 es_query_cxt 前必须先 reset tuple table，避免 pin/refcount 遗留。
```

## 6. 状态随时间推进的故事

把源码主线换成一个对象的生命周期，会更容易看出本节 tension。

### 6.1. 创建前

在“创建前”阶段，关键观察点是：ExecInitScanTupleSlot() 或 ExecInitResultTupleSlotTL() 在节点初始化时创建 slot。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

### 6.2. 创建时

在“创建时”阶段，关键观察点是：MakeTupleTableSlot() 根据 tupleDesc 和 tts_ops 分配 slot 结构和 Datum/isnull 数组。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

### 6.3. 第一次使用前

在“第一次使用前”阶段，关键观察点是：tupleDesc 存在时 slot 标记为 fixed，并 PinTupleDesc。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

### 6.4. 正常推进中

在“正常推进中”阶段，关键观察点是：slot->tts_ops->init(slot) 执行类型特定初始化。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

### 6.5. 遇到边界时

在“遇到边界时”阶段，关键观察点是：SeqScan 从 table AM 拿到物理 tuple 后，可能调用 ExecStoreBufferHeapTuple() 放入 buffer-backed slot。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

### 6.6. 提前停止时

在“提前停止时”阶段，关键观察点是：Projection 可能把表达式结果写入 Datum/isnull 数组，再调用 ExecStoreVirtualTuple()。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

### 6.7. 收尾时

在“收尾时”阶段，关键观察点是：上层节点通过 slot_getattr 或 ExecCopySlot* 访问数据，slot ops 按需 deform 或 copy。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

### 6.8. ERROR 后

在“ERROR 后”阶段，关键观察点是：如果 tuple 要跨越下层 slot 生命周期或 buffer pin 生命周期，调用 ExecMaterializeSlot()。

这一步的诊断问题是：如果断点停在这里，哪些字段应该已经有效，哪些字段还不能读取。

## 7. 生命周期 / ownership / cleanup

- slot 通常在 ExecInit* 阶段创建并加入 estate->es_tupleTable。
- slot 结构活到 ExecutorEnd；slot 内容活到下一次 ExecClearTuple、rescan 或节点覆写。
- buffer-backed slot 内容依赖 buffer pin，pin 活到 slot clear。
- virtual slot 内容依赖 values/isnull 和其中 Datum 指向的内存，pass-by-reference 值可能更短命。
- TupleDesc refcount 在 slot 创建或设置 descriptor 时增加，在 ExecResetTupleTable 或 drop slot 时释放。

用 ownership 语言重写本节：

| 问题 | 判断方式 |
| --- | --- |
| 谁创建 | 看入口函数和 memory context 切换点。 |
| 谁持有 | 看 QueryDesc、EState、PlanState 或 slot 列表中的指针。 |
| 谁释放 | 看 ExecutorFinish、ExecutorEnd、ExecEndNode、ExecResetTupleTable 或 ResourceOwner。 |
| ERROR 怎么办 | 看资源是否有 ResourceOwner、MemoryContext、snapshot register 或 callback 兜底。 |
| 能否跨边界保存 | 看对象是否 backend-local、是否依赖 slot 内容、buffer pin 或 per-tuple context。 |

## 8. 正确性机制层次

- slot 类型不是优化标签，而是 ownership 协议。
- TTS_EMPTY 表示没有当前 tuple；全 NULL tuple 是有内容的 tuple，不等于 empty。
- virtual tuple 避免物理构造，但不能自动延长 pass-by-reference Datum 的生命周期。
- buffer-backed slot 必须持有 pin，否则底层 page tuple 可能被移动或回收。
- ExecMaterializeSlot 把 slot 内容转成只依赖 slot 自己的状态。
- ExecResetTupleTable 必须在 context delete 前执行，释放非内存资源。

这些机制互相补位。snapshot 解决可见性，不解决内存生命周期；MemoryContext 解决内存批量释放，不解决 buffer pin；函数指针解决分派成本，不解决节点协议正确性。

| 层次 | 保证 | 不能误解为 |
| --- | --- | --- |
| snapshot / command id | tuple 可见性和写入标记边界 | 并发互斥或资源释放 |
| PlanState / slot 协议 | 节点间传递和推进方式一致 | 所有节点都非阻塞 |
| MemoryContext | backend-local 内存归属清楚 | 外部资源自动释放 |
| ResourceOwner / pin / refcount | 外部资源不会被过早回收 | 对象语义仍然新鲜 |
| hook / instrumentation | 观测或扩展入口 | 可以任意改变核心语义 |

## 9. 错误路径 / 异常路径 / fallback

- ExecForceStoreHeapTuple 和 ExecForceStoreMinimalTuple 可以在 slot 类型不匹配时转换，但更昂贵。
- MakeSingleTupleTableSlot 创建的 slot 不在 EState tuple table 中，需要 ExecDropSingleTupleTableSlot。
- 系统属性访问不是所有 slot 类型都支持，不支持时 slot ops 会报错。
- minimal tuple 适合 tuplestore/sort 等路径，但不是 heap page 上的完整 tuple。
- fixed descriptor slot 不能随意 ExecSetSlotDescriptor。

PostgreSQL executor 的异常路径不能按 C++ 析构模型理解。`ereport(ERROR)` 可能 longjmp 到上层，普通 C 栈上的清理代码不会自然逐层执行。

因此本节所有状态都要问一句：如果这个点 ERROR，已经注册的资源由谁释放，尚未注册的资源是否会泄漏或变成悬挂引用。

## 10. 成本、资源与跨模块传播

- virtual slot 减少 tuple 构造和复制，但可能把 materialize 成本推迟到边界处。
- heap/buffer slot 支持按需 deform，访问少量列时可减少全部属性展开。
- slot_getattr 的 deform 成本与访问到的最大列号相关。
- materialize 和 copy 会增加内存分配与数据复制，但换来清晰 ownership。
- buffer-backed slot 持有 pin 时间过长可能影响 VACUUM、page cleanup 或并发等待。

executor 的成本常常不是单个函数慢，而是边界次数被 workload 放大：每行、每节点、每次 FETCH、每个 trigger、每个 slot clear，都可能成为可见成本。

| 放大因素 | 典型表现 |
| --- | --- |
| tuple 数 | ExecProcNode、slot deform、per-tuple reset 和 receiver 调用被放大。 |
| plan node 数 | ExecInitNode、ExecEndNode、instrumentation 对象和 EXPLAIN 输出被放大。 |
| relation 数 | range table relation 打开、ResultRelInfo、锁和权限检查被放大。 |
| FETCH 次数 | ExecutorRun 边界、DestReceiver startup/shutdown 和 Portal 状态切换被放大。 |
| 副作用数量 | trigger、constraint、FDW flush、ModifyTable drain 被放大。 |

## 11. 观测与诊断入口

- gdb 打印 slot->tts_ops 可判断当前 slot 类型。
- 打印 slot->tts_flags 和 tts_nvalid 可以判断是否 empty、已 deform 到哪一列。
- 断在 ExecStoreBufferHeapTuple 可观察 SeqScan 如何绑定 buffer pin。
- 断在 ExecMaterializeSlot 可观察哪些边界需要复制。
- 断在 ExecResetTupleTable 可验证 End 阶段释放 pin 和 TupleDesc ref。

建议的断点顺序：

1. `TupleTableSlot`
2. `TupleTableSlotOps`
3. `MakeTupleTableSlot`
4. `ExecAllocTableSlot`
5. `ExecStoreVirtualTuple`
6. `ExecStoreBufferHeapTuple`
7. `ExecMaterializeSlot`
8. `ExecResetTupleTable`

断点不只是为了停住程序。每个断点都应该带一个问题：这个状态是否已经初始化，owner 是谁，下一步是否会被改写。

## 12. 课堂实验

### 实验 1

执行 SELECT * FROM t，断在 ExecStoreBufferHeapTuple，打印 slot->tts_ops。

观察点：
- 断点前后同一个字段是否发生变化。
- 这个变化是否发生在本节主流程上。
- 如果把 SQL 改成 cursor、EXPLAIN、DML 或带 trigger，边界是否改变。

### 实验 2

执行 SELECT a+1 FROM t，断在 ExecStoreVirtualTuple，观察 projection result slot。

观察点：
- 断点前后同一个字段是否发生变化。
- 这个变化是否发生在本节主流程上。
- 如果把 SQL 改成 cursor、EXPLAIN、DML 或带 trigger，边界是否改变。

### 实验 3

在 slot_getattr 前后打印 tts_nvalid，观察按需 deform。

观察点：
- 断点前后同一个字段是否发生变化。
- 这个变化是否发生在本节主流程上。
- 如果把 SQL 改成 cursor、EXPLAIN、DML 或带 trigger，边界是否改变。

### 实验 4

执行 ORDER BY 或 HashAgg，观察 minimal slot 在阻塞节点中的使用。

观察点：
- 断点前后同一个字段是否发生变化。
- 这个变化是否发生在本节主流程上。
- 如果把 SQL 改成 cursor、EXPLAIN、DML 或带 trigger，边界是否改变。

### 实验 5

在 ExecMaterializeSlot 设置断点，执行 INSERT INTO target SELECT ... 观察写入前 materialize。

观察点：
- 断点前后同一个字段是否发生变化。
- 这个变化是否发生在本节主流程上。
- 如果把 SQL 改成 cursor、EXPLAIN、DML 或带 trigger，边界是否改变。

### 实验 6

关闭 cursor 前后观察 buffer pin 或 tuple table reset 相关断点。

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

### 练习 1：`TupleTableSlot`

在 `/home/highgo/postgres` 中定位 `TupleTableSlot`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

### 练习 2：`TupleTableSlotOps`

在 `/home/highgo/postgres` 中定位 `TupleTableSlotOps`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

### 练习 3：`MakeTupleTableSlot`

在 `/home/highgo/postgres` 中定位 `MakeTupleTableSlot`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

### 练习 4：`ExecAllocTableSlot`

在 `/home/highgo/postgres` 中定位 `ExecAllocTableSlot`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

### 练习 5：`ExecStoreVirtualTuple`

在 `/home/highgo/postgres` 中定位 `ExecStoreVirtualTuple`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

### 练习 6：`ExecStoreBufferHeapTuple`

在 `/home/highgo/postgres` 中定位 `ExecStoreBufferHeapTuple`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

### 练习 7：`ExecMaterializeSlot`

在 `/home/highgo/postgres` 中定位 `ExecMaterializeSlot`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

### 练习 8：`ExecResetTupleTable`

在 `/home/highgo/postgres` 中定位 `ExecResetTupleTable`，先读调用者，再读被调用者。

记录三件事：入口条件、改变的状态、失败时的 cleanup 或错误传播边界。

如果这个函数只是转发 hook 或 wrapper，也要继续追到标准实现或真实节点函数。

## 14. 常见误区

- 不要把 TupleTableSlot 当成 tuple 本身；它是 tuple 的容器和 ownership 协议。
- 不要认为 virtual slot 一定安全跨节点保存。
- 不要在 slot clear 后继续使用从 slot 取出的指针。
- 不要把 TTS_EMPTY 和所有列为 NULL 混淆。
- 不要绕过 slot ops 直接解释具体 slot 结构，除非已经确认类型。

这些误区的共同点是把 runtime 状态当成静态结构，或者把静态计划当成可以直接推进的状态机。

## 15. 讨论题

1. 为什么 executor 宁愿维护多种 slot ops，也不统一转换成 HeapTuple？
2. 哪些节点最容易因为 slot 生命周期误判产生隐蔽 bug？
3. buffer pin 与 materialize 的边界如何影响长事务和 VACUUM？
4. 如果新增一种 table AM，slot callbacks 需要保证哪些协议？

讨论时要尽量回到一个具体 runtime 现象，比如一次 FETCH、一次 EXPLAIN ANALYZE、一个 trigger 或一个 slot materialize，而不是停留在抽象偏好。

## 16. 本节小结

本节唯一主问题是：TupleTableSlot 为什么同时支持 virtual、heap、minimal、buffer-backed 多种形态，并且要求节点传 slot 而不是直接传 HeapTuple？

可以带走的一句话模型是：TupleTableSlot 是 executor 的 tuple 容器协议；slot ops 定义 clear、deform、materialize、copy 等行为，让上层节点按统一接口消费 tuple，同时把物理表示和资源释放留给具体 slot 类型。

- 源码阅读应先找状态创建点，再找推进点，最后找 cleanup。
- 任何字段都要和 owner、生命周期、异常路径一起理解。
- 观测数据只有能回到具体 runtime 边界时，才适合用于诊断。

后续课程会继续进入 TupleDesc、投影、ExprContext 和具体执行节点。

## 17. 从现象回源码的诊断案例

下面的案例都只服务本节主问题。它们的共同读法是先看现象，再找 runtime 边界，最后回到 owner 和 cleanup。

### 案例 1：第一次执行停在入口但状态还没有建立

现象：

- 你在 `ExecutorRun` 附近停住，看到 SQL 还没有返回，但某些 executor 状态已经改变。
- 这时不要急着沿调用栈往下追完整执行，而要先回到本节主问题：TupleTableSlot 为什么同时支持 virtual、heap、minimal、buffer-backed 多种形态，并且要求节点传 slot 而不是直接传 HeapTuple？
- 如果现象来自 EXPLAIN、cursor、trigger 或扩展 hook，要先确认它落在哪个 executor 生命周期边界。

源码回看：

- 入口函数：`ExecutorRun`。
- 本步主线：ExecInitScanTupleSlot() 或 ExecInitResultTupleSlotTL() 在节点初始化时创建 slot。
- 关键状态：`tts_ops`，语义是：slot 实现类型，决定 clear、getsomeattrs、materialize、copy 行为。
- 相邻状态：`tts_flags`，用来判断这一步之后谁继续持有资源。

判断方式：

- 如果字段刚被写入但还没有注册到 owner，异常路径通常只能靠当前函数局部清理。
- 如果状态已经挂到 EState、PlanState、slot table 或 snapshot manager，后续 cleanup 应该能从 owner 找回。
- 如果这个状态只影响观测，不影响语义，诊断时不要把它解释成执行计划行为改变。
- 如果这个状态影响 tuple 生命周期，必须同时检查 slot、TupleDesc、buffer pin 或 per-tuple context。

### 案例 2：查询只返回少量行但内部状态已经推进

现象：

- 你在 `ExecutorRun` 附近停住，看到 SQL 还没有返回，但某些 executor 状态已经改变。
- 这时不要急着沿调用栈往下追完整执行，而要先回到本节主问题：TupleTableSlot 为什么同时支持 virtual、heap、minimal、buffer-backed 多种形态，并且要求节点传 slot 而不是直接传 HeapTuple？
- 如果现象来自 EXPLAIN、cursor、trigger 或扩展 hook，要先确认它落在哪个 executor 生命周期边界。

源码回看：

- 入口函数：`ExecutorRun`。
- 本步主线：MakeTupleTableSlot() 根据 tupleDesc 和 tts_ops 分配 slot 结构和 Datum/isnull 数组。
- 关键状态：`tts_flags`，语义是：EMPTY、SHOULDFREE、FIXED 等状态位，必须结合 slot ops 解释。
- 相邻状态：`tts_nvalid`，用来判断这一步之后谁继续持有资源。

判断方式：

- 如果字段刚被写入但还没有注册到 owner，异常路径通常只能靠当前函数局部清理。
- 如果状态已经挂到 EState、PlanState、slot table 或 snapshot manager，后续 cleanup 应该能从 owner 找回。
- 如果这个状态只影响观测，不影响语义，诊断时不要把它解释成执行计划行为改变。
- 如果这个状态影响 tuple 生命周期，必须同时检查 slot、TupleDesc、buffer pin 或 per-tuple context。

### 案例 3：EXPLAIN ANALYZE 时间与节点时间不一致

现象：

- 你在 `ExecutorRun` 附近停住，看到 SQL 还没有返回，但某些 executor 状态已经改变。
- 这时不要急着沿调用栈往下追完整执行，而要先回到本节主问题：TupleTableSlot 为什么同时支持 virtual、heap、minimal、buffer-backed 多种形态，并且要求节点传 slot 而不是直接传 HeapTuple？
- 如果现象来自 EXPLAIN、cursor、trigger 或扩展 hook，要先确认它落在哪个 executor 生命周期边界。

源码回看：

- 入口函数：`ExecutorRun`。
- 本步主线：tupleDesc 存在时 slot 标记为 fixed，并 PinTupleDesc。
- 关键状态：`tts_nvalid`，语义是：values/isnull 中已经 deform 出来的属性数量。
- 相邻状态：`tts_tupleDescriptor`，用来判断这一步之后谁继续持有资源。

判断方式：

- 如果字段刚被写入但还没有注册到 owner，异常路径通常只能靠当前函数局部清理。
- 如果状态已经挂到 EState、PlanState、slot table 或 snapshot manager，后续 cleanup 应该能从 owner 找回。
- 如果这个状态只影响观测，不影响语义，诊断时不要把它解释成执行计划行为改变。
- 如果这个状态影响 tuple 生命周期，必须同时检查 slot、TupleDesc、buffer pin 或 per-tuple context。

### 案例 4：cursor 或 receiver 让执行被分成多段

现象：

- 你在 `ExecutorRun` 附近停住，看到 SQL 还没有返回，但某些 executor 状态已经改变。
- 这时不要急着沿调用栈往下追完整执行，而要先回到本节主问题：TupleTableSlot 为什么同时支持 virtual、heap、minimal、buffer-backed 多种形态，并且要求节点传 slot 而不是直接传 HeapTuple？
- 如果现象来自 EXPLAIN、cursor、trigger 或扩展 hook，要先确认它落在哪个 executor 生命周期边界。

源码回看：

- 入口函数：`ExecutorRun`。
- 本步主线：slot->tts_ops->init(slot) 执行类型特定初始化。
- 关键状态：`tts_tupleDescriptor`，语义是：slot 的 tuple 形状，可能被 fixed，也可能运行时设置。
- 相邻状态：`tts_values / tts_isnull`，用来判断这一步之后谁继续持有资源。

判断方式：

- 如果字段刚被写入但还没有注册到 owner，异常路径通常只能靠当前函数局部清理。
- 如果状态已经挂到 EState、PlanState、slot table 或 snapshot manager，后续 cleanup 应该能从 owner 找回。
- 如果这个状态只影响观测，不影响语义，诊断时不要把它解释成执行计划行为改变。
- 如果这个状态影响 tuple 生命周期，必须同时检查 slot、TupleDesc、buffer pin 或 per-tuple context。

### 案例 5：ERROR 发生后没有按普通调用栈返回

现象：

- 你在 `ExecutorRun` 附近停住，看到 SQL 还没有返回，但某些 executor 状态已经改变。
- 这时不要急着沿调用栈往下追完整执行，而要先回到本节主问题：TupleTableSlot 为什么同时支持 virtual、heap、minimal、buffer-backed 多种形态，并且要求节点传 slot 而不是直接传 HeapTuple？
- 如果现象来自 EXPLAIN、cursor、trigger 或扩展 hook，要先确认它落在哪个 executor 生命周期边界。

源码回看：

- 入口函数：`ExecutorRun`。
- 本步主线：SeqScan 从 table AM 拿到物理 tuple 后，可能调用 ExecStoreBufferHeapTuple() 放入 buffer-backed slot。
- 关键状态：`tts_values / tts_isnull`，语义是：virtual tuple 或 deform 后属性数组。
- 相邻状态：`tts_mcxt`，用来判断这一步之后谁继续持有资源。

判断方式：

- 如果字段刚被写入但还没有注册到 owner，异常路径通常只能靠当前函数局部清理。
- 如果状态已经挂到 EState、PlanState、slot table 或 snapshot manager，后续 cleanup 应该能从 owner 找回。
- 如果这个状态只影响观测，不影响语义，诊断时不要把它解释成执行计划行为改变。
- 如果这个状态影响 tuple 生命周期，必须同时检查 slot、TupleDesc、buffer pin 或 per-tuple context。

### 案例 6：扩展 hook 看到的边界与核心 executor 边界不同

现象：

- 你在 `ExecutorRun` 附近停住，看到 SQL 还没有返回，但某些 executor 状态已经改变。
- 这时不要急着沿调用栈往下追完整执行，而要先回到本节主问题：TupleTableSlot 为什么同时支持 virtual、heap、minimal、buffer-backed 多种形态，并且要求节点传 slot 而不是直接传 HeapTuple？
- 如果现象来自 EXPLAIN、cursor、trigger 或扩展 hook，要先确认它落在哪个 executor 生命周期边界。

源码回看：

- 入口函数：`ExecutorRun`。
- 本步主线：Projection 可能把表达式结果写入 Datum/isnull 数组，再调用 ExecStoreVirtualTuple()。
- 关键状态：`tts_mcxt`，语义是：slot 自身和可变数组分配所在 memory context。
- 相邻状态：`buffer-backed tuple`，用来判断这一步之后谁继续持有资源。

判断方式：

- 如果字段刚被写入但还没有注册到 owner，异常路径通常只能靠当前函数局部清理。
- 如果状态已经挂到 EState、PlanState、slot table 或 snapshot manager，后续 cleanup 应该能从 owner 找回。
- 如果这个状态只影响观测，不影响语义，诊断时不要把它解释成执行计划行为改变。
- 如果这个状态影响 tuple 生命周期，必须同时检查 slot、TupleDesc、buffer pin 或 per-tuple context。

## 18. 源码断点与字段记录表

下面这张表用于课堂现场跟读。每一行都应该实际在 gdb 中停一次，记录进入前和返回后的字段差异。

| 序号 | 断点 | 进入前看什么 | 返回后看什么 | 常见误判 |
| --- | --- | --- | --- | --- |
| 1 | `ExecutorRun` | `tts_ops` 是否已经有效 | ExecInitScanTupleSlot() 或 ExecInitResultTupleSlotTL() 在节点初始化时创建 slot。 | 把字段存在误认为语义已经完成 |
| 2 | `ExecutorRun` | `tts_flags` 是否已经有效 | MakeTupleTableSlot() 根据 tupleDesc 和 tts_ops 分配 slot 结构和 Datum/isnull 数组。 | 把字段存在误认为语义已经完成 |
| 3 | `ExecutorRun` | `tts_nvalid` 是否已经有效 | tupleDesc 存在时 slot 标记为 fixed，并 PinTupleDesc。 | 把字段存在误认为语义已经完成 |
| 4 | `ExecutorRun` | `tts_tupleDescriptor` 是否已经有效 | slot->tts_ops->init(slot) 执行类型特定初始化。 | 把字段存在误认为语义已经完成 |
| 5 | `ExecutorRun` | `tts_values / tts_isnull` 是否已经有效 | SeqScan 从 table AM 拿到物理 tuple 后，可能调用 ExecStoreBufferHeapTuple() 放入 buffer-backed slot。 | 把字段存在误认为语义已经完成 |
| 6 | `ExecutorRun` | `tts_mcxt` 是否已经有效 | Projection 可能把表达式结果写入 Datum/isnull 数组，再调用 ExecStoreVirtualTuple()。 | 把字段存在误认为语义已经完成 |
| 7 | `ExecutorRun` | `buffer-backed tuple` 是否已经有效 | 上层节点通过 slot_getattr 或 ExecCopySlot* 访问数据，slot ops 按需 deform 或 copy。 | 把字段存在误认为语义已经完成 |
| 8 | `ExecutorRun` | `materialized tuple` 是否已经有效 | 如果 tuple 要跨越下层 slot 生命周期或 buffer pin 生命周期，调用 ExecMaterializeSlot()。 | 把字段存在误认为语义已经完成 |
| 9 | `ExecutorRun` | `tts_ops` 是否已经有效 | ExecClearTuple(slot) 调用 slot ops clear，只清内容，不清 tuple descriptor。 | 把字段存在误认为语义已经完成 |
| 10 | `ExecutorRun` | `tts_flags` 是否已经有效 | ExecResetTupleTable() 遍历 EState slot 列表，clear、release、ReleaseTupleDesc。 | 把字段存在误认为语义已经完成 |
| 11 | `ExecutorRun` | `tts_nvalid` 是否已经有效 | shouldFree、transfer_pin、materialize 决定 clear 时释放内存还是释放 buffer pin。 | 把字段存在误认为语义已经完成 |
| 12 | `ExecutorRun` | `tts_tupleDescriptor` 是否已经有效 | ExecutorEnd 删除 es_query_cxt 前必须先 reset tuple table，避免 pin/refcount 遗留。 | 把字段存在误认为语义已经完成 |

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
- 只观察一个状态变化：ExecInitScanTupleSlot() 或 ExecInitResultTupleSlotTL() 在节点初始化时创建 slot。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 2：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：MakeTupleTableSlot() 根据 tupleDesc 和 tts_ops 分配 slot 结构和 Datum/isnull 数组。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 3：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：tupleDesc 存在时 slot 标记为 fixed，并 PinTupleDesc。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 4：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：slot->tts_ops->init(slot) 执行类型特定初始化。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 5：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：SeqScan 从 table AM 拿到物理 tuple 后，可能调用 ExecStoreBufferHeapTuple() 放入 buffer-backed slot。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 6：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：Projection 可能把表达式结果写入 Datum/isnull 数组，再调用 ExecStoreVirtualTuple()。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 7：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：上层节点通过 slot_getattr 或 ExecCopySlot* 访问数据，slot ops 按需 deform 或 copy。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 8：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：如果 tuple 要跨越下层 slot 生命周期或 buffer pin 生命周期，调用 ExecMaterializeSlot()。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 9：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：ExecClearTuple(slot) 调用 slot ops clear，只清内容，不清 tuple descriptor。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 10：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：ExecResetTupleTable() 遍历 EState slot 列表，clear、release、ReleaseTupleDesc。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 11：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：shouldFree、transfer_pin、materialize 决定 clear 时释放内存还是释放 buffer pin。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 12：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：ExecutorEnd 删除 es_query_cxt 前必须先 reset tuple table，避免 pin/refcount 遗留。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 13：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：ExecInitScanTupleSlot() 或 ExecInitResultTupleSlotTL() 在节点初始化时创建 slot。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 14：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：MakeTupleTableSlot() 根据 tupleDesc 和 tts_ops 分配 slot 结构和 Datum/isnull 数组。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 15：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：tupleDesc 存在时 slot 标记为 fixed，并 PinTupleDesc。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 16：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：slot->tts_ops->init(slot) 执行类型特定初始化。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 17：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：SeqScan 从 table AM 拿到物理 tuple 后，可能调用 ExecStoreBufferHeapTuple() 放入 buffer-backed slot。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 18：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：Projection 可能把表达式结果写入 Datum/isnull 数组，再调用 ExecStoreVirtualTuple()。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 19：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：上层节点通过 slot_getattr 或 ExecCopySlot* 访问数据，slot ops 按需 deform 或 copy。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 20：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：如果 tuple 要跨越下层 slot 生命周期或 buffer pin 生命周期，调用 ExecMaterializeSlot()。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 21：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：ExecClearTuple(slot) 调用 slot ops clear，只清内容，不清 tuple descriptor。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 22：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：ExecResetTupleTable() 遍历 EState slot 列表，clear、release、ReleaseTupleDesc。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 23：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：shouldFree、transfer_pin、materialize 决定 clear 时释放内存还是释放 buffer pin。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 24：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：ExecutorEnd 删除 es_query_cxt 前必须先 reset tuple table，避免 pin/refcount 遗留。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 25：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：ExecInitScanTupleSlot() 或 ExecInitResultTupleSlotTL() 在节点初始化时创建 slot。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 26：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：MakeTupleTableSlot() 根据 tupleDesc 和 tts_ops 分配 slot 结构和 Datum/isnull 数组。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。

### 复走 27：`ExecutorRun`

- 重新从 `ExecutorRun` 进入源码。
- 只观察一个状态变化：tupleDesc 存在时 slot 标记为 fixed，并 PinTupleDesc。
- 写下 owner、cleanup 和观测入口。
- 如果这一轮读到了第二个主问题，把它留给后续课程。
## 20. 一页纸定位模板

这部分用于把课程内容压缩成排查动作。不要从这里开始背函数名；先带着现象进入，再沿状态边界回源码。

### 定位 1：执行还没返回，但 gdb 已经停在 executor 内部

- 先停在 `TupleTableSlot`，确认当前调用属于 `PostgreSQL TupleTableSlot 的虚拟 tuple 与物理 tuple 边界` 的哪一个生命周期边界。
- 打印 `tts_ops`，不要只看字段值，还要解释它的语义：slot 实现类型，决定 clear、getsomeattrs、materialize、copy 行为。
- 向上追一层调用者，确认是谁要求 executor 进入这个边界。
- 向下追一层被调用者，确认这一步是否注册了资源、改写了状态或返回了 slot。
- 如果现象来自观测指标，判断指标采样是在进入本步之前、执行过程中，还是返回之后。
- 如果现象来自异常路径，判断资源是否已经挂到 owner；没有 owner 的对象必须由当前路径处理。

### 定位 2：EXPLAIN ANALYZE 的时间和直觉不一致

- 先停在 `TupleTableSlotOps`，确认当前调用属于 `PostgreSQL TupleTableSlot 的虚拟 tuple 与物理 tuple 边界` 的哪一个生命周期边界。
- 打印 `tts_flags`，不要只看字段值，还要解释它的语义：EMPTY、SHOULDFREE、FIXED 等状态位，必须结合 slot ops 解释。
- 向上追一层调用者，确认是谁要求 executor 进入这个边界。
- 向下追一层被调用者，确认这一步是否注册了资源、改写了状态或返回了 slot。
- 如果现象来自观测指标，判断指标采样是在进入本步之前、执行过程中，还是返回之后。
- 如果现象来自异常路径，判断资源是否已经挂到 owner；没有 owner 的对象必须由当前路径处理。

### 定位 3：cursor、FETCH 或客户端输出让执行分段

- 先停在 `MakeTupleTableSlot`，确认当前调用属于 `PostgreSQL TupleTableSlot 的虚拟 tuple 与物理 tuple 边界` 的哪一个生命周期边界。
- 打印 `tts_nvalid`，不要只看字段值，还要解释它的语义：values/isnull 中已经 deform 出来的属性数量。
- 向上追一层调用者，确认是谁要求 executor 进入这个边界。
- 向下追一层被调用者，确认这一步是否注册了资源、改写了状态或返回了 slot。
- 如果现象来自观测指标，判断指标采样是在进入本步之前、执行过程中，还是返回之后。
- 如果现象来自异常路径，判断资源是否已经挂到 owner；没有 owner 的对象必须由当前路径处理。

### 定位 4：ERROR 后怀疑 executor 资源没有按预期释放

- 先停在 `ExecAllocTableSlot`，确认当前调用属于 `PostgreSQL TupleTableSlot 的虚拟 tuple 与物理 tuple 边界` 的哪一个生命周期边界。
- 打印 `tts_tupleDescriptor`，不要只看字段值，还要解释它的语义：slot 的 tuple 形状，可能被 fixed，也可能运行时设置。
- 向上追一层调用者，确认是谁要求 executor 进入这个边界。
- 向下追一层被调用者，确认这一步是否注册了资源、改写了状态或返回了 slot。
- 如果现象来自观测指标，判断指标采样是在进入本步之前、执行过程中，还是返回之后。
- 如果现象来自异常路径，判断资源是否已经挂到 owner；没有 owner 的对象必须由当前路径处理。

### 定位 5：扩展 hook 或 profiler 捕获到一个边界事件

- 先停在 `ExecStoreVirtualTuple`，确认当前调用属于 `PostgreSQL TupleTableSlot 的虚拟 tuple 与物理 tuple 边界` 的哪一个生命周期边界。
- 打印 `tts_values / tts_isnull`，不要只看字段值，还要解释它的语义：virtual tuple 或 deform 后属性数组。
- 向上追一层调用者，确认是谁要求 executor 进入这个边界。
- 向下追一层被调用者，确认这一步是否注册了资源、改写了状态或返回了 slot。
- 如果现象来自观测指标，判断指标采样是在进入本步之前、执行过程中，还是返回之后。
- 如果现象来自异常路径，判断资源是否已经挂到 owner；没有 owner 的对象必须由当前路径处理。

### 定位 6：源码阅读时分不清字段属于 Plan 还是 runtime state

- 先停在 `ExecStoreBufferHeapTuple`，确认当前调用属于 `PostgreSQL TupleTableSlot 的虚拟 tuple 与物理 tuple 边界` 的哪一个生命周期边界。
- 打印 `tts_mcxt`，不要只看字段值，还要解释它的语义：slot 自身和可变数组分配所在 memory context。
- 向上追一层调用者，确认是谁要求 executor 进入这个边界。
- 向下追一层被调用者，确认这一步是否注册了资源、改写了状态或返回了 slot。
- 如果现象来自观测指标，判断指标采样是在进入本步之前、执行过程中，还是返回之后。
- 如果现象来自异常路径，判断资源是否已经挂到 owner；没有 owner 的对象必须由当前路径处理。

定位结束时只保留三句话：

1. 这个现象落在哪个 executor 生命周期边界。
2. 这个边界改变了哪个核心状态，以及 owner 是谁。
3. 正常路径和 ERROR 路径分别由哪个函数或机制收尾。
