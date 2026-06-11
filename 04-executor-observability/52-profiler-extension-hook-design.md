# PostgreSQL Profiler extension 的最小 hook 设计

## 课程定位

前置知识：已经理解 executor hook 链、auto_explain 的安全边界，也能把 profiler 栈映射回执行器热点。

本节唯一主问题：

```text
如何用 executor hook 搭建一个只观测不改语义的轻量 profiler，记录 query id、计划节点、开始结束时间和错误路径？
```

核心矛盾：扩展越靠近 executor 现场，越容易拿到有价值状态；但 hook 运行在核心执行路径中，任何漏调用 previous hook、异常路径状态残留或过度插桩都会破坏线上语义。

学完后应能审查一个 profiler extension 的最小 hook 设计是否只观测、不改语义、可异常恢复、可和其它 hook 共存。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

上一节用外部 profiler 解释 CPU 栈。
从本节开始，04 目录进入轻量 Executor profiler extension 主题。
第一步不是 ring buffer，也不是复杂 UI。
第一步是 hook 设计：在哪里进入，记录什么，如何保证无论成功失败都把执行器交还给核心代码。
本节唯一主问题是最小 executor hook profiler 的生命周期和异常边界。
下一节会继续追问节点级计时和嵌套调用栈。

```text
_PG_init
  -> DefineCustomGUCs
  -> save previous Executor hooks
  -> install profiler hooks
  -> ExecutorStart/Run/Finish/End wrapper
  -> record query event without changing semantics
```

本节只处理自己这一环，不把相邻主题扩成第二个主问题。

阅读时要保持一条线：先从用户可见现象进入，再回到状态写入点，最后再回到可验证实验。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
扩展在 `_PG_init()` 中保存 previous executor hooks 并安装自己的 hook；每个 hook 先做 cheap guard，再记录最少状态，随后调用 previous 或 standard executor；跨 hook 状态放在 backend-local profiler context 中，ERROR 路径用 PG_TRY/PG_FINALLY 或明确 cleanup 恢复。
```

这里的 tension 是：要在执行器现场拿到 query id 和 PlanState，又不能让 profiler 成为新的执行器语义层。

本节只问一个问题：最小 hook profiler 的边界怎么画。

这个模型的诊断含义是：

- 不要先背函数名，先判断状态属于哪个生命周期边界。

- 不要把用户可见指标当成源码真相，指标只是某个状态边界的投影。

- 不要把一次采样、一次累计值和一次执行剖面混在一起解释。

- 每个观测值都要回答：谁写入，何时可见，谁清理，异常时是否仍可靠。

本节后面的源码 walkthrough 都围绕这一个模型展开。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/executor/executor.h` | ExecutorStart/Run/Finish/End hook 类型和 standard_* 声明。 |
| 2 | `src/backend/executor/execMain.c` | 四个 hook 的分派位置和 executor 标准生命周期。 |
| 3 | `contrib/auto_explain/auto_explain.c` | 真实扩展示例：保存 previous hook、定义 GUC、调用 standard/previous、End 阶段输出。 |
| 4 | `src/include/executor/execdesc.h` | `QueryDesc`、`query_instr`、`instrument_options`。 |
| 5 | `src/include/nodes/plannodes.h` | `PlannedStmt.queryId` 和 `Plan.plan_node_id`。 |
| 6 | `src/backend/parser/analyze.c` | `JumbleQuery()` 后 query id 写入路径。 |
| 7 | `src/backend/optimizer/plan/planner.c` | `PlannedStmt.queryId` 继承 parse query id。 |
| 8 | `src/backend/utils/misc/guc.c` | `DefineCustomBoolVariable()` 等自定义 GUC API。 |
| 9 | `src/backend/utils/mmgr/mcxt.c` | MemoryContext 作为扩展临时状态生命周期工具。 |

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

### 4.1. `prev_ExecutorStart/Run/Finish/End`

扩展安装前保存的 hook 指针。

它的持有者是：_PG_init。

它的读取者是：扩展自己的 hook 函数。

