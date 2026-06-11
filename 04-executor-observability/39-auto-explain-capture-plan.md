# PostgreSQL auto_explain 如何捕获慢查询计划

## 课程定位

前置知识：已经理解 executor hook 链；本节以 contrib/auto_explain 为例，看一个真实扩展如何用 hook 捕获慢查询计划。

本节唯一主问题：

```text
auto_explain 如何在 executor hook 中测量执行、判断阈值、调用 explain 输出计划，并处理 nested statement、log level、buffers、timing 等选项？
```

核心矛盾：慢查询计划需要在线上自动捕获，但不能对所有查询无条件开启昂贵 instrumentation，也不能改变 SQL 的执行语义。

学完后应能从 auto_explain 日志反推它在 Start、Run、Finish、End 哪些边界做了什么，以及哪些 GUC 会改变采集成本和输出内容。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

上一节讲 hook 链的通用协议。

auto_explain 是这套协议最常见的真实用例。

它不是重新执行 EXPLAIN，也不是 planner 阶段保存一份计划文本。

它在 executor 生命周期中打开必要 instrumentation，等待真实执行结束后基于 QueryDesc 和 PlanState 输出计划。

本节只讲 capture plan 的主线。

下一节会继续讲 hook 异常安全和开销控制。

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
auto_explain 在 _PG_init() 定义 GUC 并安装 executor hooks；Start 阶段按阈值和采样率决定是否启用 query / node instrumentation；Run 和 Finish 只维护 nesting_level 并调用 previous；End 阶段读取 query_instr 总耗时，超过 log_min_duration 时构造 ExplainState 并调用 ExplainPrintPlan() 写日志。
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
| 1 | `contrib/auto_explain/auto_explain.c` | _PG_init()、explain_ExecutorStart/Run/Finish/End、GUC、采样和输出主线。 |
| 2 | `src/include/executor/executor.h` | executor hook 类型和 QueryDesc 入口声明。 |
| 3 | `src/backend/executor/execMain.c` | 标准 executor 调用 hook 并创建 query_instr / PlanState instrumentation。 |
| 4 | `src/backend/executor/instrument.c` | query-level 和 node-level instrumentation 计时。 |
| 5 | `src/backend/commands/explain.c` | ExplainPrintPlan()、ExplainQueryText()、ExplainQueryParameters()、ExplainPrintTriggers()。 |
| 6 | `src/backend/commands/explain_state.c` | NewExplainState() 与 ExplainState 输出状态。 |
| 7 | `src/include/commands/explain_state.h` | ExplainState 字段，控制 analyze、buffers、wal、timing、format 等输出。 |

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

### 4.1. `auto_explain_log_min_duration`

阈值；-1 禁用，0 记录所有采样命中的语句。

读这个状态时先问四个问题：

- 它是 backend-local、shared memory、文件状态，还是只在调用栈中短暂存在？
- 谁负责初始化它，初始化失败时有没有后续 cleanup 责任？
- 谁能读取或修改它，读取是否需要锁、pin、snapshot 或 owner？
- 它何时失效，失效后用户可见指标会如何表现？

在本节主问题中，`auto_explain_log_min_duration` 的价值是帮助判断一个观测现象到底落在哪个状态边界。

### 4.2. `auto_explain_sample_rate`

采样率，只让部分顶层查询进入 auto_explain 处理。

读这个状态时先问四个问题：

- 它是 backend-local、shared memory、文件状态，还是只在调用栈中短暂存在？
- 谁负责初始化它，初始化失败时有没有后续 cleanup 责任？
- 谁能读取或修改它，读取是否需要锁、pin、snapshot 或 owner？
- 它何时失效，失效后用户可见指标会如何表现？

在本节主问题中，`auto_explain_sample_rate` 的价值是帮助判断一个观测现象到底落在哪个状态边界。

### 4.3. `current_query_sampled`

当前顶层语句是否被采样命中的 backend-local 状态。

读这个状态时先问四个问题：

