# PostgreSQL 从 profiler 栈回到执行器热点

## 课程定位

前置知识：已经能从 Buffers 和 wait event 判断 I/O 与等待；本节处理 backend active 但不等待的 CPU 热点。

本节唯一主问题：

```text
如何用 perf、gprof、eBPF 或 extension profiler 的栈样本，把 CPU 热点映射回表达式求值、tuple deform、hash compare、sort compare、JIT 或节点调度？
```

核心矛盾：profiler 给的是函数栈和采样比例，执行器诊断需要的是 plan node、tuple 流、表达式、数据分布和 runtime state；两者天然不在同一层。

学完后应能看到一个 C symbol 热点后，判断它属于哪个 executor 节点边界，以及还需要哪类 SQL/EXPLAIN 证据才能闭环。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

前两节分别处理 I/O 和等待。
如果慢 SQL 既没有明显 wait event，也不能由 Buffers 解释，通常要进入 CPU profile。
profile 栈不会告诉你 SQL 的语义。
它只告诉你采样时 CPU 正在哪些符号附近消耗。
本节唯一主问题是把这些符号重新映射回 executor 的节点、表达式、tuple 和算法状态。
下一节会基于这个思路设计一个轻量 profiler extension。

```text
active backend pid
  -> profiler samples
  -> C symbols
  -> executor subsystem
  -> PlanState / ExprState / TupleTableSlot
  -> EXPLAIN node and workload fact
```

本节只处理自己这一环，不把相邻主题扩成第二个主问题。

阅读时要保持一条线：先从用户可见现象进入，再回到状态写入点，最后再回到可验证实验。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
profiler 先给出 backend 的采样栈；我们按符号族把样本归到 `ExecProcNode` 调度、表达式解释/JIT、tuple deform、hash/sort 比较、table/index scan、内存分配等边界；再用 EXPLAIN 和源码确认这些 CPU 是否服务同一个 plan node。
```

这里的 tension 是：采样越低侵入，语义越少；语义越多，插桩成本越高。

本节只问一个问题：在不先写扩展的情况下，如何把外部 profiler 的栈样本解释成执行器热点。

这个模型的诊断含义是：

- 不要先背函数名，先判断状态属于哪个生命周期边界。

- 不要把用户可见指标当成源码真相，指标只是某个状态边界的投影。

- 不要把一次采样、一次累计值和一次执行剖面混在一起解释。

- 每个观测值都要回答：谁写入，何时可见，谁清理，异常时是否仍可靠。

本节后面的源码 walkthrough 都围绕这一个模型展开。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/executor/executor.h` | `ExecProcNode()` inline 调度边界。 |
| 2 | `src/backend/executor/execProcnode.c` | `ExecInitNode()`、`ExecSetExecProcNode()`、`ExecProcNodeFirst()` 的节点函数安装。 |
| 3 | `src/backend/executor/instrument.c` | `ExecProcNodeInstr()`、`InstrStartNode()`、`InstrStopNode()` 的节点级计时 wrapper。 |
| 4 | `src/backend/executor/execExprInterp.c` | 表达式解释执行热点，例如 qual、projection、函数调用。 |
| 5 | `src/backend/executor/execExpr.c` | 表达式状态初始化和编译边界。 |
| 6 | `src/backend/executor/execTuples.c` | TupleTableSlot 存储、materialize、deform 边界。 |
| 7 | `src/backend/access/common/heaptuple.c` | `heap_deform_tuple()`、`heap_form_tuple()` 等 tuple 形态转换。 |
| 8 | `src/backend/executor/nodeHash.c` | hash table build/probe、hash compare、batch/spill 相关 CPU。 |
| 9 | `src/backend/utils/sort/tuplesort.c` | Sort 和 comparison 热点来源。 |
| 10 | `src/backend/executor/nodeAgg.c` | aggregate transition、hash/sorted aggregation 热点。 |
| 11 | `src/backend/jit/llvm/llvmjit_expr.c` | JIT 表达式生成后热点的源码边界。 |

