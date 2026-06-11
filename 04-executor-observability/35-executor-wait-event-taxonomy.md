# PostgreSQL Executor 常见等待类型定位

## 课程定位

前置知识：已经知道 wait_event_info 如何设置和清理；本节把用户可见的 wait_event_type / wait_event 映射回执行 SQL 时的源码路径。

本节唯一主问题：

```text
执行 SQL 时常见的 LWLock、Lock、BufferPin、IO、Client、IPC 等 wait event 分别对应哪些执行器、存储和通信路径？
```

核心矛盾：pg_stat_activity 只给出一个短名称，但慢 SQL 的等待可能来自执行器节点、buffer manager、lock manager、I/O 层、并行通信或客户端背压。

学完后应能从 wait_event_type/name 推断下一步应该看哪个源码模块、哪个系统视图和哪类实验。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

上一节讲的是单个 wait event 如何设置与清理。

本节不再停留在 start/end 协议，而是建立一张诊断地图。

执行器本身很多节点并不直接等待 OS I/O 或锁。

它们通常调用 table AM、index AM、buffer、lock、latch、parallel infrastructure 或 libpq 层。

因此 pg_stat_activity 上的 wait event 必须沿调用链向下追。

本节唯一目标是把等待类型和源码层次对应起来。

下一节会继续讨论 wait、CPU 和阻塞链如何组合判断。

04 目录到这里的观测链路可以压缩成：

```text
Executor runtime state
  -> Instrumentation / EXPLAIN ANALYZE
  -> pg_stat cumulative counters
  -> wait event current state
  -> executor hook / auto_explain extension boundary
```

本节只处理自己这一环，不把相邻主题扩成第二个主问题。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
一个 SQL backend 在 active 状态下如果 wait_event 非空，先按 wait_event_type 定位等待类别，再按具体 name 回到 wait_event_names.txt 与设置点；最后结合计划节点、pg_locks、I/O 统计、并行 worker 和客户端状态判断等待属于哪条运行路径。
```

这个模型的诊断含义是：

- 不要先背函数名，先判断状态属于哪个生命周期边界。
- 不要把用户可见指标当成源码真相，指标只是某个状态边界的投影。
- 不要把一次采样、一次累计值和一次执行剖面混在一起解释。
- 每个观测值都要回答：谁写入，何时可见，谁清理，异常时是否仍可靠。

本节后面的源码 walkthrough 都围绕这一个模型展开。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/utils/activity/wait_event_names.txt` | 内置 wait event 名称、类型和描述的源数据。 |
| 2 | `src/backend/utils/activity/wait_event.c` | raw wait_event_info 到类型和名称的解析。 |
| 3 | `src/backend/utils/activity/wait_event_funcs.c` | pg_get_wait_events() 输出 wait event 元数据。 |
| 4 | `src/backend/utils/adt/pgstatfuncs.c` | pg_stat_activity 中 wait_event_type / wait_event 的读取逻辑。 |
| 5 | `src/backend/storage/lmgr/lwlock.c` | LWLockAcquire() 等轻量锁等待路径。 |
| 6 | `src/backend/storage/lmgr/proc.c` | ProcSleep()、ProcWaitForSignal() 等 heavyweight lock 和信号等待。 |
| 7 | `src/backend/storage/smgr/md.c` | DataFileRead / DataFileWrite / DataFileExtend 等文件 I/O 等待点。 |
| 8 | `src/backend/executor/nodeHash.c` | Parallel Hash build / batch 等执行器节点相关 IPC 等待。 |
| 9 | `src/backend/executor/nodeBitmapHeapscan.c` | Parallel Bitmap Heap Scan 的 ConditionVariable 等待。 |
| 10 | `src/backend/libpq/be-secure.c` | ClientRead / ClientWrite 等客户端通信等待。 |

阅读顺序按 mental model 排列，不按文件名排序。

建议先从入口和状态结构读起，再追 ownership、cleanup、异常路径和观测输出。

本节源码核对基线：

```text
cd /home/highgo/postgres
git rev-parse --abbrev-ref HEAD
git rev-parse HEAD
```

