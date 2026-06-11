# PostgreSQL 自定义等待点与扩展可观测性

## 课程定位

前置知识：已经能用内置 wait event 诊断执行器和存储路径；本节讨论扩展或新执行路径如何把自己的等待暴露出来。

本节唯一主问题：

```text
扩展或新执行路径何时应该注册 wait event，如何命名、设置和清理，避免让用户只能看到模糊的 active 状态？
```

核心矛盾：扩展需要暴露自己的阻塞点以便诊断，但 wait event 注册和命名必须稳定、低成本、不会污染内置分类。

学完后应能为一个扩展等待路径设计 wait event 名称、注册时机、start/end 边界和诊断文档。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

内置 wait event 覆盖了核心 lock、I/O、IPC、Client 等路径。

扩展代码如果执行网络请求、外部服务等待、共享队列等待或自定义后台任务等待，也应该给用户一个可观测名称。

否则 pg_stat_activity 只显示 active，诊断者会误以为 backend 在 CPU 跑。

本节只讲 custom wait event 的注册与使用边界。

它不讲扩展 hooks 的完整设计；那是下一节 executor hook chain 的主题。

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
扩展调用 WaitEventExtensionNew(name) 为稳定名称取得 wait_event_info；真实等待前调用 pgstat_report_wait_start(wait_event_info)，返回后调用 pgstat_report_wait_end()；pg_get_wait_events() 和 pg_stat_activity 会把该 id 显示为 Extension 类型。
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
| 1 | `src/include/utils/wait_event.h` | 扩展 wait event 的公开入口 WaitEventExtensionNew() 与 start/end 函数。 |
| 2 | `src/backend/utils/activity/wait_event.c` | WaitEventCustomNew()、共享 hash、重复名称检查和 id 分配。 |
| 3 | `src/backend/utils/activity/wait_event_funcs.c` | pg_get_wait_events() 如何输出 Extension / InjectionPoint 自定义名称。 |
| 4 | `src/include/storage/subsystemlist.h` | WaitEventCustomShmemCallbacks 注册到启动期 shared memory 初始化。 |
| 5 | `src/include/storage/lwlocklist.h` | WaitEventCustomLock 保护自定义 wait event hash。 |
| 6 | `src/backend/utils/adt/pgstatfuncs.c` | pg_stat_activity 读取并解码自定义 wait event。 |

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

### 4.1. `WaitEventCustomCounterData`

共享计数器，为自定义 wait event 分配 id。

读这个状态时先问四个问题：

- 它是 backend-local、shared memory、文件状态，还是只在调用栈中短暂存在？
- 谁负责初始化它，初始化失败时有没有后续 cleanup 责任？
- 谁能读取或修改它，读取是否需要锁、pin、snapshot 或 owner？
- 它何时失效，失效后用户可见指标会如何表现？

在本节主问题中，`WaitEventCustomCounterData` 的价值是帮助判断一个观测现象到底落在哪个状态边界。

### 4.2. `WaitEventCustomHashByName`

按名称查找已注册 wait event，避免重复分配。

读这个状态时先问四个问题：

- 它是 backend-local、shared memory、文件状态，还是只在调用栈中短暂存在？
- 谁负责初始化它，初始化失败时有没有后续 cleanup 责任？
- 谁能读取或修改它，读取是否需要锁、pin、snapshot 或 owner？
- 它何时失效，失效后用户可见指标会如何表现？

在本节主问题中，`WaitEventCustomHashByName` 的价值是帮助判断一个观测现象到底落在哪个状态边界。

### 4.3. `WaitEventCustomHashByInfo`

按 wait_event_info 查回名称，用于读取端显示。

读这个状态时先问四个问题：

- 它是 backend-local、shared memory、文件状态，还是只在调用栈中短暂存在？
- 谁负责初始化它，初始化失败时有没有后续 cleanup 责任？
- 谁能读取或修改它，读取是否需要锁、pin、snapshot 或 owner？
- 它何时失效，失效后用户可见指标会如何表现？

