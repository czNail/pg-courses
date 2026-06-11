# PostgreSQL Executor hook 链与扩展插桩边界

## 课程定位

前置知识：已经理解执行器生命周期和可观测指标入口；本节从内核提供的 hook 点看扩展如何包裹 executor。

本节唯一主问题：

```text
ExecutorStart_hook、ExecutorRun_hook、ExecutorFinish_hook、ExecutorEnd_hook 如何形成扩展插桩入口，hook 链为什么必须保存并调用 previous hook？
```

核心矛盾：扩展需要在不修改核心代码的情况下观测执行器，但多个扩展共享全局 hook 指针，任何一个扩展不遵守链式协议都会破坏其他扩展或标准 executor 语义。

学完后应能判断一个 executor hook 扩展应该在哪个生命周期点做事、如何保存 previous hook、何时调用 standard_*，以及哪些行为不适合放在 executor hook。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

前面几节讲的是执行器自身状态和可观测输出。

hook 是扩展生态进入这些状态边界的方式。

它不是事件总线，也不是自动多播。

PostgreSQL 采用的是全局函数指针链：后加载的扩展接管入口，并负责调用之前的 hook 或标准实现。

这给扩展极大自由，也带来纪律要求。

本节只讲 executor 四个主 hook 的链式边界。

下一节会用 auto_explain 作为完整例子，看它如何捕获慢查询计划。

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
ExecutorStart/Run/Finish/End 在进入 standard_* 前先检查对应全局 hook 指针；扩展加载时保存当前 hook 到 prev_*，再把全局 hook 改成自己的函数；自己的函数完成插桩后必须调用 prev_* 或 standard_*，从而把多个扩展串成一条链。
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
| 1 | `src/include/executor/executor.h` | ExecutorStart_hook_type、ExecutorRun_hook_type、ExecutorFinish_hook_type、ExecutorEnd_hook_type 声明。 |
| 2 | `src/backend/executor/execMain.c` | 四个 hook 全局变量和 ExecutorStart/Run/Finish/End 的分派逻辑。 |
| 3 | `contrib/auto_explain/auto_explain.c` | 保存 prev hook、安装 hook、调用 previous 或 standard implementation 的真实扩展示例。 |
| 4 | `src/backend/executor/instrument.c` | hook 常操作的 instrumentation 边界。 |
| 5 | `src/backend/commands/explain.c` | hook 扩展可能调用的 ExplainPrintPlan() 等格式化入口。 |

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

### 4.1. `ExecutorStart_hook`

启动阶段 hook，适合修改 QueryDesc instrumentation options 或记录开始状态。

读这个状态时先问四个问题：

- 它是 backend-local、shared memory、文件状态，还是只在调用栈中短暂存在？
- 谁负责初始化它，初始化失败时有没有后续 cleanup 责任？
- 谁能读取或修改它，读取是否需要锁、pin、snapshot 或 owner？
- 它何时失效，失效后用户可见指标会如何表现？

在本节主问题中，`ExecutorStart_hook` 的价值是帮助判断一个观测现象到底落在哪个状态边界。

### 4.2. `ExecutorRun_hook`

运行阶段 hook，包裹 tuple 拉取和 DestReceiver 输出。

读这个状态时先问四个问题：

- 它是 backend-local、shared memory、文件状态，还是只在调用栈中短暂存在？
- 谁负责初始化它，初始化失败时有没有后续 cleanup 责任？
- 谁能读取或修改它，读取是否需要锁、pin、snapshot 或 owner？
- 它何时失效，失效后用户可见指标会如何表现？

在本节主问题中，`ExecutorRun_hook` 的价值是帮助判断一个观测现象到底落在哪个状态边界。

### 4.3. `ExecutorFinish_hook`

副作用 drain 阶段 hook，观察触发器和 queued side effect 收尾。

读这个状态时先问四个问题：

- 它是 backend-local、shared memory、文件状态，还是只在调用栈中短暂存在？
- 谁负责初始化它，初始化失败时有没有后续 cleanup 责任？
- 谁能读取或修改它，读取是否需要锁、pin、snapshot 或 owner？
- 它何时失效，失效后用户可见指标会如何表现？