期望分支为 `master`，提交为本课开头列出的完整提交号。

## 4. 关键状态与边界

状态不是字段清单。

一个字段只有和创建时机、访问者、生命周期、异常路径一起看，才有诊断语义。

### 4.1. `wait_event_type`

第一层分类，决定先看 Lock、LWLock、IO、Client、IPC、BufferPin 还是 Extension。

读这个状态时先问四个问题：

- 它是 backend-local、shared memory、文件状态，还是只在调用栈中短暂存在？
- 谁负责初始化它，初始化失败时有没有后续 cleanup 责任？
- 谁能读取或修改它，读取是否需要锁、pin、snapshot 或 owner？
- 它何时失效，失效后用户可见指标会如何表现？

在本节主问题中，`wait_event_type` 的价值是帮助判断一个观测现象到底落在哪个状态边界。

### 4.2. `wait_event`

第二层名称，通常能回到一个具体等待点或资源类型。

读这个状态时先问四个问题：

- 它是 backend-local、shared memory、文件状态，还是只在调用栈中短暂存在？
- 谁负责初始化它，初始化失败时有没有后续 cleanup 责任？
- 谁能读取或修改它，读取是否需要锁、pin、snapshot 或 owner？
- 它何时失效，失效后用户可见指标会如何表现？

在本节主问题中，`wait_event` 的价值是帮助判断一个观测现象到底落在哪个状态边界。

### 4.3. `PlanState 节点`

告诉你 SQL 正在执行哪类算子，但不一定告诉等待来自哪个底层模块。

读这个状态时先问四个问题：

- 它是 backend-local、shared memory、文件状态，还是只在调用栈中短暂存在？
- 谁负责初始化它，初始化失败时有没有后续 cleanup 责任？
- 谁能读取或修改它，读取是否需要锁、pin、snapshot 或 owner？
- 它何时失效，失效后用户可见指标会如何表现？

在本节主问题中，`PlanState 节点` 的价值是帮助判断一个观测现象到底落在哪个状态边界。

### 4.4. `PGPROC wait fields`

其他 backend 读取当前等待状态的共享入口。

读这个状态时先问四个问题：

- 它是 backend-local、shared memory、文件状态，还是只在调用栈中短暂存在？
- 谁负责初始化它，初始化失败时有没有后续 cleanup 责任？
- 谁能读取或修改它，读取是否需要锁、pin、snapshot 或 owner？
- 它何时失效，失效后用户可见指标会如何表现？

在本节主问题中，`PGPROC wait fields` 的价值是帮助判断一个观测现象到底落在哪个状态边界。

### 4.5. `pg_locks row`

Lock 类型等待需要结合 locktag、mode、granted 和 waitstart。

读这个状态时先问四个问题：

- 它是 backend-local、shared memory、文件状态，还是只在调用栈中短暂存在？
- 谁负责初始化它，初始化失败时有没有后续 cleanup 责任？
- 谁能读取或修改它，读取是否需要锁、pin、snapshot 或 owner？
- 它何时失效，失效后用户可见指标会如何表现？

在本节主问题中，`pg_locks row` 的价值是帮助判断一个观测现象到底落在哪个状态边界。

### 4.6. `I/O counters`

IO wait 需要与 pg_stat_io、Buffers 和系统 I/O 指标一起解释。

读这个状态时先问四个问题：

- 它是 backend-local、shared memory、文件状态，还是只在调用栈中短暂存在？
- 谁负责初始化它，初始化失败时有没有后续 cleanup 责任？
- 谁能读取或修改它，读取是否需要锁、pin、snapshot 或 owner？
- 它何时失效，失效后用户可见指标会如何表现？

在本节主问题中，`I/O counters` 的价值是帮助判断一个观测现象到底落在哪个状态边界。

本节最重要的不变量是：

```text
raw field 不是语义；field + lifecycle + writer + reader + cleanup 才是语义。
```

## 5. 主流程源码 walkthrough

下面按时间顺序走主链路。

每一步都回答“状态发生了什么变化”，而不是把源码读成函数名清单。

### 5.1. 步骤 1

