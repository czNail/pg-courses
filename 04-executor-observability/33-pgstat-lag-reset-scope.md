# PostgreSQL pg_stat 刷新延迟与 reset scope

## 课程定位

前置知识：已经理解执行器节点、Instrumentation 和常见 scan / DML 计数来源；本节进入 pg_stat 累计统计的读写一致性边界。

本节唯一主问题：

```text
为什么 pg_stat 视图可能存在刷新延迟、事务可见性和 reset scope 差异，排查执行器问题时如何避免把累计指标误读成单条 SQL 事实？
```

核心矛盾：执行路径需要低成本累计大量计数 vs 诊断者希望立刻看到全局一致、按语句切分、可重置的指标。

学完后应能判断一个 pg_stat 数值是 backend-local pending、共享统计、当前 backend snapshot，还是 reset 后重新开始累计的结果。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

前面几节已经从 EXPLAIN ANALYZE 看到单次执行中的节点级指标。

pg_stat 的目标不同：它不是一次计划树的剖面，而是长期运行中跨语句、跨事务、跨 backend 的累计事实。

这个差异会制造一个常见误判：看到某个 pg_stat 计数没有变化，就以为刚执行的 SQL 没有触发对应路径。

实际源码中，很多计数先写入本 backend 的 pending state，再由 flush 边界推进到共享统计。

同时，读取端还可能持有当前 backend 的 stats snapshot。

因此 pg_stat 不是“每次 SELECT 都实时扫描所有 backend 的最新局部变量”。

本节只讨论刷新延迟、读取一致性和 reset scope。

它不重复讲上一节 pg_stat relation / index / function 计数与执行节点的对应关系。

下一组课程会转向 wait event，因为 wait event 是“当前正在等什么”，与 pg_stat 的累计模型形成对照。

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
执行路径先把计数累加到 backend-local pending state；pgstat_report_stat() 在事务边界、空闲间隔或强制刷新时把 pending state 合并到共享统计；读取端按 stats_fetch_consistency 选择直接读、缓存或 snapshot；reset 按数据库、kind、对象或 backend scope 更新共享统计与 reset timestamp。
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
| 1 | `src/backend/utils/activity/pgstat.c` | 累计统计基础设施主文件；关注 pgstat_report_stat()、pgstat_clear_snapshot()、pgstat_build_snapshot()、assign_stats_fetch_consistency()。 |
| 2 | `src/backend/utils/activity/pgstat_shmem.c` | 共享统计 hash、entry ref、reset 和 GC；关注 StatsShmemCallbacks、pgstat_reset_entry()、pgstat_reset_entries_of_kind()。 |
| 3 | `src/backend/utils/activity/pgstat_xact.c` | 事务和子事务结束时的统计合并；关注 AtEOXact_PgStat()、AtEOSubXact_PgStat()。 |
| 4 | `src/backend/utils/activity/pgstat_relation.c` | 表级 pending 计数入口；关注 pgstat_assoc_relation() 与 heap insert/update/delete 计数函数。 |
| 5 | `src/backend/utils/adt/pgstatfuncs.c` | SQL 层 reset、clear snapshot、force flush 和 pg_stat 视图函数入口。 |
| 6 | `src/include/pgstat.h` | 对外统计结构、fetch consistency 枚举和 pgstat_* 函数声明。 |
| 7 | `src/include/utils/pgstat_internal.h` | 内部 PgStat_EntryRef、PgStat_LocalState、kind info 与 pending 标记。 |

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

### 4.1. `PgStat_LocalState`

backend-local 根状态，连接共享统计控制区、本地 entry ref cache、snapshot 和当前读取策略。

读这个状态时先问四个问题：

- 它是 backend-local、shared memory、文件状态，还是只在调用栈中短暂存在？
- 谁负责初始化它，初始化失败时有没有后续 cleanup 责任？
- 谁能读取或修改它，读取是否需要锁、pin、snapshot 或 owner？
- 它何时失效，失效后用户可见指标会如何表现？

在本节主问题中，`PgStat_LocalState` 的价值是帮助判断一个观测现象到底落在哪个状态边界。

### 4.2. `pgStatPending`

本 backend 中尚未 flush 的 PgStat_EntryRef 链表；它解释了“刚执行完但视图暂时不变”。

读这个状态时先问四个问题：

- 它是 backend-local、shared memory、文件状态，还是只在调用栈中短暂存在？
- 谁负责初始化它，初始化失败时有没有后续 cleanup 责任？
- 谁能读取或修改它，读取是否需要锁、pin、snapshot 或 owner？
- 它何时失效，失效后用户可见指标会如何表现？

