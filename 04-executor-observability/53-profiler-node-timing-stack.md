# PostgreSQL Profiler 节点级计时与嵌套调用栈

## 课程定位

前置知识：已经有最小 executor hook profiler，并理解 PlanState、ExecProcNode 和 core instrumentation wrapper。

本节唯一主问题：

```text
如何在 ExecProcNode 边界或 wrapper 中记录节点级耗时、调用次数和父子关系，避免把子节点时间重复解释成独占时间？
```

核心矛盾：执行器是父节点拉取子节点的嵌套调用栈；在父节点边界计时会天然包含子节点时间，但诊断又希望区分 inclusive 和 exclusive 成本。

学完后应能设计节点级 profiler 的 stack 语义，知道哪些成本可直接记录，哪些必须通过进入/退出栈和子节点扣减近似。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

上一节只设计了 query 级 hook 生命周期。
profiler 真正有价值时，通常需要落到节点级。
但节点级不是简单给每个 `Exec*` 函数外面加计时。
因为执行器采用 pull 模型，父节点调用子节点时，父节点调用栈仍然在场。
本节唯一主问题是如何记录节点耗时和嵌套关系，避免把 inclusive time 当成 exclusive time。
下一节会把这些事件放进共享内存 ring buffer。

```text
ExecutorStart hook
  -> walk PlanState tree
  -> attach profiler metadata
  -> ExecProcNode entry
  -> push node frame
  -> call real node
  -> pop node frame
  -> update inclusive / child / exclusive time
```

本节只处理自己这一环，不把相邻主题扩成第二个主问题。

阅读时要保持一条线：先从用户可见现象进入，再回到状态写入点，最后再回到可验证实验。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
节点级 profiler 需要一个 backend-local 调用栈：进入节点时 push frame 并记录 start time，退出时计算 elapsed，把 elapsed 加到当前节点 inclusive time，同时加到父 frame 的 child time；exclusive time 只能由 inclusive 减 child 得到。
```

这里的 tension 是：越接近 `ExecProcNode` 边界，节点语义越清楚，但每个 tuple 都会经过这个边界，计时和栈操作极易放大成本。

本节只问一个问题：节点级时间的栈语义怎样定义才不会误导。

这个模型的诊断含义是：

- 不要先背函数名，先判断状态属于哪个生命周期边界。

- 不要把用户可见指标当成源码真相，指标只是某个状态边界的投影。

- 不要把一次采样、一次累计值和一次执行剖面混在一起解释。

- 每个观测值都要回答：谁写入，何时可见，谁清理，异常时是否仍可靠。

本节后面的源码 walkthrough 都围绕这一个模型展开。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/executor/executor.h` | `ExecProcNode()` inline 以及 `ExecSetExecProcNode()` 声明。 |
| 2 | `src/backend/executor/execProcnode.c` | `ExecSetExecProcNode()`、`ExecProcNodeFirst()` 如何安装 real function 和 wrapper。 |
| 3 | `src/backend/executor/instrument.c` | `ExecProcNodeInstr()` 如何围绕 real node 调用 `InstrStartNode()` / `InstrStopNode()`。 |
| 4 | `src/include/nodes/execnodes.h` | `PlanState` 中 `ExecProcNode`、`ExecProcNodeReal`、`instrument` 字段。 |
| 5 | `src/include/nodes/plannodes.h` | `Plan.plan_node_id` 作为节点标识。 |
| 6 | `src/backend/commands/explain.c` | EXPLAIN 如何遍历 PlanState 并展示 actual time / rows / loops。 |
| 7 | `src/backend/executor/nodeHash.c` | MultiExec 节点和普通 ExecProcNode 边界差异。 |
| 8 | `src/backend/executor/nodeBitmapHeapscan.c` | BitmapHeapScan 等节点的额外 instrumentation。 |
| 9 | `src/backend/executor/execAsync.c` | 异步节点 instrumentation 的特殊边界。 |

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

### 4.1. `profiler_node_meta`

扩展为每个 PlanState 建立的节点元数据。