先从 pg_stat_activity 找到慢 SQL backend 的 state、wait_event_type 和 wait_event。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.2. 步骤 2

如果 wait_event_type 为空，说明当前采样瞬间没有进入 PostgreSQL wait path。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.3. 步骤 3

如果类型是 Lock，转向 pg_locks、pg_blocking_pids() 和 lock manager 源码。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.4. 步骤 4

如果类型是 LWLock，关注共享内存结构保护锁，例如 buffer、WAL、stats 或 predicate lock。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.5. 步骤 5

如果类型是 BufferPin，关注持 pin 的长查询、standby conflict 或 cleanup lock。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.6. 步骤 6

如果类型是 IO，沿 wait_event 名称回到 md.c、fd.c、WAL 或 SLRU I/O 路径。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.7. 步骤 7

如果类型是 Client，优先判断应用读取慢、网络拥塞或客户端未发送下一条消息。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.8. 步骤 8

如果类型是 IPC，检查 parallel query、message queue、barrier、condition variable 或 latch。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.9. 步骤 9

如果类型是 Timeout，确认是显式 sleep、commit delay、recovery conflict 还是 retry 间隔。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.10. 步骤 10

如果类型是 Extension，必须查扩展注册的 wait event 名称和对应代码。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.11. 步骤 11

把 wait event 映射回计划节点时，要沿节点调用的 table AM / index AM / storage 函数继续追。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.12. 步骤 12

最终判断必须回到一个可复现实验，而不是只停留在类型名称。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

主链路可以压缩成：

```text
01. 先从 pg_stat_activity 找到慢 SQL backend 的 state、wait_event_type 和 wait_event。
02. 如果 wait_event_type 为空，说明当前采样瞬间没有进入 PostgreSQL wait path。
03. 如果类型是 Lock，转向 pg_locks、pg_blocking_pids() 和 lock manager 源码。
04. 如果类型是 LWLock，关注共享内存结构保护锁，例如 buffer、WAL、stats 或 predicate lock。
05. 如果类型是 BufferPin，关注持 pin 的长查询、standby conflict 或 cleanup lock。
06. 如果类型是 IO，沿 wait_event 名称回到 md.c、fd.c、WAL 或 SLRU I/O 路径。
07. 如果类型是 Client，优先判断应用读取慢、网络拥塞或客户端未发送下一条消息。
08. 如果类型是 IPC，检查 parallel query、message queue、barrier、condition variable 或 latch。
09. 如果类型是 Timeout，确认是显式 sleep、commit delay、recovery conflict 还是 retry 间隔。
10. 如果类型是 Extension，必须查扩展注册的 wait event 名称和对应代码。
11. 把 wait event 映射回计划节点时，要沿节点调用的 table AM / index AM / storage 函数继续追。
12. 最终判断必须回到一个可复现实验，而不是只停留在类型名称。
```

## 6. 生命周期 / ownership / cleanup

生命周期是本节能否用于真实诊断的关键。

### 6.1. 发现

采样 pg_stat_activity 时发现 active backend 有 wait_event。

这里要特别区分两件事：

- 内存是否释放。
- 语义状态是否已经对其他 backend 或用户视图不可见。

很多观测问题并不是对象还在内存里，而是读取端还持有旧语义。

### 6.2. 归类

先用 wait_event_type 分出大类，避免把 Client 等待误看成数据库内部阻塞。

这里要特别区分两件事：

- 内存是否释放。
- 语义状态是否已经对其他 backend 或用户视图不可见。

很多观测问题并不是对象还在内存里，而是读取端还持有旧语义。

### 6.3. 定位

用 wait_event 名称和源码设置点确定等待资源。

这里要特别区分两件事：

- 内存是否释放。
- 语义状态是否已经对其他 backend 或用户视图不可见。

很多观测问题并不是对象还在内存里，而是读取端还持有旧语义。

### 6.4. 交叉验证

结合 EXPLAIN、pg_locks、pg_stat_io、系统 profiler 或客户端日志确认原因。

这里要特别区分两件事：