在本节主问题中，`WaitEventCustomHashByInfo` 的价值是帮助判断一个观测现象到底落在哪个状态边界。

### 4.4. `PG_WAIT_EXTENSION`

自定义扩展等待的 class。

读这个状态时先问四个问题：

- 它是 backend-local、shared memory、文件状态，还是只在调用栈中短暂存在？
- 谁负责初始化它，初始化失败时有没有后续 cleanup 责任？
- 谁能读取或修改它，读取是否需要锁、pin、snapshot 或 owner？
- 它何时失效，失效后用户可见指标会如何表现？

在本节主问题中，`PG_WAIT_EXTENSION` 的价值是帮助判断一个观测现象到底落在哪个状态边界。

### 4.5. `wait_event_name`

用户可见的稳定诊断名称，长度受 NAMEDATALEN 限制。

读这个状态时先问四个问题：

- 它是 backend-local、shared memory、文件状态，还是只在调用栈中短暂存在？
- 谁负责初始化它，初始化失败时有没有后续 cleanup 责任？
- 谁能读取或修改它，读取是否需要锁、pin、snapshot 或 owner？
- 它何时失效，失效后用户可见指标会如何表现？

在本节主问题中，`wait_event_name` 的价值是帮助判断一个观测现象到底落在哪个状态边界。

### 4.6. `extension wait_event_info`

扩展保存的 uint32 id，真实等待路径反复使用。

读这个状态时先问四个问题：

- 它是 backend-local、shared memory、文件状态，还是只在调用栈中短暂存在？
- 谁负责初始化它，初始化失败时有没有后续 cleanup 责任？
- 谁能读取或修改它，读取是否需要锁、pin、snapshot 或 owner？
- 它何时失效，失效后用户可见指标会如何表现？

在本节主问题中，`extension wait_event_info` 的价值是帮助判断一个观测现象到底落在哪个状态边界。

本节最重要的不变量是：

```text
raw field 不是语义；field + lifecycle + writer + reader + cleanup 才是语义。
```

## 5. 主流程源码 walkthrough

下面按时间顺序走主链路。

每一步都回答“状态发生了什么变化”，而不是把源码读成函数名清单。

### 5.1. 步骤 1

扩展作者先确定等待是否真的会阻塞 backend。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.2. 步骤 2

如果只是 CPU 计算，不应该注册 wait event。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.3. 步骤 3

如果会等待外部服务、共享队列、文件描述符或后台 worker，应定义稳定名称。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.4. 步骤 4

扩展在合适初始化路径调用 WaitEventExtensionNew(name)。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.5. 步骤 5

WaitEventCustomNew() 先用共享锁查名称是否已存在。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.6. 步骤 6

若不存在，再用排他锁重查并分配新的 id。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.7. 步骤 7

分配 id 时使用 WaitEventCustomCounterData 的 spinlock 保护 nextId。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.8. 步骤 8

名称和 wait_event_info 同时写入两个共享 hash。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.9. 步骤 9

真实等待前，扩展调用 pgstat_report_wait_start(saved_info)。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.10. 步骤 10

等待结束后，扩展必须调用 pgstat_report_wait_end()。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.11. 步骤 11

用户查询 pg_stat_activity 时看到 wait_event_type=Extension 和扩展名称。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

### 5.12. 步骤 12

用户也可以通过 pg_get_wait_events() 查到扩展注册的 wait event。

阅读这一段源码时，建议同时看三个边界：

- 入口边界：调用者是否已经准备好本步所需状态。
- 状态边界：本步新建、修改、读取或清理了什么。
- 观测边界：用户能否在 SQL、日志、视图或 profiler 中看到结果。

如果断点停在这里，不要只打印局部变量。

先确认 owner、lifecycle 和下一步 cleanup，再解释当前值。

主链路可以压缩成：