它的持有者是：ExecutorStart hook 遍历计划树。

它的读取者是：节点 wrapper 和结果 SRF。

生命周期边界是：QueryDesc 生命周期内有效。

诊断价值是：把 plan_node_id、node type、父子关系和计数器绑定。

读这个状态时不要只看字段名。

先问它在 backend-local、shared memory、DSM、文件系统还是调用栈里。

再问它的写入者是否和读取者在同一条时间线上。

最后问它失败时会留下错误值、缺失值，还是只能让上层退化成推断。

在本节唯一主问题中，它只服务于：如何在 ExecProcNode 边界或 wrapper 中记录节点级耗时、调用次数和父子关系，避免把子节点时间重复解释成独占时间？

如果这个状态不能解释这一点，就不应该把它扩成另一条主线。

### 4.2. `profiler_call_stack`

backend-local 当前节点调用栈。

它的持有者是：节点 wrapper。

它的读取者是：wrapper exit、ERROR cleanup。

生命周期边界是：一次 backend 执行期间随调用进入退出变化。

诊断价值是：区分 inclusive 和 exclusive time。

读这个状态时不要只看字段名。

先问它在 backend-local、shared memory、DSM、文件系统还是调用栈里。

再问它的写入者是否和读取者在同一条时间线上。

最后问它失败时会留下错误值、缺失值，还是只能让上层退化成推断。

在本节唯一主问题中，它只服务于：如何在 ExecProcNode 边界或 wrapper 中记录节点级耗时、调用次数和父子关系，避免把子节点时间重复解释成独占时间？

如果这个状态不能解释这一点，就不应该把它扩成另一条主线。

### 4.3. `frame.start_time`

节点进入时的计时起点。

它的持有者是：wrapper entry。

它的读取者是：wrapper exit。

生命周期边界是：一次节点调用内有效。

诊断价值是：计算单次调用耗时。

读这个状态时不要只看字段名。

先问它在 backend-local、shared memory、DSM、文件系统还是调用栈里。

再问它的写入者是否和读取者在同一条时间线上。

最后问它失败时会留下错误值、缺失值，还是只能让上层退化成推断。

在本节唯一主问题中，它只服务于：如何在 ExecProcNode 边界或 wrapper 中记录节点级耗时、调用次数和父子关系，避免把子节点时间重复解释成独占时间？

如果这个状态不能解释这一点，就不应该把它扩成另一条主线。

### 4.4. `frame.child_time`

当前 frame 下子节点已消耗的时间。

它的持有者是：子节点 exit。

它的读取者是：父节点 exit。

生命周期边界是：父 frame 生命周期内有效。

诊断价值是：从 inclusive 中扣减子节点时间。

读这个状态时不要只看字段名。

先问它在 backend-local、shared memory、DSM、文件系统还是调用栈里。

再问它的写入者是否和读取者在同一条时间线上。

最后问它失败时会留下错误值、缺失值，还是只能让上层退化成推断。

在本节唯一主问题中，它只服务于：如何在 ExecProcNode 边界或 wrapper 中记录节点级耗时、调用次数和父子关系，避免把子节点时间重复解释成独占时间？

如果这个状态不能解释这一点，就不应该把它扩成另一条主线。

### 4.5. `node.inclusive_time`

节点自身 wrapper 覆盖的总时间。

它的持有者是：节点 exit。

它的读取者是：SRF、日志、聚合输出。

生命周期边界是：query 生命周期或 ring event 生命周期。

诊断价值是：表示节点作为调用边界包含子节点的时间。

读这个状态时不要只看字段名。

先问它在 backend-local、shared memory、DSM、文件系统还是调用栈里。

再问它的写入者是否和读取者在同一条时间线上。

最后问它失败时会留下错误值、缺失值，还是只能让上层退化成推断。

在本节唯一主问题中，它只服务于：如何在 ExecProcNode 边界或 wrapper 中记录节点级耗时、调用次数和父子关系，避免把子节点时间重复解释成独占时间？

