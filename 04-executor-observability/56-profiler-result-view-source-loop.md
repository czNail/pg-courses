# PostgreSQL Profiler 结果视图与源码闭环

## 课程定位

前置知识：已经有 hook、节点计时、ring buffer 和 guardrail；本节把结果暴露给 SQL 并回到源码定位。

本节唯一主问题：

```text
如何把 profiler 采样结果暴露成 SRF / view，并把节点 id、plan id、source symbol 和 EXPLAIN 输出关联起来，形成从慢 SQL 到源码热点的闭环？
```

核心矛盾：共享 ring buffer 中保存的是低成本事件值；用户需要的是可查询、可关联、可解释的结果视图。输出层越友好，越容易引入权限、锁、格式化和语义误读。

学完后应能设计一个 profiler result view：短锁复制、权限可控、字段能关联 EXPLAIN 和源码，并明确哪些关联是精确的、哪些是近似的。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

上一节完成了 profiler 的线上保护。
最后一节收束到诊断闭环。
profiler 如果只能收集事件而不能被 SQL 查询，就很难和 EXPLAIN、pg_stat_statements、源码符号结合。
但 view 不是简单把共享内存结构体暴露出去。
本节唯一主问题是如何把低成本事件变成可解释的 SRF/view，同时不破坏共享状态和权限边界。

```text
profiler ring event
  -> SRF copies snapshot
  -> heap_form_tuple / tuplestore
  -> SQL view
  -> join queryId / plan_node_id / pid / timestamps
  -> EXPLAIN and source entry
  -> reproducible diagnosis
```

本节只处理自己这一环，不把相邻主题扩成第二个主问题。

阅读时要保持一条线：先从用户可见现象进入，再回到状态写入点，最后再回到可验证实验。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
结果视图在 SRF first call 中检查权限，短锁复制 ring events 到本地 memory context，释放锁后用 tuple descriptor 输出；字段以 queryId、plan_node_id、pid、database、userid、node type、inclusive/exclusive time、calls、tuples、source symbol、dropped flags 为主，再通过 EXPLAIN 和源码入口完成闭环。
```

这里的 tension 是：结果要足够解释问题，但不能把共享内存指针、原始 source text 或不稳定内部状态暴露成长期契约。

本节只问一个问题：profiler view 的字段和读取流程如何服务源码闭环。

这个模型的诊断含义是：

- 不要先背函数名，先判断状态属于哪个生命周期边界。

- 不要把用户可见指标当成源码真相，指标只是某个状态边界的投影。

- 不要把一次采样、一次累计值和一次执行剖面混在一起解释。

- 每个观测值都要回答：谁写入，何时可见，谁清理，异常时是否仍可靠。

本节后面的源码 walkthrough 都围绕这一个模型展开。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/utils/fmgr/funcapi.c` | `FuncCallContext`、SRF helper、tuple descriptor 支持。 |
| 2 | `src/backend/utils/adt/pgstatfuncs.c` | 系统统计 SRF 如何用 `ReturnSetInfo` / tuplestore 输出。 |
| 3 | `src/backend/utils/misc/guc_funcs.c` | `SRF_FIRSTCALL_INIT()`、`SRF_RETURN_NEXT()`、`SRF_RETURN_DONE()` 示例。 |
| 4 | `contrib/pg_stat_statements/pg_stat_statements.c` | 扩展 view / SRF 输出、权限、shared state 读取和 tuple forming 示例。 |
| 5 | `src/include/commands/explain.h` | `explain_per_plan_hook`、`explain_per_node_hook` 作为 EXPLAIN 扩展点。 |
| 6 | `src/backend/commands/explain.c` | `ExplainNode()` 输出 Node Type、actual rows/time、Query Identifier。 |
| 7 | `src/include/nodes/plannodes.h` | `PlannedStmt.queryId` 和 `Plan.plan_node_id`。 |
| 8 | `src/backend/nodes/queryjumblefuncs.c` | `JumbleQuery()` 如何生成 query id。 |
| 9 | `src/backend/access/common/heaptuple.c` | `heap_form_tuple()` 构造返回 tuple。 |
| 10 | `src/backend/access/common/tupdesc.c` | `BlessTupleDesc()` 相关 tuple descriptor 边界。 |

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

