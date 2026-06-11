# PostgreSQL wait_event_info 的设置与清理边界

## 课程定位

前置知识：已经知道 pg_stat 是累计统计；本节转向另一类可观测性：当前 backend 正在等待什么。

本节唯一主问题：

```text
pgstat_report_wait_start() / pgstat_report_wait_end() 如何把 backend 当前等待点暴露给 pg_stat_activity，为什么每条等待路径都必须保证清理？
```

核心矛盾：等待点上报必须足够轻，才能放在大量底层等待路径里；但一旦漏清理，用户看到的 wait event 就会变成错误诊断线索。

学完后应能判断一个等待路径应该在哪里设置 wait event、在哪里清理，以及 pg_stat_activity 中 stale wait event 可能来自哪类代码缺陷。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

上一节的 pg_stat 回答“历史累计了多少”。

wait event 回答“此刻正在等什么”。

两者的观测模型完全不同：pg_stat 允许延迟合并，wait event 需要在等待前后快速写入一个当前状态。

这也是 PostgreSQL 把 wait_event_info 放在 PGPROC 中的原因：其他 backend 可以通过 pg_stat_activity 读取当前 backend 的等待状态。

本节只讲 wait_event_info 的设置、清理和读取边界。

下一节会把这些 wait event 名称按执行器常见路径分类。

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
等待路径在进入阻塞前把 wait_event_info 写到 my_wait_event_info 指向的位置；普通 backend 初始化后该指针指向 MyProc->wait_event_info；等待结束后写 0 清理；pg_stat_activity 读取 PGPROC.wait_event_info 并把 class/id 解码成 wait_event_type 和 wait_event。
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
| 1 | `src/include/utils/wait_event.h` | 内联设置和清理函数；关注 pgstat_report_wait_start()、pgstat_report_wait_end() 与 my_wait_event_info。 |
| 2 | `src/backend/utils/activity/wait_event.c` | 本地 wait storage、PGPROC storage 切换、自定义 wait event 名称解析。 |
| 3 | `src/include/storage/proc.h` | PGPROC.wait_event_info 字段，pg_stat_activity 读取的共享状态。 |
| 4 | `src/backend/utils/adt/pgstatfuncs.c` | pg_stat_activity 和 pg_stat_get_backend_wait_event* 读取 wait_event_info。 |
| 5 | `src/backend/storage/ipc/waiteventset.c` | WaitEventSetWait() 统一等待入口，接收 wait_event_info 参数。 |
| 6 | `src/backend/storage/ipc/latch.c` | WaitLatch() 如何把 latch 等待映射到 WaitEventSetWait()。 |
| 7 | `src/backend/storage/lmgr/condition_variable.c` | ConditionVariableSleep() 使用传入的 wait_event_info 标记条件变量等待。 |
| 8 | `src/backend/storage/lmgr/proc.c` | ProcSleep() 和 ProcWaitForSignal() 中的 lock / signal 等待路径。 |

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

### 4.1. `my_wait_event_info`

当前进程写 wait event 的目标指针；初始指向本地变量，后续可切到 PGPROC 字段。

读这个状态时先问四个问题：

- 它是 backend-local、shared memory、文件状态，还是只在调用栈中短暂存在？
- 谁负责初始化它，初始化失败时有没有后续 cleanup 责任？
- 谁能读取或修改它，读取是否需要锁、pin、snapshot 或 owner？
- 它何时失效，失效后用户可见指标会如何表现？

在本节主问题中，`my_wait_event_info` 的价值是帮助判断一个观测现象到底落在哪个状态边界。

### 4.2. `local_my_wait_event_info`

MyProc 未就绪前的本地后备存储，允许早期代码安全调用 start/end。

读这个状态时先问四个问题：

- 它是 backend-local、shared memory、文件状态，还是只在调用栈中短暂存在？
- 谁负责初始化它，初始化失败时有没有后续 cleanup 责任？
- 谁能读取或修改它，读取是否需要锁、pin、snapshot 或 owner？
- 它何时失效，失效后用户可见指标会如何表现？

在本节主问题中，`local_my_wait_event_info` 的价值是帮助判断一个观测现象到底落在哪个状态边界。