在本节主问题中，`ExecutorFinish_hook` 的价值是帮助判断一个观测现象到底落在哪个状态边界。

### 4.4. `ExecutorEnd_hook`

清理前 hook，仍可访问 QueryDesc / EState / PlanState 以输出观测结果。

读这个状态时先问四个问题：

- 它是 backend-local、shared memory、文件状态，还是只在调用栈中短暂存在？
- 谁负责初始化它，初始化失败时有没有后续 cleanup 责任？
- 谁能读取或修改它，读取是否需要锁、pin、snapshot 或 owner？
- 它何时失效，失效后用户可见指标会如何表现？

在本节主问题中，`ExecutorEnd_hook` 的价值是帮助判断一个观测现象到底落在哪个状态边界。

### 4.5. `prev_Executor*`

扩展保存的前一个 hook 指针，是链式调用的关键。

读这个状态时先问四个问题：

- 它是 backend-local、shared memory、文件状态，还是只在调用栈中短暂存在？
- 谁负责初始化它，初始化失败时有没有后续 cleanup 责任？
- 谁能读取或修改它，读取是否需要锁、pin、snapshot 或 owner？
- 它何时失效，失效后用户可见指标会如何表现？

在本节主问题中，`prev_Executor*` 的价值是帮助判断一个观测现象到底落在哪个状态边界。

### 4.6. `standard_Executor*`

没有 previous hook 时必须调用的核心实现。

读这个状态时先问四个问题：

- 它是 backend-local、shared memory、文件状态，还是只在调用栈中短暂存在？
- 谁负责初始化它，初始化失败时有没有后续 cleanup 责任？
- 谁能读取或修改它，读取是否需要锁、pin、snapshot 或 owner？
- 它何时失效，失效后用户可见指标会如何表现？

在本节主问题中，`standard_Executor*` 的价值是帮助判断一个观测现象到底落在哪个状态边界。

本节最重要的不变量是：

```text
raw field 不是语义；field + lifecycle + writer + reader + cleanup 才是语义。
```

## 5. 主流程源码 walkthrough

下面按时间顺序走主链路。

每一步都回答“状态发生了什么变化”，而不是把源码读成函数名清单。

### 5.1. 步骤 1

扩展模块加载进入 _PG_init()。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.2. 步骤 2

扩展读取当前 ExecutorStart_hook，保存到 prev_ExecutorStart。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.3. 步骤 3

扩展把 ExecutorStart_hook 改成自己的 explain_ExecutorStart 或类似函数。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.4. 步骤 4

Run、Finish、End hook 重复同样保存和替换动作。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.5. 步骤 5

用户执行 SQL 时，ExecutorStart() 先报告 query id。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.6. 步骤 6

ExecutorStart() 检查 ExecutorStart_hook 是否非空。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.7. 步骤 7

如果有 hook，调用当前链头；否则调用 standard_ExecutorStart()。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.8. 步骤 8

扩展 hook 可以在调用 previous 前修改 QueryDesc 的 instrumentation options。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.9. 步骤 9

扩展随后调用 prev_ExecutorStart 或 standard_ExecutorStart。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.10. 步骤 10

ExecutorRun/Finish/End 使用同样链式分派。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.11. 步骤 11

多个扩展按加载顺序形成嵌套调用链，最后落到 standard implementation。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.12. 步骤 12

任何扩展若不调用 previous 或 standard，都会截断 executor 生命周期。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

主链路可以压缩成：