### 4.1. `profiler_event_snapshot`

SRF 复制到本地的事件数组。

它的持有者是：结果函数 first call。

它的读取者是：SRF next-call loop。

生命周期边界是：一次 SQL 函数调用内有效。

诊断价值是：避免持锁遍历共享 ring。

读这个状态时不要只看字段名。

先问它在 backend-local、shared memory、DSM、文件系统还是调用栈里。

再问它的写入者是否和读取者在同一条时间线上。

最后问它失败时会留下错误值、缺失值，还是只能让上层退化成推断。

在本节唯一主问题中，它只服务于：如何把 profiler 采样结果暴露成 SRF / view，并把节点 id、plan id、source symbol 和 EXPLAIN 输出关联起来，形成从慢 SQL 到源码热点的闭环？

如果这个状态不能解释这一点，就不应该把它扩成另一条主线。

### 4.2. `TupleDesc`

结果行结构定义。

它的持有者是：SRF 初始化或 SQL extension schema。

它的读取者是：heap_form_tuple / tuplestore。

生命周期边界是：函数调用期间或系统缓存中有效。

诊断价值是：定义 view 对外稳定字段。

读这个状态时不要只看字段名。

先问它在 backend-local、shared memory、DSM、文件系统还是调用栈里。

再问它的写入者是否和读取者在同一条时间线上。

最后问它失败时会留下错误值、缺失值，还是只能让上层退化成推断。

在本节唯一主问题中，它只服务于：如何把 profiler 采样结果暴露成 SRF / view，并把节点 id、plan id、source symbol 和 EXPLAIN 输出关联起来，形成从慢 SQL 到源码热点的闭环？

如果这个状态不能解释这一点，就不应该把它扩成另一条主线。

### 4.3. `queryId`

归一化查询标识。

它的持有者是：JumbleQuery / planner。

它的读取者是：profiler、pg_stat_statements、EXPLAIN Query Identifier。

生命周期边界是：同一规范化查询下稳定，但受版本和配置影响。

诊断价值是：关联长期统计和 profiler 事件。

读这个状态时不要只看字段名。

先问它在 backend-local、shared memory、DSM、文件系统还是调用栈里。

再问它的写入者是否和读取者在同一条时间线上。

最后问它失败时会留下错误值、缺失值，还是只能让上层退化成推断。

在本节唯一主问题中，它只服务于：如何把 profiler 采样结果暴露成 SRF / view，并把节点 id、plan id、source symbol 和 EXPLAIN 输出关联起来，形成从慢 SQL 到源码热点的闭环？

如果这个状态不能解释这一点，就不应该把它扩成另一条主线。

### 4.4. `plan_node_id`

最终计划树内节点唯一 id。

它的持有者是：planner/setrefs 后计划。

它的读取者是：executor、parallel DSM、profiler metadata。

生命周期边界是：一次计划生命周期内稳定。

诊断价值是：关联 profiler 节点事件与 EXPLAIN 计划遍历。

读这个状态时不要只看字段名。

先问它在 backend-local、shared memory、DSM、文件系统还是调用栈里。

再问它的写入者是否和读取者在同一条时间线上。

最后问它失败时会留下错误值、缺失值，还是只能让上层退化成推断。

在本节唯一主问题中，它只服务于：如何把 profiler 采样结果暴露成 SRF / view，并把节点 id、plan id、source symbol 和 EXPLAIN 输出关联起来，形成从慢 SQL 到源码热点的闭环？

如果这个状态不能解释这一点，就不应该把它扩成另一条主线。

### 4.5. `node_type`

PlanState / Plan node 的节点类型名。

它的持有者是：PlanState 元数据或 ExplainNode。

它的读取者是：结果 view 和用户。

生命周期边界是：一次事件内固定。

诊断价值是：让用户无需先解析源码枚举。

读这个状态时不要只看字段名。

