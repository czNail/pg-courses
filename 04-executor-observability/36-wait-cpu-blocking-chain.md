# PostgreSQL 等待、CPU 与阻塞链的区分

## 课程定位

前置知识：已经能把 wait_event_type 映射到大致源码路径；本节解决慢 SQL 诊断中最容易混在一起的三类原因。

本节唯一主问题：

```text
如何把 pg_stat_activity、pg_locks、wait event、EXPLAIN ANALYZE 和系统 profiler 结合起来，判断慢 SQL 是锁等待、I/O、CPU 还是上游客户端背压？
```

核心矛盾：用户只看到一次 SQL 响应时间，但数据库内部时间可能分散在等待、CPU、阻塞链、客户端通信和计划节点执行之间。

学完后应能把慢 SQL 初步归因到等待、CPU、阻塞链、I/O 或客户端，并知道每种归因需要什么证据。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

上一节把 wait event 名称映射到源码模块。

但真实诊断不会只停在一个 wait_event 字段。

一个慢 SQL 可能先等锁，然后跑 CPU，再向慢客户端写大量结果。

如果只看一次 pg_stat_activity 采样，很容易把最后一个阶段当成全部原因。

本节建立的是诊断顺序：先判断是否正在等待，再判断等待是否有 blocker，再用 EXPLAIN 和 profiler 区分 CPU 与 I/O。

本节不展开每类执行节点算法，重点是证据如何组合。

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
慢 SQL 诊断先把总耗时拆成当前状态、等待状态、阻塞关系、计划节点实际耗时和系统 CPU 栈；wait_event 非空说明采样点在等待，Lock 等待需要阻塞链，wait_event 为空但 CPU 高需要 profiler，Client 等待需要看应用消费或网络。
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
| 1 | `src/backend/utils/adt/pgstatfuncs.c` | pg_stat_activity 输出 state、wait_event、query_id 和时间戳。 |
| 2 | `src/backend/utils/adt/lockfuncs.c` | pg_locks、pg_blocking_pids() 和 lock wait 信息。 |
| 3 | `src/backend/utils/activity/backend_status.c` | pgstat_report_activity() 和 query_id / state 上报。 |
| 4 | `src/backend/storage/lmgr/proc.c` | ProcSleep() 中 heavyweight lock 等待和 latch 唤醒。 |
| 5 | `src/backend/storage/lmgr/deadlock.c` | deadlock 检测、blocking graph 和等待边处理。 |
| 6 | `src/backend/commands/explain.c` | EXPLAIN ANALYZE 输出计划节点实际时间、rows、buffers。 |
| 7 | `src/backend/executor/instrument.c` | InstrStartNode() / InstrStopNode() 节点级计时边界。 |
| 8 | `src/backend/utils/activity/pgstat_lock.c` | lock wait 累计统计。 |
| 9 | `src/backend/utils/activity/pgstat_io.c` | I/O 累计统计。 |

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

### 4.1. `backend state`

active、idle in transaction 等状态告诉你 backend 是否正在执行或等待客户端。

读这个状态时先问四个问题：

- 它是 backend-local、shared memory、文件状态，还是只在调用栈中短暂存在？
- 谁负责初始化它，初始化失败时有没有后续 cleanup 责任？
- 谁能读取或修改它，读取是否需要锁、pin、snapshot 或 owner？
- 它何时失效，失效后用户可见指标会如何表现？

在本节主问题中，`backend state` 的价值是帮助判断一个观测现象到底落在哪个状态边界。

### 4.2. `wait_event_type/name`

当前采样点是否进入 PostgreSQL wait path。

读这个状态时先问四个问题：

- 它是 backend-local、shared memory、文件状态，还是只在调用栈中短暂存在？
- 谁负责初始化它，初始化失败时有没有后续 cleanup 责任？
- 谁能读取或修改它，读取是否需要锁、pin、snapshot 或 owner？
- 它何时失效，失效后用户可见指标会如何表现？

在本节主问题中，`wait_event_type/name` 的价值是帮助判断一个观测现象到底落在哪个状态边界。

### 4.3. `pg_locks.granted/waitstart`

锁等待是否已经排队，以及等待从什么时候开始。