```text
01. 扩展模块加载进入 _PG_init()。
02. 扩展读取当前 ExecutorStart_hook，保存到 prev_ExecutorStart。
03. 扩展把 ExecutorStart_hook 改成自己的 explain_ExecutorStart 或类似函数。
04. Run、Finish、End hook 重复同样保存和替换动作。
05. 用户执行 SQL 时，ExecutorStart() 先报告 query id。
06. ExecutorStart() 检查 ExecutorStart_hook 是否非空。
07. 如果有 hook，调用当前链头；否则调用 standard_ExecutorStart()。
08. 扩展 hook 可以在调用 previous 前修改 QueryDesc 的 instrumentation options。
09. 扩展随后调用 prev_ExecutorStart 或 standard_ExecutorStart。
10. ExecutorRun/Finish/End 使用同样链式分派。
11. 多个扩展按加载顺序形成嵌套调用链，最后落到 standard implementation。
12. 任何扩展若不调用 previous 或 standard，都会截断 executor 生命周期。
```

## 6. 生命周期 / ownership / cleanup

生命周期是本节能否用于真实诊断的关键。

### 6.1. 安装

hook 在 backend 加载扩展时安装，影响当前 backend 后续 executor 调用。

这里要特别区分两件事：

- 内存是否释放。
- 语义状态是否已经对其他 backend 或用户视图不可见。

很多观测问题并不是对象还在内存里，而是读取端还持有旧语义。

### 6.2. 持有

全局 hook 指针由进程持有，不是共享内存协议。

这里要特别区分两件事：

- 内存是否释放。
- 语义状态是否已经对其他 backend 或用户视图不可见。

很多观测问题并不是对象还在内存里，而是读取端还持有旧语义。

### 6.3. 调用

每次 Executor* 入口按当前 hook 指针分派。

这里要特别区分两件事：

- 内存是否释放。
- 语义状态是否已经对其他 backend 或用户视图不可见。

很多观测问题并不是对象还在内存里，而是读取端还持有旧语义。

### 6.4. 串联

previous hook 指针把多个扩展串成嵌套链。

这里要特别区分两件事：

- 内存是否释放。
- 语义状态是否已经对其他 backend 或用户视图不可见。

很多观测问题并不是对象还在内存里，而是读取端还持有旧语义。

### 6.5. 结束

普通会话生命周期中 hook 通常随 backend 退出消失。

这里要特别区分两件事：

- 内存是否释放。
- 语义状态是否已经对其他 backend 或用户视图不可见。

很多观测问题并不是对象还在内存里，而是读取端还持有旧语义。

ERROR / abort 路径上，不能依赖普通 C 调用栈逐层返回完成清理。

因此源码里要找的是 ResourceOwner、MemoryContext、PG_TRY/PG_FINALLY、事务回调或标准 executor teardown。

## 7. 正确性机制层次

本节的正确性来自多个层次叠加，而不是单个函数。

### 7.1. 机制 1

hook 函数签名必须与 executor.h 声明一致。

把它放回运行期，可以得到两个判断：

- 正常路径中，它限制了状态何时能被推进或读取。
- 异常路径中，它决定用户看到的是空值、旧值、错误还是等待。

### 7.2. 机制 2

扩展必须保存 previous hook，否则无法把链继续向下传递。

把它放回运行期，可以得到两个判断：

- 正常路径中，它限制了状态何时能被推进或读取。
- 异常路径中，它决定用户看到的是空值、旧值、错误还是等待。

### 7.3. 机制 3

没有 previous hook 时必须调用 standard_*，否则核心 executor 不会运行。

把它放回运行期，可以得到两个判断：

- 正常路径中，它限制了状态何时能被推进或读取。
- 异常路径中，它决定用户看到的是空值、旧值、错误还是等待。

### 7.4. 机制 4

Start 中修改 instrumentation options 必须发生在 standard_ExecutorStart() 创建 PlanState 前。

把它放回运行期，可以得到两个判断：

- 正常路径中，它限制了状态何时能被推进或读取。
- 异常路径中，它决定用户看到的是空值、旧值、错误还是等待。

### 7.5. 机制 5

End 中输出计划必须发生在 standard_ExecutorEnd() 释放 EState 前。

把它放回运行期，可以得到两个判断：

- 正常路径中，它限制了状态何时能被推进或读取。
- 异常路径中，它决定用户看到的是空值、旧值、错误还是等待。

### 7.6. 机制 6