先问它在 backend-local、shared memory、DSM、文件系统还是调用栈里。

再问它的写入者是否和读取者在同一条时间线上。

最后问它失败时会留下错误值、缺失值，还是只能让上层退化成推断。

在本节唯一主问题中，它只服务于：如何把 profiler 采样结果暴露成 SRF / view，并把节点 id、plan id、source symbol 和 EXPLAIN 输出关联起来，形成从慢 SQL 到源码热点的闭环？

如果这个状态不能解释这一点，就不应该把它扩成另一条主线。

### 4.6. `source_symbol`

采样或 wrapper 记录的源码函数符号。

它的持有者是：profiler 或外部符号化。

它的读取者是：结果 view。

生命周期边界是：取决于符号质量和采样方式。

诊断价值是：把 node event 回到 C 函数入口。

读这个状态时不要只看字段名。

先问它在 backend-local、shared memory、DSM、文件系统还是调用栈里。

再问它的写入者是否和读取者在同一条时间线上。

最后问它失败时会留下错误值、缺失值，还是只能让上层退化成推断。

在本节唯一主问题中，它只服务于：如何把 profiler 采样结果暴露成 SRF / view，并把节点 id、plan id、source symbol 和 EXPLAIN 输出关联起来，形成从慢 SQL 到源码热点的闭环？

如果这个状态不能解释这一点，就不应该把它扩成另一条主线。

### 4.7. `dropped_flags`

事件是否被截断、覆盖或采样丢弃的标志。

它的持有者是：ring writer / guard。

它的读取者是：结果 view。

生命周期边界是：事件生命周期内固定。

诊断价值是：防止用户把不完整结果当完整事实。

读这个状态时不要只看字段名。

先问它在 backend-local、shared memory、DSM、文件系统还是调用栈里。

再问它的写入者是否和读取者在同一条时间线上。

最后问它失败时会留下错误值、缺失值，还是只能让上层退化成推断。

在本节唯一主问题中，它只服务于：如何把 profiler 采样结果暴露成 SRF / view，并把节点 id、plan id、source symbol 和 EXPLAIN 输出关联起来，形成从慢 SQL 到源码热点的闭环？

如果这个状态不能解释这一点，就不应该把它扩成另一条主线。

本节最重要的不变量是：

```text
raw field 不是语义；
field + writer + reader + lifecycle + cleanup 才是诊断语义。
```

## 5. 主流程源码 walkthrough

下面按一次慢查询诊断或 profiler 采样的时间线走主流程。

```text
SQL 调用 profiler_events()
SRF first call 检查权限
短锁复制 ring snapshot
释放锁后构造 tuple
view 暴露稳定字段
关联 EXPLAIN
关联源码入口
形成闭环结论
```

### 5.1. SQL 调用 profiler_events()

用户通过 SRF 或 view 请求 profiler 结果。

这一段改变的状态边界是：SQL 输出入口。

回到诊断时要验证：调用者权限和参数范围是否合格。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

### 5.2. SRF first call 检查权限

根据 superuser、pg_read_all_stats 或扩展策略决定能看哪些事件。

这一段改变的状态边界是：权限边界。

回到诊断时要验证：是否泄露其它用户 SQL 或数据库信息。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

### 5.3. 短锁复制 ring snapshot

持 profiler LWLock 复制 header、events、dropped counters 到本地。

这一段改变的状态边界是：共享读取边界。

回到诊断时要验证：是否避免持锁 form tuple。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

### 5.4. 释放锁后构造 tuple

用 TupleDesc、heap_form_tuple 或 tuplestore 在本地内存中输出行。

这一段改变的状态边界是：格式化边界。

回到诊断时要验证：共享状态是否已不再被访问。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

### 5.5. view 暴露稳定字段

SQL view 重命名字段、隐藏内部 flags、提供过滤条件。

这一段改变的状态边界是：对外契约边界。

回到诊断时要验证：字段是否能支持 queryId、plan_node_id、时间窗口过滤。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

### 5.6. 关联 EXPLAIN

用 queryId 找到 SQL，再重跑或抓取 EXPLAIN；用 node type/plan_node_id/遍历顺序关联节点。