阅读顺序按 mental model 排列，不按文件名排序。

建议先从入口和状态结构读起，再追 ownership、cleanup、异常路径和观测输出。

本节源码核对基线：

```text
cd /home/highgo/postgres
git rev-parse --abbrev-ref HEAD
git rev-parse HEAD
```

期望分支为 `master`，提交为本课开头列出的完整提交号。

## 4. 关键数据结构与状态

### 4.1. `PlanState.ExecProcNode`

每个执行节点实际拉取 tuple 的函数指针。

它的持有者是：ExecInitNode 和节点初始化代码。

它的读取者是：ExecProcNode inline 调用者。

生命周期边界是：PlanState 生命周期内可被 ExecSetExecProcNode 更新。

诊断价值是：把栈上的 `ExecSeqScan`、`ExecHashJoin` 等符号映射到 plan node。

读这个状态时不要只看字段名。

先问它在 backend-local、shared memory、DSM、文件系统还是调用栈里。

再问它的写入者是否和读取者在同一条时间线上。

最后问它失败时会留下错误值、缺失值，还是只能让上层退化成推断。

在本节唯一主问题中，它只服务于：如何用 perf、gprof、eBPF 或 extension profiler 的栈样本，把 CPU 热点映射回表达式求值、tuple deform、hash compare、sort compare、JIT 或节点调度？

如果这个状态不能解释这一点，就不应该把它扩成另一条主线。

### 4.2. `PlanState.ExecProcNodeReal`

去掉首调用和 instrumentation wrapper 后的真实节点函数。

它的持有者是：ExecSetExecProcNode。

它的读取者是：ExecProcNodeFirst / ExecProcNodeInstr。

生命周期边界是：节点函数改变时重新安装 wrapper。

诊断价值是：解释 profile 中 wrapper 和 real function 的区别。

读这个状态时不要只看字段名。

先问它在 backend-local、shared memory、DSM、文件系统还是调用栈里。

再问它的写入者是否和读取者在同一条时间线上。

最后问它失败时会留下错误值、缺失值，还是只能让上层退化成推断。

在本节唯一主问题中，它只服务于：如何用 perf、gprof、eBPF 或 extension profiler 的栈样本，把 CPU 热点映射回表达式求值、tuple deform、hash compare、sort compare、JIT 或节点调度？

如果这个状态不能解释这一点，就不应该把它扩成另一条主线。

### 4.3. `ExprState`

表达式执行状态，承载 qual、projection、targetlist、函数调用步骤。

它的持有者是：ExecInitExpr 等初始化路径。

它的读取者是：execExprInterp 或 JIT 生成代码。

生命周期边界是：随 PlanState / EState 释放。

诊断价值是：把 CPU 热点归因到过滤、投影或计算表达式。

读这个状态时不要只看字段名。

先问它在 backend-local、shared memory、DSM、文件系统还是调用栈里。

再问它的写入者是否和读取者在同一条时间线上。

最后问它失败时会留下错误值、缺失值，还是只能让上层退化成推断。

在本节唯一主问题中，它只服务于：如何用 perf、gprof、eBPF 或 extension profiler 的栈样本，把 CPU 热点映射回表达式求值、tuple deform、hash compare、sort compare、JIT 或节点调度？

如果这个状态不能解释这一点，就不应该把它扩成另一条主线。

### 4.4. `TupleTableSlot`

节点之间传递 tuple 的容器，可能是 virtual、heap、minimal、buffer-backed。

它的持有者是：执行节点和 slot 工具函数。

它的读取者是：上层节点、表达式、projection。

生命周期边界是：随 tuple table 和节点生命周期清理。

诊断价值是：解释 tuple deform/materialize 热点。

读这个状态时不要只看字段名。

先问它在 backend-local、shared memory、DSM、文件系统还是调用栈里。

再问它的写入者是否和读取者在同一条时间线上。