hook 不应改变 QueryDesc 语义，除非扩展明确承担行为修改责任。

把它放回运行期，可以得到两个判断：

- 正常路径中，它限制了状态何时能被推进或读取。
- 异常路径中，它决定用户看到的是空值、旧值、错误还是等待。

不要把这些机制抽象成“为了安全”。

更准确的说法是：它们把低成本 hot path 和可解释的观测边界同时保留下来。

## 8. 错误路径 / 异常路径 / fallback

下面这些场景不是边角料，而是现场排障最常遇到的误判来源。

### 8.1. 忘记调用 previous

后加载扩展会屏蔽先加载扩展，甚至屏蔽标准 executor。

排查时不要先改参数。

先问：当前现象是写入端没有推进，读取端没有刷新，还是观察工具选错了层次。

### 8.2. End 后访问 EState

standard_ExecutorEnd() 之后很多 executor 状态已释放。

排查时不要先改参数。

先问：当前现象是写入端没有推进，读取端没有刷新，还是观察工具选错了层次。

### 8.3. Run hook 不处理异常

递归计数或临时状态可能在 ERROR 后残留。

排查时不要先改参数。

先问：当前现象是写入端没有推进，读取端没有刷新，还是观察工具选错了层次。

### 8.4. 在 Start 后才打开节点 instrumentation

PlanState 的 instrumentation 已经分配完成，可能来不及生效。

排查时不要先改参数。

先问：当前现象是写入端没有推进，读取端没有刷新，还是观察工具选错了层次。

### 8.5. 把 planner 问题放 executor hook

executor hook 看的是已计划好的 QueryDesc，不适合重写优化器决策。

排查时不要先改参数。

先问：当前现象是写入端没有推进，读取端没有刷新，还是观察工具选错了层次。

fallback 的设计目标通常不是让结果更漂亮，而是让系统在异常路径上保持可恢复、可清理、可解释。

## 9. 成本、资源与跨模块传播

可观测性代码如果放错层次，会把诊断工具变成性能问题的一部分。

### 9.1. 成本维度 1

每次 Executor* 入口都会经过 hook 指针分支。

这个成本通常会继续传播到相邻模块：

- executor hot path 是否多一次分支、计时或锁。
- shared memory 是否出现新的争用点。
- 日志、视图或监控采样是否放大已有压力。

### 9.2. 成本维度 2

禁用态 hook 应尽量只做条件判断，然后快速调用 previous。

这个成本通常会继续传播到相邻模块：

- executor hot path 是否多一次分支、计时或锁。
- shared memory 是否出现新的争用点。
- 日志、视图或监控采样是否放大已有压力。

### 9.3. 成本维度 3

Start hook 打开 instrumentation 会增加每个节点执行成本。

这个成本通常会继续传播到相邻模块：

- executor hot path 是否多一次分支、计时或锁。
- shared memory 是否出现新的争用点。
- 日志、视图或监控采样是否放大已有压力。

### 9.4. 成本维度 4

End hook 格式化计划和写日志可能成为慢查询后的额外成本。

这个成本通常会继续传播到相邻模块：

- executor hot path 是否多一次分支、计时或锁。
- shared memory 是否出现新的争用点。
- 日志、视图或监控采样是否放大已有压力。

### 9.5. 成本维度 5

多个扩展 hook 嵌套会增加调用深度和状态交互复杂度。

这个成本通常会继续传播到相邻模块：

- executor hot path 是否多一次分支、计时或锁。
- shared memory 是否出现新的争用点。
- 日志、视图或监控采样是否放大已有压力。

因此课程里的实验默认是短时、可控、可回滚的，不建议把强制刷新或高频采样作为长期配置。

## 10. 观测与诊断入口

观测入口要按问题层次选择。

### 10.1. `auto_explain`

contrib 中最典型的 executor hook 插桩示例。

使用这个入口时要同时记录：

- 采样时间。
- backend pid 或 query_id。
- 是否处于同一事务、同一 backend 或同一执行。

### 10.2. `gdb 打印 ExecutorStart_hook`