这一段改变的状态边界是：计划关联边界。

回到诊断时要验证：计划是否因 GUC、统计信息、参数变化而改变。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

### 5.7. 关联源码入口

根据 node_type 和 source_symbol 回到 `node*.c`、`execExpr*`、`tuplesort.c` 等文件。

这一段改变的状态边界是：源码定位边界。

回到诊断时要验证：函数符号是否存在并对应当前 commit。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

### 5.8. 形成闭环结论

结论同时列出 SQL、plan node、runtime 指标、source function 和复现实验。

这一段改变的状态边界是：诊断报告边界。

回到诊断时要验证：是否把 workload-dependent 结论说成普遍规律。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

## 6. 生命周期 / ownership / cleanup

### 6.1. SRF 调用上下文

创建或进入者：SRF_FIRSTCALL_INIT 或 materialized SRF。

正常清理者：SRF_RETURN_DONE。

异常路径依赖：ERROR 时 memory context 清理本地 snapshot。

这里要区分三件事。

MemoryContext 管内存批量释放。

ResourceOwner 管 pin、refcount、文件句柄、锁等外部资源。

全局或共享状态必须有独立的可见性、锁或代际边界。

把这些混成一句“事务结束释放”，会丢掉定位问题所需的时间顺序。

### 6.2. 共享 ring 读取

创建或进入者：reader 获取 LWLock。

正常清理者：复制完成立即释放。

异常路径依赖：读取过程中不能调用可能长时间阻塞的格式化逻辑。

这里要区分三件事。

MemoryContext 管内存批量释放。

ResourceOwner 管 pin、refcount、文件句柄、锁等外部资源。

全局或共享状态必须有独立的可见性、锁或代际边界。

把这些混成一句“事务结束释放”，会丢掉定位问题所需的时间顺序。

### 6.3. 结果 tuple

创建或进入者：heap_form_tuple / tuplestore_puttuple。

正常清理者：SQL executor 消费后释放。

异常路径依赖：tuple 值不能引用共享区临时指针。

这里要区分三件事。

MemoryContext 管内存批量释放。

ResourceOwner 管 pin、refcount、文件句柄、锁等外部资源。

全局或共享状态必须有独立的可见性、锁或代际边界。

把这些混成一句“事务结束释放”，会丢掉定位问题所需的时间顺序。

### 6.4. query id 关联

创建或进入者：事件写入时保存。

正常清理者：长期统计 reset 或 query id 变化后失效。

异常路径依赖：关联必须标注时间窗口。

这里要区分三件事。

MemoryContext 管内存批量释放。

ResourceOwner 管 pin、refcount、文件句柄、锁等外部资源。

全局或共享状态必须有独立的可见性、锁或代际边界。

把这些混成一句“事务结束释放”，会丢掉定位问题所需的时间顺序。

### 6.5. plan node 关联

创建或进入者：事件写入时保存 plan_node_id。

正常清理者：计划重新生成后可能不同。

异常路径依赖：不能跨计划版本盲目 join。

这里要区分三件事。

MemoryContext 管内存批量释放。

ResourceOwner 管 pin、refcount、文件句柄、锁等外部资源。

全局或共享状态必须有独立的可见性、锁或代际边界。

把这些混成一句“事务结束释放”，会丢掉定位问题所需的时间顺序。

### 6.6. 源码闭环

创建或进入者：当前 commit 下查找函数和文件。

正常清理者：版本升级后重新核对。

异常路径依赖：结论要写明源码基线。

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
| 短锁读取 | copy snapshot then format | SRF 不持锁输出大量 tuple。 | 直接遍历共享 ring 更简单也安全。 |
| 权限控制 | stats 权限和用户过滤 | 敏感 SQL 和跨用户数据受控。 | 诊断 view 可以默认公开。 |
| 字段稳定 | SQL schema | 对外字段应是值语义。 | 暴露结构体布局。 |
| query 关联 | queryId | 同规范化查询可聚合关联。 | queryId 是跨版本永久主键。 |
| plan 关联 | plan_node_id / traversal | 一次计划内节点可定位。 | EXPLAIN 默认一定显示 plan_node_id。 |
| 源码关联 | source_symbol + commit | 函数名必须在当前源码存在。 | 符号永远不变。 |