生命周期边界是：模块加载后进程内长期有效。

诊断价值是：保证 hook 链不断裂。

读这个状态时不要只看字段名。

先问它在 backend-local、shared memory、DSM、文件系统还是调用栈里。

再问它的写入者是否和读取者在同一条时间线上。

最后问它失败时会留下错误值、缺失值，还是只能让上层退化成推断。

在本节唯一主问题中，它只服务于：如何用 executor hook 搭建一个只观测不改语义的轻量 profiler，记录 query id、计划节点、开始结束时间和错误路径？

如果这个状态不能解释这一点，就不应该把它扩成另一条主线。

### 4.2. `profiler_enabled`

自定义 GUC 的 backend-local 开关。

它的持有者是：GUC 系统和 `_PG_init()`。

它的读取者是：每个 executor hook 的 cheap guard。

生命周期边界是：backend 配置生命周期内有效。

诊断价值是：避免每条 SQL 都承担完整 profiler 成本。

读这个状态时不要只看字段名。

先问它在 backend-local、shared memory、DSM、文件系统还是调用栈里。

再问它的写入者是否和读取者在同一条时间线上。

最后问它失败时会留下错误值、缺失值，还是只能让上层退化成推断。

在本节唯一主问题中，它只服务于：如何用 executor hook 搭建一个只观测不改语义的轻量 profiler，记录 query id、计划节点、开始结束时间和错误路径？

如果这个状态不能解释这一点，就不应该把它扩成另一条主线。

### 4.3. `profiler_active`

当前 query 是否已进入 profiler 记录。

它的持有者是：ExecutorStart hook。

它的读取者是：Run/Finish/End hook。

生命周期边界是：一次 QueryDesc 生命周期内有效。

诊断价值是：防止 End 阶段输出未初始化状态。

读这个状态时不要只看字段名。

先问它在 backend-local、shared memory、DSM、文件系统还是调用栈里。

再问它的写入者是否和读取者在同一条时间线上。

最后问它失败时会留下错误值、缺失值，还是只能让上层退化成推断。

在本节唯一主问题中，它只服务于：如何用 executor hook 搭建一个只观测不改语义的轻量 profiler，记录 query id、计划节点、开始结束时间和错误路径？

如果这个状态不能解释这一点，就不应该把它扩成另一条主线。

### 4.4. `profiler_nesting_level`

递归执行或 nested statement 的保护计数。

它的持有者是：hook wrapper。

它的读取者是：所有 hook 分支。

生命周期边界是：必须在 ERROR 路径恢复。

诊断价值是：避免 SPI/nested query 污染外层记录。

读这个状态时不要只看字段名。

先问它在 backend-local、shared memory、DSM、文件系统还是调用栈里。

再问它的写入者是否和读取者在同一条时间线上。

最后问它失败时会留下错误值、缺失值，还是只能让上层退化成推断。

在本节唯一主问题中，它只服务于：如何用 executor hook 搭建一个只观测不改语义的轻量 profiler，记录 query id、计划节点、开始结束时间和错误路径？

如果这个状态不能解释这一点，就不应该把它扩成另一条主线。

### 4.5. `QueryDesc`

executor 生命周期的顶层对象，连接 PlannedStmt、EState、sourceText、params。

它的持有者是：调用 executor 的上层。

它的读取者是：executor hook、standard executor。

生命周期边界是：ExecutorStart 到 ExecutorEnd。

诊断价值是：profiler 读取 query id、计划树和执行上下文的入口。

读这个状态时不要只看字段名。

先问它在 backend-local、shared memory、DSM、文件系统还是调用栈里。

再问它的写入者是否和读取者在同一条时间线上。

最后问它失败时会留下错误值、缺失值，还是只能让上层退化成推断。

在本节唯一主问题中，它只服务于：如何用 executor hook 搭建一个只观测不改语义的轻量 profiler，记录 query id、计划节点、开始结束时间和错误路径？

如果这个状态不能解释这一点，就不应该把它扩成另一条主线。