在本节主问题中，`pgStatPending` 的价值是帮助判断一个观测现象到底落在哪个状态边界。

### 4.3. `PgStat_EntryRef`

本地缓存的共享统计 entry 引用，同时可能携带 pending data，避免每次计数都访问共享 hash。

读这个状态时先问四个问题：

- 它是 backend-local、shared memory、文件状态，还是只在调用栈中短暂存在？
- 谁负责初始化它，初始化失败时有没有后续 cleanup 责任？
- 谁能读取或修改它，读取是否需要锁、pin、snapshot 或 owner？
- 它何时失效，失效后用户可见指标会如何表现？

在本节主问题中，`PgStat_EntryRef` 的价值是帮助判断一个观测现象到底落在哪个状态边界。

### 4.4. `PgStat_SnapshotEntry`

读取端在当前 backend 内保存的统计快照条目，受 stats_fetch_consistency 影响。

读这个状态时先问四个问题：

- 它是 backend-local、shared memory、文件状态，还是只在调用栈中短暂存在？
- 谁负责初始化它，初始化失败时有没有后续 cleanup 责任？
- 谁能读取或修改它，读取是否需要锁、pin、snapshot 或 owner？
- 它何时失效，失效后用户可见指标会如何表现？

在本节主问题中，`PgStat_SnapshotEntry` 的价值是帮助判断一个观测现象到底落在哪个状态边界。

### 4.5. `stats_fetch_consistency`

读取一致性 GUC，决定函数是否直接访问、缓存访问或建立 snapshot。

读这个状态时先问四个问题：

- 它是 backend-local、shared memory、文件状态，还是只在调用栈中短暂存在？
- 谁负责初始化它，初始化失败时有没有后续 cleanup 责任？
- 谁能读取或修改它，读取是否需要锁、pin、snapshot 或 owner？
- 它何时失效，失效后用户可见指标会如何表现？

在本节主问题中，`stats_fetch_consistency` 的价值是帮助判断一个观测现象到底落在哪个状态边界。

### 4.6. `reset timestamp`

reset 不只是把计数清零，还要留下重置时间，避免诊断时丢失“从什么时候开始累计”的语义。

读这个状态时先问四个问题：

- 它是 backend-local、shared memory、文件状态，还是只在调用栈中短暂存在？
- 谁负责初始化它，初始化失败时有没有后续 cleanup 责任？
- 谁能读取或修改它，读取是否需要锁、pin、snapshot 或 owner？
- 它何时失效，失效后用户可见指标会如何表现？

在本节主问题中，`reset timestamp` 的价值是帮助判断一个观测现象到底落在哪个状态边界。

本节最重要的不变量是：

```text
raw field 不是语义；field + lifecycle + writer + reader + cleanup 才是语义。
```

## 5. 主流程源码 walkthrough

下面按时间顺序走主链路。

每一步都回答“状态发生了什么变化”，而不是把源码读成函数名清单。

### 5.1. 步骤 1

执行器或 access method 在 scan / DML 路径上调用 pgstat_count_* 宏或函数。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.2. 步骤 2

relation 统计先通过 pgstat_assoc_relation() 找到当前 backend 的 PgStat_TableStatus。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.3. 步骤 3

tuple、block、scan 等计数写入 backend-local counts，而不是直接加锁修改共享 hash。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.4. 步骤 4

事务内的 insert/update/delete 还要经过 PgStat_TableXactStatus，等待提交或回滚边界决定如何合并。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.5. 步骤 5

AtEOXact_PgStat() 在事务结束时处理数据库、relation 和 dropped stats 状态。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.6. 步骤 6

pgstat_report_stat(force) 按时间间隔、force 标志和 pending 链表决定是否 flush。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.7. 步骤 7

flush 过程把 pending data 合并到共享统计 entry，并清理本地 pending 标记。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.8. 步骤 8

读取 pg_stat 视图时，pgstat_fetch_stat_* 根据 stats_fetch_consistency 准备读取策略。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.9. 步骤 9

PGSTAT_FETCH_CONSISTENCY_SNAPSHOT 会构建当前 backend 的统计 snapshot。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.10. 步骤 10

pg_stat_clear_snapshot() 清掉当前 backend 已持有的 snapshot，使下一次读取重新取数。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.11. 步骤 11