```text
01. 扩展作者先确定等待是否真的会阻塞 backend。
02. 如果只是 CPU 计算，不应该注册 wait event。
03. 如果会等待外部服务、共享队列、文件描述符或后台 worker，应定义稳定名称。
04. 扩展在合适初始化路径调用 WaitEventExtensionNew(name)。
05. WaitEventCustomNew() 先用共享锁查名称是否已存在。
06. 若不存在，再用排他锁重查并分配新的 id。
07. 分配 id 时使用 WaitEventCustomCounterData 的 spinlock 保护 nextId。
08. 名称和 wait_event_info 同时写入两个共享 hash。
09. 真实等待前，扩展调用 pgstat_report_wait_start(saved_info)。
10. 等待结束后，扩展必须调用 pgstat_report_wait_end()。
11. 用户查询 pg_stat_activity 时看到 wait_event_type=Extension 和扩展名称。
12. 用户也可以通过 pg_get_wait_events() 查到扩展注册的 wait event。
```

## 6. 生命周期 / ownership / cleanup

生命周期是本节能否用于真实诊断的关键。

### 6.1. 命名

名称是用户诊断入口，应描述等待资源或协议阶段，而不是内部函数名。

这里要特别区分两件事：

- 内存是否释放。
- 语义状态是否已经对其他 backend 或用户视图不可见。

很多观测问题并不是对象还在内存里，而是读取端还持有旧语义。

### 6.2. 注册

注册应发生在等待路径外，避免每次等待都访问共享 hash。

这里要特别区分两件事：

- 内存是否释放。
- 语义状态是否已经对其他 backend 或用户视图不可见。

很多观测问题并不是对象还在内存里，而是读取端还持有旧语义。

### 6.3. 使用

保存返回的 wait_event_info，在真实阻塞前后成对 start/end。

这里要特别区分两件事：

- 内存是否释放。
- 语义状态是否已经对其他 backend 或用户视图不可见。

很多观测问题并不是对象还在内存里，而是读取端还持有旧语义。

### 6.4. 暴露

pg_stat_activity 和 pg_get_wait_events() 负责把 id 变成用户可见字符串。

这里要特别区分两件事：

- 内存是否释放。
- 语义状态是否已经对其他 backend 或用户视图不可见。

很多观测问题并不是对象还在内存里，而是读取端还持有旧语义。

### 6.5. 维护

名称一旦用于线上监控，不应随意重命名或复用不同语义。

这里要特别区分两件事：

- 内存是否释放。
- 语义状态是否已经对其他 backend 或用户视图不可见。

很多观测问题并不是对象还在内存里，而是读取端还持有旧语义。

ERROR / abort 路径上，不能依赖普通 C 调用栈逐层返回完成清理。

因此源码里要找的是 ResourceOwner、MemoryContext、PG_TRY/PG_FINALLY、事务回调或标准 executor teardown。

## 7. 正确性机制层次

本节的正确性来自多个层次叠加，而不是单个函数。

### 7.1. 机制 1

同名 wait event 在同一 class 内复用同一个 id。

把它放回运行期，可以得到两个判断：

- 正常路径中，它限制了状态何时能被推进或读取。
- 异常路径中，它决定用户看到的是空值、旧值、错误还是等待。

### 7.2. 机制 2

同名但不同 class 会报 duplicate object 错误。

把它放回运行期，可以得到两个判断：

- 正常路径中，它限制了状态何时能被推进或读取。
- 异常路径中，它决定用户看到的是空值、旧值、错误还是等待。

### 7.3. 机制 3

名称过长会 ERROR，避免共享 hash 中出现截断歧义。

把它放回运行期，可以得到两个判断：

- 正常路径中，它限制了状态何时能被推进或读取。
- 异常路径中，它决定用户看到的是空值、旧值、错误还是等待。

### 7.4. 机制 4

id 分配上限受 WAIT_EVENT_CUSTOM_HASH_SIZE 约束。

把它放回运行期，可以得到两个判断：

- 正常路径中，它限制了状态何时能被推进或读取。
- 异常路径中，它决定用户看到的是空值、旧值、错误还是等待。

### 7.5. 机制 5

注册成本不应进入每次等待 hot path。

把它放回运行期，可以得到两个判断：

- 正常路径中，它限制了状态何时能被推进或读取。
- 异常路径中，它决定用户看到的是空值、旧值、错误还是等待。

