# PostgreSQL Hook 中的异常安全与开销控制

## 课程定位

前置知识：已经看过 auto_explain 如何捕获慢查询计划；本节收束到所有 executor hook 扩展都必须面对的安全和成本问题。

本节唯一主问题：

```text
扩展 hook 如何避免在 ERROR 路径破坏 executor 状态，采样率、阈值、GUC、内存上下文和递归调用保护如何控制线上开销？
```

核心矛盾：hook 代码运行在核心 executor 生命周期里，越靠近问题现场越有价值，但任何异常、递归状态残留或无条件 instrumentation 都会放大线上风险。

学完后应能审查一个 executor hook 扩展是否在 ERROR-safe、递归、内存上下文、previous hook、采样和日志成本上有明确边界。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

auto_explain 展示了 hook 能做什么。

本节追问 hook 不能随便做什么。

执行器 hook 不是普通库回调，它包在 SQL 执行主路径上。

一旦 hook 在 ERROR 路径留下错误状态，后续语句可能被污染。

一旦 hook 无条件打开重 instrumentation，线上观测本身会成为瓶颈。

本节只讲异常安全和开销控制，不再扩展新的观测功能。

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
安全的 executor hook 先用 cheap guard 决定是否启用；需要跨调用维护的状态必须在 PG_TRY/PG_FINALLY 或等价机制中恢复；临时分配放入 executor 可清理的 memory context；昂贵 instrumentation 受阈值、采样率和 GUC 控制；无论是否启用，都必须调用 previous 或 standard executor。
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
| 1 | `contrib/auto_explain/auto_explain.c` | PG_TRY/PG_FINALLY、nesting_level、sample_rate、GUC、MemoryContextSwitchTo 和 logging 主线。 |
| 2 | `src/backend/executor/execMain.c` | ExecutorStart/Run/Finish/End 的标准生命周期和 hook 分派。 |
| 3 | `src/backend/executor/instrument.c` | Instrumentation 的计时和节点级开销来源。 |
| 4 | `src/backend/commands/explain.c` | ExplainPrintPlan() 等格式化输出成本。 |
| 5 | `src/include/executor/executor.h` | hook 签名和 standard_* 声明。 |
| 6 | `src/include/executor/instrument.h` | INSTRUMENT_TIMER / BUFFERS / WAL / IO 等选项。 |

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

### 4.1. `nesting_level`

递归执行保护状态，必须在 ERROR 时恢复。

读这个状态时先问四个问题：

- 它是 backend-local、shared memory、文件状态，还是只在调用栈中短暂存在？
- 谁负责初始化它，初始化失败时有没有后续 cleanup 责任？
- 谁能读取或修改它，读取是否需要锁、pin、snapshot 或 owner？
- 它何时失效，失效后用户可见指标会如何表现？

在本节主问题中，`nesting_level` 的价值是帮助判断一个观测现象到底落在哪个状态边界。

### 4.2. `current_query_sampled`

当前顶层语句是否采样，避免每条语句都启用昂贵逻辑。

读这个状态时先问四个问题：

- 它是 backend-local、shared memory、文件状态，还是只在调用栈中短暂存在？
- 谁负责初始化它，初始化失败时有没有后续 cleanup 责任？
- 谁能读取或修改它，读取是否需要锁、pin、snapshot 或 owner？
- 它何时失效，失效后用户可见指标会如何表现？

在本节主问题中，`current_query_sampled` 的价值是帮助判断一个观测现象到底落在哪个状态边界。

### 4.3. `auto_explain_log_min_duration`

阈值开关，-1 代表禁用。

读这个状态时先问四个问题：

- 它是 backend-local、shared memory、文件状态，还是只在调用栈中短暂存在？
- 谁负责初始化它，初始化失败时有没有后续 cleanup 责任？
- 谁能读取或修改它，读取是否需要锁、pin、snapshot 或 owner？
- 它何时失效，失效后用户可见指标会如何表现？

在本节主问题中，`auto_explain_log_min_duration` 的价值是帮助判断一个观测现象到底落在哪个状态边界。