- 它是 backend-local、shared memory、文件状态，还是只在调用栈中短暂存在？
- 谁负责初始化它，初始化失败时有没有后续 cleanup 责任？
- 谁能读取或修改它，读取是否需要锁、pin、snapshot 或 owner？
- 它何时失效，失效后用户可见指标会如何表现？

在本节主问题中，`current_query_sampled` 的价值是帮助判断一个观测现象到底落在哪个状态边界。

### 4.4. `nesting_level`

区分顶层与嵌套语句，配合 log_nested_statements。

读这个状态时先问四个问题：

- 它是 backend-local、shared memory、文件状态，还是只在调用栈中短暂存在？
- 谁负责初始化它，初始化失败时有没有后续 cleanup 责任？
- 谁能读取或修改它，读取是否需要锁、pin、snapshot 或 owner？
- 它何时失效，失效后用户可见指标会如何表现？

在本节主问题中，`nesting_level` 的价值是帮助判断一个观测现象到底落在哪个状态边界。

### 4.5. `query_instr_options`

query-level 总耗时计时开关。

读这个状态时先问四个问题：

- 它是 backend-local、shared memory、文件状态，还是只在调用栈中短暂存在？
- 谁负责初始化它，初始化失败时有没有后续 cleanup 责任？
- 谁能读取或修改它，读取是否需要锁、pin、snapshot 或 owner？
- 它何时失效，失效后用户可见指标会如何表现？

在本节主问题中，`query_instr_options` 的价值是帮助判断一个观测现象到底落在哪个状态边界。

### 4.6. `instrument_options`

节点级 rows/timing/buffers/io/wal instrumentation 开关。

读这个状态时先问四个问题：

- 它是 backend-local、shared memory、文件状态，还是只在调用栈中短暂存在？
- 谁负责初始化它，初始化失败时有没有后续 cleanup 责任？
- 谁能读取或修改它，读取是否需要锁、pin、snapshot 或 owner？
- 它何时失效，失效后用户可见指标会如何表现？

在本节主问题中，`instrument_options` 的价值是帮助判断一个观测现象到底落在哪个状态边界。

### 4.7. `ExplainState`

End 阶段格式化计划和可选信息的输出状态。

读这个状态时先问四个问题：

- 它是 backend-local、shared memory、文件状态，还是只在调用栈中短暂存在？
- 谁负责初始化它，初始化失败时有没有后续 cleanup 责任？
- 谁能读取或修改它，读取是否需要锁、pin、snapshot 或 owner？
- 它何时失效，失效后用户可见指标会如何表现？

在本节主问题中，`ExplainState` 的价值是帮助判断一个观测现象到底落在哪个状态边界。

本节最重要的不变量是：

```text
raw field 不是语义；field + lifecycle + writer + reader + cleanup 才是语义。
```

## 5. 主流程源码 walkthrough

下面按时间顺序走主链路。

每一步都回答“状态发生了什么变化”，而不是把源码读成函数名清单。

### 5.1. 步骤 1

_PG_init() 定义 auto_explain.* GUC。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.2. 步骤 2

_PG_init() 调用 MarkGUCPrefixReserved("auto_explain")。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.3. 步骤 3

_PG_init() 保存 previous executor hooks。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.4. 步骤 4

_PG_init() 安装 explain_ExecutorStart/Run/Finish/End。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.5. 步骤 5

ExecutorStart 进入 explain_ExecutorStart。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.6. 步骤 6

顶层语句先根据 log_min_duration、parallel worker 和 sample_rate 决定 current_query_sampled。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.7. 步骤 7

如果 auto_explain_enabled()，Start 阶段打开 query_instr_options 的 INSTRUMENT_TIMER。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.8. 步骤 8

如果 log_analyze 且不是 EXPLAIN_ONLY，按 GUC 打开 node-level timer、rows、buffers、io、wal。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.9. 步骤 9

Start hook 调用 previous 或 standard_ExecutorStart，真实构造 EState / PlanState。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.10. 步骤 10

Run hook 和 Finish hook 用 PG_TRY/PG_FINALLY 维护 nesting_level。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.11. 步骤 11