读这个状态时先问四个问题：

- 它是 backend-local、shared memory、文件状态，还是只在调用栈中短暂存在？
- 谁负责初始化它，初始化失败时有没有后续 cleanup 责任？
- 谁能读取或修改它，读取是否需要锁、pin、snapshot 或 owner？
- 它何时失效，失效后用户可见指标会如何表现？

在本节主问题中，`pg_locks.granted/waitstart` 的价值是帮助判断一个观测现象到底落在哪个状态边界。

### 4.4. `blocking pids`

heavyweight lock 的直接 blocker 集合。

读这个状态时先问四个问题：

- 它是 backend-local、shared memory、文件状态，还是只在调用栈中短暂存在？
- 谁负责初始化它，初始化失败时有没有后续 cleanup 责任？
- 谁能读取或修改它，读取是否需要锁、pin、snapshot 或 owner？
- 它何时失效，失效后用户可见指标会如何表现？

在本节主问题中，`blocking pids` 的价值是帮助判断一个观测现象到底落在哪个状态边界。

### 4.5. `Instrumentation timing`

EXPLAIN ANALYZE 节点级时间，可能包含子节点时间和等待时间。

读这个状态时先问四个问题：

- 它是 backend-local、shared memory、文件状态，还是只在调用栈中短暂存在？
- 谁负责初始化它，初始化失败时有没有后续 cleanup 责任？
- 谁能读取或修改它，读取是否需要锁、pin、snapshot 或 owner？
- 它何时失效，失效后用户可见指标会如何表现？

在本节主问题中，`Instrumentation timing` 的价值是帮助判断一个观测现象到底落在哪个状态边界。

### 4.6. `system profile samples`

wait_event 为空但 CPU 高时，用 perf / eBPF / gdb 栈解释热点。

读这个状态时先问四个问题：

- 它是 backend-local、shared memory、文件状态，还是只在调用栈中短暂存在？
- 谁负责初始化它，初始化失败时有没有后续 cleanup 责任？
- 谁能读取或修改它，读取是否需要锁、pin、snapshot 或 owner？
- 它何时失效，失效后用户可见指标会如何表现？

在本节主问题中，`system profile samples` 的价值是帮助判断一个观测现象到底落在哪个状态边界。

本节最重要的不变量是：

```text
raw field 不是语义；field + lifecycle + writer + reader + cleanup 才是语义。
```

## 5. 主流程源码 walkthrough

下面按时间顺序走主链路。

每一步都回答“状态发生了什么变化”，而不是把源码读成函数名清单。

### 5.1. 步骤 1

先定位目标 backend pid、query_id、state 和 query_start。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.2. 步骤 2

如果 state 不是 active，先判断是否 idle in transaction 或客户端未继续驱动。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.3. 步骤 3

如果 active 且 wait_event_type 非空，记录等待类型和名称。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.4. 步骤 4

如果等待类型是 Lock，立即查询 pg_blocking_pids(pid) 和 pg_locks。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.5. 步骤 5

如果 blocker 存在，继续看 blocker 是否 active、idle in transaction 或也在等待别人。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.6. 步骤 6

沿 blocker 链追到根因 backend，而不是只终止最末端等待者。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.7. 步骤 7

如果等待类型是 IO，结合 EXPLAIN Buffers、pg_stat_io 和系统 I/O 指标。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.8. 步骤 8

如果等待类型是 Client，判断服务器是否在等客户端读写，而不是继续调数据库参数。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.9. 步骤 9

如果 wait_event 为空但 SQL 仍慢，观察 CPU 利用率和 profiler 栈。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.10. 步骤 10

EXPLAIN ANALYZE 用来确认时间集中在哪些节点，但不要把它和 pg_stat 累计混淆。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.11. 步骤 11

如果节点 rows / loops 异常，回到优化器估计、统计信息或 SQL 形状。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.12. 步骤 12

最后把证据归纳成一个主因，并设计最小复现实验。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

主链路可以压缩成：