pg_stat_reset* 函数按 kind、数据库、对象、backend 或 shared scope 修改共享统计。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.12. 步骤 12

诊断者把“计数是否增加”解释为执行事实前，必须先判断 flush、snapshot 和 reset scope。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

主链路可以压缩成：

```text
01. 执行器或 access method 在 scan / DML 路径上调用 pgstat_count_* 宏或函数。
02. relation 统计先通过 pgstat_assoc_relation() 找到当前 backend 的 PgStat_TableStatus。
03. tuple、block、scan 等计数写入 backend-local counts，而不是直接加锁修改共享 hash。
04. 事务内的 insert/update/delete 还要经过 PgStat_TableXactStatus，等待提交或回滚边界决定如何合并。
05. AtEOXact_PgStat() 在事务结束时处理数据库、relation 和 dropped stats 状态。
06. pgstat_report_stat(force) 按时间间隔、force 标志和 pending 链表决定是否 flush。
07. flush 过程把 pending data 合并到共享统计 entry，并清理本地 pending 标记。
08. 读取 pg_stat 视图时，pgstat_fetch_stat_* 根据 stats_fetch_consistency 准备读取策略。
09. PGSTAT_FETCH_CONSISTENCY_SNAPSHOT 会构建当前 backend 的统计 snapshot。
10. pg_stat_clear_snapshot() 清掉当前 backend 已持有的 snapshot，使下一次读取重新取数。
11. pg_stat_reset* 函数按 kind、数据库、对象、backend 或 shared scope 修改共享统计。
12. 诊断者把“计数是否增加”解释为执行事实前，必须先判断 flush、snapshot 和 reset scope。
```

## 6. 生命周期 / ownership / cleanup

生命周期是本节能否用于真实诊断的关键。

### 6.1. 创建

统计共享内存在 server 启动期通过 StatsShmemCallbacks 创建；本地引用在 backend 首次访问某个 stats entry 时建立。

这里要特别区分两件事：

- 内存是否释放。
- 语义状态是否已经对其他 backend 或用户视图不可见。

很多观测问题并不是对象还在内存里，而是读取端还持有旧语义。

### 6.2. 持有

本地 PgStat_EntryRef 由当前 backend 持有，指向共享 hash entry 和进程本地地址。

这里要特别区分两件事：

- 内存是否释放。
- 语义状态是否已经对其他 backend 或用户视图不可见。

很多观测问题并不是对象还在内存里，而是读取端还持有旧语义。

### 6.3. 推进

执行路径只推进本地 pending；pgstat_report_stat() 才把 pending 推进到共享统计。

这里要特别区分两件事：

- 内存是否释放。
- 语义状态是否已经对其他 backend 或用户视图不可见。

很多观测问题并不是对象还在内存里，而是读取端还持有旧语义。

### 6.4. 失效

对象 drop、reset 或 stats_fetch_consistency 变化会让本地引用或 snapshot 需要释放或重建。

这里要特别区分两件事：

- 内存是否释放。
- 语义状态是否已经对其他 backend 或用户视图不可见。

很多观测问题并不是对象还在内存里，而是读取端还持有旧语义。

### 6.5. 释放

backend 退出、entry GC 或事务结束会清理对应本地状态；共享统计 entry 的释放要考虑其他 backend 的引用。

这里要特别区分两件事：

- 内存是否释放。
- 语义状态是否已经对其他 backend 或用户视图不可见。

很多观测问题并不是对象还在内存里，而是读取端还持有旧语义。

ERROR / abort 路径上，不能依赖普通 C 调用栈逐层返回完成清理。

因此源码里要找的是 ResourceOwner、MemoryContext、PG_TRY/PG_FINALLY、事务回调或标准 executor teardown。

## 7. 正确性机制层次

本节的正确性来自多个层次叠加，而不是单个函数。

### 7.1. 机制 1

计数更新不是 MVCC tuple 可见性；它是一套独立的累计统计一致性协议。

把它放回运行期，可以得到两个判断：

- 正常路径中，它限制了状态何时能被推进或读取。
- 异常路径中，它决定用户看到的是空值、旧值、错误还是等待。

### 7.2. 机制 2

本地 pending 消除了 hot path 上的共享锁竞争。

把它放回运行期，可以得到两个判断：

- 正常路径中，它限制了状态何时能被推进或读取。
- 异常路径中，它决定用户看到的是空值、旧值、错误还是等待。

### 7.3. 机制 3

共享 entry 使用每项锁和 dshash / DSA 管理可变对象统计。