### 4.3. `PGPROC.wait_event_info`

共享内存中的四字节当前等待状态，被 pg_stat_activity 无锁读取。

读这个状态时先问四个问题：

- 它是 backend-local、shared memory、文件状态，还是只在调用栈中短暂存在？
- 谁负责初始化它，初始化失败时有没有后续 cleanup 责任？
- 谁能读取或修改它，读取是否需要锁、pin、snapshot 或 owner？
- 它何时失效，失效后用户可见指标会如何表现？

在本节主问题中，`PGPROC.wait_event_info` 的价值是帮助判断一个观测现象到底落在哪个状态边界。

### 4.4. `wait event class`

高位 class 表示 LWLock、Lock、IO、Client、IPC、Timeout、Extension 等类型。

读这个状态时先问四个问题：

- 它是 backend-local、shared memory、文件状态，还是只在调用栈中短暂存在？
- 谁负责初始化它，初始化失败时有没有后续 cleanup 责任？
- 谁能读取或修改它，读取是否需要锁、pin、snapshot 或 owner？
- 它何时失效，失效后用户可见指标会如何表现？

在本节主问题中，`wait event class` 的价值是帮助判断一个观测现象到底落在哪个状态边界。

### 4.5. `wait event id`

低位 id 表示 class 内具体等待点，例如 DataFileRead、ClientRead 或 SafeSnapshot。

读这个状态时先问四个问题：

- 它是 backend-local、shared memory、文件状态，还是只在调用栈中短暂存在？
- 谁负责初始化它，初始化失败时有没有后续 cleanup 责任？
- 谁能读取或修改它，读取是否需要锁、pin、snapshot 或 owner？
- 它何时失效，失效后用户可见指标会如何表现？

在本节主问题中，`wait event id` 的价值是帮助判断一个观测现象到底落在哪个状态边界。

### 4.6. `pg_stat_activity wait_event_type/name`

SQL 层把 raw wait_event_info 解码后的用户可见结果。

读这个状态时先问四个问题：

- 它是 backend-local、shared memory、文件状态，还是只在调用栈中短暂存在？
- 谁负责初始化它，初始化失败时有没有后续 cleanup 责任？
- 谁能读取或修改它，读取是否需要锁、pin、snapshot 或 owner？
- 它何时失效，失效后用户可见指标会如何表现？

在本节主问题中，`pg_stat_activity wait_event_type/name` 的价值是帮助判断一个观测现象到底落在哪个状态边界。

本节最重要的不变量是：

```text
raw field 不是语义；field + lifecycle + writer + reader + cleanup 才是语义。
```

## 5. 主流程源码 walkthrough

下面按时间顺序走主链路。

每一步都回答“状态发生了什么变化”，而不是把源码读成函数名清单。

### 5.1. 步骤 1

backend 启动早期 my_wait_event_info 指向 local_my_wait_event_info。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.2. 步骤 2

PGPROC 初始化后，pgstat_set_wait_event_storage() 把指针切到 MyProc->wait_event_info。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.3. 步骤 3

某个等待路径准备阻塞，例如 WaitLatch()、ConditionVariableSleep() 或 ProcSleep()。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.4. 步骤 4

调用者把合适的 WAIT_EVENT_* 常量传给底层等待函数。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.5. 步骤 5

底层等待函数在进入 OS wait 或 latch wait 前调用 pgstat_report_wait_start()。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.6. 步骤 6

start 函数只做一次四字节 volatile 写入，不检查 track_activities。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.7. 步骤 7

其他 backend 查询 pg_stat_activity 时，通过 PGPROC 读取 raw_wait_event。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.8. 步骤 8

pgstat_get_wait_event_type() 根据 class 返回类型字符串。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.9. 步骤 9

pgstat_get_wait_event() 根据 class/id 返回具体名称。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.10. 步骤 10

等待返回后，路径调用 pgstat_report_wait_end() 把字段写为 0。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.11. 步骤 11

如果调用者忘记清理，backend 明明已经 active 运行，视图仍可能显示旧等待。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.12. 步骤 12

因此 wait event 的正确性边界不是锁语义，而是 start/end 成对出现。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