```text
01. 先定位目标 backend pid、query_id、state 和 query_start。
02. 如果 state 不是 active，先判断是否 idle in transaction 或客户端未继续驱动。
03. 如果 active 且 wait_event_type 非空，记录等待类型和名称。
04. 如果等待类型是 Lock，立即查询 pg_blocking_pids(pid) 和 pg_locks。
05. 如果 blocker 存在，继续看 blocker 是否 active、idle in transaction 或也在等待别人。
06. 沿 blocker 链追到根因 backend，而不是只终止最末端等待者。
07. 如果等待类型是 IO，结合 EXPLAIN Buffers、pg_stat_io 和系统 I/O 指标。
08. 如果等待类型是 Client，判断服务器是否在等客户端读写，而不是继续调数据库参数。
09. 如果 wait_event 为空但 SQL 仍慢，观察 CPU 利用率和 profiler 栈。
10. EXPLAIN ANALYZE 用来确认时间集中在哪些节点，但不要把它和 pg_stat 累计混淆。
11. 如果节点 rows / loops 异常，回到优化器估计、统计信息或 SQL 形状。
12. 最后把证据归纳成一个主因，并设计最小复现实验。
```

## 6. 生命周期 / ownership / cleanup

生命周期是本节能否用于真实诊断的关键。

### 6.1. 采样

从 pg_stat_activity 抓当前状态，知道这是瞬时证据。

这里要特别区分两件事：

- 内存是否释放。
- 语义状态是否已经对其他 backend 或用户视图不可见。

很多观测问题并不是对象还在内存里，而是读取端还持有旧语义。

### 6.2. 追链

Lock 等待要沿 blocker 链追到根，而不是只看被阻塞 SQL。

这里要特别区分两件事：

- 内存是否释放。
- 语义状态是否已经对其他 backend 或用户视图不可见。

很多观测问题并不是对象还在内存里，而是读取端还持有旧语义。

### 6.3. 剖面

CPU 问题要用 profiler 栈或 EXPLAIN 节点时间看热点。

这里要特别区分两件事：

- 内存是否释放。
- 语义状态是否已经对其他 backend 或用户视图不可见。

很多观测问题并不是对象还在内存里，而是读取端还持有旧语义。

### 6.4. 交叉

I/O 问题要同时看 wait event、Buffers、pg_stat_io 和系统设备指标。

这里要特别区分两件事：

- 内存是否释放。
- 语义状态是否已经对其他 backend 或用户视图不可见。

很多观测问题并不是对象还在内存里，而是读取端还持有旧语义。

### 6.5. 复盘

诊断结论必须能解释 SQL 总耗时的主要部分。

这里要特别区分两件事：

- 内存是否释放。
- 语义状态是否已经对其他 backend 或用户视图不可见。

很多观测问题并不是对象还在内存里，而是读取端还持有旧语义。

ERROR / abort 路径上，不能依赖普通 C 调用栈逐层返回完成清理。

因此源码里要找的是 ResourceOwner、MemoryContext、PG_TRY/PG_FINALLY、事务回调或标准 executor teardown。

## 7. 正确性机制层次

本节的正确性来自多个层次叠加，而不是单个函数。

### 7.1. 机制 1

wait_event 为空不等于没有问题，可能正在 CPU hot path。

把它放回运行期，可以得到两个判断：

- 正常路径中，它限制了状态何时能被推进或读取。
- 异常路径中，它决定用户看到的是空值、旧值、错误还是等待。

### 7.2. 机制 2

Lock wait 的根因通常是 blocker 的行为，不是等待者本身。

把它放回运行期，可以得到两个判断：

- 正常路径中，它限制了状态何时能被推进或读取。
- 异常路径中，它决定用户看到的是空值、旧值、错误还是等待。

### 7.3. 机制 3

EXPLAIN ANALYZE 的节点时间是执行期间采集的，不等同于 pg_stat 累计时间。

把它放回运行期，可以得到两个判断：

- 正常路径中，它限制了状态何时能被推进或读取。
- 异常路径中，它决定用户看到的是空值、旧值、错误还是等待。

### 7.4. 机制 4

Client wait 是 PostgreSQL 与客户端协议边界，不应误判为内部锁。

把它放回运行期，可以得到两个判断：

- 正常路径中，它限制了状态何时能被推进或读取。
- 异常路径中，它决定用户看到的是空值、旧值、错误还是等待。

### 7.5. 机制 5