把它放回运行期，可以得到两个判断：

- 正常路径中，它限制了状态何时能被推进或读取。
- 异常路径中，它决定用户看到的是空值、旧值、错误还是等待。

### 7.4. 机制 4

事务级统计通过 xact stack 区分提交、回滚和子事务合并。

把它放回运行期，可以得到两个判断：

- 正常路径中，它限制了状态何时能被推进或读取。
- 异常路径中，它决定用户看到的是空值、旧值、错误还是等待。

### 7.5. 机制 5

读取端 snapshot 保证一次诊断查询内部看到稳定统计。

把它放回运行期，可以得到两个判断：

- 正常路径中，它限制了状态何时能被推进或读取。
- 异常路径中，它决定用户看到的是空值、旧值、错误还是等待。

### 7.6. 机制 6

reset scope 必须与数据库、对象种类和对象标识一起解释。

把它放回运行期，可以得到两个判断：

- 正常路径中，它限制了状态何时能被推进或读取。
- 异常路径中，它决定用户看到的是空值、旧值、错误还是等待。

不要把这些机制抽象成“为了安全”。

更准确的说法是：它们把低成本 hot path 和可解释的观测边界同时保留下来。

## 8. 错误路径 / 异常路径 / fallback

下面这些场景不是边角料，而是现场排障最常遇到的误判来源。

### 8.1. 刚执行完查不到变化

可能是 pending 尚未 flush，也可能是当前 backend 仍持有旧 stats snapshot。

排查时不要先改参数。

先问：当前现象是写入端没有推进，读取端没有刷新，还是观察工具选错了层次。

### 8.2. reset 后还有旧值

检查读端是否需要 pg_stat_clear_snapshot()，以及 reset 目标是否覆盖了当前对象种类。

排查时不要先改参数。

先问：当前现象是写入端没有推进，读取端没有刷新，还是观察工具选错了层次。

### 8.3. 把 pg_stat 当单条 SQL 指标

pg_stat 是累计统计；单条 SQL 应优先看 EXPLAIN ANALYZE 或 auto_explain。

排查时不要先改参数。

先问：当前现象是写入端没有推进，读取端没有刷新，还是观察工具选错了层次。

### 8.4. 跨事务读取不一致

不同 backend 的 flush 时机不同，视图读取不是全局 barrier。

排查时不要先改参数。

先问：当前现象是写入端没有推进，读取端没有刷新，还是观察工具选错了层次。

### 8.5. 对象删除后的统计残留

entry ref GC 需要等待持有旧引用的 backend 释放。

排查时不要先改参数。

先问：当前现象是写入端没有推进，读取端没有刷新，还是观察工具选错了层次。

fallback 的设计目标通常不是让结果更漂亮，而是让系统在异常路径上保持可恢复、可清理、可解释。

## 9. 成本、资源与跨模块传播

可观测性代码如果放错层次，会把诊断工具变成性能问题的一部分。

### 9.1. 成本维度 1

每 tuple 直接加共享锁会破坏执行器 hot path；pending state 是成本控制。

这个成本通常会继续传播到相邻模块：

- executor hot path 是否多一次分支、计时或锁。
- shared memory 是否出现新的争用点。
- 日志、视图或监控采样是否放大已有压力。

### 9.2. 成本维度 2

pending 链表越长，flush 时合并成本越集中。

这个成本通常会继续传播到相邻模块：

- executor hot path 是否多一次分支、计时或锁。
- shared memory 是否出现新的争用点。
- 日志、视图或监控采样是否放大已有压力。

### 9.3. 成本维度 3

stats_fetch_consistency=snapshot 会增加本 backend 的内存与构建成本。

这个成本通常会继续传播到相邻模块：

- executor hot path 是否多一次分支、计时或锁。
- shared memory 是否出现新的争用点。
- 日志、视图或监控采样是否放大已有压力。

### 9.4. 成本维度 4

reset 大范围统计会访问更多共享 entry，并触发本地引用 GC。

这个成本通常会继续传播到相邻模块：

- executor hot path 是否多一次分支、计时或锁。
- shared memory 是否出现新的争用点。
- 日志、视图或监控采样是否放大已有压力。

### 9.5. 成本维度 5

诊断脚本频繁 force flush 会改变系统运行节奏，不应长期打开。

这个成本通常会继续传播到相邻模块：

- executor hot path 是否多一次分支、计时或锁。
- shared memory 是否出现新的争用点。
- 日志、视图或监控采样是否放大已有压力。