### 4.4. `instrument_options`

节点级 instrumentation 选项，直接决定执行时额外成本。

读这个状态时先问四个问题：

- 它是 backend-local、shared memory、文件状态，还是只在调用栈中短暂存在？
- 谁负责初始化它，初始化失败时有没有后续 cleanup 责任？
- 谁能读取或修改它，读取是否需要锁、pin、snapshot 或 owner？
- 它何时失效，失效后用户可见指标会如何表现？

在本节主问题中，`instrument_options` 的价值是帮助判断一个观测现象到底落在哪个状态边界。

### 4.5. `query_instr`

query-level 总耗时对象，End 阶段用于判断是否记录。

读这个状态时先问四个问题：

- 它是 backend-local、shared memory、文件状态，还是只在调用栈中短暂存在？
- 谁负责初始化它，初始化失败时有没有后续 cleanup 责任？
- 谁能读取或修改它，读取是否需要锁、pin、snapshot 或 owner？
- 它何时失效，失效后用户可见指标会如何表现？

在本节主问题中，`query_instr` 的价值是帮助判断一个观测现象到底落在哪个状态边界。

### 4.6. `es_query_cxt`

executor query context，适合放 End 阶段临时 explain 输出对象。

读这个状态时先问四个问题：

- 它是 backend-local、shared memory、文件状态，还是只在调用栈中短暂存在？
- 谁负责初始化它，初始化失败时有没有后续 cleanup 责任？
- 谁能读取或修改它，读取是否需要锁、pin、snapshot 或 owner？
- 它何时失效，失效后用户可见指标会如何表现？

在本节主问题中，`es_query_cxt` 的价值是帮助判断一个观测现象到底落在哪个状态边界。

### 4.7. `previous hook`

安全链式调用状态，任何分支都不能丢。

读这个状态时先问四个问题：

- 它是 backend-local、shared memory、文件状态，还是只在调用栈中短暂存在？
- 谁负责初始化它，初始化失败时有没有后续 cleanup 责任？
- 谁能读取或修改它，读取是否需要锁、pin、snapshot 或 owner？
- 它何时失效，失效后用户可见指标会如何表现？

在本节主问题中，`previous hook` 的价值是帮助判断一个观测现象到底落在哪个状态边界。

本节最重要的不变量是：

```text
raw field 不是语义；field + lifecycle + writer + reader + cleanup 才是语义。
```

## 5. 主流程源码 walkthrough

下面按时间顺序走主链路。

每一步都回答“状态发生了什么变化”，而不是把源码读成函数名清单。

### 5.1. 步骤 1

hook 函数进入后先做 cheap guard，例如检查阈值、采样率、嵌套层级和 parallel worker。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.2. 步骤 2

如果未启用，应尽快调用 previous 或 standard。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.3. 步骤 3

Start hook 只在确实需要时设置 query_instr_options 和 instrument_options。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.4. 步骤 4

Run hook 维护 nesting_level，因为执行过程中可能触发嵌套语句。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.5. 步骤 5

Run hook 使用 PG_TRY/PG_FINALLY 包裹 previous 或 standard 调用。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.6. 步骤 6

即使 standard_ExecutorRun() 抛 ERROR，FINALLY 仍会递减 nesting_level。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.7. 步骤 7

Finish hook 使用同样模式保护副作用 drain 阶段。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.8. 步骤 8

End hook 在 executor 状态仍有效时读取 query_instr。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.9. 步骤 9

End hook 切换到 es_query_cxt 构造 ExplainState 和输出字符串。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.10. 步骤 10

日志输出完成后切回原 memory context。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.11. 步骤 11

最后必须调用 previous End 或 standard_ExecutorEnd() 释放 executor 状态。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.12. 步骤 12

扩展自己的错误处理不能吞掉 executor ERROR，也不能跳过核心 cleanup。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

主链路可以压缩成：