确认当前 backend hook 链头是谁。

使用这个入口时要同时记录：

- 采样时间。
- backend pid 或 query_id。
- 是否处于同一事务、同一 backend 或同一执行。

### 10.3. `LOAD 顺序`

改变多个 hook 扩展的嵌套顺序。

使用这个入口时要同时记录：

- 采样时间。
- backend pid 或 query_id。
- 是否处于同一事务、同一 backend 或同一执行。

### 10.4. `日志`

hook 扩展通常在 End 或 ERROR-safe 边界输出诊断。

使用这个入口时要同时记录：

- 采样时间。
- backend pid 或 query_id。
- 是否处于同一事务、同一 backend 或同一执行。

### 10.5. `QueryDesc.instrument_options`

Start hook 常修改的观测开关。

使用这个入口时要同时记录：

- 采样时间。
- backend pid 或 query_id。
- 是否处于同一事务、同一 backend 或同一执行。

### 10.6. `query_instr_options`

query-level runtime 计时入口。

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

### 12.1. 观察 auto_explain 安装 hook

实验步骤：

```sql
LOAD 'auto_explain';
SET auto_explain.log_min_duration = 0;
SELECT 1;
```

用日志验证 hook 已包裹 executor 生命周期。

实验后回到源码，至少确认一个入口函数和一个状态字段。

### 12.2. 源码跟读 hook 分派

实验步骤：

```bash
rg -n "ExecutorStart_hook|standard_ExecutorStart" /home/highgo/postgres/src/backend/executor/execMain.c /home/highgo/postgres/src/include/executor/executor.h
```

确认核心入口只认识全局函数指针。

实验后回到源码，至少确认一个入口函数和一个状态字段。

### 12.3. 跟读 previous hook

实验步骤：

```bash
rg -n "prev_ExecutorStart|prev_ExecutorRun|prev_ExecutorFinish|prev_ExecutorEnd" /home/highgo/postgres/contrib/auto_explain/auto_explain.c
```

理解扩展如何保存和调用链上一个函数。

实验后回到源码，至少确认一个入口函数和一个状态字段。

### 12.4. 判断 hook 位置

实验步骤：

```sql
-- Start: 设置 instrumentation
-- Run: 包裹执行
-- Finish: 副作用收尾
-- End: 输出并释放前观察
```

按生命周期选择插桩点，而不是把所有逻辑塞进 Run。

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

1. 为什么 executor hook 不是注册多个 callback 的列表？

回答时必须至少引用一个源码文件和一个运行期现象。

2. 加载顺序如何影响多个 hook 扩展的嵌套？

回答时必须至少引用一个源码文件和一个运行期现象。

3. Start hook 和 End hook 分别适合做什么？

回答时必须至少引用一个源码文件和一个运行期现象。

4. 为什么 End hook 输出计划要早于 standard_ExecutorEnd()？

回答时必须至少引用一个源码文件和一个运行期现象。

5. 什么问题应该放 planner hook，而不是 executor hook？

回答时必须至少引用一个源码文件和一个运行期现象。

6. 扩展如何证明自己没有改变 executor 语义？

回答时必须至少引用一个源码文件和一个运行期现象。

讨论题不是开放闲聊，而是训练把源码机制压缩成可迁移判断。

## 15. 源码复盘清单

如果需要快速复盘，本节可以按下面的源码锚点重新走一遍。

### 15.1. `ExecutorStart_hook`

先定位 `ExecutorStart_hook` 的调用者，不要从文件顶部线性读到底。

然后确认它如何影响 `ExecutorStart_hook`。

本锚点对应的运行期动作是：扩展模块加载进入 _PG_init()。

复盘时记录三个问题：

- 这个函数在正常路径上推进了什么状态？
- 它在 ERROR、abort 或 early return 时是否需要额外清理？
- 用户能否在 SQL、日志、pg_stat、wait event、EXPLAIN 或 profiler 中看到它的后果？

### 15.2. `ExecutorRun_hook`

先定位 `ExecutorRun_hook` 的调用者，不要从文件顶部线性读到底。