如果这个状态不能解释这一点，就不应该把它扩成另一条主线。

### 4.6. `node.exclusive_time`

inclusive 减 child 的近似独占时间。

它的持有者是：节点 exit 计算。

它的读取者是：诊断输出。

生命周期边界是：受异步、MultiExec、并行影响，需要标注近似。

诊断价值是：避免把子节点成本重复算到父节点。

读这个状态时不要只看字段名。

先问它在 backend-local、shared memory、DSM、文件系统还是调用栈里。

再问它的写入者是否和读取者在同一条时间线上。

最后问它失败时会留下错误值、缺失值，还是只能让上层退化成推断。

在本节唯一主问题中，它只服务于：如何在 ExecProcNode 边界或 wrapper 中记录节点级耗时、调用次数和父子关系，避免把子节点时间重复解释成独占时间？

如果这个状态不能解释这一点，就不应该把它扩成另一条主线。

### 4.7. `node.call_count / tuple_count`

节点被调用次数和返回 tuple 数。

它的持有者是：wrapper exit。

它的读取者是：诊断输出。

生命周期边界是：query 生命周期内累加。

诊断价值是：判断时间是单次慢还是高频小成本放大。

读这个状态时不要只看字段名。

先问它在 backend-local、shared memory、DSM、文件系统还是调用栈里。

再问它的写入者是否和读取者在同一条时间线上。

最后问它失败时会留下错误值、缺失值，还是只能让上层退化成推断。

在本节唯一主问题中，它只服务于：如何在 ExecProcNode 边界或 wrapper 中记录节点级耗时、调用次数和父子关系，避免把子节点时间重复解释成独占时间？

如果这个状态不能解释这一点，就不应该把它扩成另一条主线。

本节最重要的不变量是：

```text
raw field 不是语义；
field + writer + reader + lifecycle + cleanup 才是诊断语义。
```

## 5. 主流程源码 walkthrough

下面按一次慢查询诊断或 profiler 采样的时间线走主流程。

```text
ExecutorStart hook 遍历 PlanState
安装或利用节点 wrapper
进入节点 push frame
调用真实节点函数
子节点退出累加 child_time
当前节点 pop frame
处理 MultiExec / blocking node
输出时标注语义
```

### 5.1. ExecutorStart hook 遍历 PlanState

standard_ExecutorStart 后可以访问已初始化的 planstate tree。

这一段改变的状态边界是：元数据创建边界。

回到诊断时要验证：是否已有 plan_node_id、nodeTag 和父子关系。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

### 5.2. 安装或利用节点 wrapper

扩展可以包裹节点函数，也可以借助核心 instrumentation；无论哪种，都必须保留 real function。

这一段改变的状态边界是：节点入口边界。

回到诊断时要验证：是否破坏 ExecProcNodeFirst / instrumentation wrapper。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

### 5.3. 进入节点 push frame

记录 plan_node_id、start_time，并把当前 frame 设为栈顶。

这一段改变的状态边界是：调用栈边界。

回到诊断时要验证：是否处理递归和异常退出。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

### 5.4. 调用真实节点函数

节点可能直接产出 tuple，也可能继续拉取子节点。

这一段改变的状态边界是：执行器 pull 模型边界。

回到诊断时要验证：父节点时间天然包含子节点。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

### 5.5. 子节点退出累加 child_time

子节点 elapsed 回加到父 frame 的 child_time。

这一段改变的状态边界是：inclusive/exclusive 分离边界。

回到诊断时要验证：父 frame 是否仍然匹配。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

### 5.6. 当前节点 pop frame

计算 elapsed、call_count、tuple_count、exclusive，并恢复父 frame。

这一段改变的状态边界是：节点退出边界。

回到诊断时要验证：TupIsNull、ERROR、中断是否处理。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

### 5.7. 处理 MultiExec / blocking node

Hash、BitmapIndexScan 等路径不一定通过普通 tuple 返回协议。

这一段改变的状态边界是：旁路边界。