最后问它失败时会留下错误值、缺失值，还是只能让上层退化成推断。

在本节唯一主问题中，它只服务于：如何用 perf、gprof、eBPF 或 extension profiler 的栈样本，把 CPU 热点映射回表达式求值、tuple deform、hash compare、sort compare、JIT 或节点调度？

如果这个状态不能解释这一点，就不应该把它扩成另一条主线。

### 4.5. `NodeInstrumentation`

可选的节点时间/rows/buffer 统计。

它的持有者是：ExecutorStart / ExecInitNode。

它的读取者是：EXPLAIN、auto_explain、扩展。

生命周期边界是：ExecutorEnd 前可读。

诊断价值是：用 EXPLAIN 时间校验 profiler 归因。

读这个状态时不要只看字段名。

先问它在 backend-local、shared memory、DSM、文件系统还是调用栈里。

再问它的写入者是否和读取者在同一条时间线上。

最后问它失败时会留下错误值、缺失值，还是只能让上层退化成推断。

在本节唯一主问题中，它只服务于：如何用 perf、gprof、eBPF 或 extension profiler 的栈样本，把 CPU 热点映射回表达式求值、tuple deform、hash compare、sort compare、JIT 或节点调度？

如果这个状态不能解释这一点，就不应该把它扩成另一条主线。

### 4.6. `JitInstrumentation`

JIT 编译和执行相关统计。

它的持有者是：JIT provider 和 executor。

它的读取者是：EXPLAIN JIT 输出。

生命周期边界是：查询结束汇总。

诊断价值是：区分编译成本和生成代码执行成本。

读这个状态时不要只看字段名。

先问它在 backend-local、shared memory、DSM、文件系统还是调用栈里。

再问它的写入者是否和读取者在同一条时间线上。

最后问它失败时会留下错误值、缺失值，还是只能让上层退化成推断。

在本节唯一主问题中，它只服务于：如何用 perf、gprof、eBPF 或 extension profiler 的栈样本，把 CPU 热点映射回表达式求值、tuple deform、hash compare、sort compare、JIT 或节点调度？

如果这个状态不能解释这一点，就不应该把它扩成另一条主线。

### 4.7. `tuplesort / hash table private state`

Sort/Hash/Agg 算法私有状态。

它的持有者是：对应 blocking node。

它的读取者是：节点执行函数和 EXPLAIN 输出。

生命周期边界是：节点结束或 rescan 时清理。

诊断价值是：把比较函数、hash 函数热点映射到数据分布和 work_mem。

读这个状态时不要只看字段名。

先问它在 backend-local、shared memory、DSM、文件系统还是调用栈里。

再问它的写入者是否和读取者在同一条时间线上。

最后问它失败时会留下错误值、缺失值，还是只能让上层退化成推断。

在本节唯一主问题中，它只服务于：如何用 perf、gprof、eBPF 或 extension profiler 的栈样本，把 CPU 热点映射回表达式求值、tuple deform、hash compare、sort compare、JIT 或节点调度？

如果这个状态不能解释这一点，就不应该把它扩成另一条主线。

本节最重要的不变量是：

```text
raw field 不是语义；
field + writer + reader + lifecycle + cleanup 才是诊断语义。
```

## 5. 主流程源码 walkthrough

下面按一次慢查询诊断或 profiler 采样的时间线走主流程。

```text
定位 backend pid
采样 CPU 栈
按符号族初分
回到 PlanState 调度
检查表达式状态
检查 tuple 形态
检查算法私有状态
形成诊断闭环
```

### 5.1. 定位 backend pid

先确定慢 SQL 对应的 backend，而不是对整个系统平均采样。

这一段改变的状态边界是：进程边界。

回到诊断时要验证：pid 是否仍在运行同一条 SQL。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

### 5.2. 采样 CPU 栈

用 perf、eBPF 或 gprof 采到符号栈，保留用户态符号和调用路径。