读源码时可以按这个顺序检查：

```text
先找写入点
  -> 再找读取点
  -> 再找清理点
  -> 再看 ERROR / abort / backend exit 是否能兜底
  -> 最后把用户可见输出映射回这一条状态链
```

1. `短锁读取` 这一层主要依赖 copy snapshot then format。

它保证的是：SRF 不持锁输出大量 tuple。

不要把它误读成：直接遍历共享 ring 更简单也安全。

如果一次诊断只能看到这一层的投影，就必须保留不确定性。

2. `权限控制` 这一层主要依赖 stats 权限和用户过滤。

它保证的是：敏感 SQL 和跨用户数据受控。

不要把它误读成：诊断 view 可以默认公开。

如果一次诊断只能看到这一层的投影，就必须保留不确定性。

3. `字段稳定` 这一层主要依赖 SQL schema。

它保证的是：对外字段应是值语义。

不要把它误读成：暴露结构体布局。

如果一次诊断只能看到这一层的投影，就必须保留不确定性。

4. `query 关联` 这一层主要依赖 queryId。

它保证的是：同规范化查询可聚合关联。

不要把它误读成：queryId 是跨版本永久主键。

如果一次诊断只能看到这一层的投影，就必须保留不确定性。

5. `plan 关联` 这一层主要依赖 plan_node_id / traversal。

它保证的是：一次计划内节点可定位。

不要把它误读成：EXPLAIN 默认一定显示 plan_node_id。

如果一次诊断只能看到这一层的投影，就必须保留不确定性。

6. `源码关联` 这一层主要依赖 source_symbol + commit。

它保证的是：函数名必须在当前源码存在。

不要把它误读成：符号永远不变。

如果一次诊断只能看到这一层的投影，就必须保留不确定性。

## 8. 错误路径 / 异常路径 / fallback

### 8.1. 权限不足

可见现象：查询 view 报错或隐藏字段。

源码上应先回到：SRF permission check。

正确处理方式是：按 PostgreSQL stats 权限策略处理。

这类路径通常不是“继续收集更多指标”就能解决。

要先判断观测值是在失败前、失败中还是失败后写入。

如果写入点本身在异常路径里，就要检查 cleanup 是否仍然执行。

### 8.2. ring 覆盖

可见现象：事件序号不连续或 dropped flags。

源码上应先回到：ring header。

正确处理方式是：结果 view 显示 gap，不假装完整。

这类路径通常不是“继续收集更多指标”就能解决。

要先判断观测值是在失败前、失败中还是失败后写入。

如果写入点本身在异常路径里，就要检查 cleanup 是否仍然执行。

### 8.3. 计划不一致

可见现象：用 queryId 重跑 EXPLAIN 后节点不匹配。

源码上应先回到：planner / GUC / stats。

正确处理方式是：保存 plan hash、node type、时间窗口或当场抓 EXPLAIN。

这类路径通常不是“继续收集更多指标”就能解决。

要先判断观测值是在失败前、失败中还是失败后写入。

如果写入点本身在异常路径里，就要检查 cleanup 是否仍然执行。

### 8.4. source symbol 缺失

可见现象：view 只有 node_type 没有函数符号。

源码上应先回到：profiler symbolization。

正确处理方式是：回到 node type 和 static source mapping。

这类路径通常不是“继续收集更多指标”就能解决。

要先判断观测值是在失败前、失败中还是失败后写入。

如果写入点本身在异常路径里，就要检查 cleanup 是否仍然执行。

### 8.5. 大结果读取扰动

可见现象：查询 profiler view 本身变慢。

源码上应先回到：SRF copy/output path。

正确处理方式是：限制窗口、分页、top N。

这类路径通常不是“继续收集更多指标”就能解决。

要先判断观测值是在失败前、失败中还是失败后写入。