End hook 在 standard_ExecutorEnd 前读取 queryDesc->query_instr 总耗时。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.12. 步骤 12

如果耗时超过阈值，End hook 创建 ExplainState 并调用 ExplainPrintPlan() 输出。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

主链路可以压缩成：

```text
01. _PG_init() 定义 auto_explain.* GUC。
02. _PG_init() 调用 MarkGUCPrefixReserved("auto_explain")。
03. _PG_init() 保存 previous executor hooks。
04. _PG_init() 安装 explain_ExecutorStart/Run/Finish/End。
05. ExecutorStart 进入 explain_ExecutorStart。
06. 顶层语句先根据 log_min_duration、parallel worker 和 sample_rate 决定 current_query_sampled。
07. 如果 auto_explain_enabled()，Start 阶段打开 query_instr_options 的 INSTRUMENT_TIMER。
08. 如果 log_analyze 且不是 EXPLAIN_ONLY，按 GUC 打开 node-level timer、rows、buffers、io、wal。
09. Start hook 调用 previous 或 standard_ExecutorStart，真实构造 EState / PlanState。
10. Run hook 和 Finish hook 用 PG_TRY/PG_FINALLY 维护 nesting_level。
11. End hook 在 standard_ExecutorEnd 前读取 queryDesc->query_instr 总耗时。
12. 如果耗时超过阈值，End hook 创建 ExplainState 并调用 ExplainPrintPlan() 输出。
```

## 6. 生命周期 / ownership / cleanup

生命周期是本节能否用于真实诊断的关键。

### 6.1. 加载

session_preload_libraries、shared_preload_libraries 或 LOAD 让模块进入 backend。

这里要特别区分两件事：

- 内存是否释放。
- 语义状态是否已经对其他 backend 或用户视图不可见。

很多观测问题并不是对象还在内存里，而是读取端还持有旧语义。

### 6.2. 采样

每个顶层语句开始时决定是否采样，嵌套语句按配置跟随。

这里要特别区分两件事：

- 内存是否释放。
- 语义状态是否已经对其他 backend 或用户视图不可见。

很多观测问题并不是对象还在内存里，而是读取端还持有旧语义。

### 6.3. 执行

真实 SQL 只执行一次，auto_explain 只是包裹 executor。

这里要特别区分两件事：

- 内存是否释放。
- 语义状态是否已经对其他 backend 或用户视图不可见。

很多观测问题并不是对象还在内存里，而是读取端还持有旧语义。

### 6.4. 输出

End 阶段在 executor 状态释放前格式化计划并写日志。

这里要特别区分两件事：

- 内存是否释放。
- 语义状态是否已经对其他 backend 或用户视图不可见。

很多观测问题并不是对象还在内存里，而是读取端还持有旧语义。

### 6.5. 清理

standard_ExecutorEnd() 释放 EState、PlanState、slot 和 executor context。

这里要特别区分两件事：

- 内存是否释放。
- 语义状态是否已经对其他 backend 或用户视图不可见。

很多观测问题并不是对象还在内存里，而是读取端还持有旧语义。

ERROR / abort 路径上，不能依赖普通 C 调用栈逐层返回完成清理。

因此源码里要找的是 ResourceOwner、MemoryContext、PG_TRY/PG_FINALLY、事务回调或标准 executor teardown。

## 7. 正确性机制层次

本节的正确性来自多个层次叠加，而不是单个函数。

### 7.1. 机制 1

auto_explain 不重新执行查询；它基于真实执行后的 QueryDesc 输出。

把它放回运行期，可以得到两个判断：

- 正常路径中，它限制了状态何时能被推进或读取。
- 异常路径中，它决定用户看到的是空值、旧值、错误还是等待。

### 7.2. 机制 2

Start 阶段必须早于 PlanState instrumentation 分配。

把它放回运行期，可以得到两个判断：

- 正常路径中，它限制了状态何时能被推进或读取。
- 异常路径中，它决定用户看到的是空值、旧值、错误还是等待。

### 7.3. 机制 3

Parallel worker 中 auto_explain 选择不干预，避免和父会话 EXPLAIN 汇总冲突。