这一段改变的状态边界是：外部观测边界。

回到诊断时要验证：采样时间窗口是否覆盖慢 SQL 主体。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

### 5.3. 按符号族初分

把 `Exec*`、`ExecInterp*`、`slot_deform`、`hash`、`tuplesort`、`llvm` 等符号归类。

这一段改变的状态边界是：源码子系统边界。

回到诊断时要验证：热点是否集中到一类函数。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

### 5.4. 回到 PlanState 调度

从 `ExecProcNode()` / 节点函数确认热点属于哪个 plan node 类型。

这一段改变的状态边界是：执行器节点边界。

回到诊断时要验证：EXPLAIN 中是否存在对应节点且 loops/rows 匹配。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

### 5.5. 检查表达式状态

若热点在 `execExprInterp.c` 或 JIT，回到 qual、targetlist、函数调用和数据类型。

这一段改变的状态边界是：表达式执行边界。

回到诊断时要验证：过滤率、函数成本、投影宽度是否解释热点。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

### 5.6. 检查 tuple 形态

若热点在 deform/materialize，回到 slot 类型、tuple 宽度、varlena、投影需求。

这一段改变的状态边界是：tuple 表示边界。

回到诊断时要验证：宽行、重复投影或跨节点 materialize 是否存在。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

### 5.7. 检查算法私有状态

若热点在 hash/sort/agg，回到 batch、spill、compare、transition state。

这一段改变的状态边界是：blocking node / 算法状态边界。

回到诊断时要验证：EXPLAIN 的 Sort Method、Hash Batches、Memory/Disk 是否支持。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

### 5.8. 形成诊断闭环

把 stack percentage、EXPLAIN node、源码状态和可复现实验放在同一结论里。

这一段改变的状态边界是：runtime -> reusable abstraction。

回到诊断时要验证：是否还有未验证的 I/O 或 wait 解释。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

## 6. 生命周期 / ownership / cleanup

### 6.1. profile session

创建或进入者：外部 profiler 或扩展启动采样。

正常清理者：采样结束释放数据。

异常路径依赖：采样中断不应影响 backend。

这里要区分三件事。

MemoryContext 管内存批量释放。

ResourceOwner 管 pin、refcount、文件句柄、锁等外部资源。

全局或共享状态必须有独立的可见性、锁或代际边界。

把这些混成一句“事务结束释放”，会丢掉定位问题所需的时间顺序。

### 6.2. PlanState tree

创建或进入者：ExecutorStart / ExecInitNode 创建。

正常清理者：ExecutorEnd / ExecEndNode 清理。

异常路径依赖：ERROR 由 executor teardown 和 memory context 兜底。

这里要区分三件事。

MemoryContext 管内存批量释放。

ResourceOwner 管 pin、refcount、文件句柄、锁等外部资源。

全局或共享状态必须有独立的可见性、锁或代际边界。

把这些混成一句“事务结束释放”，会丢掉定位问题所需的时间顺序。

### 6.3. expression state

创建或进入者：ExecInitExpr 构建。

正常清理者：随 PlanState / EState 释放。

异常路径依赖：函数调用异常需要回到 executor context 清理。

这里要区分三件事。

MemoryContext 管内存批量释放。

ResourceOwner 管 pin、refcount、文件句柄、锁等外部资源。

全局或共享状态必须有独立的可见性、锁或代际边界。

把这些混成一句“事务结束释放”，会丢掉定位问题所需的时间顺序。

### 6.4. slot storage

创建或进入者：节点初始化或 tuple table 创建。

正常清理者：ExecClearTuple / ExecResetTupleTable。

异常路径依赖：buffer-backed slot 还涉及 pin release。

这里要区分三件事。

MemoryContext 管内存批量释放。

ResourceOwner 管 pin、refcount、文件句柄、锁等外部资源。

全局或共享状态必须有独立的可见性、锁或代际边界。