```text
01. hook 函数进入后先做 cheap guard，例如检查阈值、采样率、嵌套层级和 parallel worker。
02. 如果未启用，应尽快调用 previous 或 standard。
03. Start hook 只在确实需要时设置 query_instr_options 和 instrument_options。
04. Run hook 维护 nesting_level，因为执行过程中可能触发嵌套语句。
05. Run hook 使用 PG_TRY/PG_FINALLY 包裹 previous 或 standard 调用。
06. 即使 standard_ExecutorRun() 抛 ERROR，FINALLY 仍会递减 nesting_level。
07. Finish hook 使用同样模式保护副作用 drain 阶段。
08. End hook 在 executor 状态仍有效时读取 query_instr。
09. End hook 切换到 es_query_cxt 构造 ExplainState 和输出字符串。
10. 日志输出完成后切回原 memory context。
11. 最后必须调用 previous End 或 standard_ExecutorEnd() 释放 executor 状态。
12. 扩展自己的错误处理不能吞掉 executor ERROR，也不能跳过核心 cleanup。
```

## 6. 生命周期 / ownership / cleanup

生命周期是本节能否用于真实诊断的关键。

### 6.1. 进入

hook 作为 executor 生命周期的一部分进入，不能假设自己是唯一扩展。

这里要特别区分两件事：

- 内存是否释放。
- 语义状态是否已经对其他 backend 或用户视图不可见。

很多观测问题并不是对象还在内存里，而是读取端还持有旧语义。

### 6.2. 启用判定

GUC、阈值、采样、嵌套和 worker 状态共同决定是否启用昂贵逻辑。

这里要特别区分两件事：

- 内存是否释放。
- 语义状态是否已经对其他 backend 或用户视图不可见。

很多观测问题并不是对象还在内存里，而是读取端还持有旧语义。

### 6.3. 执行保护

跨调用状态用 PG_TRY/PG_FINALLY 恢复。

这里要特别区分两件事：

- 内存是否释放。
- 语义状态是否已经对其他 backend 或用户视图不可见。

很多观测问题并不是对象还在内存里，而是读取端还持有旧语义。

### 6.4. 输出

日志和格式化放在状态仍有效但即将清理的 End 边界。

这里要特别区分两件事：

- 内存是否释放。
- 语义状态是否已经对其他 backend 或用户视图不可见。

很多观测问题并不是对象还在内存里，而是读取端还持有旧语义。

### 6.5. 退出

调用 previous / standard 后让 executor 自己完成资源释放。

这里要特别区分两件事：

- 内存是否释放。
- 语义状态是否已经对其他 backend 或用户视图不可见。

很多观测问题并不是对象还在内存里，而是读取端还持有旧语义。

ERROR / abort 路径上，不能依赖普通 C 调用栈逐层返回完成清理。

因此源码里要找的是 ResourceOwner、MemoryContext、PG_TRY/PG_FINALLY、事务回调或标准 executor teardown。

## 7. 正确性机制层次

本节的正确性来自多个层次叠加，而不是单个函数。

### 7.1. 机制 1

hook 不应吞掉 ERROR；它最多保证自己的状态恢复。

把它放回运行期，可以得到两个判断：

- 正常路径中，它限制了状态何时能被推进或读取。
- 异常路径中，它决定用户看到的是空值、旧值、错误还是等待。

### 7.2. 机制 2

递归计数、采样标志和临时状态必须是 backend-local。

把它放回运行期，可以得到两个判断：

- 正常路径中，它限制了状态何时能被推进或读取。
- 异常路径中，它决定用户看到的是空值、旧值、错误还是等待。

### 7.3. 机制 3

MemoryContextSwitchTo() 后必须切回 old context。

把它放回运行期，可以得到两个判断：

- 正常路径中，它限制了状态何时能被推进或读取。
- 异常路径中，它决定用户看到的是空值、旧值、错误还是等待。

### 7.4. 机制 4

End 阶段访问 EState 必须早于 standard_ExecutorEnd()。

把它放回运行期，可以得到两个判断：

- 正常路径中，它限制了状态何时能被推进或读取。
- 异常路径中，它决定用户看到的是空值、旧值、错误还是等待。