主链路可以压缩成：

```text
01. backend 启动早期 my_wait_event_info 指向 local_my_wait_event_info。
02. PGPROC 初始化后，pgstat_set_wait_event_storage() 把指针切到 MyProc->wait_event_info。
03. 某个等待路径准备阻塞，例如 WaitLatch()、ConditionVariableSleep() 或 ProcSleep()。
04. 调用者把合适的 WAIT_EVENT_* 常量传给底层等待函数。
05. 底层等待函数在进入 OS wait 或 latch wait 前调用 pgstat_report_wait_start()。
06. start 函数只做一次四字节 volatile 写入，不检查 track_activities。
07. 其他 backend 查询 pg_stat_activity 时，通过 PGPROC 读取 raw_wait_event。
08. pgstat_get_wait_event_type() 根据 class 返回类型字符串。
09. pgstat_get_wait_event() 根据 class/id 返回具体名称。
10. 等待返回后，路径调用 pgstat_report_wait_end() 把字段写为 0。
11. 如果调用者忘记清理，backend 明明已经 active 运行，视图仍可能显示旧等待。
12. 因此 wait event 的正确性边界不是锁语义，而是 start/end 成对出现。
```

## 6. 生命周期 / ownership / cleanup

生命周期是本节能否用于真实诊断的关键。

### 6.1. 创建

wait_event_info 字段随 PGPROC 创建，早期使用本地后备变量保证调用安全。

这里要特别区分两件事：

- 内存是否释放。
- 语义状态是否已经对其他 backend 或用户视图不可见。

很多观测问题并不是对象还在内存里，而是读取端还持有旧语义。

### 6.2. 持有

当前 backend 独占写自己的 wait_event_info，其他 backend 只读。

这里要特别区分两件事：

- 内存是否释放。
- 语义状态是否已经对其他 backend 或用户视图不可见。

很多观测问题并不是对象还在内存里，而是读取端还持有旧语义。

### 6.3. 推进

每次进入等待前写非零值，离开等待后写 0。

这里要特别区分两件事：

- 内存是否释放。
- 语义状态是否已经对其他 backend 或用户视图不可见。

很多观测问题并不是对象还在内存里，而是读取端还持有旧语义。

### 6.4. 失效

等待返回、ERROR 路径跳出或 storage reset 都应避免保留旧值。

这里要特别区分两件事：

- 内存是否释放。
- 语义状态是否已经对其他 backend 或用户视图不可见。

很多观测问题并不是对象还在内存里，而是读取端还持有旧语义。

### 6.5. 释放

backend 退出时 PGPROC 槽位归还，wait event 不作为长期历史保存。

这里要特别区分两件事：

- 内存是否释放。
- 语义状态是否已经对其他 backend 或用户视图不可见。

很多观测问题并不是对象还在内存里，而是读取端还持有旧语义。

ERROR / abort 路径上，不能依赖普通 C 调用栈逐层返回完成清理。

因此源码里要找的是 ResourceOwner、MemoryContext、PG_TRY/PG_FINALLY、事务回调或标准 executor teardown。

## 7. 正确性机制层次

本节的正确性来自多个层次叠加，而不是单个函数。

### 7.1. 机制 1

四字节读写天然适合轻量暴露当前等待状态。

把它放回运行期，可以得到两个判断：

- 正常路径中，它限制了状态何时能被推进或读取。
- 异常路径中，它决定用户看到的是空值、旧值、错误还是等待。

### 7.2. 机制 2

无锁读取意味着 pg_stat_activity 看到的是近似当前状态，不是事务一致 snapshot。

把它放回运行期，可以得到两个判断：

- 正常路径中，它限制了状态何时能被推进或读取。
- 异常路径中，它决定用户看到的是空值、旧值、错误还是等待。

### 7.3. 机制 3

start/end 不检查 GUC，是为了避免 hot path 上的分支成本。

把它放回运行期，可以得到两个判断：

- 正常路径中，它限制了状态何时能被推进或读取。
- 异常路径中，它决定用户看到的是空值、旧值、错误还是等待。

### 7.4. 机制 4

