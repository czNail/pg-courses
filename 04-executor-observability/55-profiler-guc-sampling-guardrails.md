# PostgreSQL Profiler GUC、采样率与线上保护

## 课程定位

前置知识：已经有 hook、节点计时和共享 ring buffer 的模型，知道 profiler 自身可能成为执行路径成本。

本节唯一主问题：

```text
如何设计 enable 开关、采样率、最小时长、用户 / database 过滤和最大记录数，让 profiler 能在线上临时开启而不制造新的瓶颈？
```

核心矛盾：线上诊断需要临时打开更多观测，但 profiler 越详细越可能放大 CPU、锁、内存和日志成本；保护栏必须先于昂贵逻辑生效。

学完后应能评价 profiler 的 GUC 设计是否默认关闭、边界清晰、采样可控、权限和资源上限合理。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

上一节把事件放进共享 ring buffer。
现在的问题是：什么时候允许写事件。
没有 GUC 和采样保护的 profiler，本质上是在所有 SQL 上默认加负载。
PostgreSQL 扩展通常通过自定义 GUC 暴露开关、阈值和格式选项。
本节唯一主问题是这些 guardrail 应该放在 hook 的哪个时间点，以及如何保证默认安全。
下一节会把采样结果暴露成 view 并闭环到源码。

```text
_PG_init
  -> DefineCustomBool/Int/Real/StringVariable
  -> MarkGUCPrefixReserved
  -> hook cheap guard
  -> sample decision
  -> min_duration / filters
  -> bounded write
```

本节只处理自己这一环，不把相邻主题扩成第二个主问题。