如果写入点本身在异常路径里，就要检查 cleanup 是否仍然执行。

## 9. 成本、资源与跨模块传播

成本分析要围绕同一个主问题。

不要把所有可能成本都列一遍。

### 9.1. SRF snapshot

主要扩张因子：ring capacity。

放大方式：复制越大，读取越慢。

控制办法：限制 capacity 和查询窗口。

诊断时不要只看绝对值。

要看它是否随 rows、loops、backend 数、buffer miss、计划节点深度或采样率线性放大。

如果它只在某个节点或某类等待里集中出现，说明问题边界比 SQL 文本更窄。

### 9.2. tuple forming

主要扩张因子：返回行数和字段数。

放大方式：heap_form_tuple/tuplestore 成本。

控制办法：只输出必要字段。

诊断时不要只看绝对值。

要看它是否随 rows、loops、backend 数、buffer miss、计划节点深度或采样率线性放大。

如果它只在某个节点或某类等待里集中出现，说明问题边界比 SQL 文本更窄。

### 9.3. 权限过滤

主要扩张因子：事件数和过滤条件复杂度。

放大方式：读取时扫描成本。

控制办法：先按时间/seq/pid 粗过滤。

诊断时不要只看绝对值。

要看它是否随 rows、loops、backend 数、buffer miss、计划节点深度或采样率线性放大。

如果它只在某个节点或某类等待里集中出现，说明问题边界比 SQL 文本更窄。

### 9.4. EXPLAIN 关联

主要扩张因子：重跑计划成本。

放大方式：可能改变缓存和负载。

控制办法：必要时只对少数事件做。

诊断时不要只看绝对值。

要看它是否随 rows、loops、backend 数、buffer miss、计划节点深度或采样率线性放大。

如果它只在某个节点或某类等待里集中出现，说明问题边界比 SQL 文本更窄。

### 9.5. 源码映射

主要扩张因子：符号数量和版本。

放大方式：离线映射比在线解析更稳。

控制办法：保存 commit 和 source path。

诊断时不要只看绝对值。

要看它是否随 rows、loops、backend 数、buffer miss、计划节点深度或采样率线性放大。

如果它只在某个节点或某类等待里集中出现，说明问题边界比 SQL 文本更窄。

### 9.6. 结果保留

主要扩张因子：ring size 和 reset 频率。

放大方式：过长窗口占 shared memory。

控制办法：通过 reset 和导出控制。

诊断时不要只看绝对值。

要看它是否随 rows、loops、backend 数、buffer miss、计划节点深度或采样率线性放大。

如果它只在某个节点或某类等待里集中出现，说明问题边界比 SQL 文本更窄。

## 10. 观测与诊断入口

观测入口要分清当前值、累计值、单次执行剖面和外部采样。

同一个慢 SQL 结论最好至少由两类信号互相校验。

### 10.1. profiler_events()

能看到：节点事件、时间、调用、标识。

看不到：未采样或被覆盖事件。

源码入口：扩展 SRF。

使用时先问它是当前状态、累计状态，还是本次执行状态。

再问它是否会因为采样、刷新延迟、并行 worker 汇总或输出格式化而偏移。

最后把结论回扣到本节主流程中的某个写入点。

### 10.2. profiler_info()

能看到：capacity、dropped、reset_time、guard counters。

看不到：单个节点细节。

源码入口：扩展 SRF。

使用时先问它是当前状态、累计状态，还是本次执行状态。

再问它是否会因为采样、刷新延迟、并行 worker 汇总或输出格式化而偏移。

最后把结论回扣到本节主流程中的某个写入点。

### 10.3. EXPLAIN (ANALYZE, VERBOSE)

能看到：节点类型、actual rows/time、Query Identifier。

看不到：profiler 独占时间。

源码入口：`explain.c`。

使用时先问它是当前状态、累计状态，还是本次执行状态。

再问它是否会因为采样、刷新延迟、并行 worker 汇总或输出格式化而偏移。

最后把结论回扣到本节主流程中的某个写入点。

### 10.4. pg_stat_statements

能看到：长期 query id 指标。