wait_event_info 只描述当前等待点，不描述等待已经持续多久。

把它放回运行期，可以得到两个判断：

- 正常路径中，它限制了状态何时能被推进或读取。
- 异常路径中，它决定用户看到的是空值、旧值、错误还是等待。

### 7.5. 机制 5

一个 wait event 名称必须回到具体源码等待点解释。

把它放回运行期，可以得到两个判断：

- 正常路径中，它限制了状态何时能被推进或读取。
- 异常路径中，它决定用户看到的是空值、旧值、错误还是等待。

### 7.6. 机制 6

清理边界错误会直接污染诊断，而不是只影响统计精度。

把它放回运行期，可以得到两个判断：

- 正常路径中，它限制了状态何时能被推进或读取。
- 异常路径中，它决定用户看到的是空值、旧值、错误还是等待。

不要把这些机制抽象成“为了安全”。

更准确的说法是：它们把低成本 hot path 和可解释的观测边界同时保留下来。

## 8. 错误路径 / 异常路径 / fallback

下面这些场景不是边角料，而是现场排障最常遇到的误判来源。

### 8.1. 漏掉 wait_end

backend 运行中仍显示旧 wait_event，排查时会把 CPU 或锁路径误判成旧等待。

排查时不要先改参数。

先问：当前现象是写入端没有推进，读取端没有刷新，还是观察工具选错了层次。

### 8.2. 等待前设置过早

如果设置后又执行大量非等待逻辑，视图会夸大等待时间。

排查时不要先改参数。

先问：当前现象是写入端没有推进，读取端没有刷新，还是观察工具选错了层次。

### 8.3. 等待后清理过晚

清理延迟会让采样监控把非等待时间归到等待事件。

排查时不要先改参数。

先问：当前现象是写入端没有推进，读取端没有刷新，还是观察工具选错了层次。

### 8.4. 传入泛化事件

过宽的 wait event 名称不能帮助用户从视图回到源码点。

排查时不要先改参数。

先问：当前现象是写入端没有推进，读取端没有刷新，还是观察工具选错了层次。

### 8.5. 把 wait_event 当阻塞者

wait_event 只说明等待类别；阻塞链仍要结合 pg_locks 或 pg_blocking_pids()。

排查时不要先改参数。

先问：当前现象是写入端没有推进，读取端没有刷新，还是观察工具选错了层次。

fallback 的设计目标通常不是让结果更漂亮，而是让系统在异常路径上保持可恢复、可清理、可解释。

## 9. 成本、资源与跨模块传播

可观测性代码如果放错层次，会把诊断工具变成性能问题的一部分。

### 9.1. 成本维度 1

start/end 是四字节写入，成本足够低才能放在 I/O、Lock、LWLock 等底层路径。

这个成本通常会继续传播到相邻模块：

- executor hot path 是否多一次分支、计时或锁。
- shared memory 是否出现新的争用点。
- 日志、视图或监控采样是否放大已有压力。

### 9.2. 成本维度 2

名称解析发生在读取端，不发生在等待 hot path。

这个成本通常会继续传播到相邻模块：

- executor hot path 是否多一次分支、计时或锁。
- shared memory 是否出现新的争用点。
- 日志、视图或监控采样是否放大已有压力。

### 9.3. 成本维度 3

无锁读取降低成本，但牺牲跨列一致性。

这个成本通常会继续传播到相邻模块：

- executor hot path 是否多一次分支、计时或锁。
- shared memory 是否出现新的争用点。
- 日志、视图或监控采样是否放大已有压力。

### 9.4. 成本维度 4

等待点越细，诊断越清楚，但维护 wait event 名称和分类的成本越高。

这个成本通常会继续传播到相邻模块：

- executor hot path 是否多一次分支、计时或锁。
- shared memory 是否出现新的争用点。
- 日志、视图或监控采样是否放大已有压力。

### 9.5. 成本维度 5

扩展 wait event 需要共享 hash 和 LWLock，只应在注册时承担成本，而不是每次等待都注册。

这个成本通常会继续传播到相邻模块：

- executor hot path 是否多一次分支、计时或锁。
- shared memory 是否出现新的争用点。
- 日志、视图或监控采样是否放大已有压力。