回到诊断时要验证：是否需要单独包裹或只用 EXPLAIN instrumentation。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

### 5.8. 输出时标注语义

结果必须明确 inclusive、exclusive、calls、tuples 和 sampling/window。

这一段改变的状态边界是：用户可见边界。

回到诊断时要验证：是否把近似独占时间说成精确事实。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

## 6. 生命周期 / ownership / cleanup

### 6.1. 节点元数据

创建或进入者：ExecutorStart hook 创建。

正常清理者：ExecutorEnd hook 输出后释放。

异常路径依赖：ERROR 由 query context 或 PG_FINALLY 清理。

这里要区分三件事。

MemoryContext 管内存批量释放。

ResourceOwner 管 pin、refcount、文件句柄、锁等外部资源。

全局或共享状态必须有独立的可见性、锁或代际边界。

把这些混成一句“事务结束释放”，会丢掉定位问题所需的时间顺序。

### 6.2. 调用栈 frame

创建或进入者：节点 wrapper entry。

正常清理者：wrapper exit。

异常路径依赖：ERROR 必须恢复栈顶。

这里要区分三件事。

MemoryContext 管内存批量释放。

ResourceOwner 管 pin、refcount、文件句柄、锁等外部资源。

全局或共享状态必须有独立的可见性、锁或代际边界。

把这些混成一句“事务结束释放”，会丢掉定位问题所需的时间顺序。

### 6.3. 计时器

创建或进入者：进入节点读当前时间。

正常清理者：退出节点做差。

异常路径依赖：高频节点下计时成本显著。

这里要区分三件事。

MemoryContext 管内存批量释放。

ResourceOwner 管 pin、refcount、文件句柄、锁等外部资源。

全局或共享状态必须有独立的可见性、锁或代际边界。

把这些混成一句“事务结束释放”，会丢掉定位问题所需的时间顺序。

### 6.4. 父子关系

创建或进入者：遍历 PlanState tree 建立。

正常清理者：query 结束释放。

异常路径依赖：SubPlan、InitPlan、CustomScan 需要特殊遍历。

这里要区分三件事。

MemoryContext 管内存批量释放。

ResourceOwner 管 pin、refcount、文件句柄、锁等外部资源。

全局或共享状态必须有独立的可见性、锁或代际边界。

把这些混成一句“事务结束释放”，会丢掉定位问题所需的时间顺序。

### 6.5. 节点函数指针

创建或进入者：ExecInitNode / ExecSetExecProcNode。

正常清理者：PlanState 清理。

异常路径依赖：扩展不能丢失 ExecProcNodeReal。

这里要区分三件事。

MemoryContext 管内存批量释放。

ResourceOwner 管 pin、refcount、文件句柄、锁等外部资源。

全局或共享状态必须有独立的可见性、锁或代际边界。

把这些混成一句“事务结束释放”，会丢掉定位问题所需的时间顺序。

### 6.6. 输出结果

创建或进入者：End 阶段或 ring buffer 读取。

正常清理者：读取后清理或覆盖。

异常路径依赖：输出失败不能改变查询结果。

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
| wrapper 链 | ExecProcNodeFirst / ExecProcNodeInstr | 扩展不能破坏核心 instrumentation。 | 直接替换函数指针总是安全。 |
| 栈匹配 | push/pop frame | 进入退出必须成对。 | ERROR 会自动恢复栈。 |
| 时间语义 | inclusive/exclusive | 父节点时间包含子节点，独占时间是推导值。 | 节点 total time 等于自身 CPU。 |
| tuple 语义 | TupIsNull / nTuples | 返回空 slot 表示该调用没有产出 tuple。 | 调用次数等于返回行数。 |
| 旁路节点 | MultiExecProcNode | 某些节点不走普通 tuple 协议。 | 包 ExecProcNode 就覆盖所有节点。 |
| 异步/并行 | execAsync / worker instrumentation | 时间归因可能跨 worker 或异步请求。 | 单 backend stack 等于全局时间。 |

读源码时可以按这个顺序检查：