看不到：节点级事件。

源码入口：`pg_stat_statements.c`。

使用时先问它是当前状态、累计状态，还是本次执行状态。

再问它是否会因为采样、刷新延迟、并行 worker 汇总或输出格式化而偏移。

最后把结论回扣到本节主流程中的某个写入点。

### 10.5. 源码 rg

能看到：函数和文件真实位置。

看不到：运行时成本。

源码入口：`rg source_symbol /home/highgo/postgres/src`。

使用时先问它是当前状态、累计状态，还是本次执行状态。

再问它是否会因为采样、刷新延迟、并行 worker 汇总或输出格式化而偏移。

最后把结论回扣到本节主流程中的某个写入点。

### 10.6. perf

能看到：符号级 CPU 样本。

看不到：query id / node id。

源码入口：外部 profiler。

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

### 12.1. SRF 短锁读取

实验目标：把可见现象压回源码状态边界。

建议准备：

```text
高并发写 profiler events。
同时查询 profiler_events()。
观察是否出现 profiler LWLock wait。
```

预期现象：正常设计下读取应短锁复制，不应长期阻塞 writer。

回到源码时检查：SRF read path 和 LWLock 持有区。

实验完成后不要只记录数值。

还要记录计划节点、GUC、并发状态、缓存冷热和是否开启 timing。

这些条件决定同一个观测值能否迁移到另一个现场。

### 12.2. queryId 关联

实验目标：把可见现象压回源码状态边界。

建议准备：

```text
运行慢 SQL。
查询 profiler_events() 得到 query_id。
在 pg_stat_statements 中按 queryid 查聚合指标。
```

预期现象：同一 query id 可把单次 profiler 事件和长期统计关联。

回到源码时检查：`PlannedStmt.queryId`、`pg_stat_statements.c`。

实验完成后不要只记录数值。

还要记录计划节点、GUC、并发状态、缓存冷热和是否开启 timing。

这些条件决定同一个观测值能否迁移到另一个现场。

### 12.3. plan_node_id 关联

实验目标：把可见现象压回源码状态边界。

建议准备：

```text
查询 profiler_events() 得到 plan_node_id 和 node_type。
重跑 EXPLAIN (ANALYZE, VERBOSE)。
按 node type 和计划树位置核对。
```

预期现象：一次计划内可定位节点，但计划变化会破坏关联。

回到源码时检查：`plannodes.h`、`explain.c`。

实验完成后不要只记录数值。

还要记录计划节点、GUC、并发状态、缓存冷热和是否开启 timing。

这些条件决定同一个观测值能否迁移到另一个现场。

### 12.4. source symbol 闭环

实验目标：把可见现象压回源码状态边界。

建议准备：

```text
从 view 取 source_symbol。
cd /home/highgo/postgres
rg "source_symbol" src contrib
```

预期现象：符号应能回到当前源码文件。

回到源码时检查：最终诊断要写明 commit。

实验完成后不要只记录数值。

还要记录计划节点、GUC、并发状态、缓存冷热和是否开启 timing。

这些条件决定同一个观测值能否迁移到另一个现场。

## 13. 讨论题

1. 为什么 view 应暴露值字段而不是共享结构体布局？

2. queryId 和 plan_node_id 分别解决什么关联问题，各自有什么不稳定性？

3. 为什么 EXPLAIN 默认不一定给出 profiler 需要的全部关联字段？

4. SRF 读取共享 ring 时，哪些操作必须在锁外完成？

5. 一个完整慢 SQL 源码闭环报告至少应包含哪些证据？

## 14. 本节小结

本节只沉淀一个模型：

```text
profiler view 是低成本事件到可解释诊断的边界。
SRF 读取共享 ring 时应短锁复制，再在本地格式化。
queryId 关联长期统计，plan_node_id 关联一次计划内节点。
源码符号必须基于明确 commit 重新核对。
结果 view 要暴露 dropped/flags，让缺失数据可解释。
```

至此 04 目录从 executor runtime、EXPLAIN、pg_stat、wait event、hook 到 profiler 闭环完成一条观测主线。