因此课程里的实验默认是短时、可控、可回滚的，不建议把强制刷新或高频采样作为长期配置。

## 10. 观测与诊断入口

观测入口要按问题层次选择。

### 10.1. `pg_stat_activity.wait_event_type`

从 class 解码出来的等待类型。

使用这个入口时要同时记录：

- 采样时间。
- backend pid 或 query_id。
- 是否处于同一事务、同一 backend 或同一执行。

### 10.2. `pg_stat_activity.wait_event`

class 内具体等待名称。

使用这个入口时要同时记录：

- 采样时间。
- backend pid 或 query_id。
- 是否处于同一事务、同一 backend 或同一执行。

### 10.3. `pg_stat_get_backend_wait_event_type()`

按 backend 编号读取等待类型。

使用这个入口时要同时记录：

- 采样时间。
- backend pid 或 query_id。
- 是否处于同一事务、同一 backend 或同一执行。

### 10.4. `pg_stat_get_backend_wait_event()`

按 backend 编号读取等待名称。

使用这个入口时要同时记录：

- 采样时间。
- backend pid 或 query_id。
- 是否处于同一事务、同一 backend 或同一执行。

### 10.5. `pg_get_wait_events()`

列出内置和自定义 wait event 名称及说明。

使用这个入口时要同时记录：

- 采样时间。
- backend pid 或 query_id。
- 是否处于同一事务、同一 backend 或同一执行。

### 10.6. `gdb 打印 MyProc->wait_event_info`

在源码调试中直接观察 raw wait event 字段。

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

### 12.1. 观察锁等待设置与清理

实验步骤：

```sql
-- session 1
BEGIN;
LOCK TABLE t_wait_demo IN ACCESS EXCLUSIVE MODE;
-- session 2
SELECT * FROM t_wait_demo;
```

在第三个会话查询 pg_stat_activity，看到 session 2 的 Lock 类型等待；session 1 提交后该字段应清理。

实验后回到源码，至少确认一个入口函数和一个状态字段。

### 12.2. 观察 ClientRead

实验步骤：

```sql
-- psql 中开启事务但暂不发送下一条命令
BEGIN;
SELECT pg_backend_pid();
```

backend 等待客户端下一条消息时可能显示 ClientRead，这不是数据库内部锁。

实验后回到源码，至少确认一个入口函数和一个状态字段。

### 12.3. 列出等待事件

实验步骤：

```sql
SELECT type, name FROM pg_get_wait_events() WHERE type IN ('Lock', 'IO', 'IPC', 'Client') ORDER BY 1, 2 LIMIT 30;
```

这一步把用户可见名称与源码里的等待分类对应起来。

实验后回到源码，至少确认一个入口函数和一个状态字段。

### 12.4. 源码断点

实验步骤：

```gdb
break pgstat_report_wait_start
break pgstat_report_wait_end
commands
bt
continue
end
```

用 gdb 查看哪些等待路径进入 start/end，确认 wait event 的调用点。

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

1. 为什么 wait event 不通过累计统计系统上报？

回答时必须至少引用一个源码文件和一个运行期现象。

2. 为什么 start/end 不检查 pgstat_track_activities？

回答时必须至少引用一个源码文件和一个运行期现象。

3. 无锁读取 wait_event_info 会带来哪些解释限制？

回答时必须至少引用一个源码文件和一个运行期现象。

4. 一个等待点应该命名为算法阶段，还是底层资源类型？

回答时必须至少引用一个源码文件和一个运行期现象。

5. 如何判断 wait_event 是否是 stale 值？

回答时必须至少引用一个源码文件和一个运行期现象。

6. wait_event 与 pg_locks 的信息边界在哪里？

回答时必须至少引用一个源码文件和一个运行期现象。

讨论题不是开放闲聊，而是训练把源码机制压缩成可迁移判断。

## 15. 源码复盘清单

如果需要快速复盘，本节可以按下面的源码锚点重新走一遍。

### 15.1. `pgstat_report_wait_start()`

先定位 `pgstat_report_wait_start()` 的调用者，不要从文件顶部线性读到底。