```text
先找写入点
  -> 再找读取点
  -> 再找清理点
  -> 再看 ERROR / abort / backend exit 是否能兜底
  -> 最后把用户可见输出映射回这一条状态链
```

1. `wrapper 链` 这一层主要依赖 ExecProcNodeFirst / ExecProcNodeInstr。

它保证的是：扩展不能破坏核心 instrumentation。

不要把它误读成：直接替换函数指针总是安全。

如果一次诊断只能看到这一层的投影，就必须保留不确定性。

2. `栈匹配` 这一层主要依赖 push/pop frame。

它保证的是：进入退出必须成对。

不要把它误读成：ERROR 会自动恢复栈。

如果一次诊断只能看到这一层的投影，就必须保留不确定性。

3. `时间语义` 这一层主要依赖 inclusive/exclusive。

它保证的是：父节点时间包含子节点，独占时间是推导值。

不要把它误读成：节点 total time 等于自身 CPU。

如果一次诊断只能看到这一层的投影，就必须保留不确定性。

4. `tuple 语义` 这一层主要依赖 TupIsNull / nTuples。

它保证的是：返回空 slot 表示该调用没有产出 tuple。

不要把它误读成：调用次数等于返回行数。

如果一次诊断只能看到这一层的投影，就必须保留不确定性。

5. `旁路节点` 这一层主要依赖 MultiExecProcNode。

它保证的是：某些节点不走普通 tuple 协议。

不要把它误读成：包 ExecProcNode 就覆盖所有节点。

如果一次诊断只能看到这一层的投影，就必须保留不确定性。

6. `异步/并行` 这一层主要依赖 execAsync / worker instrumentation。

它保证的是：时间归因可能跨 worker 或异步请求。

不要把它误读成：单 backend stack 等于全局时间。

如果一次诊断只能看到这一层的投影，就必须保留不确定性。

## 8. 错误路径 / 异常路径 / fallback

### 8.1. double wrapping

可见现象：节点函数被扩展重复包裹。

源码上应先回到：`ExecSetExecProcNode()`。

正确处理方式是：记录 wrapper installed 标记，避免重复。

这类路径通常不是“继续收集更多指标”就能解决。

要先判断观测值是在失败前、失败中还是失败后写入。

如果写入点本身在异常路径里，就要检查 cleanup 是否仍然执行。

### 8.2. exclusive time 为负

可见现象：child_time 大于 elapsed。

源码上应先回到：异步、计时精度、栈错配。

正确处理方式是：标记为近似，先查栈 push/pop。

这类路径通常不是“继续收集更多指标”就能解决。

要先判断观测值是在失败前、失败中还是失败后写入。

如果写入点本身在异常路径里，就要检查 cleanup 是否仍然执行。

### 8.3. 节点调用次数爆炸

可见现象：轻节点或 Nested Loop 内侧高频。

源码上应先回到：`ExecProcNode()` call path。

正确处理方式是：优先采样或阈值过滤。

这类路径通常不是“继续收集更多指标”就能解决。

要先判断观测值是在失败前、失败中还是失败后写入。

如果写入点本身在异常路径里，就要检查 cleanup 是否仍然执行。

### 8.4. MultiExec 缺失

可见现象：Hash build 或 BitmapIndexScan 没有普通调用计时。

源码上应先回到：`MultiExecProcNode()`。

正确处理方式是：单独处理或用 core instrumentation 辅助。

这类路径通常不是“继续收集更多指标”就能解决。

要先判断观测值是在失败前、失败中还是失败后写入。

如果写入点本身在异常路径里，就要检查 cleanup 是否仍然执行。

### 8.5. ERROR 后栈非空

可见现象：下一条查询 profiler 栈污染。

源码上应先回到：wrapper PG_TRY/PG_FINALLY。

正确处理方式是：异常路径必须 pop 到进入前状态。

这类路径通常不是“继续收集更多指标”就能解决。

要先判断观测值是在失败前、失败中还是失败后写入。

如果写入点本身在异常路径里，就要检查 cleanup 是否仍然执行。