### 4.6. `PlannedStmt.queryId`

归一化后的 query id。

它的持有者是：parser/analyze/planner。

它的读取者是：pg_stat_statements、profiler、EXPLAIN Query Identifier。

生命周期边界是：计划生命周期内稳定。

诊断价值是：把 profiler 事件与长期统计关联。

读这个状态时不要只看字段名。

先问它在 backend-local、shared memory、DSM、文件系统还是调用栈里。

再问它的写入者是否和读取者在同一条时间线上。

最后问它失败时会留下错误值、缺失值，还是只能让上层退化成推断。

在本节唯一主问题中，它只服务于：如何用 executor hook 搭建一个只观测不改语义的轻量 profiler，记录 query id、计划节点、开始结束时间和错误路径？

如果这个状态不能解释这一点，就不应该把它扩成另一条主线。

### 4.7. `profiler event`

扩展自己的观测记录，不属于核心 executor 语义。

它的持有者是：hook wrapper。

它的读取者是：共享 buffer、日志或 SRF 读取者。

生命周期边界是：取决于扩展设计。

诊断价值是：必须可以丢弃，不能阻塞核心执行。

读这个状态时不要只看字段名。

先问它在 backend-local、shared memory、DSM、文件系统还是调用栈里。

再问它的写入者是否和读取者在同一条时间线上。

最后问它失败时会留下错误值、缺失值，还是只能让上层退化成推断。

在本节唯一主问题中，它只服务于：如何用 executor hook 搭建一个只观测不改语义的轻量 profiler，记录 query id、计划节点、开始结束时间和错误路径？

如果这个状态不能解释这一点，就不应该把它扩成另一条主线。

本节最重要的不变量是：

```text
raw field 不是语义；
field + writer + reader + lifecycle + cleanup 才是诊断语义。
```

## 5. 主流程源码 walkthrough

下面按一次慢查询诊断或 profiler 采样的时间线走主流程。

```text
_PG_init() 定义 GUC
_PG_init() 保存 previous hook
_PG_init() 安装 profiler hook
ExecutorStart hook 做 cheap guard
调用 previous 或 standard_ExecutorStart
ExecutorRun/Finish hook 只记录阶段
ExecutorEnd hook 输出或落盘
ERROR 路径恢复 backend-local 状态
```

### 5.1. _PG_init() 定义 GUC

扩展加载时注册 enable、sample_rate、min_duration 等开关。

这一段改变的状态边界是：配置边界。

回到诊断时要验证：GUC 名称、上下文和默认值是否保守。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

### 5.2. _PG_init() 保存 previous hook

读取当前 `ExecutorStart_hook` 等全局指针到 prev 变量。

这一段改变的状态边界是：hook 链边界。

回到诊断时要验证：是否每个 hook 都保存了 previous。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

### 5.3. _PG_init() 安装 profiler hook

把全局 hook 指针替换为扩展函数。

这一段改变的状态边界是：扩展进入 executor 的唯一入口。

回到诊断时要验证：是否避免覆盖后不调用 previous。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

### 5.4. ExecutorStart hook 做 cheap guard

快速检查 enabled、采样、query id、nested level，决定是否记录。

这一段改变的状态边界是：开销控制边界。

回到诊断时要验证：未启用路径是否几乎只多一次分支。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

### 5.5. 调用 previous 或 standard_ExecutorStart

profiler 必须把控制权交给下一环或核心实现。

这一段改变的状态边界是：语义保持边界。

回到诊断时要验证：任何错误分支是否都不会跳过 executor。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

### 5.6. ExecutorRun/Finish hook 只记录阶段

运行和收尾阶段可记录开始结束时间、方向、count、finish 状态。

这一段改变的状态边界是：阶段事件边界。

回到诊断时要验证：是否避免在 Run 中做复杂格式化。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

### 5.7. ExecutorEnd hook 输出或落盘

End 阶段仍可读取 QueryDesc / EState / PlanState，适合完成事件。

这一段改变的状态边界是：最后可读边界。