然后确认它如何影响 `my_wait_event_info`。

本锚点对应的运行期动作是：backend 启动早期 my_wait_event_info 指向 local_my_wait_event_info。

复盘时记录三个问题：

- 这个函数在正常路径上推进了什么状态？
- 它在 ERROR、abort 或 early return 时是否需要额外清理？
- 用户能否在 SQL、日志、pg_stat、wait event、EXPLAIN 或 profiler 中看到它的后果？

### 15.2. `pgstat_report_wait_end()`

先定位 `pgstat_report_wait_end()` 的调用者，不要从文件顶部线性读到底。

然后确认它如何影响 `local_my_wait_event_info`。

本锚点对应的运行期动作是：PGPROC 初始化后，pgstat_set_wait_event_storage() 把指针切到 MyProc->wait_event_info。

复盘时记录三个问题：

- 这个函数在正常路径上推进了什么状态？
- 它在 ERROR、abort 或 early return 时是否需要额外清理？
- 用户能否在 SQL、日志、pg_stat、wait event、EXPLAIN 或 profiler 中看到它的后果？

### 15.3. `pgstat_set_wait_event_storage()`

先定位 `pgstat_set_wait_event_storage()` 的调用者，不要从文件顶部线性读到底。

然后确认它如何影响 `PGPROC.wait_event_info`。

本锚点对应的运行期动作是：某个等待路径准备阻塞，例如 WaitLatch()、ConditionVariableSleep() 或 ProcSleep()。

复盘时记录三个问题：

- 这个函数在正常路径上推进了什么状态？
- 它在 ERROR、abort 或 early return 时是否需要额外清理？
- 用户能否在 SQL、日志、pg_stat、wait event、EXPLAIN 或 profiler 中看到它的后果？

### 15.4. `WaitEventSetWait()`

先定位 `WaitEventSetWait()` 的调用者，不要从文件顶部线性读到底。

然后确认它如何影响 `wait event class`。

本锚点对应的运行期动作是：调用者把合适的 WAIT_EVENT_* 常量传给底层等待函数。

复盘时记录三个问题：

- 这个函数在正常路径上推进了什么状态？
- 它在 ERROR、abort 或 early return 时是否需要额外清理？
- 用户能否在 SQL、日志、pg_stat、wait event、EXPLAIN 或 profiler 中看到它的后果？

### 15.5. `WaitLatch()`

先定位 `WaitLatch()` 的调用者，不要从文件顶部线性读到底。

然后确认它如何影响 `wait event id`。

本锚点对应的运行期动作是：底层等待函数在进入 OS wait 或 latch wait 前调用 pgstat_report_wait_start()。

复盘时记录三个问题：

- 这个函数在正常路径上推进了什么状态？
- 它在 ERROR、abort 或 early return 时是否需要额外清理？
- 用户能否在 SQL、日志、pg_stat、wait event、EXPLAIN 或 profiler 中看到它的后果？

### 15.6. `ConditionVariableSleep()`

先定位 `ConditionVariableSleep()` 的调用者，不要从文件顶部线性读到底。

然后确认它如何影响 `pg_stat_activity wait_event_type/name`。

本锚点对应的运行期动作是：start 函数只做一次四字节 volatile 写入，不检查 track_activities。

复盘时记录三个问题：

- 这个函数在正常路径上推进了什么状态？
- 它在 ERROR、abort 或 early return 时是否需要额外清理？
- 用户能否在 SQL、日志、pg_stat、wait event、EXPLAIN 或 profiler 中看到它的后果？

## 16. 本节小结

- wait_event_info 是当前等待状态，不是历史统计。
- 它用轻量 start/end 协议把 backend 当前等待点暴露给 pg_stat_activity。
- 诊断时要把名称反查到源码等待点，并确认等待已经正确清理。
- 下一节会按 executor 常见慢 SQL 场景建立 wait event 分类地图。

本节最后沉淀出的可迁移规律是：

```text
观测值不是事实本身；观测值是某个源码状态在某个生命周期边界上的投影。
```

读 PostgreSQL 内核时，始终把 runtime 现象压回 state、owner、cleanup 和 cost。