然后确认它如何影响 `ExecutorRun_hook`。

本锚点对应的运行期动作是：扩展读取当前 ExecutorStart_hook，保存到 prev_ExecutorStart。

复盘时记录三个问题：

- 这个函数在正常路径上推进了什么状态？
- 它在 ERROR、abort 或 early return 时是否需要额外清理？
- 用户能否在 SQL、日志、pg_stat、wait event、EXPLAIN 或 profiler 中看到它的后果？

### 15.3. `ExecutorFinish_hook`

先定位 `ExecutorFinish_hook` 的调用者，不要从文件顶部线性读到底。

然后确认它如何影响 `ExecutorFinish_hook`。

本锚点对应的运行期动作是：扩展把 ExecutorStart_hook 改成自己的 explain_ExecutorStart 或类似函数。

复盘时记录三个问题：

- 这个函数在正常路径上推进了什么状态？
- 它在 ERROR、abort 或 early return 时是否需要额外清理？
- 用户能否在 SQL、日志、pg_stat、wait event、EXPLAIN 或 profiler 中看到它的后果？

### 15.4. `ExecutorEnd_hook`

先定位 `ExecutorEnd_hook` 的调用者，不要从文件顶部线性读到底。

然后确认它如何影响 `ExecutorEnd_hook`。

本锚点对应的运行期动作是：Run、Finish、End hook 重复同样保存和替换动作。

复盘时记录三个问题：

- 这个函数在正常路径上推进了什么状态？
- 它在 ERROR、abort 或 early return 时是否需要额外清理？
- 用户能否在 SQL、日志、pg_stat、wait event、EXPLAIN 或 profiler 中看到它的后果？

### 15.5. `standard_ExecutorStart()`

先定位 `standard_ExecutorStart()` 的调用者，不要从文件顶部线性读到底。

然后确认它如何影响 `prev_Executor*`。

本锚点对应的运行期动作是：用户执行 SQL 时，ExecutorStart() 先报告 query id。

复盘时记录三个问题：

- 这个函数在正常路径上推进了什么状态？
- 它在 ERROR、abort 或 early return 时是否需要额外清理？
- 用户能否在 SQL、日志、pg_stat、wait event、EXPLAIN 或 profiler 中看到它的后果？

### 15.6. `prev_ExecutorStart`

先定位 `prev_ExecutorStart` 的调用者，不要从文件顶部线性读到底。

然后确认它如何影响 `standard_Executor*`。

本锚点对应的运行期动作是：ExecutorStart() 检查 ExecutorStart_hook 是否非空。

复盘时记录三个问题：

- 这个函数在正常路径上推进了什么状态？
- 它在 ERROR、abort 或 early return 时是否需要额外清理？
- 用户能否在 SQL、日志、pg_stat、wait event、EXPLAIN 或 profiler 中看到它的后果？

### 15.7. `ExecutorStart_hook`

先定位 `ExecutorStart_hook` 的调用者，不要从文件顶部线性读到底。

然后确认它如何影响 `ExecutorStart_hook`。

本锚点对应的运行期动作是：如果有 hook，调用当前链头；否则调用 standard_ExecutorStart()。

复盘时记录三个问题：

- 这个函数在正常路径上推进了什么状态？
- 它在 ERROR、abort 或 early return 时是否需要额外清理？
- 用户能否在 SQL、日志、pg_stat、wait event、EXPLAIN 或 profiler 中看到它的后果？

## 16. 本节小结

- executor hook 是扩展插桩入口，也是全局函数指针链。
- previous hook 协议是扩展共存的核心纪律。
- Start、Run、Finish、End 对应不同生命周期边界，不能混用。
- 下一节用 auto_explain 具体说明 hook 如何捕获慢查询计划。

本节最后沉淀出的可迁移规律是：

```text
观测值不是事实本身；观测值是某个源码状态在某个生命周期边界上的投影。
```

读 PostgreSQL 内核时，始终把 runtime 现象压回 state、owner、cleanup 和 cost。