回到诊断时要验证：是否在 standard_ExecutorEnd 前读取必要状态。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

### 5.8. ERROR 路径恢复 backend-local 状态

用 PG_TRY/PG_FINALLY 或严格局部变量保证 nesting、active、context 恢复。

这一段改变的状态边界是：异常安全边界。

回到诊断时要验证：取消、ERROR、OOM 时是否留下 active 状态。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

## 6. 生命周期 / ownership / cleanup

### 6.1. 模块加载

创建或进入者：_PG_init。

正常清理者：进程退出卸载。

异常路径依赖：shared_preload 需求由扩展功能决定。

这里要区分三件事。

MemoryContext 管内存批量释放。

ResourceOwner 管 pin、refcount、文件句柄、锁等外部资源。

全局或共享状态必须有独立的可见性、锁或代际边界。

把这些混成一句“事务结束释放”，会丢掉定位问题所需的时间顺序。

### 6.2. GUC 状态

创建或进入者：DefineCustom*Variable。

正常清理者：GUC reset 或 backend exit。

异常路径依赖：check/assign hook 不能抛出破坏性状态。

这里要区分三件事。

MemoryContext 管内存批量释放。

ResourceOwner 管 pin、refcount、文件句柄、锁等外部资源。

全局或共享状态必须有独立的可见性、锁或代际边界。

把这些混成一句“事务结束释放”，会丢掉定位问题所需的时间顺序。

### 6.3. hook 链状态

创建或进入者：_PG_init 保存和安装。

正常清理者：进程生命周期内持续。

异常路径依赖：无 _PG_fini 场景也必须不假设动态卸载。

这里要区分三件事。

MemoryContext 管内存批量释放。

ResourceOwner 管 pin、refcount、文件句柄、锁等外部资源。

全局或共享状态必须有独立的可见性、锁或代际边界。

把这些混成一句“事务结束释放”，会丢掉定位问题所需的时间顺序。

### 6.4. query profiler state

创建或进入者：ExecutorStart hook。

正常清理者：ExecutorEnd hook 或 ERROR cleanup。

异常路径依赖：nested/error 必须恢复。

这里要区分三件事。

MemoryContext 管内存批量释放。

ResourceOwner 管 pin、refcount、文件句柄、锁等外部资源。

全局或共享状态必须有独立的可见性、锁或代际边界。

把这些混成一句“事务结束释放”，会丢掉定位问题所需的时间顺序。

### 6.5. 事件记录

创建或进入者：Run/Finish/End hook。

正常清理者：落入 ring、backend-local list 或日志后释放。

异常路径依赖：满了应丢弃或采样，不阻塞 executor。

这里要区分三件事。

MemoryContext 管内存批量释放。

ResourceOwner 管 pin、refcount、文件句柄、锁等外部资源。

全局或共享状态必须有独立的可见性、锁或代际边界。

把这些混成一句“事务结束释放”，会丢掉定位问题所需的时间顺序。

### 6.6. 临时内存

创建或进入者：CurrentMemoryContext 或 executor context。

正常清理者：MemoryContextDelete/Reset。

异常路径依赖：不要把短生命周期指针存到共享区。

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
| hook 链 | previous hook + standard executor | 所有路径必须继续执行 executor。 | 扩展可以独占 hook。 |
| 语义隔离 | 只读 QueryDesc / PlanState | profiler 不改变查询结果、snapshot、locks。 | 观测可以顺手修正执行行为。 |
| 异常恢复 | PG_TRY/PG_FINALLY | backend-local guard 必须恢复。 | ERROR 自动清所有静态变量。 |
| 采样边界 | cheap guard | 未采样查询成本极低。 | 启用扩展就必须记录所有 SQL。 |
| 内存边界 | MemoryContext | 临时状态随 query 或 backend 清理。 | 共享区可以保存指针。 |
| 标识边界 | queryId + plan_node_id | 标识用于关联，不保证跨版本绝对稳定。 | query id 等于 SQL 文本主键。 |

读源码时可以按这个顺序检查：