- 内存是否释放。
- 语义状态是否已经对其他 backend 或用户视图不可见。

很多观测问题并不是对象还在内存里，而是读取端还持有旧语义。

### 6.5. 收敛

把诊断从“慢”收敛为锁、I/O、CPU、IPC、BufferPin 或客户端背压中的一种主因。

这里要特别区分两件事：

- 内存是否释放。
- 语义状态是否已经对其他 backend 或用户视图不可见。

很多观测问题并不是对象还在内存里，而是读取端还持有旧语义。

ERROR / abort 路径上，不能依赖普通 C 调用栈逐层返回完成清理。

因此源码里要找的是 ResourceOwner、MemoryContext、PG_TRY/PG_FINALLY、事务回调或标准 executor teardown。

## 7. 正确性机制层次

本节的正确性来自多个层次叠加，而不是单个函数。

### 7.1. 机制 1

wait event 是采样时刻状态，不保证覆盖整个 SQL 耗时。

把它放回运行期，可以得到两个判断：

- 正常路径中，它限制了状态何时能被推进或读取。
- 异常路径中，它决定用户看到的是空值、旧值、错误还是等待。

### 7.2. 机制 2

同一个 SQL 可以在不同阶段显示不同 wait event。

把它放回运行期，可以得到两个判断：

- 正常路径中，它限制了状态何时能被推进或读取。
- 异常路径中，它决定用户看到的是空值、旧值、错误还是等待。

### 7.3. 机制 3

执行器节点名不是等待类型；底层 access method 和 storage 层才常设置等待。

把它放回运行期，可以得到两个判断：

- 正常路径中，它限制了状态何时能被推进或读取。
- 异常路径中，它决定用户看到的是空值、旧值、错误还是等待。

### 7.4. 机制 4

Lock 类型等待不等于死锁；死锁检测是另一个 lock manager 路径。

把它放回运行期，可以得到两个判断：

- 正常路径中，它限制了状态何时能被推进或读取。
- 异常路径中，它决定用户看到的是空值、旧值、错误还是等待。

### 7.5. 机制 5

IO wait 不等于计划错误；也可能是缓存状态、checkpoint 或存储层问题。

把它放回运行期，可以得到两个判断：

- 正常路径中，它限制了状态何时能被推进或读取。
- 异常路径中，它决定用户看到的是空值、旧值、错误还是等待。

### 7.6. 机制 6

ClientWrite 常说明服务器在等客户端收结果，而不是数据库内部慢。

把它放回运行期，可以得到两个判断：

- 正常路径中，它限制了状态何时能被推进或读取。
- 异常路径中，它决定用户看到的是空值、旧值、错误还是等待。

不要把这些机制抽象成“为了安全”。

更准确的说法是：它们把低成本 hot path 和可解释的观测边界同时保留下来。

## 8. 错误路径 / 异常路径 / fallback

下面这些场景不是边角料，而是现场排障最常遇到的误判来源。

### 8.1. 看到 LWLock 就调 work_mem

LWLock 通常是共享结构争用，和单节点内存限制不一定相关。

排查时不要先改参数。

先问：当前现象是写入端没有推进，读取端没有刷新，还是观察工具选错了层次。

### 8.2. 看到 IO 就重建索引

IO wait 只说明当前在 I/O 路径，必须结合 Buffers 和访问模式。

排查时不要先改参数。

先问：当前现象是写入端没有推进，读取端没有刷新，还是观察工具选错了层次。

### 8.3. 看到 ClientRead 责怪数据库

ClientRead 常是后端等待客户端下一条消息。

排查时不要先改参数。

先问：当前现象是写入端没有推进，读取端没有刷新，还是观察工具选错了层次。

### 8.4. 看到 Lock 不查 blocker

Lock 类型必须继续看 pg_blocking_pids() 或 pg_locks。

排查时不要先改参数。

先问：当前现象是写入端没有推进，读取端没有刷新，还是观察工具选错了层次。

### 8.5. 把一次采样当全程画像

短查询或抖动场景需要多次采样或日志采样。

排查时不要先改参数。

先问：当前现象是写入端没有推进，读取端没有刷新，还是观察工具选错了层次。