把它放回运行期，可以得到两个判断：

- 正常路径中，它限制了状态何时能被推进或读取。
- 异常路径中，它决定用户看到的是空值、旧值、错误还是等待。

### 7.4. 机制 4

Run / Finish 使用 PG_FINALLY 保证 nesting_level 在 ERROR 时恢复。

把它放回运行期，可以得到两个判断：

- 正常路径中，它限制了状态何时能被推进或读取。
- 异常路径中，它决定用户看到的是空值、旧值、错误还是等待。

### 7.5. 机制 5

End 阶段切到 es_query_cxt，输出临时对象随 ExecutorEnd 清理。

把它放回运行期，可以得到两个判断：

- 正常路径中，它限制了状态何时能被推进或读取。
- 异常路径中，它决定用户看到的是空值、旧值、错误还是等待。

### 7.6. 机制 6

JSON 输出需要把 EXPLAIN 结果修成对象形式，源码中有明确处理。

把它放回运行期，可以得到两个判断：

- 正常路径中，它限制了状态何时能被推进或读取。
- 异常路径中，它决定用户看到的是空值、旧值、错误还是等待。

不要把这些机制抽象成“为了安全”。

更准确的说法是：它们把低成本 hot path 和可解释的观测边界同时保留下来。

## 8. 错误路径 / 异常路径 / fallback

下面这些场景不是边角料，而是现场排障最常遇到的误判来源。

### 8.1. 以为 auto_explain 是 planner hook

它主要在 executor hook 中打开 instrumentation 并输出执行后计划。

排查时不要先改参数。

先问：当前现象是写入端没有推进，读取端没有刷新，还是观察工具选错了层次。

### 8.2. 开启 log_analyze 后忽略开销

节点级 timer、buffers、wal、io 都会增加执行成本。

排查时不要先改参数。

先问：当前现象是写入端没有推进，读取端没有刷新，还是观察工具选错了层次。

### 8.3. 误读 nested statements

默认只记录顶层，log_nested_statements 才扩展到嵌套语句。

排查时不要先改参数。

先问：当前现象是写入端没有推进，读取端没有刷新，还是观察工具选错了层次。

### 8.4. 忘记 sample_rate

采样未命中的慢 SQL 不会记录。

排查时不要先改参数。

先问：当前现象是写入端没有推进，读取端没有刷新，还是观察工具选错了层次。

### 8.5. 把 auto_explain 日志当客户端结果

输出走 server log，不改变 SQL 返回。

排查时不要先改参数。

先问：当前现象是写入端没有推进，读取端没有刷新，还是观察工具选错了层次。

fallback 的设计目标通常不是让结果更漂亮，而是让系统在异常路径上保持可恢复、可清理、可解释。

## 9. 成本、资源与跨模块传播

可观测性代码如果放错层次，会把诊断工具变成性能问题的一部分。

### 9.1. 成本维度 1

禁用时主要是 hook 条件判断成本。

这个成本通常会继续传播到相邻模块：

- executor hot path 是否多一次分支、计时或锁。
- shared memory 是否出现新的争用点。
- 日志、视图或监控采样是否放大已有压力。

### 9.2. 成本维度 2

query_instr_options 只记录 query-level 总时间，成本低于节点级 instrumentation。

这个成本通常会继续传播到相邻模块：

- executor hot path 是否多一次分支、计时或锁。
- shared memory 是否出现新的争用点。
- 日志、视图或监控采样是否放大已有压力。

### 9.3. 成本维度 3

log_analyze 开启后每个节点可能记录 rows、timing、buffers、io、wal。

这个成本通常会继续传播到相邻模块：

- executor hot path 是否多一次分支、计时或锁。
- shared memory 是否出现新的争用点。
- 日志、视图或监控采样是否放大已有压力。

### 9.4. 成本维度 4

log_timing 关闭可以减少高频计时开销。

这个成本通常会继续传播到相邻模块：

- executor hot path 是否多一次分支、计时或锁。
- shared memory 是否出现新的争用点。
- 日志、视图或监控采样是否放大已有压力。