### 7.6. 机制 6

start/end 的清理规则与内置 wait event 完全相同。

把它放回运行期，可以得到两个判断：

- 正常路径中，它限制了状态何时能被推进或读取。
- 异常路径中，它决定用户看到的是空值、旧值、错误还是等待。

不要把这些机制抽象成“为了安全”。

更准确的说法是：它们把低成本 hot path 和可解释的观测边界同时保留下来。

## 8. 错误路径 / 异常路径 / fallback

下面这些场景不是边角料，而是现场排障最常遇到的误判来源。

### 8.1. 每次等待都注册

会把共享 hash 和 LWLock 成本放进 hot path。

排查时不要先改参数。

先问：当前现象是写入端没有推进，读取端没有刷新，还是观察工具选错了层次。

### 8.2. 名称过宽

例如 ExtensionWait 不能帮助用户定位外部资源。

排查时不要先改参数。

先问：当前现象是写入端没有推进，读取端没有刷新，还是观察工具选错了层次。

### 8.3. 名称带动态对象

把 relation、host 或用户输入拼进名称会耗尽 id，也会破坏监控稳定性。

排查时不要先改参数。

先问：当前现象是写入端没有推进，读取端没有刷新，还是观察工具选错了层次。

### 8.4. 漏掉 wait_end

Extension wait event 会在 pg_stat_activity 中残留，误导诊断。

排查时不要先改参数。

先问：当前现象是写入端没有推进，读取端没有刷新，还是观察工具选错了层次。

### 8.5. 把 CPU 阶段标成等待

wait event 应描述阻塞等待，不是算法阶段标签。

排查时不要先改参数。

先问：当前现象是写入端没有推进，读取端没有刷新，还是观察工具选错了层次。

fallback 的设计目标通常不是让结果更漂亮，而是让系统在异常路径上保持可恢复、可清理、可解释。

## 9. 成本、资源与跨模块传播

可观测性代码如果放错层次，会把诊断工具变成性能问题的一部分。

### 9.1. 成本维度 1

注册阶段需要 WaitEventCustomLock 和共享 hash 操作。

这个成本通常会继续传播到相邻模块：

- executor hot path 是否多一次分支、计时或锁。
- shared memory 是否出现新的争用点。
- 日志、视图或监控采样是否放大已有压力。

### 9.2. 成本维度 2

等待阶段只使用已保存的 uint32 id，成本与内置 wait event 相同。

这个成本通常会继续传播到相邻模块：

- executor hot path 是否多一次分支、计时或锁。
- shared memory 是否出现新的争用点。
- 日志、视图或监控采样是否放大已有压力。

### 9.3. 成本维度 3

名称解析发生在读取端，扩展等待 hot path 不做字符串处理。

这个成本通常会继续传播到相邻模块：

- executor hot path 是否多一次分支、计时或锁。
- shared memory 是否出现新的争用点。
- 日志、视图或监控采样是否放大已有压力。

### 9.4. 成本维度 4

自定义 wait event 数量有限，应该按语义设计，不按对象实例设计。

这个成本通常会继续传播到相邻模块：

- executor hot path 是否多一次分支、计时或锁。
- shared memory 是否出现新的争用点。
- 日志、视图或监控采样是否放大已有压力。

### 9.5. 成本维度 5

监控系统依赖名称稳定性，重命名会制造观测断层。

这个成本通常会继续传播到相邻模块：

- executor hot path 是否多一次分支、计时或锁。
- shared memory 是否出现新的争用点。
- 日志、视图或监控采样是否放大已有压力。

因此课程里的实验默认是短时、可控、可回滚的，不建议把强制刷新或高频采样作为长期配置。

## 10. 观测与诊断入口

观测入口要按问题层次选择。

### 10.1. `pg_stat_activity`

看到 wait_event_type=Extension 和扩展名称。

使用这个入口时要同时记录：

- 采样时间。
- backend pid 或 query_id。
- 是否处于同一事务、同一 backend 或同一执行。