fallback 的设计目标通常不是让结果更漂亮，而是让系统在异常路径上保持可恢复、可清理、可解释。

## 9. 成本、资源与跨模块传播

可观测性代码如果放错层次，会把诊断工具变成性能问题的一部分。

### 9.1. 成本维度 1

wait event 分类本身成本低，但过度采样 pg_stat_activity 会增加诊断查询负担。

这个成本通常会继续传播到相邻模块：

- executor hot path 是否多一次分支、计时或锁。
- shared memory 是否出现新的争用点。
- 日志、视图或监控采样是否放大已有压力。

### 9.2. 成本维度 2

Lock / LWLock 等待成本随并发 backend 和共享热点扩张。

这个成本通常会继续传播到相邻模块：

- executor hot path 是否多一次分支、计时或锁。
- shared memory 是否出现新的争用点。
- 日志、视图或监控采样是否放大已有压力。

### 9.3. 成本维度 3

IO wait 成本受数据局部性、缓存命中、文件系统和 checkpoint 影响。

这个成本通常会继续传播到相邻模块：

- executor hot path 是否多一次分支、计时或锁。
- shared memory 是否出现新的争用点。
- 日志、视图或监控采样是否放大已有压力。

### 9.4. 成本维度 4

IPC wait 成本常随 parallel worker 数、tuple queue 和 leader participation 改变。

这个成本通常会继续传播到相邻模块：

- executor hot path 是否多一次分支、计时或锁。
- shared memory 是否出现新的争用点。
- 日志、视图或监控采样是否放大已有压力。

### 9.5. 成本维度 5

Client wait 成本可能来自网络或应用消费速度，不一定属于数据库内部资源。

这个成本通常会继续传播到相邻模块：

- executor hot path 是否多一次分支、计时或锁。
- shared memory 是否出现新的争用点。
- 日志、视图或监控采样是否放大已有压力。

因此课程里的实验默认是短时、可控、可回滚的，不建议把强制刷新或高频采样作为长期配置。

## 10. 观测与诊断入口

观测入口要按问题层次选择。

### 10.1. `pg_stat_activity`

第一入口：state、wait_event_type、wait_event、query_id。

使用这个入口时要同时记录：

- 采样时间。
- backend pid 或 query_id。
- 是否处于同一事务、同一 backend 或同一执行。

### 10.2. `pg_get_wait_events()`

确认 wait event 名称属于哪个类型以及官方描述。

使用这个入口时要同时记录：

- 采样时间。
- backend pid 或 query_id。
- 是否处于同一事务、同一 backend 或同一执行。

### 10.3. `pg_locks`

Lock 类型等待的 locktag、mode 和 waitstart。

使用这个入口时要同时记录：

- 采样时间。
- backend pid 或 query_id。
- 是否处于同一事务、同一 backend 或同一执行。

### 10.4. `pg_blocking_pids()`

快速得到 heavyweight lock blocker。

使用这个入口时要同时记录：

- 采样时间。
- backend pid 或 query_id。
- 是否处于同一事务、同一 backend 或同一执行。

### 10.5. `EXPLAIN (ANALYZE, BUFFERS)`

把等待阶段和计划节点的访问模式联系起来。

使用这个入口时要同时记录：

- 采样时间。
- backend pid 或 query_id。
- 是否处于同一事务、同一 backend 或同一执行。

### 10.6. `pg_stat_io`

IO 类型等待的后台累计证据。

使用这个入口时要同时记录：

- 采样时间。
- backend pid 或 query_id。
- 是否处于同一事务、同一 backend 或同一执行。

如果一个入口不能回答本节唯一主问题，就不要把它加入诊断路径。

## 11. 常见误区

### 11.1. 误区 1

把用户可见字段当成源码内部状态本身。

更稳的做法是把现象拆成：写入端、传播边界、读取端、清理端。

### 11.2. 误区 2

把一次采样当成完整执行历史。

更稳的做法是把现象拆成：写入端、传播边界、读取端、清理端。

### 11.3. 误区 3

把累计统计当成单条 SQL 的执行剖面。