### 9.5. 成本维度 5

End 阶段格式化和日志写入发生在慢查询之后，也会占用 backend 时间。

这个成本通常会继续传播到相邻模块：

- executor hot path 是否多一次分支、计时或锁。
- shared memory 是否出现新的争用点。
- 日志、视图或监控采样是否放大已有压力。

因此课程里的实验默认是短时、可控、可回滚的，不建议把强制刷新或高频采样作为长期配置。

## 10. 观测与诊断入口

观测入口要按问题层次选择。

### 10.1. `auto_explain.log_min_duration`

决定记录阈值。

使用这个入口时要同时记录：

- 采样时间。
- backend pid 或 query_id。
- 是否处于同一事务、同一 backend 或同一执行。

### 10.2. `auto_explain.log_analyze`

是否输出执行实际指标。

使用这个入口时要同时记录：

- 采样时间。
- backend pid 或 query_id。
- 是否处于同一事务、同一 backend 或同一执行。

### 10.3. `auto_explain.log_buffers / log_wal / log_io`

控制额外资源指标。

使用这个入口时要同时记录：

- 采样时间。
- backend pid 或 query_id。
- 是否处于同一事务、同一 backend 或同一执行。

### 10.4. `auto_explain.log_timing`

控制节点计时。

使用这个入口时要同时记录：

- 采样时间。
- backend pid 或 query_id。
- 是否处于同一事务、同一 backend 或同一执行。

### 10.5. `auto_explain.sample_rate`

控制采样比例。

使用这个入口时要同时记录：

- 采样时间。
- backend pid 或 query_id。
- 是否处于同一事务、同一 backend 或同一执行。

### 10.6. `server log`

最终计划输出位置。

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

### 12.1. 记录所有查询

实验步骤：

```sql
LOAD 'auto_explain';
SET auto_explain.log_min_duration = 0;
SELECT count(*) FROM generate_series(1,1000);
```

确认 auto_explain 通过 executor hook 输出计划到日志。

实验后回到源码，至少确认一个入口函数和一个状态字段。

### 12.2. 打开 analyze 和 buffers

实验步骤：

```sql
SET auto_explain.log_analyze = on;
SET auto_explain.log_buffers = on;
SELECT count(*) FROM t_pgstat_demo;
```

观察日志里 actual rows、loops 和 buffers。

实验后回到源码，至少确认一个入口函数和一个状态字段。

### 12.3. 关闭 timing 降低扰动

实验步骤：

```sql
SET auto_explain.log_timing = off;
SET auto_explain.log_analyze = on;
SELECT count(*) FROM t_pgstat_demo;
```

对比日志字段和执行成本。

实验后回到源码，至少确认一个入口函数和一个状态字段。

### 12.4. 验证采样

实验步骤：

```sql
SET auto_explain.sample_rate = 0.1;
SET auto_explain.log_min_duration = 0;
SELECT generate_series(1,20);
```

多次执行时只有采样命中的语句记录。

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

1. auto_explain 为什么要在 Start 阶段设置 instrumentation？

回答时必须至少引用一个源码文件和一个运行期现象。

2. 为什么它不在 Run hook 里直接输出计划？

回答时必须至少引用一个源码文件和一个运行期现象。

3. log_analyze、log_buffers、log_timing 的成本差别在哪里？

回答时必须至少引用一个源码文件和一个运行期现象。

4. sample_rate 和 log_min_duration 谁先决定是否记录？

回答时必须至少引用一个源码文件和一个运行期现象。

5. Parallel worker 中为什么不由 worker 自己记录 auto_explain？

回答时必须至少引用一个源码文件和一个运行期现象。

6. auto_explain 与 EXPLAIN ANALYZE 的观测扰动有什么不同？

回答时必须至少引用一个源码文件和一个运行期现象。

讨论题不是开放闲聊，而是训练把源码机制压缩成可迁移判断。

## 15. 源码复盘清单

如果需要快速复盘，本节可以按下面的源码锚点重新走一遍。

### 15.1. `explain_ExecutorStart()`