把这些混成一句“事务结束释放”，会丢掉定位问题所需的时间顺序。

### 6.5. JIT code

创建或进入者：表达式 JIT 或 deform JIT 生成。

正常清理者：JIT context / query end 清理。

异常路径依赖：编译失败可 fallback 到解释执行。

这里要区分三件事。

MemoryContext 管内存批量释放。

ResourceOwner 管 pin、refcount、文件句柄、锁等外部资源。

全局或共享状态必须有独立的可见性、锁或代际边界。

把这些混成一句“事务结束释放”，会丢掉定位问题所需的时间顺序。

### 6.6. profiling conclusion

创建或进入者：采样和 EXPLAIN 合并形成。

正常清理者：随诊断报告保存，不进入内核状态。

异常路径依赖：结论必须标注采样窗口和 GUC。

这里要区分三件事。

MemoryContext 管内存批量释放。

ResourceOwner 管 pin、refcount、文件句柄、锁等外部资源。

全局或共享状态必须有独立的可见性、锁或代际边界。

把这些混成一句“事务结束释放”，会丢掉定位问题所需的时间顺序。

## 7. 正确性机制层次

本节的正确性不是一个机制单独保证的。

下面这些层次组合起来，才让观测结果既能靠近现场，又不破坏执行语义。

| 层次 | 机制 | 本节不变量 | 常见误读 |
| --- | --- | --- | --- |
| 采样边界 | perf/eBPF/gprof | 样本近似表示 CPU 时间分布。 | 样本百分比就是精确耗时。 |
| 节点边界 | `ExecProcNode` and PlanState | 节点函数代表 tuple 拉取协议。 | 栈上一个函数名等于完整节点成本。 |
| 表达式边界 | ExprState / execExprInterp / JIT | 表达式热点要回到 qual/projection/function。 | 所有 CPU 都是 planner 估计错误。 |
| tuple 边界 | TupleTableSlot / deform | tuple 表示影响 CPU 成本。 | Buffers hit 高就没有存储相关 CPU。 |
| 算法边界 | Sort/Hash/Agg private state | 比较、hash、transition 会随数据分布放大。 | 只调 work_mem 一定解决。 |
| 验证边界 | EXPLAIN + experiment | profile 结论需要 runtime 事实支持。 | 采样栈可以单独作为根因。 |

读源码时可以按这个顺序检查：

```text
先找写入点
  -> 再找读取点
  -> 再找清理点
  -> 再看 ERROR / abort / backend exit 是否能兜底
  -> 最后把用户可见输出映射回这一条状态链
```

1. `采样边界` 这一层主要依赖 perf/eBPF/gprof。

它保证的是：样本近似表示 CPU 时间分布。

不要把它误读成：样本百分比就是精确耗时。

如果一次诊断只能看到这一层的投影，就必须保留不确定性。

2. `节点边界` 这一层主要依赖 `ExecProcNode` and PlanState。

它保证的是：节点函数代表 tuple 拉取协议。

不要把它误读成：栈上一个函数名等于完整节点成本。

如果一次诊断只能看到这一层的投影，就必须保留不确定性。

3. `表达式边界` 这一层主要依赖 ExprState / execExprInterp / JIT。

它保证的是：表达式热点要回到 qual/projection/function。

不要把它误读成：所有 CPU 都是 planner 估计错误。

如果一次诊断只能看到这一层的投影，就必须保留不确定性。

4. `tuple 边界` 这一层主要依赖 TupleTableSlot / deform。

它保证的是：tuple 表示影响 CPU 成本。

不要把它误读成：Buffers hit 高就没有存储相关 CPU。

如果一次诊断只能看到这一层的投影，就必须保留不确定性。

5. `算法边界` 这一层主要依赖 Sort/Hash/Agg private state。

它保证的是：比较、hash、transition 会随数据分布放大。

不要把它误读成：只调 work_mem 一定解决。

如果一次诊断只能看到这一层的投影，就必须保留不确定性。