更稳的做法是把现象拆成：写入端、传播边界、读取端、清理端。

### 11.4. 误区 4

只看视图名称，不沿源码回到写入点和清理点。

更稳的做法是把现象拆成：写入端、传播边界、读取端、清理端。

### 11.5. 误区 5

为了观测打开所有开关，却不评估观测扰动。

更稳的做法是把现象拆成：写入端、传播边界、读取端、清理端。

### 11.6. 误区 6

看到异常现象后直接调参，而不是先确认生命周期边界。

更稳的做法是把现象拆成：写入端、传播边界、读取端、清理端。

这些误区的共同点，是把 observability 当成事实本身，而不是事实的投影。

## 12. 课堂实验

实验目标不是跑出固定数字，而是看到一个状态如何从源码边界投影到用户视图。

### 12.1. Lock wait

实验步骤：

```sql
-- session 1
BEGIN;
UPDATE t_wait_demo SET id = id WHERE id = 1;
-- session 2
UPDATE t_wait_demo SET id = id WHERE id = 1;
```

观察 session 2 的 Lock wait，并用 pg_blocking_pids() 找到 session 1。

实验后回到源码，至少确认一个入口函数和一个状态字段。

### 12.2. Client wait

实验步骤：

```sql
COPY (SELECT generate_series(1,10000000)) TO STDOUT;
```

让客户端读取变慢，观察是否出现 ClientWrite。

实验后回到源码，至少确认一个入口函数和一个状态字段。

### 12.3. IO wait

实验步骤：

```sql
DISCARD PLANS;
EXPLAIN (ANALYZE, BUFFERS) SELECT count(*) FROM large_table;
```

结合 Buffers 和 wait event 判断是否真的卡在数据文件读取。

实验后回到源码，至少确认一个入口函数和一个状态字段。

### 12.4. IPC wait

实验步骤：

```sql
SET max_parallel_workers_per_gather = 4;
EXPLAIN (ANALYZE) SELECT count(*) FROM large_table;
```

并行查询中观察 leader 与 worker 的 IPC 等待，注意采样窗口。

实验后回到源码，至少确认一个入口函数和一个状态字段。

实验时请在测试库执行，并记录 PostgreSQL 版本、配置和是否存在并发负载。

## 13. 诊断剧本

现场排查时可以按下面顺序收敛。

1. 先确认现象属于单次执行、长期累计、当前等待，还是扩展插桩。

2. 再确认目标 backend、query_id、事务边界和采样时间。

3. 第三步回到本节第 3 节列出的源码入口，找写入点和清理点。

4. 第四步用最小 SQL 或 gdb 断点复现状态变化。

5. 第五步评估观测工具自身是否改变了执行成本。

6. 最后把结论写成“现象 -> 状态 -> 源码 -> 复现”的闭环。

这套剧本的价值在于避免在多个指标之间横跳。

## 14. 讨论题

1. 为什么 wait_event_type 是第一判断，而不是直接看 wait_event 名称？

回答时必须至少引用一个源码文件和一个运行期现象。

2. 执行器节点和 wait event 之间为什么不是一一对应？

回答时必须至少引用一个源码文件和一个运行期现象。

3. ClientWrite 为什么常常不是数据库内部瓶颈？

回答时必须至少引用一个源码文件和一个运行期现象。

4. LWLock 等待怎样进一步定位到共享结构？

回答时必须至少引用一个源码文件和一个运行期现象。

5. BufferPin 与 heavyweight Lock 的诊断路线有什么不同？

回答时必须至少引用一个源码文件和一个运行期现象。

6. wait event 采样需要和哪类指标一起使用？

回答时必须至少引用一个源码文件和一个运行期现象。

讨论题不是开放闲聊，而是训练把源码机制压缩成可迁移判断。

## 15. 源码复盘清单

如果需要快速复盘，本节可以按下面的源码锚点重新走一遍。

### 15.1. `pg_get_wait_events()`

先定位 `pg_get_wait_events()` 的调用者，不要从文件顶部线性读到底。

然后确认它如何影响 `wait_event_type`。