并行查询中 leader 和 worker 可能有不同等待，需要分开看。

把它放回运行期，可以得到两个判断：

- 正常路径中，它限制了状态何时能被推进或读取。
- 异常路径中，它决定用户看到的是空值、旧值、错误还是等待。

### 7.6. 机制 6

一次采样只描述采样时刻，长 SQL 要多次采样或用日志。

把它放回运行期，可以得到两个判断：

- 正常路径中，它限制了状态何时能被推进或读取。
- 异常路径中，它决定用户看到的是空值、旧值、错误还是等待。

不要把这些机制抽象成“为了安全”。

更准确的说法是：它们把低成本 hot path 和可解释的观测边界同时保留下来。

## 8. 错误路径 / 异常路径 / fallback

下面这些场景不是边角料，而是现场排障最常遇到的误判来源。

### 8.1. 只看等待者

Lock wait 的根因在 blocker 或 blocker 链根部。

排查时不要先改参数。

先问：当前现象是写入端没有推进，读取端没有刷新，还是观察工具选错了层次。

### 8.2. 只看 EXPLAIN

EXPLAIN 可能没有捕捉线上锁等待或客户端背压。

排查时不要先改参数。

先问：当前现象是写入端没有推进，读取端没有刷新，还是观察工具选错了层次。

### 8.3. 只看 CPU

高 CPU 可能是锁释放后的后续阶段，不能解释前面的等待。

排查时不要先改参数。

先问：当前现象是写入端没有推进，读取端没有刷新，还是观察工具选错了层次。

### 8.4. 把 wait_event 空当作正常

CPU-bound 查询通常 wait_event 为空。

排查时不要先改参数。

先问：当前现象是写入端没有推进，读取端没有刷新，还是观察工具选错了层次。

### 8.5. 把 pg_stat 累计当现场

pg_stat 适合趋势，不适合还原单次等待链。

排查时不要先改参数。

先问：当前现象是写入端没有推进，读取端没有刷新，还是观察工具选错了层次。

fallback 的设计目标通常不是让结果更漂亮，而是让系统在异常路径上保持可恢复、可清理、可解释。

## 9. 成本、资源与跨模块传播

可观测性代码如果放错层次，会把诊断工具变成性能问题的一部分。

### 9.1. 成本维度 1

pg_stat_activity 采样成本低，适合第一现场。

这个成本通常会继续传播到相邻模块：

- executor hot path 是否多一次分支、计时或锁。
- shared memory 是否出现新的争用点。
- 日志、视图或监控采样是否放大已有压力。

### 9.2. 成本维度 2

pg_locks 和 pg_blocking_pids() 需要遍历 lock table，频繁调用要节制。

这个成本通常会继续传播到相邻模块：

- executor hot path 是否多一次分支、计时或锁。
- shared memory 是否出现新的争用点。
- 日志、视图或监控采样是否放大已有压力。

### 9.3. 成本维度 3

EXPLAIN ANALYZE 会真实执行 SQL，并带来观测扰动。

这个成本通常会继续传播到相邻模块：

- executor hot path 是否多一次分支、计时或锁。
- shared memory 是否出现新的争用点。
- 日志、视图或监控采样是否放大已有压力。

### 9.4. 成本维度 4

system profiler 需要权限和采样窗口，但能解释 wait_event 看不到的 CPU。

这个成本通常会继续传播到相邻模块：

- executor hot path 是否多一次分支、计时或锁。
- shared memory 是否出现新的争用点。
- 日志、视图或监控采样是否放大已有压力。

### 9.5. 成本维度 5

诊断脚本越复杂，越要控制采样频率，避免制造额外负载。

这个成本通常会继续传播到相邻模块：

- executor hot path 是否多一次分支、计时或锁。
- shared memory 是否出现新的争用点。
- 日志、视图或监控采样是否放大已有压力。

因此课程里的实验默认是短时、可控、可回滚的，不建议把强制刷新或高频采样作为长期配置。

## 10. 观测与诊断入口

观测入口要按问题层次选择。

### 10.1. `pg_stat_activity`

当前 state、wait event、query_start、state_change。

使用这个入口时要同时记录：