6. `验证边界` 这一层主要依赖 EXPLAIN + experiment。

它保证的是：profile 结论需要 runtime 事实支持。

不要把它误读成：采样栈可以单独作为根因。

如果一次诊断只能看到这一层的投影，就必须保留不确定性。

## 8. 错误路径 / 异常路径 / fallback

### 8.1. 符号缺失

可见现象：profile 里只有地址或内核符号。

源码上应先回到：编译符号、perf map、JIT dump 设置。

正确处理方式是：先修复符号化，再解释。

这类路径通常不是“继续收集更多指标”就能解决。

要先判断观测值是在失败前、失败中还是失败后写入。

如果写入点本身在异常路径里，就要检查 cleanup 是否仍然执行。

### 8.2. 热点在 ExecProcNode wrapper

可见现象：wrapper 占比异常。

源码上应先回到：`instrument.c`。

正确处理方式是：检查是否无条件开启 timing 或 profiler wrapper。

这类路径通常不是“继续收集更多指标”就能解决。

要先判断观测值是在失败前、失败中还是失败后写入。

如果写入点本身在异常路径里，就要检查 cleanup 是否仍然执行。

### 8.3. 热点在 execExprInterp

可见现象：表达式解释器占比高。

源码上应先回到：`execExprInterp.c`、`execExpr.c`。

正确处理方式是：回到 qual、函数、类型转换、投影。

这类路径通常不是“继续收集更多指标”就能解决。

要先判断观测值是在失败前、失败中还是失败后写入。

如果写入点本身在异常路径里，就要检查 cleanup 是否仍然执行。

### 8.4. 热点在 deform

可见现象：slot/heap tuple deform 占比高。

源码上应先回到：`execTuples.c`、`heaptuple.c`。

正确处理方式是：检查宽行、列访问模式、slot materialize。

这类路径通常不是“继续收集更多指标”就能解决。

要先判断观测值是在失败前、失败中还是失败后写入。

如果写入点本身在异常路径里，就要检查 cleanup 是否仍然执行。

### 8.5. 热点在 tuplesort/hash

可见现象：比较或 hash 函数占比高。

源码上应先回到：`tuplesort.c`、`nodeHash.c`。

正确处理方式是：结合数据分布、collation、batch、spill。

这类路径通常不是“继续收集更多指标”就能解决。

要先判断观测值是在失败前、失败中还是失败后写入。

如果写入点本身在异常路径里，就要检查 cleanup 是否仍然执行。

## 9. 成本、资源与跨模块传播

成本分析要围绕同一个主问题。

不要把所有可能成本都列一遍。

### 9.1. 采样开销

主要扩张因子：采样频率和栈深。

放大方式：高频采样会扰动 CPU-bound 查询。

控制办法：线上使用低频和短窗口。

诊断时不要只看绝对值。

要看它是否随 rows、loops、backend 数、buffer miss、计划节点深度或采样率线性放大。

如果它只在某个节点或某类等待里集中出现，说明问题边界比 SQL 文本更窄。

### 9.2. 符号化成本

主要扩张因子：debug symbols、JIT maps、kernel config。

放大方式：无符号会导致误判。

控制办法：先保证符号质量。

诊断时不要只看绝对值。

要看它是否随 rows、loops、backend 数、buffer miss、计划节点深度或采样率线性放大。

如果它只在某个节点或某类等待里集中出现，说明问题边界比 SQL 文本更窄。

### 9.3. 节点调度

主要扩张因子：节点深度和 tuple 数。

放大方式：小 tuple 高频拉取放大 wrapper 和函数指针成本。

控制办法：看 rows/loops 和轻节点组合。

诊断时不要只看绝对值。

要看它是否随 rows、loops、backend 数、buffer miss、计划节点深度或采样率线性放大。

如果它只在某个节点或某类等待里集中出现，说明问题边界比 SQL 文本更窄。

### 9.4. 表达式解释