### 10.2. `pg_get_wait_events()`

列出已注册 Extension wait event。

使用这个入口时要同时记录：

- 采样时间。
- backend pid 或 query_id。
- 是否处于同一事务、同一 backend 或同一执行。

### 10.3. `日志`

扩展可以在等待超时时记录同一个 wait event 名称，帮助关联。

使用这个入口时要同时记录：

- 采样时间。
- backend pid 或 query_id。
- 是否处于同一事务、同一 backend 或同一执行。

### 10.4. `gdb`

打印 MyProc->wait_event_info 验证 start/end 是否成对。

使用这个入口时要同时记录：

- 采样时间。
- backend pid 或 query_id。
- 是否处于同一事务、同一 backend 或同一执行。

### 10.5. `源码 grep`

用 WaitEventExtensionNew 定位扩展注册点。

使用这个入口时要同时记录：

- 采样时间。
- backend pid 或 query_id。
- 是否处于同一事务、同一 backend 或同一执行。

### 10.6. `监控告警`

按稳定名称聚合扩展等待，而不是按 SQL 文本猜测。

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

### 12.1. 阅读注册路径

实验步骤：

```bash
rg -n "WaitEventExtensionNew|WaitEventInjectionPointNew" /home/highgo/postgres/src /home/highgo/postgres/contrib
```

先看核心和 contrib 是否已有注册示例。

实验后回到源码，至少确认一个入口函数和一个状态字段。

### 12.2. 设计名称

实验步骤：

```sql
-- 好：RemoteServicePoll
-- 好：ExtensionQueueReceive
-- 差：WaitForThing123
```

名称应稳定描述资源或阶段，不包含动态对象。

实验后回到源码，至少确认一个入口函数和一个状态字段。

### 12.3. 验证 SQL 输出

实验步骤：

```sql
SELECT type, name FROM pg_get_wait_events() WHERE type = 'Extension' ORDER BY name;
```

扩展加载并注册后，名称应出现在这里。

实验后回到源码，至少确认一个入口函数和一个状态字段。

### 12.4. 验证清理

实验步骤：

```sql
SELECT wait_event_type, wait_event FROM pg_stat_activity WHERE pid = pg_backend_pid();
```

等待结束后当前 backend 不应保留 Extension wait event。

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

1. 扩展等待点应该按外部资源命名，还是按内部阶段命名？

回答时必须至少引用一个源码文件和一个运行期现象。

2. 为什么不能把动态对象名拼进 wait event 名称？

回答时必须至少引用一个源码文件和一个运行期现象。

3. 注册失败时扩展应该 FATAL、ERROR 还是降级？

回答时必须至少引用一个源码文件和一个运行期现象。

4. 什么场景应该用 InjectionPoint 而不是 Extension？

回答时必须至少引用一个源码文件和一个运行期现象。

5. 自定义 wait event 与扩展自己的 pg_stat 视图如何互补？

回答时必须至少引用一个源码文件和一个运行期现象。

6. 如何给运维文档解释一个新的 Extension wait event？

回答时必须至少引用一个源码文件和一个运行期现象。

讨论题不是开放闲聊，而是训练把源码机制压缩成可迁移判断。

## 15. 源码复盘清单

如果需要快速复盘，本节可以按下面的源码锚点重新走一遍。

### 15.1. `WaitEventExtensionNew()`

先定位 `WaitEventExtensionNew()` 的调用者，不要从文件顶部线性读到底。

然后确认它如何影响 `WaitEventCustomCounterData`。

本锚点对应的运行期动作是：扩展作者先确定等待是否真的会阻塞 backend。

复盘时记录三个问题：

- 这个函数在正常路径上推进了什么状态？
- 它在 ERROR、abort 或 early return 时是否需要额外清理？
- 用户能否在 SQL、日志、pg_stat、wait event、EXPLAIN 或 profiler 中看到它的后果？

### 15.2. `WaitEventCustomNew()`

先定位 `WaitEventCustomNew()` 的调用者，不要从文件顶部线性读到底。