### 7.5. 机制 5

instrumentation 选项必须在 PlanState 初始化前确定。

把它放回运行期，可以得到两个判断：

- 正常路径中，它限制了状态何时能被推进或读取。
- 异常路径中，它决定用户看到的是空值、旧值、错误还是等待。

### 7.6. 机制 6

采样率和阈值只能控制观测，不应改变查询结果或执行路径语义。

把它放回运行期，可以得到两个判断：

- 正常路径中，它限制了状态何时能被推进或读取。
- 异常路径中，它决定用户看到的是空值、旧值、错误还是等待。

不要把这些机制抽象成“为了安全”。

更准确的说法是：它们把低成本 hot path 和可解释的观测边界同时保留下来。

## 8. 错误路径 / 异常路径 / fallback

下面这些场景不是边角料，而是现场排障最常遇到的误判来源。

### 8.1. 没有 PG_FINALLY

ERROR 后 nesting_level 可能不回落，后续语句被当成嵌套语句。

排查时不要先改参数。

先问：当前现象是写入端没有推进，读取端没有刷新，还是观察工具选错了层次。

### 8.2. 无条件 instrumentation

所有查询都承担 timer、buffers、wal 或 io 统计成本。

排查时不要先改参数。

先问：当前现象是写入端没有推进，读取端没有刷新，还是观察工具选错了层次。

### 8.3. 错误 memory context

End 阶段构造的大对象挂到过长生命周期，形成 session 级 retention。

排查时不要先改参数。

先问：当前现象是写入端没有推进，读取端没有刷新，还是观察工具选错了层次。

### 8.4. 跳过 previous hook

破坏其他扩展和标准 executor 生命周期。

排查时不要先改参数。

先问：当前现象是写入端没有推进，读取端没有刷新，还是观察工具选错了层次。

### 8.5. 日志中再次触发递归

输出或辅助查询如果进入 executor，必须有递归保护。

排查时不要先改参数。

先问：当前现象是写入端没有推进，读取端没有刷新，还是观察工具选错了层次。

fallback 的设计目标通常不是让结果更漂亮，而是让系统在异常路径上保持可恢复、可清理、可解释。

## 9. 成本、资源与跨模块传播

可观测性代码如果放错层次，会把诊断工具变成性能问题的一部分。

### 9.1. 成本维度 1

disabled path 应接近一个分支和一次 previous 调用。

这个成本通常会继续传播到相邻模块：

- executor hot path 是否多一次分支、计时或锁。
- shared memory 是否出现新的争用点。
- 日志、视图或监控采样是否放大已有压力。

### 9.2. 成本维度 2

sample_rate 在顶层语句开始时决定，避免每个节点反复抽样。

这个成本通常会继续传播到相邻模块：

- executor hot path 是否多一次分支、计时或锁。
- shared memory 是否出现新的争用点。
- 日志、视图或监控采样是否放大已有压力。

### 9.3. 成本维度 3

log_min_duration 让短查询不承担日志格式化成本，但 instrumentation 成本取决于是否提前开启。

这个成本通常会继续传播到相邻模块：

- executor hot path 是否多一次分支、计时或锁。
- shared memory 是否出现新的争用点。
- 日志、视图或监控采样是否放大已有压力。

### 9.4. 成本维度 4

log_timing、buffers、io、wal 分别增加不同观测维度的采集成本。

这个成本通常会继续传播到相邻模块：

- executor hot path 是否多一次分支、计时或锁。
- shared memory 是否出现新的争用点。
- 日志、视图或监控采样是否放大已有压力。

### 9.5. 成本维度 5

ExplainPrintPlan() 和 ereport 日志输出可能在高并发慢查询风暴中放大压力。

这个成本通常会继续传播到相邻模块：

- executor hot path 是否多一次分支、计时或锁。
- shared memory 是否出现新的争用点。
- 日志、视图或监控采样是否放大已有压力。

因此课程里的实验默认是短时、可控、可回滚的，不建议把强制刷新或高频采样作为长期配置。