阅读时要保持一条线：先从用户可见现象进入，再回到状态写入点，最后再回到可验证实验。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
profiler GUC 分两类：启动期 GUC 决定 shared memory 和最大容量；运行期 GUC 决定是否启用、采样率、最小时长、过滤条件和输出限制。hook 入口必须先执行 cheap guard，只有命中采样且满足过滤条件后才开启节点级计时和 ring 写入。
```

这里的 tension 是：诊断需要可动态调整，资源上界又必须稳定。

本节只问一个问题：怎样让 profiler 在线上可临时开启，同时不失控。

这个模型的诊断含义是：

- 不要先背函数名，先判断状态属于哪个生命周期边界。

- 不要把用户可见指标当成源码真相，指标只是某个状态边界的投影。

- 不要把一次采样、一次累计值和一次执行剖面混在一起解释。

- 每个观测值都要回答：谁写入，何时可见，谁清理，异常时是否仍可靠。

本节后面的源码 walkthrough 都围绕这一个模型展开。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `contrib/auto_explain/auto_explain.c` | 自定义 GUC、sample_rate、log_min_duration、timing/buffers/io/wal 开关。 |
| 2 | `src/backend/utils/misc/guc.c` | `DefineCustomBoolVariable()`、`DefineCustomIntVariable()`、`DefineCustomRealVariable()`、`MarkGUCPrefixReserved()`。 |
| 3 | `src/backend/utils/misc/guc_tables.c` | 核心 GUC 定义和上下文语义参考。 |
| 4 | `src/backend/utils/misc/guc_parameters.dat` | GUC 元数据和示例参数。 |
| 5 | `src/include/utils/guc.h` | GUC context、flags、check/assign/show hook 类型。 |
| 6 | `src/backend/executor/execMain.c` | Executor hook 调用时机，cheap guard 放置位置。 |
| 7 | `src/backend/utils/init/miscinit.c` | 用户、database、session 状态相关入口。 |
| 8 | `contrib/pg_stat_statements/pg_stat_statements.c` | 启动期最大条目、shared memory 和 GUC 组合示例。 |

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

### 4.1. `profiler.enabled`

运行期开关，默认关闭。

它的持有者是：自定义 GUC。

它的读取者是：hook cheap guard。

生命周期边界是：backend 配置生命周期内有效。

诊断价值是：第一道保护栏，未开启时只付极低成本。

读这个状态时不要只看字段名。

先问它在 backend-local、shared memory、DSM、文件系统还是调用栈里。

再问它的写入者是否和读取者在同一条时间线上。

最后问它失败时会留下错误值、缺失值，还是只能让上层退化成推断。

在本节唯一主问题中，它只服务于：如何设计 enable 开关、采样率、最小时长、用户 / database 过滤和最大记录数，让 profiler 能在线上临时开启而不制造新的瓶颈？

如果这个状态不能解释这一点，就不应该把它扩成另一条主线。

### 4.2. `profiler.sample_rate`

0 到 1 的采样比例。

它的持有者是：DefineCustomRealVariable。

它的读取者是：ExecutorStart hook。

生命周期边界是：每条 query 做一次采样决定。

诊断价值是：控制命中比例，而不是事件写入后再丢弃。

读这个状态时不要只看字段名。

先问它在 backend-local、shared memory、DSM、文件系统还是调用栈里。

再问它的写入者是否和读取者在同一条时间线上。

最后问它失败时会留下错误值、缺失值，还是只能让上层退化成推断。

在本节唯一主问题中，它只服务于：如何设计 enable 开关、采样率、最小时长、用户 / database 过滤和最大记录数，让 profiler 能在线上临时开启而不制造新的瓶颈？

如果这个状态不能解释这一点，就不应该把它扩成另一条主线。

### 4.3. `profiler.min_duration`

低于阈值的查询不输出或不保留。

它的持有者是：DefineCustomIntVariable。

它的读取者是：ExecutorEnd hook。

生命周期边界是：query 生命周期内判断。

诊断价值是：避免大量短查询事件淹没 ring。

读这个状态时不要只看字段名。

先问它在 backend-local、shared memory、DSM、文件系统还是调用栈里。

再问它的写入者是否和读取者在同一条时间线上。

最后问它失败时会留下错误值、缺失值，还是只能让上层退化成推断。

在本节唯一主问题中，它只服务于：如何设计 enable 开关、采样率、最小时长、用户 / database 过滤和最大记录数，让 profiler 能在线上临时开启而不制造新的瓶颈？

如果这个状态不能解释这一点，就不应该把它扩成另一条主线。

### 4.4. `profiler.max_events_per_query`

单条查询最多记录的节点事件数。

它的持有者是：GUC 和 query-local counter。

它的读取者是：node wrapper。

生命周期边界是：每条查询重置。

诊断价值是：防止复杂计划或高频节点输出爆炸。

读这个状态时不要只看字段名。

先问它在 backend-local、shared memory、DSM、文件系统还是调用栈里。

再问它的写入者是否和读取者在同一条时间线上。

最后问它失败时会留下错误值、缺失值，还是只能让上层退化成推断。

在本节唯一主问题中，它只服务于：如何设计 enable 开关、采样率、最小时长、用户 / database 过滤和最大记录数，让 profiler 能在线上临时开启而不制造新的瓶颈？

如果这个状态不能解释这一点，就不应该把它扩成另一条主线。

### 4.5. `profiler.user_filter`

允许按 user 限定采样范围。

它的持有者是：GUC string 或 Oid set。

它的读取者是：hook guard。

生命周期边界是：配置变化后重新解析。

诊断价值是：减少无关租户或系统会话的扰动。

读这个状态时不要只看字段名。

先问它在 backend-local、shared memory、DSM、文件系统还是调用栈里。

再问它的写入者是否和读取者在同一条时间线上。

最后问它失败时会留下错误值、缺失值，还是只能让上层退化成推断。

在本节唯一主问题中，它只服务于：如何设计 enable 开关、采样率、最小时长、用户 / database 过滤和最大记录数，让 profiler 能在线上临时开启而不制造新的瓶颈？

如果这个状态不能解释这一点，就不应该把它扩成另一条主线。

### 4.6. `profiler.db_filter`

允许按 database 限定采样范围。

它的持有者是：GUC string 或 Oid set。

它的读取者是：hook guard。

生命周期边界是：backend database 生命周期内稳定。

诊断价值是：避免跨库噪声。

读这个状态时不要只看字段名。

先问它在 backend-local、shared memory、DSM、文件系统还是调用栈里。

再问它的写入者是否和读取者在同一条时间线上。

最后问它失败时会留下错误值、缺失值，还是只能让上层退化成推断。

在本节唯一主问题中，它只服务于：如何设计 enable 开关、采样率、最小时长、用户 / database 过滤和最大记录数，让 profiler 能在线上临时开启而不制造新的瓶颈？

如果这个状态不能解释这一点，就不应该把它扩成另一条主线。

### 4.7. `profiler.dropped_by_guard`

因 GUC、采样、阈值、max events 被跳过的计数。

它的持有者是：hook guard 和 writer。

它的读取者是：info view。

生命周期边界是：扩展生命周期内累计。

诊断价值是：解释没有事件时到底是未命中还是没有慢 SQL。

读这个状态时不要只看字段名。

先问它在 backend-local、shared memory、DSM、文件系统还是调用栈里。

再问它的写入者是否和读取者在同一条时间线上。

最后问它失败时会留下错误值、缺失值，还是只能让上层退化成推断。

在本节唯一主问题中，它只服务于：如何设计 enable 开关、采样率、最小时长、用户 / database 过滤和最大记录数，让 profiler 能在线上临时开启而不制造新的瓶颈？

如果这个状态不能解释这一点，就不应该把它扩成另一条主线。

本节最重要的不变量是：

```text
raw field 不是语义；
field + writer + reader + lifecycle + cleanup 才是诊断语义。
```

## 5. 主流程源码 walkthrough

下面按一次慢查询诊断或 profiler 采样的时间线走主流程。

```text
_PG_init() 定义保守默认值
MarkGUCPrefixReserved()
启动期 GUC 决定容量
ExecutorStart hook 做 cheap guard
采样命中后开启细粒度状态
ExecutorEnd hook 判断 min_duration
写 ring 前检查 max events
暴露 guard 计数
```

### 5.1. _PG_init() 定义保守默认值

默认 disabled，sample_rate 合法范围 0..1，min_duration 默认非零或关闭。

这一段改变的状态边界是：配置创建边界。

回到诊断时要验证：默认安装扩展是否不会改变所有 SQL 成本。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

### 5.2. MarkGUCPrefixReserved()

定义完扩展 GUC 后保留前缀，帮助发现拼写错误。

这一段改变的状态边界是：配置命名边界。

回到诊断时要验证：是否和 auto_explain/pg_stat_statements 一样清晰。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

### 5.3. 启动期 GUC 决定容量

ring size、最大 shared memory 等不能运行期随便改变。

这一段改变的状态边界是：资源上界边界。

回到诊断时要验证：是否需要 postmaster restart。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

### 5.4. ExecutorStart hook 做 cheap guard

先检查 enabled、用户、database、queryId、nested、sample_rate。

这一段改变的状态边界是：热路径成本边界。

回到诊断时要验证：未命中是否避免 PlanState 遍历和计时。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

### 5.5. 采样命中后开启细粒度状态

只对命中 query 安装/启用节点级计时、初始化 query state。

这一段改变的状态边界是：观测启动边界。

回到诊断时要验证：采样决定是否每条 query 只做一次。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

### 5.6. ExecutorEnd hook 判断 min_duration

查询结束后再根据总耗时决定是否保留事件。

这一段改变的状态边界是：输出过滤边界。

回到诊断时要验证：短查询事件是否可直接丢弃或只累计 drop。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

### 5.7. 写 ring 前检查 max events

节点事件超过上限时停止写入并增加 dropped 计数。

这一段改变的状态边界是：输出体积边界。

回到诊断时要验证：复杂计划不会无限写共享区。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

### 5.8. 暴露 guard 计数

info view 展示 skipped、sampled、dropped_by_limit、dropped_by_ring。

这一段改变的状态边界是：可解释性边界。

回到诊断时要验证：用户能否判断缺失事件原因。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

## 6. 生命周期 / ownership / cleanup

### 6.1. GUC 注册

创建或进入者：_PG_init。

正常清理者：backend 退出。

异常路径依赖：重复加载或占位 GUC 需处理。

这里要区分三件事。

MemoryContext 管内存批量释放。

ResourceOwner 管 pin、refcount、文件句柄、锁等外部资源。

全局或共享状态必须有独立的可见性、锁或代际边界。

把这些混成一句“事务结束释放”，会丢掉定位问题所需的时间顺序。

### 6.2. 启动期配置

创建或进入者：postmaster 启动读取。

正常清理者：需要重启改变 shared memory 大小。

异常路径依赖：SIGHUP 不能改变已分配 ring 容量。

这里要区分三件事。

MemoryContext 管内存批量释放。

ResourceOwner 管 pin、refcount、文件句柄、锁等外部资源。

全局或共享状态必须有独立的可见性、锁或代际边界。

把这些混成一句“事务结束释放”，会丢掉定位问题所需的时间顺序。

### 6.3. 运行期开关

创建或进入者：SET/ALTER SYSTEM/SIGHUP。

正常清理者：会话结束或 reset。

异常路径依赖：改变应只影响后续 query。

这里要区分三件事。

MemoryContext 管内存批量释放。

ResourceOwner 管 pin、refcount、文件句柄、锁等外部资源。

全局或共享状态必须有独立的可见性、锁或代际边界。

把这些混成一句“事务结束释放”，会丢掉定位问题所需的时间顺序。

### 6.4. 采样决定

创建或进入者：ExecutorStart。

正常清理者：ExecutorEnd。

异常路径依赖：ERROR 路径按未完成 query 处理。

这里要区分三件事。

MemoryContext 管内存批量释放。

ResourceOwner 管 pin、refcount、文件句柄、锁等外部资源。

全局或共享状态必须有独立的可见性、锁或代际边界。

把这些混成一句“事务结束释放”，会丢掉定位问题所需的时间顺序。

### 6.5. 阈值过滤

创建或进入者：ExecutorEnd。

正常清理者：事件丢弃或保留。

异常路径依赖：如果 ERROR 发生，也要决定是否记录 error event。

这里要区分三件事。

MemoryContext 管内存批量释放。

ResourceOwner 管 pin、refcount、文件句柄、锁等外部资源。

全局或共享状态必须有独立的可见性、锁或代际边界。

把这些混成一句“事务结束释放”，会丢掉定位问题所需的时间顺序。

### 6.6. 计数器

创建或进入者：guard 或 writer 更新。

正常清理者：reset 函数清零。

异常路径依赖：需要锁或 atomic 保证共享一致性。

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
| 默认关闭 | enabled=false | 安装扩展不应默认扰动 workload。 | 观测扩展应该默认全量开启。 |
| 合法范围 | GUC min/max/check hook | 采样率和容量必须有硬边界。 | 配置错误可以等运行时再处理。 |
| 启动/运行区分 | GUC context | 容量类参数必须重启或固定。 | 所有参数都能在线调整。 |
| cheap guard | hook 前置分支 | 未命中路径不做昂贵工作。 | 先遍历计划再决定采样。 |
| 可解释丢弃 | dropped counters | 缺失数据有原因可查。 | 没有记录就是没有问题。 |
| 权限边界 | SUSET / USERSET / view 权限 | 普通用户不应观察其它用户敏感 SQL。 | 诊断视图可以公开所有 source text。 |

读源码时可以按这个顺序检查：

```text
先找写入点
  -> 再找读取点
  -> 再找清理点
  -> 再看 ERROR / abort / backend exit 是否能兜底
  -> 最后把用户可见输出映射回这一条状态链