然后确认它如何影响 `WaitEventCustomHashByName`。

本锚点对应的运行期动作是：如果只是 CPU 计算，不应该注册 wait event。

复盘时记录三个问题：

- 这个函数在正常路径上推进了什么状态？
- 它在 ERROR、abort 或 early return 时是否需要额外清理？
- 用户能否在 SQL、日志、pg_stat、wait event、EXPLAIN 或 profiler 中看到它的后果？

### 15.3. `pgstat_report_wait_start()`

先定位 `pgstat_report_wait_start()` 的调用者，不要从文件顶部线性读到底。

然后确认它如何影响 `WaitEventCustomHashByInfo`。

本锚点对应的运行期动作是：如果会等待外部服务、共享队列、文件描述符或后台 worker，应定义稳定名称。

复盘时记录三个问题：

- 这个函数在正常路径上推进了什么状态？
- 它在 ERROR、abort 或 early return 时是否需要额外清理？
- 用户能否在 SQL、日志、pg_stat、wait event、EXPLAIN 或 profiler 中看到它的后果？

### 15.4. `pgstat_report_wait_end()`

先定位 `pgstat_report_wait_end()` 的调用者，不要从文件顶部线性读到底。

然后确认它如何影响 `PG_WAIT_EXTENSION`。

本锚点对应的运行期动作是：扩展在合适初始化路径调用 WaitEventExtensionNew(name)。

复盘时记录三个问题：

- 这个函数在正常路径上推进了什么状态？
- 它在 ERROR、abort 或 early return 时是否需要额外清理？
- 用户能否在 SQL、日志、pg_stat、wait event、EXPLAIN 或 profiler 中看到它的后果？

### 15.5. `pg_get_wait_events()`

先定位 `pg_get_wait_events()` 的调用者，不要从文件顶部线性读到底。

然后确认它如何影响 `wait_event_name`。

本锚点对应的运行期动作是：WaitEventCustomNew() 先用共享锁查名称是否已存在。

复盘时记录三个问题：

- 这个函数在正常路径上推进了什么状态？
- 它在 ERROR、abort 或 early return 时是否需要额外清理？
- 用户能否在 SQL、日志、pg_stat、wait event、EXPLAIN 或 profiler 中看到它的后果？

### 15.6. `WaitEventCustomShmemCallbacks`

先定位 `WaitEventCustomShmemCallbacks` 的调用者，不要从文件顶部线性读到底。

然后确认它如何影响 `extension wait_event_info`。

本锚点对应的运行期动作是：若不存在，再用排他锁重查并分配新的 id。

复盘时记录三个问题：

- 这个函数在正常路径上推进了什么状态？
- 它在 ERROR、abort 或 early return 时是否需要额外清理？
- 用户能否在 SQL、日志、pg_stat、wait event、EXPLAIN 或 profiler 中看到它的后果？

### 15.7. `WaitEventExtensionNew()`

先定位 `WaitEventExtensionNew()` 的调用者，不要从文件顶部线性读到底。

然后确认它如何影响 `WaitEventCustomCounterData`。

本锚点对应的运行期动作是：分配 id 时使用 WaitEventCustomCounterData 的 spinlock 保护 nextId。

复盘时记录三个问题：

- 这个函数在正常路径上推进了什么状态？
- 它在 ERROR、abort 或 early return 时是否需要额外清理？
- 用户能否在 SQL、日志、pg_stat、wait event、EXPLAIN 或 profiler 中看到它的后果？

## 16. 本节小结

- 自定义 wait event 让扩展等待进入 PostgreSQL 标准诊断面。
- 设计重点是稳定命名、一次注册、等待路径成对 start/end。
- 它不记录历史，只暴露当前等待点。
- 下一节进入 executor hook chain，看扩展如何在执行器生命周期中插入观测逻辑。

本节最后沉淀出的可迁移规律是：

```text
观测值不是事实本身；观测值是某个源码状态在某个生命周期边界上的投影。
```

读 PostgreSQL 内核时，始终把 runtime 现象压回 state、owner、cleanup 和 cost。