```text
先找写入点
  -> 再找读取点
  -> 再找清理点
  -> 再看 ERROR / abort / backend exit 是否能兜底
  -> 最后把用户可见输出映射回这一条状态链
```

1. `hook 链` 这一层主要依赖 previous hook + standard executor。

它保证的是：所有路径必须继续执行 executor。

不要把它误读成：扩展可以独占 hook。

如果一次诊断只能看到这一层的投影，就必须保留不确定性。

2. `语义隔离` 这一层主要依赖 只读 QueryDesc / PlanState。

它保证的是：profiler 不改变查询结果、snapshot、locks。

不要把它误读成：观测可以顺手修正执行行为。

如果一次诊断只能看到这一层的投影，就必须保留不确定性。

3. `异常恢复` 这一层主要依赖 PG_TRY/PG_FINALLY。

它保证的是：backend-local guard 必须恢复。

不要把它误读成：ERROR 自动清所有静态变量。

如果一次诊断只能看到这一层的投影，就必须保留不确定性。

4. `采样边界` 这一层主要依赖 cheap guard。

它保证的是：未采样查询成本极低。

不要把它误读成：启用扩展就必须记录所有 SQL。

如果一次诊断只能看到这一层的投影，就必须保留不确定性。

5. `内存边界` 这一层主要依赖 MemoryContext。

它保证的是：临时状态随 query 或 backend 清理。

不要把它误读成：共享区可以保存指针。

如果一次诊断只能看到这一层的投影，就必须保留不确定性。

6. `标识边界` 这一层主要依赖 queryId + plan_node_id。

它保证的是：标识用于关联，不保证跨版本绝对稳定。

不要把它误读成：query id 等于 SQL 文本主键。

如果一次诊断只能看到这一层的投影，就必须保留不确定性。

## 8. 错误路径 / 异常路径 / fallback

### 8.1. 忘记调用 previous hook

可见现象：其它扩展或 standard executor 行为丢失。

源码上应先回到：`auto_explain.c` hook 示例。

正确处理方式是：每个 hook 分支都必须调用 prev 或 standard。

这类路径通常不是“继续收集更多指标”就能解决。

要先判断观测值是在失败前、失败中还是失败后写入。

如果写入点本身在异常路径里，就要检查 cleanup 是否仍然执行。

### 8.2. Start 成功前记录过多

可见现象：standard_ExecutorStart ERROR 后 profiler 状态残留。

源码上应先回到：`execMain.c`、hook wrapper。

正确处理方式是：只记录可恢复状态，异常路径恢复 active/nesting。

这类路径通常不是“继续收集更多指标”就能解决。

要先判断观测值是在失败前、失败中还是失败后写入。

如果写入点本身在异常路径里，就要检查 cleanup 是否仍然执行。

### 8.3. End 后读取 PlanState

可见现象：访问已释放 EState 或 PlanState。

源码上应先回到：`standard_ExecutorEnd()`。

正确处理方式是：在调用 previous/standard End 前完成需要的读取。

这类路径通常不是“继续收集更多指标”就能解决。

要先判断观测值是在失败前、失败中还是失败后写入。

如果写入点本身在异常路径里，就要检查 cleanup 是否仍然执行。

### 8.4. 采样逻辑抛错

可见现象：profiler 让业务 SQL 失败。

源码上应先回到：GUC check、random、memory allocation。

正确处理方式是：采样路径保持简单，失败时禁用或丢弃记录。

这类路径通常不是“继续收集更多指标”就能解决。

要先判断观测值是在失败前、失败中还是失败后写入。

如果写入点本身在异常路径里，就要检查 cleanup 是否仍然执行。

### 8.5. 跨进程共享指针

可见现象：SRF 读取崩溃或脏数据。

源码上应先回到：shmem / DSM 规则。

正确处理方式是：共享区只保存值、offset、id，不保存 backend-local 指针。

这类路径通常不是“继续收集更多指标”就能解决。

要先判断观测值是在失败前、失败中还是失败后写入。