先定位 `explain_ExecutorStart()` 的调用者，不要从文件顶部线性读到底。

然后确认它如何影响 `auto_explain_log_min_duration`。

本锚点对应的运行期动作是：_PG_init() 定义 auto_explain.* GUC。

复盘时记录三个问题：

- 这个函数在正常路径上推进了什么状态？
- 它在 ERROR、abort 或 early return 时是否需要额外清理？
- 用户能否在 SQL、日志、pg_stat、wait event、EXPLAIN 或 profiler 中看到它的后果？

### 15.2. `explain_ExecutorRun()`

先定位 `explain_ExecutorRun()` 的调用者，不要从文件顶部线性读到底。

然后确认它如何影响 `auto_explain_sample_rate`。

本锚点对应的运行期动作是：_PG_init() 调用 MarkGUCPrefixReserved("auto_explain")。

复盘时记录三个问题：

- 这个函数在正常路径上推进了什么状态？
- 它在 ERROR、abort 或 early return 时是否需要额外清理？
- 用户能否在 SQL、日志、pg_stat、wait event、EXPLAIN 或 profiler 中看到它的后果？

### 15.3. `explain_ExecutorFinish()`

先定位 `explain_ExecutorFinish()` 的调用者，不要从文件顶部线性读到底。

然后确认它如何影响 `current_query_sampled`。

本锚点对应的运行期动作是：_PG_init() 保存 previous executor hooks。

复盘时记录三个问题：

- 这个函数在正常路径上推进了什么状态？
- 它在 ERROR、abort 或 early return 时是否需要额外清理？
- 用户能否在 SQL、日志、pg_stat、wait event、EXPLAIN 或 profiler 中看到它的后果？

### 15.4. `explain_ExecutorEnd()`

先定位 `explain_ExecutorEnd()` 的调用者，不要从文件顶部线性读到底。

然后确认它如何影响 `nesting_level`。

本锚点对应的运行期动作是：_PG_init() 安装 explain_ExecutorStart/Run/Finish/End。

复盘时记录三个问题：

- 这个函数在正常路径上推进了什么状态？
- 它在 ERROR、abort 或 early return 时是否需要额外清理？
- 用户能否在 SQL、日志、pg_stat、wait event、EXPLAIN 或 profiler 中看到它的后果？

### 15.5. `ExplainPrintPlan()`

先定位 `ExplainPrintPlan()` 的调用者，不要从文件顶部线性读到底。

然后确认它如何影响 `query_instr_options`。

本锚点对应的运行期动作是：ExecutorStart 进入 explain_ExecutorStart。

复盘时记录三个问题：

- 这个函数在正常路径上推进了什么状态？
- 它在 ERROR、abort 或 early return 时是否需要额外清理？
- 用户能否在 SQL、日志、pg_stat、wait event、EXPLAIN 或 profiler 中看到它的后果？

### 15.6. `auto_explain_enabled()`

先定位 `auto_explain_enabled()` 的调用者，不要从文件顶部线性读到底。

然后确认它如何影响 `instrument_options`。

本锚点对应的运行期动作是：顶层语句先根据 log_min_duration、parallel worker 和 sample_rate 决定 current_query_sampled。

复盘时记录三个问题：

- 这个函数在正常路径上推进了什么状态？
- 它在 ERROR、abort 或 early return 时是否需要额外清理？
- 用户能否在 SQL、日志、pg_stat、wait event、EXPLAIN 或 profiler 中看到它的后果？

## 16. 本节小结

- auto_explain 是 executor hook 插桩的完整样本。
- 它在 Start 决定采样和 instrumentation，在 End 读取真实执行后的 QueryDesc 并输出计划。
- GUC 控制阈值、采样、节点指标和日志格式，也控制线上成本。
- 下一节继续看 hook 在 ERROR、递归和高频路径上的安全边界。

本节最后沉淀出的可迁移规律是：

```text
观测值不是事实本身；观测值是某个源码状态在某个生命周期边界上的投影。
```

读 PostgreSQL 内核时，始终把 runtime 现象压回 state、owner、cleanup 和 cost。