- 采样时间。
- backend pid 或 query_id。
- 是否处于同一事务、同一 backend 或同一执行。

### 10.2. `pg_blocking_pids()`

heavyweight lock 的 blocker 入口。

使用这个入口时要同时记录：

- 采样时间。
- backend pid 或 query_id。
- 是否处于同一事务、同一 backend 或同一执行。

### 10.3. `pg_locks`

等待锁和持有锁的 locktag / mode。

使用这个入口时要同时记录：

- 采样时间。
- backend pid 或 query_id。
- 是否处于同一事务、同一 backend 或同一执行。

### 10.4. `EXPLAIN (ANALYZE, BUFFERS)`

单次执行计划节点时间和 buffer 访问。

使用这个入口时要同时记录：

- 采样时间。
- backend pid 或 query_id。
- 是否处于同一事务、同一 backend 或同一执行。

### 10.5. `pg_stat_io / pg_stat_database`

I/O 与数据库级趋势。

使用这个入口时要同时记录：

- 采样时间。
- backend pid 或 query_id。
- 是否处于同一事务、同一 backend 或同一执行。

### 10.6. `perf / eBPF / gdb`

wait_event 为空时回到 CPU 栈。

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

### 12.1. 构造 blocker 链

实验步骤：

```sql
-- session 1
BEGIN; UPDATE t_wait_demo SET id = id WHERE id = 1;
-- session 2
UPDATE t_wait_demo SET id = id WHERE id = 1;
SELECT pg_blocking_pids(<session2_pid>);
```

目标是看到等待者、直接 blocker 和 blocker 状态。

实验后回到源码，至少确认一个入口函数和一个状态字段。

### 12.2. 构造 CPU-bound 查询

实验步骤：

```sql
SELECT count(*) FROM generate_series(1, 100000000) g WHERE g % 7 = 0;
```

观察 wait_event 可能为空，但 CPU 栈集中在表达式或聚合路径。

实验后回到源码，至少确认一个入口函数和一个状态字段。

### 12.3. 构造 ClientWrite

实验步骤：

```sql
COPY (SELECT repeat(x, 1000) FROM generate_series(1,1000000)) TO STDOUT;
```

放慢客户端读取，观察服务器是否等待客户端消费。

实验后回到源码，至少确认一个入口函数和一个状态字段。

### 12.4. EXPLAIN 与现场对比