本锚点对应的运行期动作是：先从 pg_stat_activity 找到慢 SQL backend 的 state、wait_event_type 和 wait_event。

复盘时记录三个问题：

- 这个函数在正常路径上推进了什么状态？
- 它在 ERROR、abort 或 early return 时是否需要额外清理？
- 用户能否在 SQL、日志、pg_stat、wait event、EXPLAIN 或 profiler 中看到它的后果？

### 15.2. `pgstat_get_wait_event_type()`

先定位 `pgstat_get_wait_event_type()` 的调用者，不要从文件顶部线性读到底。

然后确认它如何影响 `wait_event`。

本锚点对应的运行期动作是：如果 wait_event_type 为空，说明当前采样瞬间没有进入 PostgreSQL wait path。

复盘时记录三个问题：

- 这个函数在正常路径上推进了什么状态？
- 它在 ERROR、abort 或 early return 时是否需要额外清理？
- 用户能否在 SQL、日志、pg_stat、wait event、EXPLAIN 或 profiler 中看到它的后果？

### 15.3. `LWLockAcquire()`

先定位 `LWLockAcquire()` 的调用者，不要从文件顶部线性读到底。

然后确认它如何影响 `PlanState 节点`。

本锚点对应的运行期动作是：如果类型是 Lock，转向 pg_locks、pg_blocking_pids() 和 lock manager 源码。

复盘时记录三个问题：

- 这个函数在正常路径上推进了什么状态？
- 它在 ERROR、abort 或 early return 时是否需要额外清理？
- 用户能否在 SQL、日志、pg_stat、wait event、EXPLAIN 或 profiler 中看到它的后果？

### 15.4. `ProcSleep()`

先定位 `ProcSleep()` 的调用者，不要从文件顶部线性读到底。

然后确认它如何影响 `PGPROC wait fields`。

本锚点对应的运行期动作是：如果类型是 LWLock，关注共享内存结构保护锁，例如 buffer、WAL、stats 或 predicate lock。

复盘时记录三个问题：

- 这个函数在正常路径上推进了什么状态？
- 它在 ERROR、abort 或 early return 时是否需要额外清理？
- 用户能否在 SQL、日志、pg_stat、wait event、EXPLAIN 或 profiler 中看到它的后果？

### 15.5. `FileReadV()`

先定位 `FileReadV()` 的调用者，不要从文件顶部线性读到底。

然后确认它如何影响 `pg_locks row`。

本锚点对应的运行期动作是：如果类型是 BufferPin，关注持 pin 的长查询、standby conflict 或 cleanup lock。

复盘时记录三个问题：

- 这个函数在正常路径上推进了什么状态？
- 它在 ERROR、abort 或 early return 时是否需要额外清理？
- 用户能否在 SQL、日志、pg_stat、wait event、EXPLAIN 或 profiler 中看到它的后果？

### 15.6. `WaitLatch()`

先定位 `WaitLatch()` 的调用者，不要从文件顶部线性读到底。

然后确认它如何影响 `I/O counters`。

本锚点对应的运行期动作是：如果类型是 IO，沿 wait_event 名称回到 md.c、fd.c、WAL 或 SLRU I/O 路径。

复盘时记录三个问题：

- 这个函数在正常路径上推进了什么状态？
- 它在 ERROR、abort 或 early return 时是否需要额外清理？
- 用户能否在 SQL、日志、pg_stat、wait event、EXPLAIN 或 profiler 中看到它的后果？

## 16. 本节小结

- wait event taxonomy 是慢 SQL 诊断地图，不是最终答案。
- 正确路线是类型归类、名称回源、跨视图验证、实验复现。
- 执行器等待常发生在下层 storage、lock、I/O、IPC 或 client 边界。
- 下一节会把 wait event 与 CPU、阻塞链和 EXPLAIN 时间合并判断。

本节最后沉淀出的可迁移规律是：

```text
观测值不是事实本身；观测值是某个源码状态在某个生命周期边界上的投影。
```

读 PostgreSQL 内核时，始终把 runtime 现象压回 state、owner、cleanup 和 cost。