```

1. `默认关闭` 这一层主要依赖 enabled=false。

它保证的是：安装扩展不应默认扰动 workload。

不要把它误读成：观测扩展应该默认全量开启。

如果一次诊断只能看到这一层的投影，就必须保留不确定性。

2. `合法范围` 这一层主要依赖 GUC min/max/check hook。

它保证的是：采样率和容量必须有硬边界。

不要把它误读成：配置错误可以等运行时再处理。

如果一次诊断只能看到这一层的投影，就必须保留不确定性。

3. `启动/运行区分` 这一层主要依赖 GUC context。

它保证的是：容量类参数必须重启或固定。

不要把它误读成：所有参数都能在线调整。

如果一次诊断只能看到这一层的投影，就必须保留不确定性。

4. `cheap guard` 这一层主要依赖 hook 前置分支。

它保证的是：未命中路径不做昂贵工作。

不要把它误读成：先遍历计划再决定采样。

如果一次诊断只能看到这一层的投影，就必须保留不确定性。

5. `可解释丢弃` 这一层主要依赖 dropped counters。

它保证的是：缺失数据有原因可查。

不要把它误读成：没有记录就是没有问题。

如果一次诊断只能看到这一层的投影，就必须保留不确定性。

6. `权限边界` 这一层主要依赖 SUSET / USERSET / view 权限。

它保证的是：普通用户不应观察其它用户敏感 SQL。

不要把它误读成：诊断视图可以公开所有 source text。

如果一次诊断只能看到这一层的投影，就必须保留不确定性。

## 8. 错误路径 / 异常路径 / fallback

### 8.1. sample_rate 拼错或越界

可见现象：GUC 设置失败或 silently ignored。

源码上应先回到：`guc.c` check/min/max。

正确处理方式是：用 DefineCustomRealVariable 设置范围并保留前缀。

这类路径通常不是“继续收集更多指标”就能解决。

要先判断观测值是在失败前、失败中还是失败后写入。

如果写入点本身在异常路径里，就要检查 cleanup 是否仍然执行。

### 8.2. min_duration 过低

可见现象：ring 迅速满、dropped 上升。

源码上应先回到：hook guard / ring info。

正确处理方式是：提高阈值或降低采样率。

这类路径通常不是“继续收集更多指标”就能解决。

要先判断观测值是在失败前、失败中还是失败后写入。

如果写入点本身在异常路径里，就要检查 cleanup 是否仍然执行。

### 8.3. 用户过滤失效

可见现象：采到不应采的会话。

源码上应先回到：GetUserId / MyDatabaseId 边界。

正确处理方式是：在 Start hook 早期检查稳定身份。

这类路径通常不是“继续收集更多指标”就能解决。

要先判断观测值是在失败前、失败中还是失败后写入。

如果写入点本身在异常路径里，就要检查 cleanup 是否仍然执行。

### 8.4. 运行期改变 capacity

可见现象：视图和 writer 对布局理解不一致。

源码上应先回到：shared memory sizing。

正确处理方式是：capacity 只允许启动期确定。

这类路径通常不是“继续收集更多指标”就能解决。

要先判断观测值是在失败前、失败中还是失败后写入。

如果写入点本身在异常路径里，就要检查 cleanup 是否仍然执行。

### 8.5. guard 计数缺失

可见现象：用户无法解释为何没有事件。

源码上应先回到：info view。

正确处理方式是：暴露 skipped/sampled/dropped 分类。

这类路径通常不是“继续收集更多指标”就能解决。

要先判断观测值是在失败前、失败中还是失败后写入。

如果写入点本身在异常路径里，就要检查 cleanup 是否仍然执行。

## 9. 成本、资源与跨模块传播

成本分析要围绕同一个主问题。

不要把所有可能成本都列一遍。

### 9.1. GUC 读取

主要扩张因子：每条 query hook 次数。

放大方式：过多复杂解析会放大。

控制办法：assign hook 预解析，hook 只读简单值。

诊断时不要只看绝对值。

要看它是否随 rows、loops、backend 数、buffer miss、计划节点深度或采样率线性放大。

如果它只在某个节点或某类等待里集中出现，说明问题边界比 SQL 文本更窄。

### 9.2. 随机采样

主要扩张因子：query 数。

放大方式：每条 query 一次随机成本。

控制办法：只在 enabled 后执行。

诊断时不要只看绝对值。

要看它是否随 rows、loops、backend 数、buffer miss、计划节点深度或采样率线性放大。

如果它只在某个节点或某类等待里集中出现，说明问题边界比 SQL 文本更窄。

### 9.3. 过滤

主要扩张因子：用户/db 列表大小。

放大方式：字符串匹配昂贵。

控制办法：预解析 Oid 或简单模式。

诊断时不要只看绝对值。

要看它是否随 rows、loops、backend 数、buffer miss、计划节点深度或采样率线性放大。

如果它只在某个节点或某类等待里集中出现，说明问题边界比 SQL 文本更窄。

### 9.4. 阈值保留

主要扩张因子：已记录事件数。

放大方式：End 阶段可能需要丢弃 query-local buffer。

控制办法：先本地暂存，达标再写共享。

诊断时不要只看绝对值。

要看它是否随 rows、loops、backend 数、buffer miss、计划节点深度或采样率线性放大。

如果它只在某个节点或某类等待里集中出现，说明问题边界比 SQL 文本更窄。

### 9.5. 最大事件数

主要扩张因子：plan 节点和 loops。

放大方式：限制不足会输出爆炸。

控制办法：per-query counter 和 top N。

诊断时不要只看绝对值。

要看它是否随 rows、loops、backend 数、buffer miss、计划节点深度或采样率线性放大。

如果它只在某个节点或某类等待里集中出现，说明问题边界比 SQL 文本更窄。

### 9.6. 权限检查

主要扩张因子：view 查询频率。

放大方式：复杂权限判断影响读取。

控制办法：在 SRF 入口一次性检查。

诊断时不要只看绝对值。

要看它是否随 rows、loops、backend 数、buffer miss、计划节点深度或采样率线性放大。

如果它只在某个节点或某类等待里集中出现，说明问题边界比 SQL 文本更窄。

## 10. 观测与诊断入口

观测入口要分清当前值、累计值、单次执行剖面和外部采样。

同一个慢 SQL 结论最好至少由两类信号互相校验。

### 10.1. SHOW profiler.*

能看到：当前 GUC 值。

看不到：采样命中历史。

源码入口：`guc.c`。

使用时先问它是当前状态、累计状态，还是本次执行状态。

再问它是否会因为采样、刷新延迟、并行 worker 汇总或输出格式化而偏移。

最后把结论回扣到本节主流程中的某个写入点。

### 10.2. profiler_info()

能看到：sampled/skipped/dropped 计数。

看不到：具体节点热点。

源码入口：扩展 SRF。

使用时先问它是当前状态、累计状态，还是本次执行状态。

再问它是否会因为采样、刷新延迟、并行 worker 汇总或输出格式化而偏移。

最后把结论回扣到本节主流程中的某个写入点。

### 10.3. pg_settings

能看到：GUC context、source、pending_restart。

看不到：运行时事件。

源码入口：`guc_funcs.c`。

使用时先问它是当前状态、累计状态，还是本次执行状态。

再问它是否会因为采样、刷新延迟、并行 worker 汇总或输出格式化而偏移。

最后把结论回扣到本节主流程中的某个写入点。

### 10.4. pg_stat_activity

能看到：profiler 是否造成 wait。

看不到：guard 逻辑。

源码入口：`wait_event.c`。

使用时先问它是当前状态、累计状态，还是本次执行状态。

再问它是否会因为采样、刷新延迟、并行 worker 汇总或输出格式化而偏移。

最后把结论回扣到本节主流程中的某个写入点。

### 10.5. 日志

能看到：阈值命中的慢查询事件。

看不到：被采样丢弃的查询。

源码入口：扩展输出。

使用时先问它是当前状态、累计状态，还是本次执行状态。

再问它是否会因为采样、刷新延迟、并行 worker 汇总或输出格式化而偏移。

最后把结论回扣到本节主流程中的某个写入点。

### 10.6. EXPLAIN

能看到：被诊断 query 的计划形态。

看不到：采样决策。

源码入口：`explain.c`。

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

### 12.1. 默认关闭验证

实验目标：把可见现象压回源码状态边界。

建议准备：

```text
LOAD 'profiler_extension';
运行短查询。
查询 profiler_info()。
```

预期现象：未开启时 sampled 不应增长，执行成本接近无扩展。

回到源码时检查：hook cheap guard。

实验完成后不要只记录数值。

还要记录计划节点、GUC、并发状态、缓存冷热和是否开启 timing。

这些条件决定同一个观测值能否迁移到另一个现场。

### 12.2. 采样率验证

实验目标：把可见现象压回源码状态边界。

建议准备：

```text
SET profiler.enabled = on;
SET profiler.sample_rate = 0.1;
运行 1000 条相似查询。
```

预期现象：sampled 数应近似采样比例，但有随机波动。

回到源码时检查：Start hook sampling decision。

实验完成后不要只记录数值。

还要记录计划节点、GUC、并发状态、缓存冷热和是否开启 timing。

这些条件决定同一个观测值能否迁移到另一个现场。

### 12.3. min_duration 验证

实验目标：把可见现象压回源码状态边界。

建议准备：

```text
SET profiler.min_duration = '100ms';
运行一条 1ms 查询和一条 500ms 查询。
```

预期现象：短查询被 skipped_by_duration，慢查询保留。

回到源码时检查：End hook duration check。

实验完成后不要只记录数值。

还要记录计划节点、GUC、并发状态、缓存冷热和是否开启 timing。

这些条件决定同一个观测值能否迁移到另一个现场。

### 12.4. max events 验证

实验目标：把可见现象压回源码状态边界。

建议准备：

```text
设置很低 max_events_per_query。
运行复杂 join 计划。
```

预期现象：事件被截断且 dropped_by_limit 增加。

回到源码时检查：node wrapper event counter。

实验完成后不要只记录数值。

还要记录计划节点、GUC、并发状态、缓存冷热和是否开启 timing。

这些条件决定同一个观测值能否迁移到另一个现场。

## 13. 讨论题

1. 为什么 sample decision 应该在 ExecutorStart 做一次，而不是每个节点都重新采样？

2. 哪些 GUC 必须是启动期参数，哪些可以运行期调整？

3. 默认关闭和默认低采样率各有什么取舍？

4. 为什么 guard 计数本身也是 profiler 可观测性的一部分？

5. 普通用户能否看到其它用户的 profiler 结果，应由什么边界决定？

## 14. 本节小结

本节只沉淀一个模型：

```text
profiler 的线上安全来自默认关闭、采样率、阈值、过滤和上限。
容量类 GUC 决定 shared memory，必须启动期固定。
运行期 GUC 只改变是否采样和如何保留。
cheap guard 必须早于计划遍历、计时和共享写入。
所有丢弃都应可计数、可解释。
```

下一节把 profiler 结果暴露为 view，并和 EXPLAIN / source loop 关联。