实验步骤：

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM t_wait_demo WHERE id = 1;
SELECT wait_event_type, wait_event FROM pg_stat_activity WHERE pid = <pid>;
```

理解 EXPLAIN 是执行剖面，pg_stat_activity 是现场采样。

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

1. 为什么 Lock wait 的根因通常不是等待者？

回答时必须至少引用一个源码文件和一个运行期现象。

2. wait_event 为空时为什么更应该看 CPU profiler？

回答时必须至少引用一个源码文件和一个运行期现象。

3. EXPLAIN ANALYZE 与线上等待事件可能有哪些不一致？

回答时必须至少引用一个源码文件和一个运行期现象。

4. ClientRead 和 ClientWrite 分别意味着什么诊断方向？

回答时必须至少引用一个源码文件和一个运行期现象。

5. 并行查询中如何分别观察 leader 和 worker？

回答时必须至少引用一个源码文件和一个运行期现象。

6. 什么时候应该停止数据库内诊断，转向应用或操作系统？

回答时必须至少引用一个源码文件和一个运行期现象。

讨论题不是开放闲聊，而是训练把源码机制压缩成可迁移判断。

## 15. 源码复盘清单

如果需要快速复盘，本节可以按下面的源码锚点重新走一遍。

### 15.1. `pg_stat_activity`

先定位 `pg_stat_activity` 的调用者，不要从文件顶部线性读到底。

然后确认它如何影响 `backend state`。

本锚点对应的运行期动作是：先定位目标 backend pid、query_id、state 和 query_start。

复盘时记录三个问题：

- 这个函数在正常路径上推进了什么状态？
- 它在 ERROR、abort 或 early return 时是否需要额外清理？
- 用户能否在 SQL、日志、pg_stat、wait event、EXPLAIN 或 profiler 中看到它的后果？

### 15.2. `pg_blocking_pids()`

先定位 `pg_blocking_pids()` 的调用者，不要从文件顶部线性读到底。

然后确认它如何影响 `wait_event_type/name`。

本锚点对应的运行期动作是：如果 state 不是 active，先判断是否 idle in transaction 或客户端未继续驱动。

复盘时记录三个问题：

- 这个函数在正常路径上推进了什么状态？
- 它在 ERROR、abort 或 early return 时是否需要额外清理？
- 用户能否在 SQL、日志、pg_stat、wait event、EXPLAIN 或 profiler 中看到它的后果？

### 15.3. `pg_lock_status()`

先定位 `pg_lock_status()` 的调用者，不要从文件顶部线性读到底。

然后确认它如何影响 `pg_locks.granted/waitstart`。

本锚点对应的运行期动作是：如果 active 且 wait_event_type 非空，记录等待类型和名称。

复盘时记录三个问题：

- 这个函数在正常路径上推进了什么状态？
- 它在 ERROR、abort 或 early return 时是否需要额外清理？
- 用户能否在 SQL、日志、pg_stat、wait event、EXPLAIN 或 profiler 中看到它的后果？

### 15.4. `InstrStartNode()`

先定位 `InstrStartNode()` 的调用者，不要从文件顶部线性读到底。

然后确认它如何影响 `blocking pids`。

本锚点对应的运行期动作是：如果等待类型是 Lock，立即查询 pg_blocking_pids(pid) 和 pg_locks。

复盘时记录三个问题：

- 这个函数在正常路径上推进了什么状态？
- 它在 ERROR、abort 或 early return 时是否需要额外清理？
- 用户能否在 SQL、日志、pg_stat、wait event、EXPLAIN 或 profiler 中看到它的后果？

### 15.5. `InstrStopNode()`

先定位 `InstrStopNode()` 的调用者，不要从文件顶部线性读到底。

然后确认它如何影响 `Instrumentation timing`。

本锚点对应的运行期动作是：如果 blocker 存在，继续看 blocker 是否 active、idle in transaction 或也在等待别人。

复盘时记录三个问题：

- 这个函数在正常路径上推进了什么状态？
- 它在 ERROR、abort 或 early return 时是否需要额外清理？
- 用户能否在 SQL、日志、pg_stat、wait event、EXPLAIN 或 profiler 中看到它的后果？

### 15.6. `pgstat_report_activity()`

先定位 `pgstat_report_activity()` 的调用者，不要从文件顶部线性读到底。

然后确认它如何影响 `system profile samples`。

本锚点对应的运行期动作是：沿 blocker 链追到根因 backend，而不是只终止最末端等待者。

复盘时记录三个问题：

- 这个函数在正常路径上推进了什么状态？
- 它在 ERROR、abort 或 early return 时是否需要额外清理？
- 用户能否在 SQL、日志、pg_stat、wait event、EXPLAIN 或 profiler 中看到它的后果？

### 15.7. `pg_stat_activity`

先定位 `pg_stat_activity` 的调用者，不要从文件顶部线性读到底。

然后确认它如何影响 `backend state`。

本锚点对应的运行期动作是：如果等待类型是 IO，结合 EXPLAIN Buffers、pg_stat_io 和系统 I/O 指标。

复盘时记录三个问题：

- 这个函数在正常路径上推进了什么状态？
- 它在 ERROR、abort 或 early return 时是否需要额外清理？
- 用户能否在 SQL、日志、pg_stat、wait event、EXPLAIN 或 profiler 中看到它的后果？

## 16. 本节小结

- 慢 SQL 诊断要把响应时间拆成等待、CPU、阻塞链、I/O 和客户端阶段。
- wait_event 是第一线索，但不是完整解释。
- Lock 等待看 blocker，CPU 热点看 profiler，I/O 看 Buffers 和 pg_stat_io，Client 等待看应用消费。
- 下一节会讲扩展如何添加自己的 wait event，让自定义路径也能进入这套诊断地图。

本节最后沉淀出的可迁移规律是：

```text
观测值不是事实本身；观测值是某个源码状态在某个生命周期边界上的投影。
```

读 PostgreSQL 内核时，始终把 runtime 现象压回 state、owner、cleanup 和 cost。