如果写入点本身在异常路径里，就要检查 cleanup 是否仍然执行。

## 9. 成本、资源与跨模块传播

成本分析要围绕同一个主问题。

不要把所有可能成本都列一遍。

### 9.1. hook 分支

主要扩张因子：每条查询进入次数。

放大方式：即使关闭也多一次函数调用和分支。

控制办法：cheap guard 放最前。

诊断时不要只看绝对值。

要看它是否随 rows、loops、backend 数、buffer miss、计划节点深度或采样率线性放大。

如果它只在某个节点或某类等待里集中出现，说明问题边界比 SQL 文本更窄。

### 9.2. 计时

主要扩张因子：clock_gettime 或 instr_time 调用次数。

放大方式：频繁 Run 或节点级 wrapper 会放大。

控制办法：只在采样命中后计时。

诊断时不要只看绝对值。

要看它是否随 rows、loops、backend 数、buffer miss、计划节点深度或采样率线性放大。

如果它只在某个节点或某类等待里集中出现，说明问题边界比 SQL 文本更窄。

### 9.3. 计划遍历

主要扩张因子：PlanState 节点数。

放大方式：Start/End 遍历会随计划复杂度增加。

控制办法：延迟到需要输出时遍历。

诊断时不要只看绝对值。

要看它是否随 rows、loops、backend 数、buffer miss、计划节点深度或采样率线性放大。

如果它只在某个节点或某类等待里集中出现，说明问题边界比 SQL 文本更窄。

### 9.4. 事件写入

主要扩张因子：事件大小和频率。

放大方式：共享锁或 ring 写入可能竞争。

控制办法：先 backend-local，再批量或 lock-light 写入。

诊断时不要只看绝对值。

要看它是否随 rows、loops、backend 数、buffer miss、计划节点深度或采样率线性放大。

如果它只在某个节点或某类等待里集中出现，说明问题边界比 SQL 文本更窄。

### 9.5. 递归保护

主要扩张因子：nested statement 数量。

放大方式：错误 nesting 会污染外层。

控制办法：用计数和 PG_FINALLY。

诊断时不要只看绝对值。

要看它是否随 rows、loops、backend 数、buffer miss、计划节点深度或采样率线性放大。

如果它只在某个节点或某类等待里集中出现，说明问题边界比 SQL 文本更窄。

### 9.6. 日志/输出

主要扩张因子：慢查询数量和格式化成本。

放大方式：End 阶段格式化可能很重。

控制办法：只记录结构化最小事件。

诊断时不要只看绝对值。

要看它是否随 rows、loops、backend 数、buffer miss、计划节点深度或采样率线性放大。

如果它只在某个节点或某类等待里集中出现，说明问题边界比 SQL 文本更窄。

## 10. 观测与诊断入口

观测入口要分清当前值、累计值、单次执行剖面和外部采样。

同一个慢 SQL 结论最好至少由两类信号互相校验。

### 10.1. auto_explain 源码

能看到：成熟 hook 模式。

看不到：不是 profiler 的完整实现。

源码入口：`contrib/auto_explain/auto_explain.c`。

使用时先问它是当前状态、累计状态，还是本次执行状态。

再问它是否会因为采样、刷新延迟、并行 worker 汇总或输出格式化而偏移。

最后把结论回扣到本节主流程中的某个写入点。

### 10.2. EXPLAIN Query Identifier

能看到：query id 是否可见。

看不到：不能显示 profiler 私有事件。

源码入口：`explain.c`。

使用时先问它是当前状态、累计状态，还是本次执行状态。

再问它是否会因为采样、刷新延迟、并行 worker 汇总或输出格式化而偏移。

最后把结论回扣到本节主流程中的某个写入点。

### 10.3. pg_stat_statements

能看到：query id 聚合。

看不到：没有自定义节点栈。

源码入口：`pg_stat_statements.c`。

使用时先问它是当前状态、累计状态，还是本次执行状态。

再问它是否会因为采样、刷新延迟、并行 worker 汇总或输出格式化而偏移。