## 9. 成本、资源与跨模块传播

成本分析要围绕同一个主问题。

不要把所有可能成本都列一遍。

### 9.1. 每 tuple 计时

主要扩张因子：返回 tuple 数和 loops。

放大方式：小行高频节点会放大 clock 成本。

控制办法：采样、阈值、只记录慢节点。

诊断时不要只看绝对值。

要看它是否随 rows、loops、backend 数、buffer miss、计划节点深度或采样率线性放大。

如果它只在某个节点或某类等待里集中出现，说明问题边界比 SQL 文本更窄。

### 9.2. 栈操作

主要扩张因子：计划深度和调用频率。

放大方式：push/pop 成为 CPU 开销。

控制办法：backend-local 固定数组或轻量 frame。

诊断时不要只看绝对值。

要看它是否随 rows、loops、backend 数、buffer miss、计划节点深度或采样率线性放大。

如果它只在某个节点或某类等待里集中出现，说明问题边界比 SQL 文本更窄。

### 9.3. 元数据遍历

主要扩张因子：plan node 数。

放大方式：Start/End 遍历增加启动/收尾成本。

控制办法：只在采样命中时遍历。

诊断时不要只看绝对值。

要看它是否随 rows、loops、backend 数、buffer miss、计划节点深度或采样率线性放大。

如果它只在某个节点或某类等待里集中出现，说明问题边界比 SQL 文本更窄。

### 9.4. exclusive 计算

主要扩张因子：子节点数量和异步模式。

放大方式：复杂节点可能产生近似误差。

控制办法：输出里明确 inclusive/exclusive 语义。

诊断时不要只看绝对值。

要看它是否随 rows、loops、backend 数、buffer miss、计划节点深度或采样率线性放大。

如果它只在某个节点或某类等待里集中出现，说明问题边界比 SQL 文本更窄。

### 9.5. 旁路支持

主要扩张因子：MultiExec 节点数量。

放大方式：额外 wrapper 增加维护成本。

控制办法：先覆盖普通 ExecProcNode，再按价值补充。

诊断时不要只看绝对值。

要看它是否随 rows、loops、backend 数、buffer miss、计划节点深度或采样率线性放大。

如果它只在某个节点或某类等待里集中出现，说明问题边界比 SQL 文本更窄。

### 9.6. 输出体积

主要扩张因子：节点数和采样次数。

放大方式：结果 view 或 ring 可能膨胀。

控制办法：只保留 top N 或聚合。

诊断时不要只看绝对值。

要看它是否随 rows、loops、backend 数、buffer miss、计划节点深度或采样率线性放大。

如果它只在某个节点或某类等待里集中出现，说明问题边界比 SQL 文本更窄。

## 10. 观测与诊断入口

观测入口要分清当前值、累计值、单次执行剖面和外部采样。

同一个慢 SQL 结论最好至少由两类信号互相校验。

### 10.1. EXPLAIN actual time

能看到：核心 inclusive 节点时间。

看不到：独占时间。

源码入口：`instrument.c`、`explain.c`。

使用时先问它是当前状态、累计状态，还是本次执行状态。

再问它是否会因为采样、刷新延迟、并行 worker 汇总或输出格式化而偏移。

最后把结论回扣到本节主流程中的某个写入点。

### 10.2. 自定义 profiler view

能看到：inclusive/exclusive/calls/tuples。

看不到：未采样节点。

源码入口：扩展 SRF。

使用时先问它是当前状态、累计状态，还是本次执行状态。

再问它是否会因为采样、刷新延迟、并行 worker 汇总或输出格式化而偏移。

最后把结论回扣到本节主流程中的某个写入点。

### 10.3. perf 栈

能看到：真实 CPU 栈。

看不到：PlanState 标识。

源码入口：外部 profiler。

使用时先问它是当前状态、累计状态，还是本次执行状态。

再问它是否会因为采样、刷新延迟、并行 worker 汇总或输出格式化而偏移。