## 10. 观测与诊断入口

观测入口要按问题层次选择。

### 10.1. `auto_explain.log_min_duration`

关闭或限制记录阈值。

使用这个入口时要同时记录：

- 采样时间。
- backend pid 或 query_id。
- 是否处于同一事务、同一 backend 或同一执行。

### 10.2. `auto_explain.sample_rate`

降低被完整观测的查询比例。

使用这个入口时要同时记录：

- 采样时间。
- backend pid 或 query_id。
- 是否处于同一事务、同一 backend 或同一执行。

### 10.3. `auto_explain.log_timing`

控制高频计时开销。

使用这个入口时要同时记录：

- 采样时间。
- backend pid 或 query_id。
- 是否处于同一事务、同一 backend 或同一执行。

### 10.4. `auto_explain.log_buffers / log_wal / log_io`

控制额外资源计数。

使用这个入口时要同时记录：

- 采样时间。
- backend pid 或 query_id。
- 是否处于同一事务、同一 backend 或同一执行。

### 10.5. `server log volume`

观察 hook 输出是否本身成为压力源。

使用这个入口时要同时记录：

- 采样时间。
- backend pid 或 query_id。
- 是否处于同一事务、同一 backend 或同一执行。

### 10.6. `perf 栈`

确认 hook 或 ExplainPrintPlan 是否出现在热点路径。

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

### 12.1. 验证 disabled path

实验步骤：

```sql
LOAD 'auto_explain';
SET auto_explain.log_min_duration = -1;
SELECT count(*) FROM generate_series(1,100000);
```

观察关闭时不应输出计划，成本主要是 hook 分支。

实验后回到源码，至少确认一个入口函数和一个状态字段。

### 12.2. 验证采样保护

实验步骤：

```sql
SET auto_explain.log_min_duration = 0;
SET auto_explain.sample_rate = 0.01;
SELECT count(*) FROM generate_series(1,1000);
```

多次执行，确认不是每条查询都输出。

实验后回到源码，至少确认一个入口函数和一个状态字段。

### 12.3. 验证 timing 开销

实验步骤：

```sql
SET auto_explain.log_analyze = on;
SET auto_explain.log_timing = off;
SELECT count(*) FROM t_pgstat_demo;
```

关闭节点计时后仍可记录 rows，降低 timer 成本。

实验后回到源码，至少确认一个入口函数和一个状态字段。

### 12.4. 阅读异常保护

实验步骤：

```bash
rg -n "PG_TRY|PG_FINALLY|nesting_level" /home/highgo/postgres/contrib/auto_explain/auto_explain.c
```

确认 Run / Finish 中跨调用状态在 ERROR 时恢复。

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

1. hook 扩展为什么通常只恢复自己的状态，而不处理 executor cleanup？

回答时必须至少引用一个源码文件和一个运行期现象。

2. disabled path 应该承担多少成本才算可接受？

回答时必须至少引用一个源码文件和一个运行期现象。

3. sample_rate 与 log_min_duration 哪个控制日志量，哪个控制 instrumentation 成本？

回答时必须至少引用一个源码文件和一个运行期现象。

4. 为什么 End hook 要切到 es_query_cxt？

回答时必须至少引用一个源码文件和一个运行期现象。

5. 多个 hook 扩展同时打开 instrumentation 时如何评估叠加成本？

回答时必须至少引用一个源码文件和一个运行期现象。

6. 什么情况下 hook 观测应被替换为外部 profiler？

回答时必须至少引用一个源码文件和一个运行期现象。

讨论题不是开放闲聊，而是训练把源码机制压缩成可迁移判断。

## 15. 源码复盘清单

如果需要快速复盘，本节可以按下面的源码锚点重新走一遍。

### 15.1. `PG_TRY()`

先定位 `PG_TRY()` 的调用者，不要从文件顶部线性读到底。

然后确认它如何影响 `nesting_level`。

本锚点对应的运行期动作是：hook 函数进入后先做 cheap guard，例如检查阈值、采样率、嵌套层级和 parallel worker。