主要扩张因子：qual 数、函数调用、类型转换。

放大方式：每行重复执行。

控制办法：用 projection/qual 简化实验验证。

诊断时不要只看绝对值。

要看它是否随 rows、loops、backend 数、buffer miss、计划节点深度或采样率线性放大。

如果它只在某个节点或某类等待里集中出现，说明问题边界比 SQL 文本更窄。

### 9.5. tuple deform

主要扩张因子：宽行、访问列数、varlena。

放大方式：每行取列时放大 CPU。

控制办法：只取必要列或覆盖索引对比。

诊断时不要只看绝对值。

要看它是否随 rows、loops、backend 数、buffer miss、计划节点深度或采样率线性放大。

如果它只在某个节点或某类等待里集中出现，说明问题边界比 SQL 文本更窄。

### 9.6. JIT

主要扩张因子：编译阈值和表达式复杂度。

放大方式：编译成本和执行收益可能错配。

控制办法：对比 jit on/off 和 EXPLAIN JIT 输出。

诊断时不要只看绝对值。

要看它是否随 rows、loops、backend 数、buffer miss、计划节点深度或采样率线性放大。

如果它只在某个节点或某类等待里集中出现，说明问题边界比 SQL 文本更窄。

## 10. 观测与诊断入口

观测入口要分清当前值、累计值、单次执行剖面和外部采样。

同一个慢 SQL 结论最好至少由两类信号互相校验。

### 10.1. perf top/record

能看到：CPU 热点符号和调用栈。

看不到：SQL 语义和 plan node id。

源码入口：外部 profiler。

使用时先问它是当前状态、累计状态，还是本次执行状态。

再问它是否会因为采样、刷新延迟、并行 worker 汇总或输出格式化而偏移。

最后把结论回扣到本节主流程中的某个写入点。

### 10.2. EXPLAIN ANALYZE

能看到：节点 rows/loops/time。

看不到：函数级 CPU 分布。

源码入口：`explain.c`、`instrument.c`。

使用时先问它是当前状态、累计状态，还是本次执行状态。

再问它是否会因为采样、刷新延迟、并行 worker 汇总或输出格式化而偏移。

最后把结论回扣到本节主流程中的某个写入点。

### 10.3. auto_explain

能看到：慢查询计划和 runtime 指标。

看不到：持续 profiler 栈。

源码入口：`contrib/auto_explain/auto_explain.c`。

使用时先问它是当前状态、累计状态，还是本次执行状态。

再问它是否会因为采样、刷新延迟、并行 worker 汇总或输出格式化而偏移。

最后把结论回扣到本节主流程中的某个写入点。

### 10.4. pg_stat_statements

能看到：长期 query id 热点。

看不到：节点级或符号级细节。

源码入口：`pg_stat_statements.c`。

使用时先问它是当前状态、累计状态，还是本次执行状态。

再问它是否会因为采样、刷新延迟、并行 worker 汇总或输出格式化而偏移。

最后把结论回扣到本节主流程中的某个写入点。

### 10.5. gdb

能看到：现场栈和状态结构。

看不到：统计意义。

源码入口：PlanState / ExprState 调试。

使用时先问它是当前状态、累计状态，还是本次执行状态。

再问它是否会因为采样、刷新延迟、并行 worker 汇总或输出格式化而偏移。

最后把结论回扣到本节主流程中的某个写入点。

### 10.6. JIT EXPLAIN

能看到：JIT 编译阶段时间。

看不到：生成代码内部细粒度热点。

源码入口：`llvmjit_expr.c`、EXPLAIN JIT 输出。

使用时先问它是当前状态、累计状态，还是本次执行状态。

再问它是否会因为采样、刷新延迟、并行 worker 汇总或输出格式化而偏移。

最后把结论回扣到本节主流程中的某个写入点。

## 11. 常见误区

下面这些误区都来自把观测值当成源码事实。

误区一：看到一个指标就直接给结论。