因此课程里的实验默认是短时、可控、可回滚的，不建议把强制刷新或高频采样作为长期配置。

## 10. 观测与诊断入口

观测入口要按问题层次选择。

### 10.1. `pg_stat_get_snapshot_timestamp()`

判断当前 backend 持有的 stats snapshot 是何时建立。

使用这个入口时要同时记录：

- 采样时间。
- backend pid 或 query_id。
- 是否处于同一事务、同一 backend 或同一执行。

### 10.2. `pg_stat_clear_snapshot()`

清理当前 backend stats snapshot，用来验证读取缓存是否导致旧值。

使用这个入口时要同时记录：

- 采样时间。
- backend pid 或 query_id。
- 是否处于同一事务、同一 backend 或同一执行。

### 10.3. `pg_stat_force_next_flush()`

测试中强制下一次统计刷新，用来缩短 pending 延迟。

使用这个入口时要同时记录：

- 采样时间。
- backend pid 或 query_id。
- 是否处于同一事务、同一 backend 或同一执行。

### 10.4. `pg_stat_reset()`

重置当前数据库范围统计，适合实验前清空累计背景。

使用这个入口时要同时记录：

- 采样时间。
- backend pid 或 query_id。
- 是否处于同一事务、同一 backend 或同一执行。

### 10.5. `pg_stat_reset_single_table_counters()`

只重置指定 relation，适合隔离表级 scan / DML 计数。

使用这个入口时要同时记录：

- 采样时间。
- backend pid 或 query_id。
- 是否处于同一事务、同一 backend 或同一执行。

### 10.6. `pg_stat_all_tables`

观察 seq_scan、idx_scan、n_tup_ins、n_tup_upd、n_tup_del 等累计结果。

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

### 12.1. 观察 snapshot 缓存

实验步骤：

```sql
SELECT pg_stat_get_snapshot_timestamp();
SELECT * FROM pg_stat_all_tables WHERE relname = 't_pgstat_demo';
SELECT pg_stat_clear_snapshot();
SELECT pg_stat_get_snapshot_timestamp();
```

同一个会话里先读统计再清 snapshot，比较 timestamp 和计数是否重新取数。

实验后回到源码，至少确认一个入口函数和一个状态字段。

### 12.2. 隔离表级 reset

实验步骤：

```sql
CREATE TABLE t_pgstat_demo(id int);
SELECT pg_stat_reset_single_table_counters('t_pgstat_demo'::regclass);
INSERT INTO t_pgstat_demo SELECT generate_series(1,1000);
SELECT pg_stat_force_next_flush();
```

实验目标是把背景累计清掉，只看当前表的计数推进。

实验后回到源码，至少确认一个入口函数和一个状态字段。

### 12.3. 区分单次执行和累计统计