最后把结论回扣到本节主流程中的某个写入点。

### 10.4. gdb PlanState

能看到：函数指针和 instrument 字段。

看不到：统计意义。

源码入口：`execnodes.h`。

使用时先问它是当前状态、累计状态，还是本次执行状态。

再问它是否会因为采样、刷新延迟、并行 worker 汇总或输出格式化而偏移。

最后把结论回扣到本节主流程中的某个写入点。

### 10.5. 日志 sample

能看到：慢节点事件。

看不到：完整计划树。

源码入口：扩展输出。

使用时先问它是当前状态、累计状态，还是本次执行状态。

再问它是否会因为采样、刷新延迟、并行 worker 汇总或输出格式化而偏移。

最后把结论回扣到本节主流程中的某个写入点。

### 10.6. per-worker EXPLAIN

能看到：worker 节点时间。

看不到：leader 本地 wrapper 之外的全部细节。

源码入口：`execParallel.c`。

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

### 12.1. inclusive 与 exclusive 对比

实验目标：把可见现象压回源码状态边界。

建议准备：

```text
构造 Nested Loop + Index Scan。
记录父节点和内侧节点时间。
比较父 inclusive 与父 exclusive。
```

预期现象：父节点 inclusive 应包含内侧调用时间，exclusive 才接近自身调度/qual 成本。

回到源码时检查：`ExecProcNode()` 嵌套调用。

实验完成后不要只记录数值。

还要记录计划节点、GUC、并发状态、缓存冷热和是否开启 timing。

这些条件决定同一个观测值能否迁移到另一个现场。

### 12.2. 高频轻节点开销

实验目标：把可见现象压回源码状态边界。

建议准备：

```text
SELECT * FROM generate_series(1,1000000) g WHERE g > 0;
开启节点级 profiler 后对比耗时。
```

预期现象：每 tuple wrapper 可能显著扰动轻节点。

回到源码时检查：`ExecProcNodeInstr()` 的核心开销模型。

实验完成后不要只记录数值。

还要记录计划节点、GUC、并发状态、缓存冷热和是否开启 timing。

这些条件决定同一个观测值能否迁移到另一个现场。

### 12.3. MultiExec 缺口

实验目标：把可见现象压回源码状态边界。

建议准备：

```text
运行 Hash Join 或 Bitmap Heap Scan。
对比 profiler 节点事件和 EXPLAIN。
```

预期现象：某些 build/bitmap 路径不完全等同普通 tuple 边界。

回到源码时检查：`MultiExecProcNode()`。

实验完成后不要只记录数值。

还要记录计划节点、GUC、并发状态、缓存冷热和是否开启 timing。

这些条件决定同一个观测值能否迁移到另一个现场。

### 12.4. ERROR 栈恢复

实验目标：把可见现象压回源码状态边界。

建议准备：

```text
让表达式函数中途 ERROR。
确认下一条 SQL 的 profiler stack 为空。
```

预期现象：异常后不应残留 frame。

回到源码时检查：wrapper cleanup。

实验完成后不要只记录数值。

还要记录计划节点、GUC、并发状态、缓存冷热和是否开启 timing。

这些条件决定同一个观测值能否迁移到另一个现场。

## 13. 讨论题

1. 为什么父节点 actual time 通常不能解释为父节点独占 CPU？

2. 节点级 profiler 应默认记录 inclusive、exclusive 还是二者都记录？

3. 为什么 MultiExec 节点会破坏简单 wrapper 假设？

4. 高频轻节点下，采样和完整计时哪种更适合线上？

5. 如何向用户解释 exclusive time 的近似性质？

## 14. 本节小结

本节只沉淀一个模型：

```text
节点级 profiler 的核心是调用栈语义。
inclusive time 来自节点 wrapper 覆盖区间。
exclusive time 需要从子节点时间扣减，是推导值。
ExecProcNode 边界不覆盖所有 MultiExec 和异步细节。
高频节点的计时成本必须受采样和阈值控制。
```

下一节把节点事件放入共享内存 ring buffer。