正确做法是先找到写入点、读取点和清理点。

误区二：把单次执行剖面当成长期平均。

正确做法是把 EXPLAIN、pg_stat、wait event 和 profiler 的时间窗口写清楚。

误区三：把父节点时间当成父节点独占 CPU。

正确做法是区分 inclusive、exclusive、loops 和子节点成本。

误区四：把没有观测值解释成没有问题。

正确做法是检查是否关闭开关、未命中采样、被阈值过滤、被 ring 覆盖或权限隐藏。

误区五：把当前源码实现说成跨版本契约。

正确做法是写明本课基于 `/home/highgo/postgres` 的 `master` 分支和 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`。

## 12. 课堂实验

### 12.1. 表达式热点

实验目标：把可见现象压回源码状态边界。

建议准备：

```text
EXPLAIN (ANALYZE) SELECT count(*) FROM t WHERE expensive_func(a);
对 backend pid 采样 perf。
替换为简单谓词后重跑。
```

预期现象：热点应从函数调用或 execExprInterp 下降。

回到源码时检查：`execExprInterp.c` 和函数调用栈。

实验完成后不要只记录数值。

还要记录计划节点、GUC、并发状态、缓存冷热和是否开启 timing。

这些条件决定同一个观测值能否迁移到另一个现场。

### 12.2. tuple deform 热点

实验目标：把可见现象压回源码状态边界。

建议准备：

```text
SELECT many_columns FROM wide_table;
对比 SELECT only_one_column FROM wide_table;
采样 backend CPU 栈。
```

预期现象：宽行多列访问更容易出现 deform/materialize 热点。

回到源码时检查：`execTuples.c`、`heaptuple.c`。

实验完成后不要只记录数值。

还要记录计划节点、GUC、并发状态、缓存冷热和是否开启 timing。

这些条件决定同一个观测值能否迁移到另一个现场。

### 12.3. Sort compare 热点

实验目标：把可见现象压回源码状态边界。

建议准备：

```text
EXPLAIN (ANALYZE) SELECT * FROM t ORDER BY text_col;
换成 int_col 或调整 collation 后对比。
```

预期现象：比较函数和 collation 可能进入 CPU 热点。

回到源码时检查：`tuplesort.c` 和类型比较函数。

实验完成后不要只记录数值。

还要记录计划节点、GUC、并发状态、缓存冷热和是否开启 timing。

这些条件决定同一个观测值能否迁移到另一个现场。

### 12.4. JIT on/off 对比

实验目标：把可见现象压回源码状态边界。

建议准备：

```text
SET jit = off;
运行复杂表达式聚合。
SET jit = on;
对比 EXPLAIN JIT 和 profiler。
```

预期现象：JIT 可能降低执行 CPU，也可能被编译成本抵消。

回到源码时检查：`llvmjit_expr.c` 和 EXPLAIN JIT。

实验完成后不要只记录数值。

还要记录计划节点、GUC、并发状态、缓存冷热和是否开启 timing。

这些条件决定同一个观测值能否迁移到另一个现场。

## 13. 讨论题

1. 为什么 profile 栈不能直接替代 EXPLAIN？

2. 热点在 `ExecProcNodeInstr` 时应先怀疑什么？

3. tuple deform 热点和 I/O hit 高之间有什么关系？

4. JIT 编译时间和生成代码执行时间如何区分？

5. 如何把一次 profile 结论变成可复现的 SQL 实验？

## 14. 本节小结

本节只沉淀一个模型：

```text
CPU profile 给函数栈，执行器诊断要把函数栈映射回 PlanState 和 runtime state。
先排除 I/O 和 wait，再解释 active CPU 热点更稳。
表达式、tuple deform、hash/sort、JIT 是常见 CPU 热点族。
wrapper 或 instrumentation 热点可能是观测扰动。
profile 结论必须用 EXPLAIN 和实验闭环。
```

下一节开始设计只观测不改语义的轻量 profiler extension。