复盘时记录三个问题：

- 这个函数在正常路径上推进了什么状态？
- 它在 ERROR、abort 或 early return 时是否需要额外清理？
- 用户能否在 SQL、日志、pg_stat、wait event、EXPLAIN 或 profiler 中看到它的后果？

### 15.2. `PG_FINALLY()`

先定位 `PG_FINALLY()` 的调用者，不要从文件顶部线性读到底。

然后确认它如何影响 `current_query_sampled`。

本锚点对应的运行期动作是：如果未启用，应尽快调用 previous 或 standard。

复盘时记录三个问题：

- 这个函数在正常路径上推进了什么状态？
- 它在 ERROR、abort 或 early return 时是否需要额外清理？
- 用户能否在 SQL、日志、pg_stat、wait event、EXPLAIN 或 profiler 中看到它的后果？

### 15.3. `nesting_level`

先定位 `nesting_level` 的调用者，不要从文件顶部线性读到底。

然后确认它如何影响 `auto_explain_log_min_duration`。

本锚点对应的运行期动作是：Start hook 只在确实需要时设置 query_instr_options 和 instrument_options。

复盘时记录三个问题：

- 这个函数在正常路径上推进了什么状态？
- 它在 ERROR、abort 或 early return 时是否需要额外清理？
- 用户能否在 SQL、日志、pg_stat、wait event、EXPLAIN 或 profiler 中看到它的后果？

### 15.4. `MemoryContextSwitchTo()`

先定位 `MemoryContextSwitchTo()` 的调用者，不要从文件顶部线性读到底。

然后确认它如何影响 `instrument_options`。

本锚点对应的运行期动作是：Run hook 维护 nesting_level，因为执行过程中可能触发嵌套语句。

复盘时记录三个问题：

- 这个函数在正常路径上推进了什么状态？
- 它在 ERROR、abort 或 early return 时是否需要额外清理？
- 用户能否在 SQL、日志、pg_stat、wait event、EXPLAIN 或 profiler 中看到它的后果？

### 15.5. `INSTRUMENT_TIMER`

先定位 `INSTRUMENT_TIMER` 的调用者，不要从文件顶部线性读到底。

然后确认它如何影响 `query_instr`。

本锚点对应的运行期动作是：Run hook 使用 PG_TRY/PG_FINALLY 包裹 previous 或 standard 调用。

复盘时记录三个问题：

- 这个函数在正常路径上推进了什么状态？
- 它在 ERROR、abort 或 early return 时是否需要额外清理？
- 用户能否在 SQL、日志、pg_stat、wait event、EXPLAIN 或 profiler 中看到它的后果？

### 15.6. `ExplainPrintPlan()`

先定位 `ExplainPrintPlan()` 的调用者，不要从文件顶部线性读到底。

然后确认它如何影响 `es_query_cxt`。

本锚点对应的运行期动作是：即使 standard_ExecutorRun() 抛 ERROR，FINALLY 仍会递减 nesting_level。

复盘时记录三个问题：

- 这个函数在正常路径上推进了什么状态？
- 它在 ERROR、abort 或 early return 时是否需要额外清理？
- 用户能否在 SQL、日志、pg_stat、wait event、EXPLAIN 或 profiler 中看到它的后果？

## 16. 本节小结

- executor hook 的价值来自靠近执行现场，风险也来自靠近执行现场。
- 安全 hook 必须守住 previous 调用、ERROR 状态恢复、正确 memory context 和低成本 disabled path。
- auto_explain 提供了采样、阈值、GUC 和 PG_TRY/PG_FINALLY 的实际样板。
- 到这里，04 目录的 pg_stat、wait event、hook 与 auto_explain 主线已经形成完整观测闭环。

本节最后沉淀出的可迁移规律是：

```text
观测值不是事实本身；观测值是某个源码状态在某个生命周期边界上的投影。
```

读 PostgreSQL 内核时，始终把 runtime 现象压回 state、owner、cleanup 和 cost。