实验步骤：

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT count(*) FROM t_pgstat_demo;
SELECT seq_scan, seq_tup_read FROM pg_stat_all_tables WHERE relname = 't_pgstat_demo';
```

EXPLAIN 解释单次执行；pg_stat 解释多次执行后的累计事实。

实验后回到源码，至少确认一个入口函数和一个状态字段。

### 12.4. reset scope 对比

实验步骤：

```sql
SELECT pg_stat_reset();
SELECT pg_stat_reset_shared('io');
SELECT pg_stat_reset_shared('wal');
```

不同 reset 函数覆盖的 kind 不同，不能把 reset 当作一个全局清零按钮。

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

1. 为什么 PostgreSQL 不让每次 tuple 访问都直接更新共享统计？

回答时必须至少引用一个源码文件和一个运行期现象。

2. stats_fetch_consistency=snapshot 对监控脚本有什么好处和坏处？

回答时必须至少引用一个源码文件和一个运行期现象。

3. pg_stat 和 EXPLAIN ANALYZE 在排查慢 SQL 时应该如何分工？

回答时必须至少引用一个源码文件和一个运行期现象。

4. reset timestamp 为什么比单纯清零更重要？

回答时必须至少引用一个源码文件和一个运行期现象。

5. 如果一个 backend 长时间不空闲，pending stats 可能怎样影响观测？

回答时必须至少引用一个源码文件和一个运行期现象。

6. 什么时候应该使用 pg_stat_force_next_flush()，什么时候不应该？

回答时必须至少引用一个源码文件和一个运行期现象。

讨论题不是开放闲聊，而是训练把源码机制压缩成可迁移判断。

## 15. 源码复盘清单

如果需要快速复盘，本节可以按下面的源码锚点重新走一遍。

### 15.1. `pgstat_report_stat()`

先定位 `pgstat_report_stat()` 的调用者，不要从文件顶部线性读到底。

然后确认它如何影响 `PgStat_LocalState`。

本锚点对应的运行期动作是：执行器或 access method 在 scan / DML 路径上调用 pgstat_count_* 宏或函数。

复盘时记录三个问题：

- 这个函数在正常路径上推进了什么状态？
- 它在 ERROR、abort 或 early return 时是否需要额外清理？
- 用户能否在 SQL、日志、pg_stat、wait event、EXPLAIN 或 profiler 中看到它的后果？

### 15.2. `pgstat_clear_snapshot()`

先定位 `pgstat_clear_snapshot()` 的调用者，不要从文件顶部线性读到底。

然后确认它如何影响 `pgStatPending`。

本锚点对应的运行期动作是：relation 统计先通过 pgstat_assoc_relation() 找到当前 backend 的 PgStat_TableStatus。

复盘时记录三个问题：

- 这个函数在正常路径上推进了什么状态？
- 它在 ERROR、abort 或 early return 时是否需要额外清理？
- 用户能否在 SQL、日志、pg_stat、wait event、EXPLAIN 或 profiler 中看到它的后果？

### 15.3. `pgstat_build_snapshot()`

先定位 `pgstat_build_snapshot()` 的调用者，不要从文件顶部线性读到底。

然后确认它如何影响 `PgStat_EntryRef`。

本锚点对应的运行期动作是：tuple、block、scan 等计数写入 backend-local counts，而不是直接加锁修改共享 hash。

复盘时记录三个问题：

- 这个函数在正常路径上推进了什么状态？
- 它在 ERROR、abort 或 early return 时是否需要额外清理？
- 用户能否在 SQL、日志、pg_stat、wait event、EXPLAIN 或 profiler 中看到它的后果？

### 15.4. `AtEOXact_PgStat()`

先定位 `AtEOXact_PgStat()` 的调用者，不要从文件顶部线性读到底。

然后确认它如何影响 `PgStat_SnapshotEntry`。

本锚点对应的运行期动作是：事务内的 insert/update/delete 还要经过 PgStat_TableXactStatus，等待提交或回滚边界决定如何合并。

复盘时记录三个问题：

- 这个函数在正常路径上推进了什么状态？
- 它在 ERROR、abort 或 early return 时是否需要额外清理？
- 用户能否在 SQL、日志、pg_stat、wait event、EXPLAIN 或 profiler 中看到它的后果？

### 15.5. `pgstat_reset_entry()`

先定位 `pgstat_reset_entry()` 的调用者，不要从文件顶部线性读到底。

然后确认它如何影响 `stats_fetch_consistency`。

本锚点对应的运行期动作是：AtEOXact_PgStat() 在事务结束时处理数据库、relation 和 dropped stats 状态。

复盘时记录三个问题：

- 这个函数在正常路径上推进了什么状态？
- 它在 ERROR、abort 或 early return 时是否需要额外清理？
- 用户能否在 SQL、日志、pg_stat、wait event、EXPLAIN 或 profiler 中看到它的后果？

### 15.6. `pg_stat_force_next_flush()`

先定位 `pg_stat_force_next_flush()` 的调用者，不要从文件顶部线性读到底。

然后确认它如何影响 `reset timestamp`。

本锚点对应的运行期动作是：pgstat_report_stat(force) 按时间间隔、force 标志和 pending 链表决定是否 flush。

复盘时记录三个问题：

- 这个函数在正常路径上推进了什么状态？
- 它在 ERROR、abort 或 early return 时是否需要额外清理？
- 用户能否在 SQL、日志、pg_stat、wait event、EXPLAIN 或 profiler 中看到它的后果？

## 16. 本节小结

- pg_stat 的核心不是实时精确，而是在低执行成本下提供长期累计事实。
- pending flush、读取 snapshot 和 reset scope 是解释统计值的三层边界。
- 排查执行器问题时，先确认指标属于单次执行、累计统计还是当前等待。
- 下一节会转向 wait event：它不累计历史，而暴露 backend 当前等待点。

本节最后沉淀出的可迁移规律是：

```text
观测值不是事实本身；观测值是某个源码状态在某个生命周期边界上的投影。
```

读 PostgreSQL 内核时，始终把 runtime 现象压回 state、owner、cleanup 和 cost。