最后把结论回扣到本节主流程中的某个写入点。

### 10.4. gdb 查看 hook 指针

能看到：全局 hook 是否被安装。

看不到：线上不宜常用。

源码入口：`ExecutorStart_hook` 等变量。

使用时先问它是当前状态、累计状态，还是本次执行状态。

再问它是否会因为采样、刷新延迟、并行 worker 汇总或输出格式化而偏移。

最后把结论回扣到本节主流程中的某个写入点。

### 10.5. 日志

能看到：最小事件是否落地。

看不到：过量日志会反噬系统。

源码入口：扩展自己的输出路径。

使用时先问它是当前状态、累计状态，还是本次执行状态。

再问它是否会因为采样、刷新延迟、并行 worker 汇总或输出格式化而偏移。

最后把结论回扣到本节主流程中的某个写入点。

### 10.6. 单元 SQL

能看到：启用/禁用/采样效果。

看不到：不能证明高并发开销。

源码入口：扩展回归测试。

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

### 12.1. hook 链审查

实验目标：把可见现象压回源码状态边界。

建议准备：

```text
cd /home/highgo/postgres
rg "prev_ExecutorStart|ExecutorStart_hook" contrib/auto_explain src
```

预期现象：能看到 auto_explain 保存 previous 并调用 previous 或 standard。

回到源码时检查：`auto_explain.c`。

实验完成后不要只记录数值。

还要记录计划节点、GUC、并发状态、缓存冷热和是否开启 timing。

这些条件决定同一个观测值能否迁移到另一个现场。

### 12.2. 最小事件字段设计

实验目标：把可见现象压回源码状态边界。

建议准备：

```text
记录 queryId
记录 start_ts / end_ts
记录 nesting_level
记录 whether_error
```

预期现象：字段足够支持生命周期检查，但不碰查询语义。

回到源码时检查：`QueryDesc`、`PlannedStmt.queryId`。

实验完成后不要只记录数值。

还要记录计划节点、GUC、并发状态、缓存冷热和是否开启 timing。

这些条件决定同一个观测值能否迁移到另一个现场。

### 12.3. End 前读取 PlanState

实验目标：把可见现象压回源码状态边界。

建议准备：

```text
在 ExecutorEnd hook 中先读取 plannedstmt/queryDesc/estate
再调用 previous/standard End
```

预期现象：读取顺序必须早于标准清理。

回到源码时检查：`standard_ExecutorEnd()`。

实验完成后不要只记录数值。

还要记录计划节点、GUC、并发状态、缓存冷热和是否开启 timing。

这些条件决定同一个观测值能否迁移到另一个现场。

### 12.4. ERROR 恢复演练

实验目标：把可见现象压回源码状态边界。

建议准备：

```text
在被采样 SQL 中触发 ERROR。
确认下一条 SQL 的 profiler_active/nesting 正常。
```

预期现象：异常不应污染后续查询。

回到源码时检查：PG_TRY/PG_FINALLY 边界。

实验完成后不要只记录数值。

还要记录计划节点、GUC、并发状态、缓存冷热和是否开启 timing。

这些条件决定同一个观测值能否迁移到另一个现场。

## 13. 讨论题

1. 为什么 profiler extension 的第一目标是语义隔离而不是功能完整？

2. ExecutorEnd hook 为什么要在调用 standard End 前读取状态？

3. 哪些事件字段可以跨 backend 保存，哪些只能在本地使用？

4. 为什么 previous hook 链是扩展生态的正确性问题？

5. 采样命中前能做哪些事，哪些事必须延后？

## 14. 本节小结

本节只沉淀一个模型：

```text
最小 profiler extension 从 executor hook 链开始。
所有 hook 必须保存并调用 previous 或 standard executor。
cheap guard、采样和 GUC 控制未启用路径成本。
ERROR-safe 的 backend-local 状态恢复比事件字段更多更重要。
只观测不改语义是 profiler 的第一正确性边界。
```

下一节会讨论如何记录节点级计时和嵌套调用栈。
